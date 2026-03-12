---
title: aplys API 仕様
description:
author: atsushifx
version: 0.2.0
---

## aplys API 仕様

aplys is a tool runner that orchestrates developer and documentation tools.
It does not manage tool versions or environments.
Those concerns are delegated to external tools such as mise or asdf.

aplys は bash shell script を使用した **tool execution router** です。
`domain/target/action` モデルでツールの discovery・execution を統合します。
本ドキュメントは aplys の API インターフェース・実行環境・バージョンポリシーを定義します。

### aplys のレイヤー構造

aplys は 2つのレイヤーに分かれます。

```text
Core runtime (必須)
  kernel / dispatch / runner / plugin
  — tool の実行・routing を担う

Optional bootstrap (補助)
  bundle / provider
  — tool のインストール環境を構築するためのレイヤー
  — aplys の本質的な機能ではない
```

bundle / provider は tool 導入補助にすぎません。
aplys の本質は `runner` による tool execution です。

| レイヤー  | 構成要素                         | 役割                    |
| --------- | -------------------------------- | ----------------------- |
| Core      | kernel, dispatch, runner, plugin | tool の実行・routing    |
| Bootstrap | bundle, provider                 | tool のインストール補助 |

## インターフェース仕様

```bash
aplys <domain>/<target> <action> [files...]
aplys list [<domain>[/<target>]]
aplys install <bundle-tools>
aplys upgrade all
aplys help [<command>]
```

- `domain/target`: スラッシュ区切りのパス (例: `dev/shell`, `doc/markdown`, `doc/prose`)
- `action`: 実行するアクション (例: `lint`, `format`, `style`)
- `files...`: 省略可能な対象ファイル・ディレクトリ

## runtime dependency

aplys の実行に必要な環境とツールを定義します。

### required runtime environment

- bash >= 4.x: aplys は bash shell script で実装されています。`bash --version` で確認してください。

### required external tools

| ツール         | 用途                            | 検証タイミング    | 未検出時の動作 |
| -------------- | ------------------------------- | ----------------- | -------------- |
| `rg` (ripgrep) | ファイル一覧取得 (`rg --files`) | dispatcher 起動時 | exit 2         |

`rg` は dispatcher 起動時にチェックします。
`files...` 引数の有無にかかわらず、dispatcher 実行には `rg` が必要です。
`rg` が PATH 上に存在しない場合、aplys は直ちに exit code 2 で終了し、stderr にエラーメッセージを出力します。

```bash
# 起動時チェック例
command -v rg >/dev/null 2>&1 || { echo "aplys: rg (ripgrep) not found in PATH" >&2; exit 2; }
```

### provider dependencies

`install` / `upgrade` / `bootstrap` コマンドは provider を使用します。
provider の検証は bundle resolve 後、使用する provider が確定した時点で行います。
未インストールの場合は exit 2 とします。

```text
bundle
 ↓
tool list
 ↓
registry resolve → provider set 確定
 ↓
provider check (ここで初めて検証)
 ↓
install / upgrade 実行
```

使用する provider は tool レジストリの `providers.<os>` で決定します。
詳細は `aplys-provider.spec.md` を参照してください。

#### node_installer

Node ツール用 provider は `APLYS_NODE_INSTALLER` 環境変数で指定します。

```bash
export APLYS_NODE_INSTALLER=pnpm   # pnpm / npm / yarn
```

未設定の場合は `$APLYS_CONFIG_HOME/config.yaml` の `node_installer` を参照します。
それも未設定の場合は `pnpm` をデフォルトとして使用します。

自動検出 (PATH 上の存在確認) は行いません。環境差異・CI 再現性低下を防ぐため、installer は明示的に固定します。

#### default_domain

`default_domain` を設定すると、`aplys <target> <action>` の短縮形で runner を呼び出せます。

```yaml
# $APLYS_CONFIG_HOME/config.yaml
default_domain: ops
```

`APLYS_DEFAULT_DOMAIN` 環境変数でも設定できます。環境変数が優先されます。
未設定の場合、plugin が見つからなければ exit 2 とします。
詳細は `aplys-cli.spec.md` の「default domain」を参照してください。

CI 環境では `bash` のバージョン・`rg` のインストール・使用する provider を事前に確認してください。

### 内部コマンド

#### aplys install

```bash
aplys install <bundle-tools>
```

指定した `<bundle-tools>` (例: `doc-tools`) をインストールします。

##### bundle-tools の命名規則

`<bundle-tools>` は以下のパターンのみを許可:

```regex
^[a-z][a-z0-9_-]*$
```

内部実装の詳細 (bundle 定義形式・インストールアルゴリズム・exit code) は
`aplys-cli.spec.md` の「bundle システム」を参照してください。

#### aplys upgrade

