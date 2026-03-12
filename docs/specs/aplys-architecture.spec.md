---
title: aplys アーキテクチャ仕様
description: aplys の全体設計・レイヤー構造・コンポーネント間の関係を定義する
author: atsushifx
version: 0.1.0
---

## aplys アーキテクチャ仕様

本ドキュメントは aplys の全体設計・レイヤー構造・コンポーネント間の依存関係を定義します。
各コンポーネントの実装詳細は個別の spec ドキュメントを参照してください。

## 設計思想

aplys は **tool execution router** です。

```text
aplys <domain>/<target> <action> [files...]
```

`domain/target/action` の 3 層モデルでツールの discovery・execution を統合します。
ツールのバージョン管理・環境構築は mise / asdf 等の外部ツールに委譲します。

### 設計原則

| 原則                         | 意味                                                  |
| ---------------------------- | ----------------------------------------------------- |
| stateless                    | 同じ引数で同じ結果を返す                              |
| installer is authority       | install / upgrade の実行はすべて provider に委譲      |
| bundle is grouping only      | bundle は tool 名の集合定義にすぎない                 |
| do one thing                 | runner はツールを起動するだけ                         |
| delegate complexity          | sudo / ロック / バージョン管理は provider / OS に委譲 |
| prefer idempotent operations | 繰り返し実行しても副作用のない操作を選ぶ              |
| filesystem is truth          | domain の存在確認はディレクトリ存在で行う             |

## レイヤー構造

aplys は 2 つのレイヤーに分かれます。

```text
┌──────────────────────────────────────────────────┐
│               Core runtime (必須)                │
│   kernel / dispatch / runner / plugin            │
│   providers  (package manager adapter)           │
│         tool の実行・routing を担う              │
├──────────────────────────────────────────────────┤
│           Optional bootstrap (補助)              │
│              bundle / registry                   │
│       tool のインストール環境を構築する          │
└──────────────────────────────────────────────────┘
```

bundle / registry は tool 導入補助にすぎません。
aplys の本質は `runner` による tool execution です。

`providers` は package manager adapter です。Core runtime (`lib/providers/`) に配置されますが、
Bootstrap フローからのみ呼び出されます。

| レイヤー  | 構成要素                                    | 役割                    |
| --------- | ------------------------------------------- | ----------------------- |
| Core      | kernel, dispatch, runner, plugin, providers | tool の実行・routing    |
| Bootstrap | bundle, registry                            | tool のインストール補助 |

## コンポーネント概要

### Core コンポーネント

| コンポーネント | 配置                                    | 役割                                              |
| -------------- | --------------------------------------- | ------------------------------------------------- |
| kernel         | `lib/kernel/main.sh`                    | CLI routing (validate → lookup → dispatch → exec) |
| dispatch       | `lib/dispatch/`                         | builtin / plugin / runner のパス解決              |
| runner         | `runners/<domain>/<target>/<action>.sh` | 外部ツールの薄いラッパー                          |
| plugin         | `plugins/aplys-<name>`                  | `aplys <name>` で呼び出される管理系拡張           |
| builtin        | `builtin/<cmd>.sh`                      | `install` / `upgrade` / `help` の内部実装         |

### Bootstrap コンポーネント

| コンポーネント | 配置                         | 役割                                     |
| -------------- | ---------------------------- | ---------------------------------------- |
| bundle         | `tooling/bundles/<bundle>`   | tool name の集合定義 (1行1エントリ)      |
| tool registry  | `tools-registry/<tool>.yaml` | tool → (provider, packages) のマッピング |

> `providers` は Bootstrap から呼び出されますが、runtime plugin として Core (`lib/providers/`) に配置します。

## コンポーネント間の依存関係

```text
bin/aplys
  └─ lib/kernel/main.sh          (validate → lookup → dispatch → exec)
       ├─ lib/dispatch/resolve_builtin.sh
       │    └─ builtin/<cmd>.sh
       │         └─ lib/providers/<provider>.sh  (install/upgrade 時のみ)
       ├─ lib/dispatch/resolve_plugin.sh
       │    └─ plugins/aplys-<name>
       └─ lib/dispatch/resolve_runner.sh
            ├─ $APLYS_DATA_DIR/tools/<domain>/<target>/<action>.sh  (user優先)
            └─ runners/<domain>/<target>/<action>.sh                (builtin)
```

Bootstrap レイヤーは Core から一方向に依存します。逆方向の依存はありません。

```text
builtin/install.sh
  └─ tooling/bundles/<bundle>          bundle ファイル読み込み
       └─ tools-registry/<tool>.yaml   registry 解決
            └─ lib/providers/<provider>.sh  provider 実行
```

## ディレクトリ構造

