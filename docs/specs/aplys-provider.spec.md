---
title: aplys Provider
description: aplys provider インターフェース・OS 別検出・package 解決仕様
author: atsushifx
version: 0.2.0
---

## aplys Provider Specification (Draft)

本ドキュメントは aplys の provider インターフェース・OS 別検出順序・package 解決の仕様を定義します。
bundle / tool レジストリの詳細は `aplys-bundler.spec.md` を参照してください。

> **provider は Optional bootstrap レイヤーです。**
> tool の実行 (runner) には不要です。`aplys install` / `aplys upgrade` 時のみ使用します。

## provider の役割

provider は OS・パッケージマネージャーの差異を吸収します。

```text
tool レジストリ
 ↓
OS 検出 → provider 決定
 ↓
provider batching (同一 provider の packages をまとめる)
 ↓
provider_install <packages...>
```

provider は **install / upgrade の実行手段**のみを担います。
idempotent はパッケージマネージャー自身に委譲します。

## provider 一覧

### system providers

OS ネイティブツールのインストールに使用します。

| provider | 対象 OS | 主な用途             |
| -------- | ------- | -------------------- |
| `scoop`  | Windows | Windows CLI ツール   |
| `winget` | Windows | Windows アプリ       |
| `brew`   | macOS   | macOS ツール         |
| `apt`    | Linux   | Debian/Ubuntu ツール |

### node provider (仮想)

Node.js CLI ツールのインストールに使用します。

| provider | 実体                                        |
| -------- | ------------------------------------------- |
| `node`   | `node_installer` 設定値 (pnpm / npm / yarn) |

tool レジストリで `providers.default: node` と指定されたツールは、
bundle resolve 後に kernel が以下の手順で実 provider に解決します。

```text
provider = node
 ↓
node_installer 設定を読み込む ($APLYS_CONFIG_HOME/config.yaml)
 ↓
provider = pnpm | npm | yarn  (実 provider に置換)
```

解決後は通常の provider と同様に `${provider}_install` が呼び出されます。
`node_install` という関数は存在しません。

`node_installer` の起動時チェック:

```bash
command -v "$node_installer" >/dev/null 2>&1 \
  || { echo "aplys: $node_installer not found" >&2; exit 2; }
```

## provider 検証

provider の存在確認は kernel が `<provider>_is_installed` を呼び出して行います。
provider スクリプト内部では確認しません。

`install` / `upgrade` / `bootstrap` の bundle resolve 後、使用する provider が確定した時点で kernel が検証します。
未インストールの場合は exit 2 とします。

aplys は provider 自身をインストールしません。
事前インストールはユーザーの責務です。

| provider | `_is_installed` の実装 |
| -------- | ---------------------- |
| `pnpm`   | `command -v pnpm`      |
| `npm`    | `command -v npm`       |
| `yarn`   | `command -v yarn`      |
| `scoop`  | `command -v scoop`     |
| `winget` | `command -v winget`    |
| `brew`   | `command -v brew`      |
| `apt`    | `command -v apt-get`   |

## provider インターフェース

各 provider スクリプトは `<provider>_` プレフィックスを付けた関数を実装します。

```bash
<provider>_is_installed()        # provider コマンドが利用可能か確認
<provider>_install <package...>  # パッケージをインストール (複数可)
<provider>_upgrade <package...>  # パッケージを更新 (複数可)
<provider>_upgrade_all()         # provider 管理下の全パッケージを一括更新
```

`scoop` provider の実装例を以下に示します。

```bash
scoop_is_installed()
scoop_install <package...>
scoop_upgrade <package...>
scoop_upgrade_all()
```

aplys kernel は `source` でファイルを読み込み、`${provider}_install "$@"` のように呼び出します。

```bash
# kernel 側の呼び出し例
source "lib/providers/${provider}.sh"

# provider 存在確認 (kernel が統一して行う)
if ! "${provider}_is_installed"; then
  echo "aplys: provider '${provider}' not found" >&2
  exit 2
fi

"${provider}_install" "${packages[@]}"
```

### `<provider>_is_installed`

```bash
<provider>_is_installed()
# 戻り値: 0=利用可能 / 1=利用不可
```

`command -v <provider>` で検出します。
provider 内部ではチェックを行いません。kernel が統一して呼び出し、失敗時は exit 2 とします。

### `<provider>_install`

```bash
<provider>_install <package...>
# 複数パッケージを一度に受け取る (batching)
# 戻り値: package manager の exit code をそのまま返す
```

idempotent はパッケージマネージャーに委譲します。

| provider | 関数名           | 実行コマンド例                    |
| -------- | ---------------- | --------------------------------- |
| `pnpm`   | `pnpm_install`   | `pnpm add -g <package...>`        |
| `npm`    | `npm_install`    | `npm install -g <package...>`     |
| `yarn`   | `yarn_install`   | `yarn global add <package...>`    |
| `scoop`  | `scoop_install`  | `scoop install <package...>`      |
| `winget` | `winget_install` | `winget install <package>` (逐次) |
| `brew`   | `brew_install`   | `brew install <package...>`       |
| `apt`    | `apt_install`    | `apt-get install -y <package...>` |

`winget` は複数パッケージの一括指定に対応していません。`winget_install` 内部でループして 1 件ずつ実行します。

```bash
winget_install() {
  for pkg in "$@"; do
    winget install "$pkg"
  done
}
```

