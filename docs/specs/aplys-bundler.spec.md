---
title: aplys Bundler
description: aplys toolchain 定義・インストール・bootstrap 仕様
author: atsushifx
version: 0.3.0
---

<!-- textlint-disable
  ja-technical-writing/sentence-length -->

## aplys Bundler Specification (Draft)

本ドキュメントは aplys の bundle 定義・tool レジストリ・インストール・bootstrap の仕様を定義します。
provider の詳細は `aplys-provider.spec.md` を参照してください。
CLI インターフェース・exit code・環境変数の詳細は `aplys-api.spec.md` を参照してください。

> **bundle / provider は Optional bootstrap レイヤーです。**
> aplys のコアは `kernel / dispatch / runner / plugin` による tool execution です。
> bundle / provider は tool 導入環境を構築するための補助機能であり、tool の実行には不要です。

## aplys の責務

aplys はツールチェインのオーケストレーターです。責務は次の 4つのみです。

```text
1. bundle 解決        — bundle ファイルから tool name リストを読み込む
2. tool registry 解決 — tool → (provider, packages) に変換
3. provider batching  — 同一 provider の packages をまとめる
4. provider 実行      — provider コマンドを呼び出す
```

以下は aplys の管理対象外:

- `sudo` / root 権限管理 (provider が必要な場合に使用)
- OS パッケージマネージャー自体のインストール
- Node.js 環境管理 (node バージョン管理は mise/asdf 等に委譲)
- 言語ランタイム (Go/Python/Rust 等)

## ツール分類

aplys は 2種類のツールを扱います。

| 分類         | 例                                  | provider           |
| ------------ | ----------------------------------- | ------------------ |
| system tools | shellcheck, shfmt, shellspec, git   | apt / brew / scoop |
| node tools   | textlint, markdownlint-cli2, eslint | pnpm / npm / yarn  |

node tools の provider は `$APLYS_CONFIG_HOME/config.yaml` の `node_installer` で固定します。

## node_installer 設定

Node ツールに使用する installer を設定ファイルで固定します。

```yaml
# ~/.config/aplys/config.yaml
node_installer: pnpm
```

許可値: `pnpm` / `npm` / `yarn`

`aplys install` / `upgrade all` 開始時に `command -v <node_installer>` で存在を検証します。
未インストールの場合は exit 2 とします。

tool レジストリで `providers.default: node` と指定されたツールは
この `node_installer` 設定を使用します。

## Bundle

bundle は**ツール名の集合定義**です。

```text
bundle = tool name の一覧 (1行1エントリ)
```

bundle は用途ベースの名前にします。

使用例:

```text
dev-tools
doc-tools
```

### 命名規則

`<bundle>` は以下のパターンのみを許可:

```regex
^[a-z][a-z0-9_-]*$
```

## bundle ファイル形式

bundle ファイルはツール名の一覧 (1行1エントリ):

```text
# bundles/dev-tools
shellcheck
shfmt
shellspec
```

```text
# bundles/doc-tools
markdownlint-cli2
textlint
```

- `#` で始まる行はコメント
- `$ @(#)` で`what`用ヘッダコメント
- 空行は無視

### tool naming rule

```text
tool name = executable command name
```

tool name は npm package name ではなく、インストール後に `command -v` で確認できる**実行可能コマンド名**です。
`install` 後に `command -v <tool>` が必ず成功します。

npm scope 付きパッケージ (`@scope/pkg`) のように package name とコマンド名が異なる場合は、registry の `executable` フィールドでコマンド名を明示します。

| tool name           | 実行コマンド        | package name        | 分類        |
| ------------------- | ------------------- | ------------------- | ----------- |
| `shellcheck`        | `shellcheck`        | `shellcheck`        | system tool |
| `shfmt`             | `shfmt`             | `shfmt`             | system tool |
| `markdownlint-cli2` | `markdownlint-cli2` | `markdownlint-cli2` | node tool   |
| `textlint`          | `textlint`          | `textlint`          | node tool   |

## bundle ファイルの格納場所

| 優先度 | パス                                  | 説明           |
| ------ | ------------------------------------- | -------------- |
| 1      | `$APLYS_CONFIG_HOME/bundles/<bundle>` | user bundle    |
| 2      | `$APLYS_ROOT/bundles/<bundle>`        | builtin bundle |

## tool レジストリ

bundle の各 tool エントリは `tools-registry/<tool>.yaml` に対応します。
tool レジストリは OS 別の provider とインストールするパッケージ一覧を定義します。

### tool レジストリの格納場所

| 優先度 | パス                                            | 説明                  |
| ------ | ----------------------------------------------- | --------------------- |
| 1      | `$APLYS_CONFIG_HOME/tools-registry/<tool>.yaml` | user tool registry    |
| 2      | `$APLYS_ROOT/tools-registry/<tool>.yaml`        | builtin tool registry |

### tool レジストリの形式

`providers` に OS 別の provider を指定します。
`packages` にインストールするパッケージを列挙します (複数可)。

