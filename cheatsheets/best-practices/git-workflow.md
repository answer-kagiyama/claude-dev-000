# Git ワークフロー ベストプラクティス

## ブランチ戦略

### GitHub Flow（推奨：小規模〜中規模）

```bash
# シンプルで継続的デプロイに最適
main (常にデプロイ可能)
 └─ feature/add-login
 └─ feature/fix-bug-123
 └─ hotfix/security-patch

# ワークフロー
1. main から feature ブランチ作成
2. コミット・プッシュ
3. Pull Request 作成
4. レビュー・承認
5. main にマージ
6. 自動デプロイ
```

```bash
# 実践例
git checkout main
git pull origin main
git checkout -b feature/add-user-profile

# 開発...
git add .
git commit -m "feat: add user profile page"
git push -u origin feature/add-user-profile

# PR作成 → レビュー → マージ
# マージ後
git checkout main
git pull origin main
git branch -d feature/add-user-profile
```

### Git Flow（複雑なリリース管理）

```bash
# 複数バージョンの並行開発・保守
main (本番)
 └─ develop (開発)
     ├─ feature/new-feature
     ├─ release/v1.2.0
     └─ hotfix/critical-bug

# ブランチの役割
main      : 本番環境（タグ付き）
develop   : 次期リリースの統合
feature/* : 機能開発
release/* : リリース準備
hotfix/*  : 緊急修正
```

```bash
# Feature 開発
git checkout develop
git checkout -b feature/add-payment
# 開発...
git checkout develop
git merge --no-ff feature/add-payment
git branch -d feature/add-payment

# Release 準備
git checkout -b release/v1.2.0 develop
# バグ修正、バージョン更新...
git checkout main
git merge --no-ff release/v1.2.0
git tag -a v1.2.0 -m "Release version 1.2.0"
git checkout develop
git merge --no-ff release/v1.2.0
git branch -d release/v1.2.0

# Hotfix
git checkout -b hotfix/security-fix main
# 修正...
git checkout main
git merge --no-ff hotfix/security-fix
git tag -a v1.2.1 -m "Hotfix: security patch"
git checkout develop
git merge --no-ff hotfix/security-fix
git branch -d hotfix/security-fix
```

### Trunk-Based Development（大規模チーム）

```bash
# main ブランチ中心、短命フィーチャーブランチ
main (常にリリース可能)
 └─ feature/short-lived-1 (1-2日)
 └─ feature/short-lived-2

# 原則
- フィーチャーブランチは短命（1-2日）
- 頻繁に main にマージ
- Feature Flag で未完成機能を隠す
- CI/CD が必須
```

## ブランチ命名規則

```bash
# 推奨フォーマット
{type}/{issue-number}-{description}

# 例
feature/123-add-user-authentication
bugfix/456-fix-memory-leak
hotfix/789-security-vulnerability
refactor/reduce-technical-debt
docs/update-api-documentation
test/add-integration-tests
chore/update-dependencies

# Type の種類
feature  : 新機能
bugfix   : バグ修正
hotfix   : 緊急修正
refactor : リファクタリング
docs     : ドキュメント
test     : テスト追加・修正
chore    : ビルド・ツール等
perf     : パフォーマンス改善
```

## コミットメッセージ

### Conventional Commits（推奨）

```bash
# フォーマット
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]

# 例
feat(auth): add JWT authentication

Implement JWT token generation and validation
for API authentication.

Closes #123
```

### Type 一覧

```bash
feat:     新機能
fix:      バグ修正
docs:     ドキュメント変更
style:    コードフォーマット（機能変更なし）
refactor: リファクタリング
perf:     パフォーマンス改善
test:     テスト追加・修正
build:    ビルドシステム変更
ci:       CI設定変更
chore:    その他の変更
revert:   コミット取り消し
```

### 良いコミットメッセージ