```bash
aplys upgrade all
```

provider の `provider_upgrade_all()` を直接呼び出します。
`bundles/` の走査は行いません。詳細は `aplys-bundler.spec.md` を参照してください。

#### aplys list

```bash
aplys list                    # 全 aplys を列挙
aplys list <domain>           # domain 配下の aplys を列挙
aplys list <domain>/<target>  # target 配下の aplys を列挙
```

`$APLYS_ROOT/runners` 配下の実行可能ファイルを探索し、`domain/target/action` 形式で stdout に出力:

```bash
dev/shell/format
dev/shell/lint
dev/shell/test
doc/markdown/format
doc/markdown/lint
doc/prose/lint
doc/prose/style
```

出力仕様:

- 1行1エントリ
- 形式: `<domain>/<target>/<action>` (スラッシュ区切り、ファイルパスと対応)
- アルファベット順でソート
- builtin (`$APLYS_ROOT/runners`) を先に出力し、user-installed (`$APLYS_DATA_DIR/tools`) を後に出力する
- builtin が存在する場合、同名の user-installed は出力しない (shadowing 防止)

### ファイルパス規則

ディレクトリ構造の詳細は `aplys-cli.spec.md` の「ディレクトリ構造」を参照してください。

例:

```bash
aplys/dev/shell/lint      # => aplys dev/shell lint
aplys/dev/shell/format    # => aplys dev/shell format
aplys/dev/shell/test      # => aplys dev/shell test
aplys/doc/markdown/lint   # => aplys doc/markdown lint
aplys/doc/prose/lint      # => aplys doc/prose lint
```

## exit code 規約

### runner コマンド (`<domain>/<target> <action>`)

| code | 意味                                   |
| ---- | -------------------------------------- |
| 0    | success                                |
| 1    | lint / test error (ツール起因のエラー) |
| 2    | aplys error (スクリプト内部エラー)     |
| 3    | invalid argument (引数不正)            |
| 127  | runner script not found                |

CI 統合では exit code 1 をビルド失敗として扱います。

exit code `127` は shell の `command not found` POSIX 慣習に準拠します。
ただし dispatcher がスクリプトの不在を検出した場合は exit 2 を返します。
exit 127 はシェルが直接返す場合のみ発生します。

dispatcher はスクリプト実行前に存在確認を行わなければなりません (SHOULD)。
実行可能確認 (`test -x`) は Linux / macOS のみで行います。Windows では `test -x` が信頼できないためスキップします。

```bash
# 存在確認 (全 OS)
test -f "$script" || { echo "aplys: $script not found" >&2; exit 2; }
# 実行可能確認 (Linux / macOS のみ)
case "$(get_os)" in
  linux|macos) test -x "$script" || { echo "aplys: $script not executable" >&2; exit 2; } ;;
esac
```

### install / upgrade / bootstrap コマンド

| code | 意味                                                        |
| ---- | ----------------------------------------------------------- |
| 0    | success                                                     |
| 2    | runtime error (bundle 未発見 / installer 未検出 / 実行失敗) |
| 3    | invalid argument (引数不正 / エントリ形式不正)              |

install/upgrade/bootstrap は exit code `1` を使用しません。
これらのコマンドで発生するエラーは aplys 内部エラー (2) か引数不正 (3) のいずれかです。

## stdout / stderr 責務

- `stdout`: machine-readable output (ツールの出力をそのまま渡す)
- `stderr`: human-readable diagnostics (aplys 自身のエラーメッセージ)

CI ログ解析では stdout をパース対象とします。

## aplys の責務

aplys は **toolchain orchestrator** です。責務は次の 4つのみです。

```text
1. bundle 解決       — tool name リストに展開
2. tool registry 解決 — tool → (provider, packages) に変換
3. provider batching — 同一 provider の packages をまとめる
4. provider 実行     — provider コマンドを呼び出す
```

aplys itself は **stateless** です。同じ引数で実行した場合、同じ結果 (exit code・出力) を返します。
ただし stateless は deterministic filesystem state を意味しません。ツール結果はツール自身の状態やファイルシステムの状態に依存します。その責務は tool 側にあります。

### 管理対象外

以下は aplys のスコープ外です。

```text
sudo / root 権限管理     (provider が必要に応じて使用)
OS パッケージマネージャー自体のインストール
Node.js / Go / Python / Rust 等の言語ランタイム管理
CI ワークフロー制御
複雑な依存関係解決
```

### CLI design principles

```text
aplys is stateless          — 同じ引数で同じ結果を返す
installer is authority      — install/upgrade の実行はすべて provider に委譲
bundle is grouping only     — bundle は tool 名の集合定義にすぎない
do one thing                — runner はツールを起動するだけ
delegate complexity         — sudo / ロック / バージョン管理は provider / OS に委譲
prefer idempotent operations — 繰り返し実行しても副作用のない操作を選ぶ
```

