---
title: aplys CLI アーキテクチャ仕様
description: aplys の CLI 設計・ディレクトリ構造・dispatch アルゴリズムを定義する
author: atsushifx
version: 0.3.0
---

## aplys CLI アーキテクチャ仕様

本ドキュメントは aplys の CLI 設計・ディレクトリ構造・dispatch アルゴリズムを定義します。
API インターフェース・exit code・環境変数の詳細は `aplys-api.spec.md` を参照してください。

## CLI 構造

aplys は 4 系統のコマンドを持ちます。

| 系統       | コマンド例                         | 備考                                    |
| ---------- | ---------------------------------- | --------------------------------------- |
| `bundle`   | `aplys install <bundle>`           | `bundler`の管理および設定ファイルコピー |
| `builtin`  | `aplys help <command>`             | `aplys`に組み込まれているコマンド       |
| `<plugin>` | `aplys list`                       | `aplys plugin`を実行                    |
| `<runner>` | `aplys <domain>/<target> <action>` | `<target>`用の`<action>`を実行          |

## 4 概念

| 概念     | 役割                                                  |
| -------- | ----------------------------------------------------- |
| runner   | `<domain>/<target>/<action>` を実行するツールラッパー |
| plugin   | `aplys <name>` で呼び出される管理系拡張コマンド       |
| bundle   | インストール対象ツール名の集合定義 (1行1ツール名)     |
| provider | パッケージマネージャーの抽象化アダプター              |

aplys は **tool execution router** です。`domain/target/action` モデルでツールの discovery・execution を統合します。
bundle / provider は tool 導入補助 (optional bootstrap) であり、aplys のコア機能ではありません。

### domain 体系

`domain/target/action` の 3 層モデルは次の役割を持ちます。

| 層       | 意味         | 例                  |
| -------- | ------------ | ------------------- |
| `domain` | カテゴリ     | `docs`, `ops`       |
| `target` | 対象ツール   | `markdown`, `shell` |
| `action` | サブコマンド | `lint`, `format`    |

`target` は現在ツール種別 (`markdown`, `text`) で抽象化していますが、
ツール名 (`markdownlint`, `textlint`) に寄せることも可能です。

| domain | 役割           | 対応ツール例                      |
| ------ | -------------- | --------------------------------- |
| `ops`  | 運用ツール     | terraform, ansible, kubectl       |
| `code` | 開発ツール     | shellcheck, shfmt, shellspec      |
| `docs` | ドキュメント系 | textlint, vale, markdownlint-cli2 |

### runner execution API

runner は統一された実行インターフェースを持ちます。

- stdin: 標準入力をそのままツールに渡す
- argv (`$@`): `files...` 引数をそのまま渡す
- env: aplys dispatcher が以下の環境変数をセットして渡す

| 変数名              | 値の例                 | 説明                   |
| ------------------- | ---------------------- | ---------------------- |
| `APLYS_ROOT`        | `/path/to/aplys`       | aplys ルートの絶対パス |
| `APLYS_DATA_DIR`    | `~/.local/share/aplys` | データディレクトリ     |
| `APLYS_CONFIG_HOME` | `~/.config/aplys`      | 設定ディレクトリ       |
| `APLYS_CACHE_DIR`   | `~/.cache/aplys`       | キャッシュディレクトリ |
| `APLYS_DOMAIN`      | `docs`                 | domain 部分            |
| `APLYS_TARGET`      | `prose`                | target 部分            |
| `APLYS_ACTION`      | `lint`                 | action 部分            |

runner は POSIX script として実装できます。ツール呼び出しの例:

```bash
#!/usr/bin/env bash
# runners/docs/prose/lint.sh — textlint wrapper
set -euo pipefail
exec textlint "$@"
```

## ディレクトリ構造

