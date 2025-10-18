# Actions: Import issues from another repository

このドキュメントは、`.github/workflows/import-issues.yml` ワークフローを使って、他のリポジトリの Issue を手動で取り込む手順をまとめたマニュアルです。

## 目的

外部リポジトリ（public / private 両対応）の Issue を、このリポジトリにコピーしてアーカイブまたは移行するための手順を明確にすること。

## 前提条件

- このリポジトリに `main` ブランチが存在しており、Actions が有効になっていること。
- (プライベートソースを読む場合) 個人用アクセストークン（PAT）を作成し、このリポジトリの Settings → Secrets に `SOURCE_TOKEN` として追加しておくこと。推奨スコープ: `repo`（読み取り）または `repo:read` 相当。
- ワークフローはこのリポジトリで Issue を作成するため、デフォルトの `GITHUB_TOKEN` で十分です（デフォルトでリポジトリの権限を持ちます）。

## ワークフローの概要

ファイル: `.github/workflows/import-issues.yml`

主な機能:
- `workflow_dispatch`（手動トリガー）を使い、`source_owner` と `source_repo` を指定して実行します。
- ソースの Issue を順に取得して、タイトル・本文（本文には `Imported from: owner/repo#N` マーカーを付与）・ラベル・コメント（オプション）・クローズ状態をコピーします。
- 既にインポート済みの Issue は本文内のマーカーで検出し重複作成を避けます。

## 実行手順（GitHub UI）

1. リポジトリの GitHub ページで `Actions` タブを開きます。
2. サイドバーから「Import issues from other repo (manual)」を選択します。
3. 右側の `Run workflow` ボタンをクリックします。
4. 入力を指定します:
   - `source_owner`: ソースリポジトリの所有者（例: `octocat`）
   - `source_repo`: ソースのリポジトリ名（例: `hello-world`）
   - `include_comments`（任意）: `true`（デフォルト）または `false`。コメントをコピーしたくない場合は `false` を指定。
5. `Run workflow` を押して実行します。
6. 実行ログは Actions の実行ページで確認できます。問題があればログ（実行ステップの出力）をここに貼ってください。

## 実行手順（gh CLI）

ローカル環境に `gh` がインストールされ、リポジトリへの認証が済んでいる必要があります。

1. (プライベートソースで `SOURCE_TOKEN` を使う場合) リポジトリのシークレットは GitHub ウェブ UI で設定してください。`gh` から直接シークレットを作る場合は `gh secret set SOURCE_TOKEN --body '...'` を使えます。
2. ワークフローを手動で起動するコマンドの例:

```bash
gh workflow run import-issues.yml --ref main --field source_owner=octocat --field source_repo=hello-world --field include_comments=true
```

3. 実行状況を確認する:

```bash
gh run list --workflow=import-issues.yml
gh run view <run-id> --log
```

## 入力と環境変数

- `source_owner` (required): ソース repo の owner
- `source_repo` (required): ソース repo 名
- `include_comments` (optional): `true` または `false`（デフォルト `true`）

Secrets:
- `SOURCE_TOKEN` (optional): プライベートソースの読み取り用 PAT。設定がなければ `GITHUB_TOKEN` が代用されますが、`GITHUB_TOKEN` では他リポジトリの private データにアクセスできません。

## 振る舞いの詳細

- ラベル: ソースで付与されているラベル名を、そのまま新しい issue に付与します。リポジトリに同名のラベルが無ければ自動的に作成されます。
- コメント: `include_comments` が `true` のとき、ソースの issueコメントをすべて（ページネーション対応）コピーします。各コメントには「Imported comment from USER at TIMESTAMP」というヘッダが付きます。
- 既存チェック: 既に取り込まれた issue は本文内の `Imported from: owner/repo#N` で判定しスキップします。
- クローズ状態: ソースが `closed` の場合は作成後にクローズします。

## トラブルシューティング

- トークン不足エラー: プライベートリポジトリへアクセスしようとして `Not Found` や `403` が出る場合、`SOURCE_TOKEN` が不足しています。PAT の権限を確認して再設定してください。
- レート制限: 大量の Issue を短時間で取得する場合、GitHub API レート制限に到達する可能性があります。エラーが出る場合は少し時間をあけて再実行してください。
- 重複作成: 同じソースを複数回インポートすると重複する可能性があります。本文のマーカーが消えていたり改変されていると判別できません。

## 追加でできる改善案（必要なら実装します）

- dry-run モード（何をインポートするか一覧だけ出す）
- assignee / milestone のコピー（ユーザーのマッピングが必要）
- インポート対象のフィルタ（例: created_after、labels で絞る）
- Slack 等への通知

---

問題や追加機能の希望があれば教えてください。必要ならこのマニュアルを日本語/英語の別バージョンで整備します。