### runner コマンドの副作用

副作用 (ファイル書き換え等) は `format` アクションのみ許可:

| action   | 副作用 |
| -------- | ------ |
| `lint`   | なし   |
| `test`   | なし   |
| `format` | あり   |

## input validation

`<domain>`, `<target>`, `<action>`は、次のパターンだけを許可します。

```regex
^[a-z][a-z0-9_-]*$
```

バリデーション失敗時は exit code 3 で終了し、stderr にエラーメッセージを出力します。

```bash
# 有効な例
ops
shell
my-target
my_target2

# 無効な例 (exit code 3)
Ops         # 大文字不可
-ops        # ハイフン始まり不可
_libs       # アンダースコア始まり不可 (内部予約)
ops/shell/  # trailing slash
```

`action` も同じパターンを適用します。`lint`、`format`、`test` はすべて適合します。

各引数のバリデーション正規表現を明示:

| 引数     | パターン             |
| -------- | -------------------- |
| `domain` | `^[a-z][a-z0-9_-]*$` |
| `target` | `^[a-z][a-z0-9_-]*$` |
| `action` | `^[a-z][a-z0-9_-]*$` |

## 予約語

以下の語は aplys が予約します。bundle 名・domain・target・action として使用できません。

| 予約語    | 種別             | 用途                                       |
| --------- | ---------------- | ------------------------------------------ |
| `help`    | builtin コマンド | ヘルプ表示 (`aplys help [<command>]`)      |
| `list`    | plugin           | runner 一覧表示 (`aplys list`)             |
| `usage`   | plugin           | 使用方法表示 (`aplys usage`)               |
| `install` | builtin コマンド | bundle インストール (`aplys install`)      |
| `upgrade` | builtin コマンド | ツール更新 (`aplys upgrade`)               |
| `all`     | 引数予約語       | `upgrade all` での一括更新を意味する特殊値 |

### `_` プレフィックス規則

`_` で始まる名前は aplys 内部用として予約します。
`domain` / `target` / `action` / bundle 名に `_` で始まる名前は使用できません。

```text
# 無効な例 (exit code 3)
_libs
_internal
_tools
```

現在の内部ディレクトリ:

| 名前    | 用途                                       |
| ------- | ------------------------------------------ |
| `_libs` | ライブラリ格納ディレクトリ (`$APLYS_LIBS`) |

バリデーション時に予約語チェックおよび `_` プレフィックスチェックを行い、違反時は exit code 3 で終了します。

## dispatcher アルゴリズム

実装の詳細は `aplys-cli.spec.md` の「dispatcher アルゴリズム」を参照してください。

## tool discovery rule

実装の詳細は `aplys-cli.spec.md` の「tool dispatch 用パス解決」を参照してください。

## 環境変数仕様

dispatcher が aplys スクリプトを起動する際に以下の環境変数をセットします。aplys 実装はこれらを参照することで `SCRIPT_DIR=$(dirname "$0")` のような自己参照処理を省略できます。

### コア変数

| 変数名         | 値の例              | 説明                            |
| -------------- | ------------------- | ------------------------------- |
| `APLYS_ROOT`   | `/path/to/aplys`    | aplys/ ディレクトリの絶対パス   |
| `APLYS_DOMAIN` | `ops`               | domain 部分 (例: `ops`, `docs`) |
| `APLYS_TARGET` | `shell`             | target 部分 (例: `shell`)       |
| `APLYS_ACTION` | `lint`              | action 部分 (例: `lint`)        |
| `APLYS_LIBS`   | `$APLYS_ROOT/_libs` | ライブラリ読み込みパス          |

`APLYS_ROOT` は絶対パスで渡す。相対パスは `cd` や `PWD` の変更で壊れるため使用しない。

`APLYS_LIBS` は `$APLYS_ROOT/_libs` に固定し、ライブラリは `$APLYS_LIBS/` 経由で読み込みます。

```bash
# shellcheck source=aplys/_libs/common.sh
. "$APLYS_LIBS/common.sh"
```

### XDG ディレクトリ変数

| 変数名              | デフォルト値                             | 説明                     |
| ------------------- | ---------------------------------------- | ------------------------ |
| `APLYS_CONFIG_HOME` | `${XDG_CONFIG_HOME:-~/.config}/aplys`    | 設定ファイルディレクトリ |
| `APLYS_DATA_DIR`    | `${XDG_DATA_HOME:-~/.local/share}/aplys` | データディレクトリ       |
| `APLYS_CACHE_DIR`   | `${XDG_CACHE_HOME:-~/.cache}/aplys`      | キャッシュディレクトリ   |

XDG Base Directory Specification に準拠します。`XDG_*` 変数が未設定の場合はデフォルト値を使用します。