```bash
# ✅ 良い例
feat(api): add user registration endpoint

Add POST /api/users endpoint for new user registration.
Includes input validation and password hashing.

Closes #123

# ✅ 良い例（シンプル）
fix: resolve memory leak in image processing

# ✅ Breaking Change
feat!: remove deprecated API v1 endpoints

BREAKING CHANGE: API v1 endpoints have been removed.
Migrate to v2 endpoints.

# ❌ 悪い例
fix bug              # 何のバグ？
update               # 何を更新？
WIP                  # 作業中のコミットは避ける
misc changes         # 具体性がない
```

### コミットの粒度

```bash
# ✅ 適切な粒度
git commit -m "feat: add user model"
git commit -m "feat: add user repository"
git commit -m "feat: add user service"
git commit -m "test: add user service tests"

# ❌ 粗すぎる
git commit -m "feat: implement entire user feature"

# ❌ 細かすぎる
git commit -m "fix: fix typo"
git commit -m "fix: fix another typo"
git commit -m "fix: fix spacing"
```

## Pull Request

### 良いPRの条件

```markdown
# PR タイトル
feat(auth): implement OAuth2 authentication

# PR 説明テンプレート
## 概要
OAuth2認証機能を実装しました。

## 変更内容
- OAuth2プロバイダー（Google, GitHub）対応
- トークンリフレッシュ機能
- ユーザー情報取得API

## 関連Issue
Closes #123

## テスト
- [ ] ユニットテスト追加
- [ ] 統合テスト追加
- [ ] 手動テスト完了

## スクリーンショット
（必要に応じて）

## チェックリスト
- [ ] コードレビュー依頼済み
- [ ] テストが通る
- [ ] ドキュメント更新済み
- [ ] Breaking Change なし
```

### PRサイズ

```bash
# ✅ 適切なサイズ
+150 -50 lines  # レビューしやすい

# ⚠️ 大きすぎる
+1500 -800 lines  # 分割を検討

# 目安
- 小: ~200行
- 中: 200-500行
- 大: 500-1000行
- 超大: 1000行以上（分割推奨）
```

### Draft PR

```bash
# 早期フィードバック用
1. Draft PR を作成
2. 設計レビュー
3. Ready for Review に変更
4. 最終レビュー
5. マージ
```

## コードレビュー

### レビュアー視点

```markdown
# チェック項目

## 機能
- [ ] 要件を満たしているか
- [ ] エッジケースを考慮しているか
- [ ] エラーハンドリングが適切か

## コード品質
- [ ] 読みやすいコードか
- [ ] 命名が適切か
- [ ] DRY原則に従っているか
- [ ] SOLIDの原則に従っているか

## テスト
- [ ] 十分なテストがあるか
- [ ] テストが意味のあるものか

## セキュリティ
- [ ] SQL injection 対策
- [ ] XSS 対策
- [ ] 認証・認可の確認
- [ ] 機密情報の漏洩がないか

## パフォーマンス
- [ ] N+1問題がないか
- [ ] 不要な処理がないか
- [ ] キャッシュを活用しているか

## ドキュメント
- [ ] コメントは適切か
- [ ] API仕様が更新されているか
```

### コメントの書き方

```markdown
# ✅ 建設的なコメント
この処理は O(n²) になっています。
Map を使うと O(n) に改善できます：

​```js
const userMap = new Map(users.map(u => [u.id, u]));
const result = items.map(item => userMap.get(item.userId));
​```

# ✅ 質問形式
この場合、null チェックは必要ですか？
エッジケースで問題が起きる可能性があります。

# ❌ 否定的
これは間違っています。

# ❌ 曖昧
ここを直してください。
```

### レビューレベル

```markdown
# コメント接頭辞
[nit]:       些細な指摘（必須でない）
[question]:  質問
[suggestion]: 提案
[important]: 重要な指摘
[blocking]:  修正必須

# 例
[nit] 変数名を `userData` から `user` にすると読みやすいです
[important] エラーハンドリングが不足しています
[blocking] セキュリティ上の問題があります
```

## マージ戦略

### Merge Commit（推奨：履歴保持）

