# zoho-deluge スキル（Cursor Skills 形式）

このフォルダは **Cursor Remote Rule (GitHub)**（方法4）用の構造です。

- **SKILL.md**: スキル本文。言語制約・API ルールに加え、**実際の使用例**（invokeurl、関数テンプレート、COQL、カスタムボタン等）を記載。Cursor がセマンティックマッチで選択したときに読み込まれる。
- **references/usage-examples.md**: 追加の実装例（ループ代替、attachFile、ワークフロー無効化、リスト型判定等）。実装時は SKILL.md とあわせて参照すること。
- プロジェクト別の仕様書インデックス（03）は含めない。各プロジェクトの `skills/03_zoho-deluge-project-specs-index.md` で管理する。

## GitHub で共有する場合の手順

1. この `.claude/skills/` フォルダごと、**新しいリポジトリ**のルートに置く（リポジトリルート直下に `.claude/skills/zoho-deluge/` が存在する形にする）。
2. そのリポジトリを GitHub に push する。
3. Cursor IDE を **2.4.0-pre.11.patch.0 以降**に更新する。
4. Cursor の設定 → **Remote Rule (Github)** を開き、上記リポジトリの URL を入力してインポートする。
5. チャットや .deluge 編集時に、Zoho Deluge 用のスキルが自動選択されるか確認する。

詳細はプロジェクト内 `skills/共同管理ガイド.md` の「方法4」を参照。
