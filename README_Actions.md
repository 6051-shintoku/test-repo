# Actions: Import Issues from Another Repository

このドキュメントは、`.github/workflows/import-issues.yml` ワークフローを使って、他のリポジトリの Issue を手動で取り込む手順をまとめたマニュアルです。

## 目的

外部リポジトリ（public / private 両対応）の Issue を、このリポジトリにコピーしてアーカイブまたは移行するための手順を明確にすること。

## 重要な制限事項

### プライベートリポジトリアクセスについて

**Fine-grained Personal Access Token (推奨されるが制限あり):**
- ✅ 自分が所有するリポジトリにアクセス可能
- ✅ パブリックリポジトリに読み取り専用でアクセス可能
- ❌ **他人のプライベートリポジトリにはアクセス不可**

**Classic Personal Access Token (他人のプライベートリポジトリ用):**
- ✅ 適切な権限があれば他人のプライベートリポジトリにもアクセス可能
- ⚠️ より広範囲な権限が必要（repo スコープ）

### 対応状況
- ✅ パブリックリポジトリ: SOURCE_TOKEN不要（GITHUB_TOKEN で十分）
- ✅ 自分のプライベートリポジトリ: Fine-grained PAT 可能
- ✅ 他人のプライベートリポジトリ: Classic PAT または コラボレーター権限が必要

## 前提条件

- このリポジトリに `main` ブランチが存在しており、Actions が有効になっていること
- **プライベートリポジトリからインポートする場合**: 適切なPersonal Access Tokenが必要
- ワークフローはこのリポジトリで Issue を作成するため、デフォルトの `GITHUB_TOKEN` で十分

## Personal Access Token の設定

### パターン1: パブリックリポジトリのみ
設定不要。`GITHUB_TOKEN` が自動的に使用されます。

### パターン2: 自分のプライベートリポジトリ
1. GitHub → Settings → Developer settings → Personal access tokens → **Fine-grained tokens**
2. **Generate new token** をクリック
3. **Repository access**: "Selected repositories" → 対象リポジトリを選択
4. **Permissions**: 
   - Repository permissions → **Issues**: Read
   - Repository permissions → **Metadata**: Read
5. **Generate token** をクリックしてトークンをコピー

### パターン3: 他人のプライベートリポジトリ
1. GitHub → Settings → Developer settings → Personal access tokens → **Tokens (classic)**
2. **Generate new token (classic)** をクリック
3. **Select scopes**: `repo` (Full control of private repositories) をチェック
4. **Generate token** をクリックしてトークンをコピー

**または**、リポジトリ所有者に以下を依頼：
- コラボレーターとして招待してもらう
- リポジトリをパブリックに設定してもらう

### SOURCE_TOKEN の設定

1. `6051-shintoku/test-repo` リポジトリページ → **Settings** → **Secrets and variables** → **Actions**
2. **New repository secret** をクリック
3. **Name**: `SOURCE_TOKEN`
4. **Secret**: 作成したPersonal Access Tokenを貼り付け
5. **Add secret** をクリック

## ワークフローの概要

ファイル: `.github/workflows/import-issues.yml`

主な機能:
- `workflow_dispatch`（手動トリガー）でソースリポジトリを指定して実行
- 認証の自動判定（SOURCE_TOKEN → GITHUB_TOKEN の順で試行）
- 重複防止機能（本文内の `Imported from: owner/repo#N` マーカーで判定）
- Issues、コメント、ラベル、クローズ状態の完全コピー
- 詳細なエラーメッセージと解決方法の提示

## 実行手順（GitHub Web UI）

1. リポジトリの GitHub ページで **Actions** タブを開く
2. サイドバーから **「Import issues from other repo (manual)」** を選択
3. 右側の **Run workflow** ボタンをクリック
4. パラメータを入力:
   - **source_owner**: ソースリポジトリの所有者（例: `octocat`）
   - **source_repo**: ソースのリポジトリ名（例: `hello-world`）
   - **include_comments**: コメントもコピーする場合は ✅チェック（デフォルト: ON）
5. **Run workflow** を押して実行
6. 実行結果は Actions ページの実行ログで確認可能

### 実行例
```
source_owner: skawashin1122
source_repo: team-project-2025  
include_comments: ✅
```

## 実行手順（gh CLI）

```bash
# 基本的な実行
gh workflow run import-issues.yml \
  --ref main \
  --field source_owner=octocat \
  --field source_repo=hello-world \
  --field include_comments=true

# 実行状況の確認
gh run list --workflow=import-issues.yml
gh run view <run-id> --log

# SECRET の設定（必要に応じて）
gh secret set SOURCE_TOKEN --body 'github_pat_xxxxxxxxxx'
```

## 入力パラメータ

| パラメータ | 必須 | デフォルト | 説明 |
|-----------|------|-----------|------|
| `source_owner` | ✅ | - | ソースリポジトリの所有者名 |
| `source_repo` | ✅ | - | ソースリポジトリ名 |
| `include_comments` | ❌ | `true` | コメントもコピーするか |

## 環境変数・シークレット