```text
aplys/
  bin/
    aplys                 # CLI エントリポイント

  lib/                    # Core runtime
    kernel/
      main.sh             # CLI routing のみ (lookup / dispatch / exec)

    dispatch/
      resolve_builtin.sh  # builtin コマンド解決
      resolve_plugin.sh   # plugin 解決
      resolve_runner.sh   # runner パス解決・存在確認・実行可能確認

    providers/            # package manager adapter
      scoop.sh
      winget.sh
      brew.sh
      apt.sh
      pnpm.sh
      npm.sh
      yarn.sh

  runners/                # Core: runner スクリプト群 (tool wrapper)
    ops/
      shell/
        lint.sh           # => shellcheck
        format.sh         # => shfmt
        test.sh           # => shellspec
    docs/
      markdown/
        lint.sh           # => markdownlint-cli2
        format.sh         # => dprint
      text/
        lint.sh           # => textlint
        format.sh         # => dprint

  builtin/               # builtin コマンド専用
    install.sh
    upgrade.sh
    help.sh

  plugins/                # Core: plugin スクリプト群
    aplys-list
    aplys-usage
    aplys-what

  tooling/                # Optional (tool 導入補助)
    bundles/              # bundle 定義ファイル (tool name 一覧)
      dev-tools
      doc-tools
```

### ディレクトリと CLI の対応

| ディレクトリ                    | CLI 形式 (スラッシュ)      | CLI 形式 (スペース)        | レイヤー  |
| ------------------------------- | -------------------------- | -------------------------- | --------- |
| `runners/ops/shell/lint.sh`     | `aplys ops/shell lint`     | `aplys ops shell lint`     | Core      |
| `runners/docs/markdown/lint.sh` | `aplys docs/markdown lint` | `aplys docs markdown lint` | Core      |
| `runners/docs/text/lint.sh`     | `aplys docs/text lint`     | `aplys docs text lint`     | Core      |
| `builtin/install.sh`            | `aplys install`            | —                          | Core      |
| `builtin/upgrade.sh`            | `aplys upgrade`            | —                          | Core      |
| `builtin/help.sh`               | `aplys help`               | —                          | Core      |
| `plugins/aplys-list`            | `aplys list`               | —                          | Core      |
| `tooling/bundles/dev-tools`     | `aplys install dev-tools`  | —                          | Bootstrap |

## dispatcher アルゴリズム

Kernel (`lib/kernel/main.sh`) は routing のみ行います。ロジックは dispatch 層に委譲します。

### Kernel の責務

```text
Kernel: validate → lookup → dispatch → exec
  ↓
dispatch/
  resolve_builtin.sh  # builtin コマンド解決
  resolve_plugin.sh   # plugin 解決
```

Kernel が行う操作は次の 4 つです。

1. 実行環境の前提条件確認 (`validate_env`)
2. コマンドを判定し、スクリプトパスを解決
3. 実行可能確認 (不可時 exit 2)
4. `exec "$script" "$@"` で起動

### dispatch 優先順位

```bash
cmd="$1"

case "$cmd" in
  install|upgrade|help)
    # 1. builtin コマンド (builtin/ ディレクトリから直接実行)
    exec "$APLYS_ROOT/builtin/$cmd.sh" "${@:2}"
    ;;
  */*)
    # 2. runner (domain/target を含む — resolve_runner.sh に委譲)
    domain="${cmd%%/*}"
    target="${cmd#*/}"
    action="$2"
    script=$(lib/dispatch/resolve_runner.sh "$domain" "$target" "$action")
    exec "$script" "${@:3}"
    ;;
  *)
    # 2b. domain target action 形式 (引数が3個以上で第1引数がドメイン名の場合)
    if [[ $# -ge 3 ]] && is_domain "$1"; then
      domain="$1" target="$2" action="$3"
      script=$(lib/dispatch/resolve_runner.sh "$domain" "$target" "$action")
      exec "$script" "${@:4}"
    fi
    # 3. plugin 優先 (plugins/ 探索)
    if script=$(lib/dispatch/resolve_plugin.sh "$cmd" 2>/dev/null); then
      exec "$script" "${@:2}"
    fi
    echo "aplys: unknown command: $cmd" >&2
    exit 2
    ;;
esac
```

判定順序の根拠:

1. `install`/`upgrade`/`help` は**予約済みコマンド**として最優先 (plugin と名前が衝突しても builtin が勝つ)
2. `/` を含む引数は runner として判定。`domain/target` からパスを直接計算する
3. それ以外は plugin を優先して探索 (PATH 経由も含む)
4. plugin が見つからなければ exit 2

