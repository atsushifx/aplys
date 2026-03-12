# CLAUDE.md

## プロジェクト概要

`aplys` は bash shell script ベースの toolchain runner。
`aplys <domain>/<target> <action> [files...]` でツールを起動する CLI ラッパー。
現在は仕様策定フェーズ。主要成果物: `docs/specs/aplys-api.spec.md`。

## コマンド

- 開発環境セットアップ: `bash scripts/setup-dev-env.sh`
- Markdown lint:
  `markdownlint-cli2 --config "${XDG_CONFIG_HOME}/linters/markdownlint/.markdownlint-cli2.yaml" <file>`
- textlint:
  `textlint --config "${XDG_CONFIG_HOME}/linters/textlint/textlintrc.yaml" <file>`
- フォーマット: `dprint fmt`

## aplys 設計原則

- builtin first: `$APLYS_ROOT/<domain>/<target>/<action>` → `$APLYS_DATA_DIR/tools/...` の順で探索
- idempotent: `aplys install` は SHA256 比較で既存ファイルをスキップ
- exit code: 0=success / 1=tool error / 2=aplys internal / 3=invalid arg / 127=not found
- バリデーション: `domain`/`target`/`action`/`bundle-tools` は `^[a-z][a-z0-9_-]*$` のみ許可

## コミットメッセージ規約

- Conventional Commits 準拠、ヘッダー最大 76 文字
- 許可 type: `feat` `fix` `chore` `docs` `test` `refactor` `perf` `ci` `config` `release` `merge` `build` `style` `deps`
- ボディは**日本語**で記述 (ファイルパス・技術用語は英語のまま)
- `lefthook` の `prepare-commit-msg` フックが AI でメッセージを自動生成する

## 参照ドキュメント

- API 仕様: `docs/specs/aplys-api.spec.md`
- Git フック設定: `lefthook.yml`
- コミットメッセージ生成テンプレート: `.claude/agents/commit-message-generator.md`