### files... 引数の受け渡し

`files...` 引数は `exec "$script" "$@"` でスクリプトに直接渡す。

```bash
# dispatcher 側
exec "$script" "$@"
```

bash 配列の `export` は POSIX 非準拠であり、子プロセスに正しく伝わりません。`exec` による直接渡しが唯一安全な方法です。

## files... 引数解決ルール

`files...` 引数の解決は dispatcher と action script で責務を分離します。

```text
dispatcher  → ファイル一覧取得 (rg --files)
action script → 拡張子フィルタ指定 (-g '*.sh' 等)
```

dispatcher は `files...` の有無を判定し、なければ `rg --files` でファイル一覧を取得します。
拡張子の指定は action script の責務です。dispatcher は拡張子でフィルタしてはなりません (MUST NOT)。

| 入力             | dispatcher の処理                                                 |
| ---------------- | ----------------------------------------------------------------- |
| なし             | `rg --files` を実行 (拡張子フィルタは action script が付与)       |
| ファイル指定     | そのまま action script に渡す                                     |
| ディレクトリ指定 | `rg --files <dir>` を実行 (拡張子フィルタは action script が付与) |
| 混在             | ファイルはそのまま + ディレクトリは `rg --files` で展開して結合   |

### rg --files の動作

`rg --files` はデフォルトで `.gitignore` を尊重し、`.git` / `node_modules` / `vendor` 等を自動除外します。
また、`.hidden` ファイルはデフォルトで除外されます (`rg --files` default behavior)。

```bash
# action script 側での使用例
# runners/ops/shell/lint.sh
rg --files -g '*.sh'           # カレントディレクトリ配下
rg --files -g '*.sh' scripts/  # ディレクトリ指定
```

`find` は使用しません。`rg --files` により `.gitignore` 除外が自動適用されます。

### extension mapping

拡張子は **action script が決定します**。dispatcher は拡張子を関知しません。

拡張子の例:

| action script                | 拡張子 | rg オプション |
| ---------------------------- | ------ | ------------- |
| `runners/ops/shell/lint`     | `sh`   | `-g '*.sh'`   |
| `runners/docs/markdown/lint` | `md`   | `-g '*.md'`   |

## セキュリティ仕様

### 入力サニタイズ

- `domain`/`target`/`action` は `^[a-z][a-z0-9_-]*$` でバリデーションする
- ファイルパスのシェル展開をしません (glob はシェル展開に依存する)
- スクリプトパスは `$APLYS_ROOT` 配下のみ許可する
- resolved script path は `$APLYS_ROOT` から始まることを検証する (path traversal 防御)

```bash
# path traversal 防御の実装例
# canonical path resolve: cd + pwd で portable に絶対パスを取得
resolved=$(cd "$(dirname "$script")" && pwd)/$(basename "$script")
case "$resolved" in
  "$APLYS_ROOT"/*) ;;  # OK
  *) echo "aplys: script path outside APLYS_ROOT" >&2; exit 2 ;;
esac
# スクリプトの実行可能確認 (Linux / macOS のみ。Windows は test -x が信頼できないためスキップ)
case "$(get_os)" in
  linux|macos) test -x "$resolved" || { echo "aplys: $resolved not executable" >&2; exit 2; } ;;
esac
```

execution shell・OS 別起動方法・PATH policy の実装詳細は `aplys-cli.spec.md` の「セキュリティ実装仕様」を参照してください。

## shellcheck オプション仕様

実装の詳細は `aplys-cli.spec.md` の「開発ツール仕様」を参照してください。

## aplys version policy

aplys は [Semantic Versioning 2.0.0](https://semver.org/) (SemVer) を採用します。

### バージョン形式

```bash
MAJOR.MINOR.PATCH
```

| バージョン | 意味                             |
| ---------- | -------------------------------- |
| `MAJOR`    | 後方互換性なし (breaking change) |
| `MINOR`    | 後方互換あり (機能追加)          |
| `PATCH`    | バグ修正のみ                     |

Go / Linux kernel スタイルの SemVer に準拠:

- `v0.x.x`: 開発中 (API は安定保証なし)
- `v1.0.0` 以降: 同一 MINOR バージョン内で後方互換を保証 (compatibility guaranteed within same MINOR)
- breaking change (API 削除・引数変更・exit code 変更) は MAJOR バージョンを上げる

### 後方互換の定義

以下の変更は後方互換 (MINOR):

- 新しい `domain/target/action` の追加
- 新しい環境変数の追加
- 新しい exit code の追加 (既存の値は変更しない)

以下は breaking change (MAJOR):

- 既存の exit code の意味変更
- 環境変数名の変更
- インターフェース (`aplys <domain>/<target> <action>`) の変更