#### 予約済みコマンド

以下のコマンド名は builtin として予約されており、plugin 名には使用できません。

| コマンド  | 実行先               |
| --------- | -------------------- |
| `install` | `builtin/install.sh` |
| `upgrade` | `builtin/upgrade.sh` |
| `help`    | `builtin/help.sh`    |

#### `is_domain` の定義

`is_domain()` は `runners/` 配下のディレクトリ存在で判定します (固定リストではなく filesystem が真実):

```bash
is_domain() {
  test -d "$APLYS_ROOT/runners/$1"
}
```

これにより、`runners/` に新しいドメインを追加するだけで自動的に認識されます。

#### `domain target action` 形式の安全性

単語 3 つのコマンドでは `$1` がドメイン名であっても `$2` (target) のディレクトリが存在しない場合は runner 扱いにしません。`is_domain` + `resolve_runner.sh` 内の存在確認が二重の安全弁になります。

### runner パス解決

runner は `domain/target/action` から直接パスを計算します。filesystem scan は行いません。

```text
$APLYS_DATA_DIR/tools/<domain>/<target>/<action>.sh   (user-installed、優先)
$APLYS_ROOT/runners/<domain>/<target>/<action>.sh     (builtin、user が存在しない場合のみ)
```

`APLYS_ALLOW_OVERRIDE=0` を設定すると builtin 優先に戻ります (セキュリティ制約環境向け):

```bash
# APLYS_ALLOW_OVERRIDE=0 の場合は builtin 優先
if [[ "${APLYS_ALLOW_OVERRIDE:-1}" == "0" ]]; then
  # builtin 優先 (旧動作)
  script="$APLYS_ROOT/runners/$domain/$target/$action.sh"
  test -f "$script" || script="$APLYS_DATA_DIR/tools/$domain/$target/$action.sh"
else
  # user 優先 (デフォルト)
  script="$APLYS_DATA_DIR/tools/$domain/$target/$action.sh"
  test -f "$script" || script="$APLYS_ROOT/runners/$domain/$target/$action.sh"
fi
```

パス解決・存在確認・実行可能確認は `lib/dispatch/resolve_runner.sh` が担います。

```bash
# lib/dispatch/resolve_runner.sh
resolve_runner() {
  local domain="$1" target="$2" action="$3"

  # input validation (directory traversal 防御)
  [[ "$domain" =~ ^[a-z][a-z0-9_-]*$ ]] \
    || { echo "aplys: invalid domain: $domain" >&2; exit 3; }
  [[ "$target" =~ ^[a-z][a-z0-9_-]*$ ]] \
    || { echo "aplys: invalid target: $target" >&2; exit 3; }
  [[ "$action" =~ ^[a-z][a-z0-9_-]*$ ]] \
    || { echo "aplys: invalid action: $action" >&2; exit 3; }

  # direct path 計算 (user 優先)
  local script="$APLYS_DATA_DIR/tools/$domain/$target/$action.sh"
  test -f "$script" || script="$APLYS_ROOT/runners/$domain/$target/$action.sh"

  # 存在確認
  test -f "$script" \
    || { echo "aplys: runner not found: $domain/$target $action" >&2; exit 2; }

  # 実行可能確認 (Linux / macOS のみ)
  case "$(get_os)" in
    linux|macos)
      test -x "$script" \
        || { echo "aplys: not executable: $script" >&2; exit 2; }
      ;;
  esac

  echo "$script"
}
```

## bundle システム

bundle の定義形式・格納場所・install/upgrade アルゴリズム・tool レジストリの詳細は
`aplys-bundler.spec.md` を参照してください。

### bundle 概要

bundle はツール名の一覧 (1行1エントリ) で、`bundles/` に格納します。

```text
bundles/
  dev-tools     # shellcheck, shfmt, shellspec
  doc-tools     # markdownlint-cli2, textlint
```

```bash
aplys install <bundle>    # bundle のツールをインストール
aplys upgrade all         # 全ツールを provider に委譲して一括更新
```

## provider システム