### `<provider>_upgrade`

```bash
<provider>_upgrade <package...>
# 複数パッケージを一度に受け取る (batching)
# 戻り値: package manager の exit code をそのまま返す
```

| provider | 関数名           | 実行コマンド例                                |
| -------- | ---------------- | --------------------------------------------- |
| `pnpm`   | `pnpm_upgrade`   | `pnpm update -g <package...>`                 |
| `npm`    | `npm_upgrade`    | `npm update -g <package...>`                  |
| `yarn`   | `yarn_upgrade`   | `yarn global upgrade <package...>`            |
| `scoop`  | `scoop_upgrade`  | `scoop update <package...>`                   |
| `winget` | `winget_upgrade` | `winget upgrade <package>` (逐次)             |
| `brew`   | `brew_upgrade`   | `brew upgrade <package...>`                   |
| `apt`    | `apt_upgrade`    | `apt-get install --only-upgrade <package...>` |

`apt_upgrade` は `apt-get upgrade` ではなく `apt-get install --only-upgrade` を使用します。
`apt-get upgrade` はシステム全体の更新になるためです。

### `<provider>_upgrade_all`

```bash
<provider>_upgrade_all()
# 戻り値: package manager の exit code をそのまま返す
```

`aplys upgrade all` から呼び出されます。
provider 管理下の全パッケージを一括更新します。`bundles/` の走査は行いません。

| provider | 関数名               | 実行コマンド例         |
| -------- | -------------------- | ---------------------- |
| `pnpm`   | `pnpm_upgrade_all`   | `pnpm update -g`       |
| `npm`    | `npm_upgrade_all`    | `npm update -g`        |
| `yarn`   | `yarn_upgrade_all`   | `yarn global upgrade`  |
| `scoop`  | `scoop_upgrade_all`  | `scoop update *`       |
| `winget` | `winget_upgrade_all` | `winget upgrade --all` |
| `brew`   | `brew_upgrade_all`   | `brew upgrade`         |
| `apt`    | `apt_upgrade_all`    | `apt-get upgrade -y`   |

## provider ファイル構成

```text
lib/
  providers/
    pnpm.sh
    npm.sh
    yarn.sh
    scoop.sh
    winget.sh
    brew.sh
    apt.sh
```

各ファイルは `<provider>_` プレフィックス付きの関数を実装した bash スクリプトです。
実装する関数は次の 4 つです。

- `<provider>_is_installed`
- `<provider>_install`
- `<provider>_upgrade`
- `<provider>_upgrade_all`

`apt` provider のみ、package index 更新のための内部関数を追加で実装します。
`apt_repository_update` は kernel からは呼び出されず、`apt_install` / `apt_upgrade` が内部で使用します。

各 provider スクリプトの先頭に次のシェバンとオプションを記述します。

```bash
#!/usr/bin/env bash
set -euo pipefail
```

## apt provider の update ポリシー

`apt-get install` は package index が古いと `package not found` になります。
特に fresh container (CI/Docker) では初回実行前に `apt-get update` が必要です。

`apt` provider は初回 `apt_install` / `apt_upgrade` 呼び出し時に 1 度だけ `apt-get update` を実行します。

```bash
# apt.sh
_apt_repository_updated=false

apt_repository_update() {
  if [[ "$_apt_repository_updated" == "false" ]]; then
    apt-get update
    _apt_repository_updated=true
  fi
}

apt_install() {
  apt_repository_update
  apt-get install -y "$@"
}

apt_upgrade() {
  apt_repository_update
  apt-get install --only-upgrade -y "$@"
}
```

`apt_repository_update` は `<provider>_` プレフィックス付きの内部関数です。
`_apt_repository_updated` フラグはプロセス内で保持され、同一 aplys 実行内で `apt-get update` は 1 度のみ実行されます。
kernel はこの関数を直接呼び出しません。`apt_install` / `apt_upgrade` が内部で呼び出します。

## sudo ポリシー

aplys は `sudo` を直接実行しません。
`apt` 等のシステム provider は、root 権限が必要なコマンド実行時に sudo を使用します。

```bash
# apt provider 内部の実装例
apt_install() {
  apt_repository_update
  apt-get install -y "$@"   # sudo が必要な場合は apt 側で処理
}
```

sudo のタイムアウト・パスワードプロンプト・キャッシュは OS の責務です。
aplys はこれらに関与しません。

## idempotent 設計

aplys は idempotent をパッケージマネージャーに委譲します。

| provider | 既存パッケージへの動作 |
| -------- | ---------------------- |
| `pnpm`   | 既存なら skip (exit 0) |
| `scoop`  | 既存なら skip (exit 0) |
| `brew`   | 既存なら skip (exit 0) |
| `apt`    | 既存なら skip (exit 0) |

aplys 側での SHA256 比較・存在チェックは行いません。
パッケージマネージャーが冪等性を保証します。

## 並列実行ポリシー

provider batching 後の実行方式:

| provider     | 実行方式 | 理由       |
| ------------ | -------- | ---------- |
| `pnpm`/`npm` | 並列     | ロックなし |
| `yarn`       | 並列     | ロックなし |
| `scoop`      | 並列     | ロックなし |
| `winget`     | 並列     | ロックなし |
| `brew`       | 並列     | ロックなし |
| `apt`        | 逐次     | dpkg lock  |