node tools には `providers.default: node` を指定します。
`node` は仮想 provider です。実行時に `node_installer` 設定値で解決されます。

```yaml
# tools-registry/textlint.yaml
executable: textlint # 省略可 (packages[0] "textlint@^15" から自動導出される)
providers:
  default: node

packages:
  # core
  - textlint@^15
  - textlint-filter-rule-allowlist
  - textlint-filter-rule-comments

  # Japanese technical writing rules
  - textlint-rule-preset-ja-technical-writing
  - textlint-rule-preset-ja-spacing
  - "@textlint-ja/textlint-rule-preset-ai-writing"
  - textlint-rule-ja-no-orthographic-variants
  - "@textlint-ja/textlint-rule-no-synonyms"
  - sudachi-synonyms-dictionary
  - "@textlint-ja/textlint-rule-morpheme-match"
  - textlint-rule-ja-hiraku
  - textlint-rule-no-mixed-zenkaku-and-hankaku-alphabet

  # proofreading
  - textlint-rule-common-misspellings
  - "@proofdict/textlint-rule-proofdict"
  - textlint-rule-prh
```

```yaml
# tools-registry/markdownlint-cli2.yaml
providers:
  default: node

packages:
  - markdownlint-cli2@^0.37
```

```yaml
# tools-registry/shellcheck.yaml
providers:
  windows: scoop
  linux: apt
  macos: brew
  default: brew

packages:
  - shellcheck
```

- `executable`: install 済み判定に使う実行可能コマンド名 (省略可)
- `providers.<os>`: OS ごとに使用する provider を指定
- `providers.default`: OS に対応する provider が未定義の場合のフォールバック (**必須**)
- `providers.default: node`: node tools に使用する仮想 provider
- `packages`: インストールするパッケージ一覧 (**1件以上必須**)
- `packages[1..]`: 依存パッケージ。`packages[0]` と合わせて provider に一括渡しする
- YAML コメント (`#`) でパッケージグループを記述可能

`executable` は `command -v` で確認できる実行可能コマンド名を指定します。
省略時は `packages[0]` から自動導出します。

```bash
# executable 省略時の導出ルール
pkg="${packages[0]%%@*}"   # バージョン指定を除去 (textlint@^15 → textlint)
pkg="${pkg##*/}"           # スコープを除去 (@scope/name → name)
executable="$pkg"
```

スコープ付きパッケージ (`@scope/pkg`) や、パッケージ名とコマンド名が一致しない場合は `executable` を明示してください。

### registry validation

tool レジストリ読み込み時に以下を検証します。違反時は exit 2 とします。

| フィールド          | 制約                                    |
| ------------------- | --------------------------------------- |
| `providers`         | 必須                                    |
| `providers.default` | 必須                                    |
| `packages`          | 必須、1件以上                           |
| `executable`        | 省略可。省略時は `packages[0]` から導出 |

### OS detection

`providers.<os>` のキーは以下の aplys OS キーを使用します。

| aplys OS キー | 判定条件 (`uname -s` の正規化結果) |
| ------------- | ---------------------------------- |
| `windows`     | `MINGW*` / `MSYS*` / `CYGWIN*`     |
| `linux`       | `Linux`                            |
| `macos`       | `Darwin`                           |

OS 判定ロジック:

```bash
get_os() {
  case "$(uname -s)" in
    Linux*)               echo "linux"   ;;
    Darwin*)              echo "macos"   ;;
    MINGW*|MSYS*|CYGWIN*) echo "windows" ;;
    *)                    echo "linux"   ;;  # fallback
  esac
}
```

`providers.<os>` に一致するキーがない場合は `providers.default` にフォールバックします。

### version pin

`packages` エントリは `<package>@<version>` 形式でバージョンを指定できます。
バージョン解決は provider に委譲します。aplys はパッケージ名をそのまま provider に渡します。

| 形式                  | 意味                             |
| --------------------- | -------------------------------- |
| `<package>`           | バージョン指定なし (latest)      |
| `<package>@<version>` | バージョン pin (provider に委譲) |

provider ごとのバージョン指定対応:

| provider     | バージョン指定 | 構文例                     |
| ------------ | -------------- | -------------------------- |
| `pnpm`/`npm` | 対応           | `pnpm add -g textlint@^15` |
| `scoop`      | 非対応 (無視)  | —                          |
| `brew`       | 非対応 (無視)  | —                          |
| `apt`        | 非対応 (無視)  | —                          |

## aplys install

```bash
aplys install <bundle>
```

idempotent はパッケージマネージャーに委譲します。
`pnpm add` / `apt install` / `brew install` はいずれも既存なら成功終了します。

### install 判定

aplys は `executable` フィールドのコマンドを `command -v` で確認し、install 済みの tool をスキップします。

```text
tool
 ↓
registry resolve
 ↓
executable (省略時: packages[0] からバージョン・スコープを除去して導出)
 ↓
command -v executable
 ↓
found     → skip (already installed)
not found → provider install へ
```

この方式は provider 非依存 (dpkg / brew list / pnpm list を解析不要) であり、PATH ベースで確認できます。