provider インターフェース・OS 別検出順序・idempotent 設計の詳細は
`aplys-provider.spec.md` を参照してください。

provider は OS・パッケージマネージャーの差異を吸収します。
各 provider は `lib/providers/` に配置します。

## tool dispatch 用パス解決

| 優先度 | パス                                                  | 説明                  |
| ------ | ----------------------------------------------------- | --------------------- |
| 1      | `$APLYS_DATA_DIR/tools/<domain>/<target>/<action>.sh` | user-installed (優先) |
| 2      | `$APLYS_ROOT/runners/<domain>/<target>/<action>.sh`   | builtin (同梱)        |

- デフォルトは user-installed 優先。`APLYS_ALLOW_OVERRIDE=0` 設定時のみ builtin 優先に戻る
- 探索対象は上記 2 箇所のみ (任意パスの実行は禁止)
- スクリプトは実行ビット (`chmod +x`) が必須
- PATH を通じた探索は行わない

## plugin 命名規則

plugin ファイルは `aplys-<name>` 形式:

```text
plugins/aplys-list
plugins/aplys-usage
plugins/aplys-what
```

CLI での呼び出しは `aplys <name>` (プレフィックス `aplys-` を除いた名前)。

### plugin 探索順序

plugin は以下の順序で探索します。PATH 全体は探索しません。

> `install` / `upgrade` / `help` は dispatch 時に `case` 文で先に捕捉され、
> `builtin/` から直接実行されます。plugin 探索には到達しません。

1. `$APLYS_ROOT/plugins/aplys-<name>`
2. `APLYS_PLUGIN_PATH` の各ディレクトリを左から順に探索 (`aplys-<name>`)

デフォルトの探索パスは `$APLYS_ROOT/plugins` のみです。
追加ディレクトリは `APLYS_PLUGIN_PATH` 環境変数 (コロン区切り、PATH スタイル) で指定します。

```bash
export APLYS_PLUGIN_PATH="$HOME/.config/aplys/plugins:$HOME/work/plugins"
```

PATH 全体を探索しないことで、`~/bin/aplys-install` 等による PATH hijack を防ぎます。

## クロスプラットフォーム

| OS      | スクリプト実行        | パッケージマネージャー |
| ------- | --------------------- | ---------------------- |
| Windows | `bash "$script" "$@"` | scoop / winget         |
| Linux   | `exec "$script" "$@"` | apt / dnf              |
| macOS   | `exec "$script" "$@"` | brew                   |

環境セットアップスクリプトは OS ごとに二系統提供:

```text
scripts/
  install-dev-tools.ps1   # Windows (PowerShell)
  install-dev-tools.sh    # Linux / macOS (bash)
  install-doc-tools.ps1
  install-doc-tools.sh
```

## セキュリティ実装仕様

### execution shell

aplys スクリプトの先頭:

```bash
#!/usr/bin/env bash
set -euo pipefail
```

- `set -e`: エラー時即時終了
- `set -u`: 未定義変数参照をエラーとして扱う
- `set -o pipefail`: パイプライン内のエラーを伝播する

### 起動時 validate

aplys は起動時に実行環境の前提条件を確認します。

```bash
validate_env() {
  # Windows 環境では bash が必須
  if [[ "$(get_os)" == "windows" ]]; then
    command -v bash >/dev/null 2>&1 \
      || { echo "aplys: bash not found (required on Windows)" >&2; exit 2; }
  fi
}
```

`validate_env` は `lib/kernel/main.sh` の先頭で呼び出します。

### OS 別の起動方法

dispatcher は OS 判定関数 `get_os()` を持ち、OS にあわせて起動方法を切り替えます。

```bash
get_os() {
  case "$OSTYPE" in
    linux*)               echo "linux"   ;;  # Linux および WSL (linux-gnu 等)
    darwin*)              echo "macos"   ;;
    msys*|cygwin*|win32*) echo "windows" ;;  # Git Bash / MSYS / Cygwin
    *)                    echo "linux"   ;;  # fallback
  esac
}
```