```bash
git checkout main
git merge --no-ff feature/add-login

# メリット
- ブランチ履歴が残る
- PRとコミットの対応が明確

# デメリット
- 履歴が複雑になる
```

### Squash and Merge（推奨：綺麗な履歴）

```bash
git checkout main
git merge --squash feature/add-login
git commit -m "feat: add login feature"

# メリット
- main ブランチが綺麗
- 1 PR = 1 コミット

# デメリット
- 詳細な履歴が失われる
```

### Rebase and Merge

```bash
git checkout feature/add-login
git rebase main
git checkout main
git merge --ff-only feature/add-login

# メリット
- 直線的な履歴
- マージコミットなし

# デメリット
- コンフリクト解決が複雑
- 履歴が書き換わる
```

## タグとリリース

### セマンティックバージョニング

```bash
# フォーマット: MAJOR.MINOR.PATCH
v1.2.3

MAJOR: 互換性のない変更
MINOR: 後方互換性のある機能追加
PATCH: 後方互換性のあるバグ修正

# 例
v1.0.0  # 初回リリース
v1.1.0  # 機能追加
v1.1.1  # バグ修正
v2.0.0  # Breaking Change
```

### タグ作成

```bash
# Annotated Tag（推奨）
git tag -a v1.2.0 -m "Release version 1.2.0"
git push origin v1.2.0

# リリース内容を含める
git tag -a v1.2.0 -m "Release v1.2.0

Features:
- Add user authentication
- Add admin dashboard

Bug Fixes:
- Fix memory leak in image processing
"

# すべてのタグをプッシュ
git push origin --tags
```

### リリースノート

```markdown
# Release v1.2.0

## ✨ Features
- Add OAuth2 authentication (#123)
- Add email notifications (#124)

## 🐛 Bug Fixes
- Fix memory leak in image processing (#125)
- Fix race condition in cache (#126)

## 📚 Documentation
- Update API documentation (#127)

## ⚠️ Breaking Changes
- Remove deprecated `/api/v1` endpoints

## 🔧 Maintenance
- Update dependencies
- Improve test coverage
```

## よくあるワークフロー

### Feature 開発

```bash
# 1. 最新の main を取得
git checkout main
git pull origin main

# 2. Feature ブランチ作成
git checkout -b feature/123-add-dark-mode

# 3. 開発
git add .
git commit -m "feat: add dark mode toggle"

# 4. main の変更を取り込む（rebase推奨）
git fetch origin
git rebase origin/main

# コンフリクト解決
git add .
git rebase --continue

# 5. プッシュ（初回）
git push -u origin feature/123-add-dark-mode

# 6. プッシュ（2回目以降、rebase後）
git push --force-with-lease

# 7. PR作成・レビュー・マージ

# 8. ローカルブランチ削除
git checkout main
git pull origin main
git branch -d feature/123-add-dark-mode
```

### Hotfix

```bash
# 1. main から hotfix ブランチ
git checkout main
git pull origin main
git checkout -b hotfix/critical-security-fix

# 2. 修正
git add .
git commit -m "fix: patch security vulnerability CVE-2024-1234"

# 3. プッシュ・PR・マージ
git push -u origin hotfix/critical-security-fix

# 4. タグ作成
git checkout main
git pull origin main
git tag -a v1.2.1 -m "Hotfix: security patch"
git push origin v1.2.1
```

### コミット修正

```bash
# 直前のコミットを修正
git add .
git commit --amend -m "feat: add user profile (updated)"

# プッシュ済みの場合
git push --force-with-lease

# ⚠️ 注意: 他の人が使っているブランチでは使わない
```

### コミット取り消し

```bash
# まだプッシュしていない
git reset --soft HEAD~1  # コミット取り消し、変更は残る
git reset --hard HEAD~1  # コミットと変更を完全削除

# プッシュ済み（revert推奨）
git revert HEAD
git push origin main

# 複数コミットをrevert
git revert HEAD~3..HEAD
```

### コンフリクト解決