```text
aplys/
  bin/
    aplys                   # CLI エントリポイント

  lib/                      # Core runtime
    kernel/
      main.sh               # validate → lookup → dispatch → exec
    dispatch/
      resolve_builtin.sh    # builtin コマンド解決
      resolve_plugin.sh     # plugin 解決
      resolve_runner.sh     # runner パス解決・存在確認・実行可能確認
    providers/              # package manager adapter (Bootstrap で使用)
      apt.sh
      brew.sh
      pnpm.sh
      npm.sh
      yarn.sh
      scoop.sh
      winget.sh

  runners/                  # Core: runner スクリプト群 (tool wrapper)
    <domain>/
      <target>/
        <action>.sh

  builtin/                  # builtin コマンド実装
    install.sh
    upgrade.sh
    help.sh

  plugins/                  # Core: plugin スクリプト群
    aplys-list
    aplys-usage
    aplys-what

  tools-registry/           # tool → provider マッピング
    <tool>.yaml

  tooling/                  # Optional (Bootstrap)
    bundles/
      dev-tools
      doc-tools

  _libs/                    # 内部共有ライブラリ (予約)
```

## dispatch フロー

kernel は受け取ったコマンドを次の優先順位で処理します。

```text
[入力]
  aplys <args>
      ↓
[validate]
  validate_env  # bash / rg の存在確認
      ↓
[dispatch]
  1. install|upgrade|help          → builtin       (予約済みコマンド)
  2. $1 == */* && $# >= 2          → runner_slash  ($1=domain/target, $2=action)
  3. is_domain $1 && $# >= 3       → runner_space  (domain target action 形式)
  4. aplys-$1 が存在する           → plugin
  5. それ以外                      → exit 2
      ↓
[exec]
  exec "$script" "$@"   (Linux/macOS)
  bash "$script" "$@"   (Windows)
```

### runner パス解決

runner は direct path 計算のみ行います。filesystem scan は行いません。

```text
user-installed: $APLYS_DATA_DIR/tools/<domain>/<target>/<action>.sh  (優先)
builtin:        $APLYS_ROOT/runners/<domain>/<target>/<action>.sh
```

`APLYS_ALLOW_OVERRIDE=0` 設定時は builtin 優先に反転します。

## Bootstrap フロー

```text
[入力]
  bundle name
      ↓
[bundle 解決]
  bundle → tool name list
      ↓
[registry 解決]
  tool name → (provider, packages[], executable) per tool
      ↓
[node provider 解決]
  provider=node → node_installer 設定 → pnpm | npm | yarn
      ↓
[install 判定]
  command -v executable → 既存 tool をスキップ
      ↓
[provider batching]
  同一 provider の packages をまとめる
      ↓
[provider 実行]
  provider ごとに 1 プロセスを起動し、packages 配列を一括渡しする
    例: pnpm add -g A B C / brew install A B / apt-get install -y A
  provider 単位で並列実行 (apt のみ dpkg lock のため逐次)
```

## セキュリティモデル

| 脅威                 | 対策                                                      |
| -------------------- | --------------------------------------------------------- |
| directory traversal  | `domain`/`target`/`action` を `^[a-z][a-z0-9_-]*$` で検証 |
| PATH hijack (plugin) | `$APLYS_ROOT/plugins/` と `APLYS_PLUGIN_PATH` のみ探索    |
| PATH hijack (runner) | `$APLYS_ROOT/runners/` と `$APLYS_DATA_DIR/tools/` のみ   |
| tool spoofing        | `prepend_path()` で trusted directory を PATH 先頭に配置  |
| builtin 上書き       | `APLYS_ALLOW_OVERRIDE=0` で builtin 優先に戻せる          |

スクリプトパスは `$APLYS_ROOT` 配下であることを検証します (path traversal 防御)。

## クロスプラットフォーム

| OS      | runner 起動方法       | provider       |
| ------- | --------------------- | -------------- |
| Linux   | `exec "$script" "$@"` | apt            |
| macOS   | `exec "$script" "$@"` | brew           |
| Windows | `bash "$script" "$@"` | scoop / winget |

`node` provider は OS 横断で使用します。実体は `node_installer` 設定 (`pnpm` / `npm` / `yarn`) です。

WSL は `$OSTYPE` が `linux-gnu` となるため、`linux` として動作します。

## 仕様ドキュメント一覧

| ドキュメント                 | 内容                                                          |
| ---------------------------- | ------------------------------------------------------------- |
| `aplys-architecture.spec.md` | 本ドキュメント。全体設計・レイヤー・依存関係                  |
| `aplys-api.spec.md`          | CLI インターフェース・exit code・環境変数・バージョンポリシー |
| `aplys-cli.spec.md`          | ディレクトリ構造・dispatch アルゴリズム・セキュリティ実装     |
| `aplys-bundler.spec.md`      | bundle 定義・tool レジストリ・install アルゴリズム            |
| `aplys-provider.spec.md`     | provider インターフェース・OS 別検出・idempotent 設計         |