| OS      | 起動方法              | 理由                                                           |
| ------- | --------------------- | -------------------------------------------------------------- |
| linux   | `exec "$script" "$@"` | shebang 有効。プロセスを置き換える                             |
| macos   | `exec "$script" "$@"` | 同上                                                           |
| windows | `bash "$script" "$@"` | shebang 非対応のため bash を明示指定 (WSL は linux として動作) |

```bash
# dispatcher での使用例
case "$(get_os)" in
  windows) bash "$script" "$@" ;;
  *)       exec "$script" "$@" ;;
esac
```

### PATH policy

dispatcher は trusted directories を PATH の先頭に置くことで tool spoofing を防ぐ。

```bash
# prepend only if not in PATH (ユーザー環境を保護)
prepend_path() {
  local dir="$1"
  case ":$PATH:" in
    *":$dir:"*) ;;       # already in PATH, skip
    *) PATH="$dir:$PATH" ;;
  esac
}

# Linux
prepend_path "/bin"
prepend_path "/usr/bin"
prepend_path "/usr/local/bin"

# macOS (Apple Silicon)
prepend_path "/bin"
prepend_path "/usr/bin"
prepend_path "/usr/local/bin"
prepend_path "/opt/homebrew/bin"

# Windows
# PATH は変更しない
```

`prepend_path()` は重複チェックを行い、すでに PATH に含まれるディレクトリはスキップします。
これにより `mise` / `asdf` / `nix` などユーザーが PATH 前方に設定したツールを上書きしません。

| ディレクトリ        | 用途                                        |
| ------------------- | ------------------------------------------- |
| `/opt/homebrew/bin` | Homebrew (macOS Apple Silicon)              |
| `/usr/local/bin`    | developer tools (brew Intel, mise, asdf 等) |
| `/usr/bin`          | OS 標準ツール                               |
| `/bin`              | 基本コマンド                                |

`/opt/homebrew/bin` は macOS Apple Silicon (M1/M2/M3) での Homebrew インストール先。
Intel Mac では `/usr/local/bin` が Homebrew のデフォルト。
Windows では PATH 操作が環境依存のため変更しない。

## 開発ツール仕様

### shellcheck オプション

aplys スクリプトの静的解析には以下のオプションを使用:

```bash
shellcheck \
  -s bash \
  -S warning \
  --format=gcc \
  --external-sources \
  "$@"
```

| オプション           | 説明                                                        |
| -------------------- | ----------------------------------------------------------- |
| `-s bash`            | bash スクリプトとして解析 (shebang による自動判定より優先)  |
| `-S warning`         | warning 以上をエラーとして扱う (info/style は無視)          |
| `--format=gcc`       | gcc 形式で出力 (GitHub Actions annotation として表示される) |
| `--external-sources` | `source` / `.` で読み込む外部ファイルを解析対象に含める     |

## CLI grammar

aplys のコマンド構文を EBNF で定義します。

```ebnf
command      ::= builtin | runner_slash | runner_space | plugin

builtin      ::= "install" bundle
               | "upgrade" "all"
               | "help" [ command_name ]

runner_slash ::= domain "/" target action files*
runner_space ::= domain target action files*

plugin       ::= name args*

bundle       ::= name
domain       ::= name
target       ::= name
action       ::= name
command_name ::= name

name         ::= [a-z][a-z0-9_-]*
args         ::= string*
files        ::= string*
```

`runner_slash` は `domain/target` をひとつのトークンとして受け取り、`action` を次の引数として取ります。
`domain/target/action` のようなスラッシュ 3 連結は正式構文ではありません。

dispatch は上から順に評価:

| 優先度 | パターン       | 判定条件                                  |
| ------ | -------------- | ----------------------------------------- |
| 1      | `builtin`      | `$1` が `install` / `upgrade` / `help`    |
| 2      | `runner_slash` | `$1` に `/` を含む                        |
| 3      | `runner_space` | `$# >= 3` かつ `is_domain "$1"`           |
| 4      | `plugin`       | `$APLYS_ROOT/plugins/aplys-$1` が存在する |

## スコープ外

以下は aplys の責務外とします。

- ツールのバージョン管理 (mise / asdf に委譲)
- 実行環境プロファイル切り替え (mise に委譲)
- パッケージレジストリの管理