```bash
# Rebase中のコンフリクト
git rebase main

# コンフリクト発生
# ファイルを手動で修正

git add .
git rebase --continue

# 中止する場合
git rebase --abort

# Merge中のコンフリクト
git merge feature-branch

# コンフリクト解決
git add .
git commit
```

### リベース（履歴整理）

```bash
# 最新の main を取り込む
git rebase main

# インタラクティブリベース（直近3コミット）
git rebase -i HEAD~3

# エディタ
pick abc123 feat: add feature A
squash def456 fix: typo
squash ghi789 fix: another typo

# 保存すると3つのコミットが1つに統合される
```

## .gitignore ベストプラクティス

```bash
# OS
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
*.swp
*.swo

# 言語・フレームワーク
node_modules/
__pycache__/
*.pyc
.env
.env.local
dist/
build/
target/

# ログ
*.log
logs/

# 一時ファイル
*.tmp
*.temp
.cache/

# 機密情報
*.key
*.pem
secrets.yml
credentials.json

# ビルド成果物
*.jar
*.war
*.ear

# テストカバレッジ
coverage/
.nyc_output/

# ✅ グローバル .gitignore も設定
git config --global core.excludesfile ~/.gitignore_global
```

## チェックリスト

### コミット前
- [ ] コードフォーマット済み
- [ ] Linter エラーなし
- [ ] テストが通る
- [ ] 不要なコメント・デバッグコード削除
- [ ] 機密情報が含まれていないか確認
- [ ] 適切なコミットメッセージ

### PR作成前
- [ ] 最新の main を rebase
- [ ] コンフリクト解決済み
- [ ] すべてのテストが通る
- [ ] PR説明が充実している
- [ ] 関連Issueをリンク
- [ ] レビュアーを指定

### マージ前
- [ ] 承認を得た
- [ ] CI/CDが通過
- [ ] コンフリクトなし
- [ ] 最新の main を反映済み

## よくある失敗と対策

### ❌ main ブランチで直接開発

```bash
# ❌ 悪い例
git checkout main
# 開発...

# ✅ 良い例
git checkout -b feature/new-feature
# 開発...
```

### ❌ 意味のないコミットメッセージ

```bash
# ❌ 悪い例
git commit -m "fix"
git commit -m "update"
git commit -m "WIP"

# ✅ 良い例
git commit -m "fix: resolve null pointer exception in user service"
```

### ❌ 巨大なPR

```bash
# ❌ 悪い例
# 10ファイル、1000行変更のPR

# ✅ 良い例
# PRを分割
# PR1: データモデル追加（200行）
# PR2: APIエンドポイント追加（300行）
# PR3: フロントエンド実装（500行）
```

### ❌ force push

```bash
# ❌ 危険
git push --force

# ✅ 安全
git push --force-with-lease  # 他の人の変更を上書きしない
```

## Git Hooks

### Pre-commit（コミット前チェック）

```bash
# .git/hooks/pre-commit
#!/bin/sh

# Linter実行
npm run lint
if [ $? -ne 0 ]; then
  echo "Linter failed. Please fix errors."
  exit 1
fi

# テスト実行
npm test
if [ $? -ne 0 ]; then
  echo "Tests failed. Please fix errors."
  exit 1
fi
```

### Commit-msg（コミットメッセージ検証）

```bash
# .git/hooks/commit-msg
#!/bin/sh

commit_msg=$(cat "$1")

# Conventional Commits チェック
if ! echo "$commit_msg" | grep -qE "^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .+"; then
  echo "Invalid commit message format."
  echo "Use: <type>[optional scope]: <description>"
  exit 1
fi
```

### Husky（推奨）

```json
// package.json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged",
      "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"
    }
  },
  "lint-staged": {
    "*.{js,ts}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md}": [
      "prettier --write"
    ]
  }
}
```

## 参考リソース

- [Git Flow](https://nvie.com/posts/a-successful-git-branching-model/)
- [GitHub Flow](https://guides.github.com/introduction/flow/)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [Semantic Versioning](https://semver.org/)
- [Trunk Based Development](https://trunkbaseddevelopment.com/)