| 変数名 | 必須 | 説明 |
|--------|------|------|
| `SOURCE_TOKEN` | 条件付き | プライベートリポジトリアクセス用PAT |
| `GITHUB_TOKEN` | 自動 | Actions で自動提供される（デフォルト） |

## 動作の詳細

### 認証の仕組み
1. `SOURCE_TOKEN` が設定されている場合はそれを使用
2. 設定されていない場合は `GITHUB_TOKEN` を使用（パブリックリポジトリのみ）
3. 認証状況とアクセス結果をログに出力

### インポート処理
1. **リポジトリアクセス確認**: ソースリポジトリへの読み取り権限を検証
2. **重複チェック**: 既存Issueの本文から `Imported from:` マーカーを検索
3. **Issues取得**: ページネーション対応でソースの全Issuesを取得
4. **Issue作成**: タイトル、本文、ラベルをコピーして新しいIssueを作成
5. **コメントコピー**: `include_comments=true` の場合、全コメントをコピー
6. **状態同期**: ソースがクローズ済みの場合、新しいIssueもクローズ

### 作成されるIssueの形式
```
タイトル: [元のタイトル]

本文:
Imported from: owner/repo#123
Original author: username
Original created_at: 2025-10-18T08:00:00Z
Original URL: https://github.com/owner/repo/issues/123

[元の本文]
```

## トラブルシューティング

### 🚨 プライベートリポジトリアクセスエラー

**エラー例**: `RequestError [HttpError]: Not Found` (404)

**原因と解決方法**:

1. **Fine-grained PAT で他人のプライベートリポジトリにアクセスしようとしている**
   ```
   ❌ Fine-grained PAT では他人のプライベートリポジトリにアクセス不可
   ✅ Classic PAT (repo スコープ) を使用
   ✅ または、リポジトリ所有者にコラボレーター招待を依頼
   ```

2. **SOURCE_TOKEN が設定されていない**
   ```
   ❌ GITHUB_TOKEN ではプライベートリポジトリにアクセス不可
   ✅ Personal Access Token を作成して SOURCE_TOKEN に設定
   ```

3. **PAT の権限不足**
   ```
   ❌ Issues: Read 権限が不足
   ✅ PAT の Repository permissions で Issues: Read を有効化
   ```

### 🔐 認証関連エラー

**エラー例**: `Bad credentials` (401), `Forbidden` (403)

**解決手順**:
1. PAT の有効期限を確認
2. PAT のスコープ・権限を確認
3. SOURCE_TOKEN の設定値を確認（コピペミスなど）
4. GitHub → Settings → Personal access tokens でトークンが Active か確認

### ⚡ レート制限エラー

**エラー例**: `API rate limit exceeded`

**対処法**:
- 少し時間をあけて再実行（1時間後など）
- 大量のIssueがある場合は分割実行を検討
- プライマリレート制限: 5,000 requests/hour (認証済み)

### 🔄 重複作成の問題

**症状**: 同じIssueが複数回作成される

**原因と対策**:
- Issue本文の `Imported from:` マーカーが削除・改変されている
- 手動でIssue本文を編集する際はマーカーを保持
- または、一度削除してから再インポート

### 🏷️ ラベル関連の問題

**症状**: ラベルが正しく設定されない

**確認点**:
- ソースリポジトリでラベルが正しく設定されているか
- デスティネーションリポジトリでのラベル作成権限
- ラベル名の文字エンコーディング（特殊文字など）

### 🔧 actions/github-script 互換性問題

**開発者向け情報**:
- `actions/github-script@v6` では `github.getOctokit` の使用に制限あり
- 現在のワークフローでは REST API の直接呼び出しを使用
- カスタム認証ヘッダーで SOURCE_TOKEN/GITHUB_TOKEN を適切に処理

### 📝 ログの確認方法

1. **Web UI**: Actions タブ → 実行履歴 → 詳細ログ
2. **CLI**: `gh run view <run-id> --log`

**重要なログメッセージ**:
```
✅ Using SOURCE_TOKEN for authentication
✅ Repository access confirmed: owner/repo, private: true
✅ created issue #N for owner/repo#M

❌ Using GITHUB_TOKEN for authentication
❌ Repository access failed: Not Found
❌ Private repository requires SOURCE_TOKEN
```

### 🆘 よくある質問

**Q: パブリックリポジトリなのにエラーが出る**
A: リポジトリ名・所有者名のタイプミス、またはリポジトリが削除されている可能性

**Q: コメントが一部しかコピーされない**
A: GitHub API のページネーション制限。ワークフローは自動対応済み

**Q: Issue の順序が変わってしまう**
A: GitHub API の仕様。作成日時順ではなく、処理順になる

**Q: Assignee や Milestone はコピーされないの？**
A: 現在未対応。ユーザーマッピングが複雑なため

---

## 今後の改善案

- [ ] Dry-run モード（インポート予定の一覧表示のみ）
- [ ] Assignee/Milestone のコピー（ユーザーマッピング機能付き）
- [ ] インポート対象フィルタ（日付範囲、ラベル条件など）
- [ ] バッチ処理モード（複数リポジトリの一括インポート）
- [ ] インポート履歴の CSV 出力

---

**問題や追加機能の希望があれば Issue を作成してください。**