### install アルゴリズム

```text
bundle
 ↓
registry resolve → (provider, packages[]) per tool
 ↓
install 判定 → executable が未インストールの tool のみ残す
 ↓
provider group → 同一 provider の packages をまとめる (batching)
 ↓
parallel execution (apt のみ逐次)
```

詳細ステップ:

1. 引数バリデーション (`^[a-z][a-z0-9_-]*$`)、失敗 → exit 3
2. bundle ファイル探索 (user → builtin)、未発見 → exit 2
3. bundle ファイルを読み込み (ツール名一覧を取得)、失敗 → exit 2
4. 各 tool エントリを検証 (`^[a-z0-9._+-]+$`)、不正形式 → exit 3
5. `tools-registry/<tool>.yaml` を探索・validation (user → builtin)、失敗 → exit 2
6. OS 検出・provider 決定 (`node` は `node_installer` に解決)、失敗 → exit 2
7. `executable` の `command -v` で install 済みの tool をスキップ
8. provider ごとに packages をまとめる (batching)
9. provider 並列実行 (`apt` のみ逐次)、失敗 → exit 2

### provider batching

同一 provider の packages を 1 コマンドにまとめて実行します。

例: `dev-tools` + `doc-tools` をまとめてインストールする場合。

```bash
# batching 後の実行イメージ
pnpm add -g textlint@^15 markdownlint-cli2@^0.37 ... &
scoop install shellcheck shfmt &
apt-get install -y shellspec
wait
```

| provider     | 並列実行    |
| ------------ | ----------- |
| `pnpm`/`npm` | 可          |
| `scoop`      | 可          |
| `winget`     | 可          |
| `brew`       | 可          |
| `apt`        | 不可 (lock) |

### sudo ポリシー

aplys は `sudo` を直接実行しません。
`apt` 等のシステム provider が sudo を使用します。
sudo のタイムアウト・パスワードプロンプト・キャッシュは OS の責務です。

### install の exit code

| code | 状況                                                              |
| ---- | ----------------------------------------------------------------- |
| 0    | 全エントリのインストール成功 (既存スキップはマネージャー側)       |
| 2    | bundle 未発見 / registry エラー / installer 未検出 / install 失敗 |
| 3    | 引数不正 (bundle 名) / エントリ形式不正                           |

## aplys upgrade all

```bash
aplys upgrade all
```

provider の一括更新機能 (`provider_upgrade_all`) を呼び出します。
bundle 解決・registry 解決は行いません。provider に全更新を委譲します。

| provider     | 実行コマンド例         |
| ------------ | ---------------------- |
| `pnpm`/`npm` | `pnpm update -g`       |
| `scoop`      | `scoop update *`       |
| `winget`     | `winget upgrade --all` |
| `brew`       | `brew upgrade`         |
| `apt`        | `apt-get upgrade -y`   |

`all` は `upgrade` の唯一の引数として予約されています。bundle 名として `all` は使用できません。

## Bootstrap

bootstrap は project 用設定を生成します。

> `$APLYS_CONFIG_HOME` 下にグローバル設定をコピー
> `<project-root>/` 下にプロジェクト設定を作成

```bash
aplys bootstrap <bundle>
```

コピー対象:

```text
.vscode/cspell.json
markdownlint.yaml
.textlintrc.yaml
.shellspec
.editorconfig
```

CI 設定 (.github/workflows) は含めない。

## Bootstrap Template System

bundle 側は template を持ちます。

```text
templates/
  .textlintrc.yaml.tpl
```

template engine は `env-subst` レベルで十分です。

使用例 (template):

```yaml
extends:
  - ${APLYS_CONFIG_DIR}/textlint/textlintrc.yaml
```

## Bootstrap Rendering

template をレンダリングし絶対パスに変換します。

使用例 (rendered):

```yaml
extends:
  - /home/<user>/.config/aplys/configs/textlint/textlintrc.yaml
```

> Windows 環境: バックスラッシュを `/` に変換して保存
> 理由: YAML config の cross-platform compatibility —
> YAML ファイルを Linux/macOS でも読み込めるよう UNIX パス区切りに統一します。

## Bootstrap Algorithm

```text
template
 ↓
env-subst で変数展開
 ↓
絶対パスに変換
 ↓
write file
```

## Bootstrap Update Policy

ファイルが存在する場合:

1. skip + warning (デフォルト)

> voift が使用可能な場合は `merge` する

## Managed Header

config / bootstrap file には管理ヘッダーを入れます。

```text
# aplys-managed: true
# aplys-bundle: doc-tools
# aplys-version: 0.2.0
```

| フィールド      | 用途                                     |
| --------------- | ---------------------------------------- |
| `aplys-managed` | `true` のとき自動更新対象と判定          |
| `aplys-bundle`  | 生成元 bundle 名 (追跡・デバッグ用)      |
| `aplys-version` | 生成時の aplys バージョン (互換性確認用) |

`aplys-managed: true` が存在するファイルのみ `aplys bootstrap --update` の対象とします。
