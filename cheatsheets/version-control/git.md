# Git チートシート

## 基本設定

```bash
# ユーザー情報設定
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# エディタ設定
git config --global core.editor "vim"

# デフォルトブランチ名
git config --global init.defaultBranch main

# 設定確認
git config --list
git config user.name
```

## リポジトリ操作

```bash
# 初期化
git init

# クローン
git clone <url>
git clone <url> <directory>
git clone --depth 1 <url>  # shallow clone（履歴なし）

# リモート確認・追加
git remote -v
git remote add origin <url>
git remote set-url origin <new-url>
```

## 基本操作

```bash
# 状態確認
git status
git status -s  # 短縮表示

# 変更確認
git diff                    # 作業ディレクトリ vs ステージング
git diff --staged          # ステージング vs 最新コミット
git diff HEAD              # 作業ディレクトリ vs 最新コミット
git diff <branch1> <branch2>

# ステージング
git add <file>
git add .                  # すべて追加
git add -p                 # 対話的に追加

# コミット
git commit -m "message"
git commit -am "message"   # add + commit（追跡済みファイルのみ）
git commit --amend         # 直前のコミットを修正
git commit --amend --no-edit  # メッセージそのまま

# 取り消し
git restore <file>         # 作業ディレクトリの変更を取り消し
git restore --staged <file>  # ステージングを取り消し
git reset HEAD <file>      # ステージングを取り消し（旧構文）
git reset --soft HEAD~1    # コミットのみ取り消し
git reset --hard HEAD~1    # コミット＋変更を完全に取り消し（危険）
```

## ブランチ操作

```bash
# ブランチ一覧
git branch                 # ローカル
git branch -r              # リモート
git branch -a              # すべて

# ブランチ作成・切替
git branch <branch>
git checkout <branch>
git checkout -b <branch>   # 作成＋切替
git switch <branch>        # 新しい切替コマンド
git switch -c <branch>     # 作成＋切替

# ブランチ削除
git branch -d <branch>     # マージ済みのみ
git branch -D <branch>     # 強制削除
git push origin --delete <branch>  # リモートブランチ削除

# ブランチ名変更
git branch -m <old> <new>
git branch -m <new>        # 現在のブランチ名変更
```

## マージ・リベース

```bash
# マージ
git merge <branch>
git merge --no-ff <branch>  # Fast-forwardしない
git merge --squash <branch>  # 1つのコミットにまとめる

# コンフリクト解決
git merge --abort          # マージ中止
git mergetool              # マージツール起動

# リベース
git rebase <branch>
git rebase -i HEAD~3       # 対話的リベース（直近3コミット）
git rebase --continue      # コンフリクト解決後続行
git rebase --abort         # リベース中止

# Cherry-pick
git cherry-pick <commit>
git cherry-pick <commit1> <commit2>
```

## リモート操作

```bash
# フェッチ
git fetch
git fetch origin
git fetch --all
git fetch --prune          # 削除されたリモートブランチを反映

# プル
git pull
git pull origin main
git pull --rebase          # rebaseしながらpull

# プッシュ
git push
git push origin <branch>
git push -u origin <branch>  # 上流ブランチ設定
git push --force-with-lease  # 安全な強制プッシュ
git push --force           # 強制プッシュ（危険）
git push --tags            # タグをプッシュ
```

## ログ・履歴

```bash
# ログ表示
git log
git log --oneline
git log --graph --oneline --all
git log -p                 # 差分も表示
git log --since="2 weeks ago"
git log --author="Name"
git log --grep="keyword"
git log <file>             # ファイルの履歴

# コミット情報
git show <commit>
git show HEAD~2

# blame（誰が変更したか）
git blame <file>
git blame -L 10,20 <file>  # 行範囲指定
```

## スタッシュ

```bash
# 一時保存
git stash
git stash save "message"
git stash -u               # 未追跡ファイルも含める

# 一覧・確認
git stash list
git stash show
git stash show -p stash@{0}

# 復元
git stash pop              # 最新を復元＋削除
git stash apply            # 最新を復元（保持）
git stash apply stash@{1}

# 削除
git stash drop stash@{0}
git stash clear            # すべて削除
```

## タグ

```bash
# タグ作成
git tag v1.0.0
git tag -a v1.0.0 -m "Release 1.0.0"  # アノテーション付き

# タグ一覧
git tag
git tag -l "v1.*"

# タグ削除
git tag -d v1.0.0
git push origin :refs/tags/v1.0.0  # リモートから削除

# タグをプッシュ
git push origin v1.0.0
git push origin --tags
```

## 便利なコマンド

```bash
# ファイル削除
git rm <file>
git rm --cached <file>     # Gitから削除（ファイルは残す）

# ファイル移動・名前変更
git mv <old> <new>

# 検索
git grep "search term"
git grep -n "search term"  # 行番号付き

# クリーンアップ
git clean -n               # 削除対象を表示
git clean -f               # 未追跡ファイル削除
git clean -fd              # 未追跡ファイル＋ディレクトリ削除

# 差分ツール
git difftool
git mergetool

# リファレンス
git reflog                 # すべての操作履歴
git reflog show HEAD

# サブモジュール
git submodule add <url> <path>
git submodule update --init --recursive
git submodule foreach git pull origin main
```

## ワークフロー例

### Feature Branch Workflow

```bash
# 1. 最新のmainを取得
git checkout main
git pull origin main

# 2. フィーチャーブランチ作成
git checkout -b feature/new-feature

# 3. 開発・コミット
git add .
git commit -m "Add new feature"

# 4. プッシュ
git push -u origin feature/new-feature

# 5. プルリクエスト作成（GitHub/GitLab等）

# 6. マージ後、ローカルをクリーンアップ
git checkout main
git pull origin main
git branch -d feature/new-feature
```

### Hotfix Workflow

```bash
# 1. mainから緊急修正ブランチ作成
git checkout main
git checkout -b hotfix/critical-bug

# 2. 修正・コミット
git add .
git commit -m "Fix critical bug"

# 3. mainにマージ
git checkout main
git merge hotfix/critical-bug
git push origin main

# 4. タグ付け
git tag -a v1.0.1 -m "Hotfix release"
git push origin v1.0.1

# 5. クリーンアップ
git branch -d hotfix/critical-bug
```

## .gitignore パターン例

```gitignore
# 依存関係
node_modules/
vendor/
venv/
__pycache__/

# ビルド成果物
dist/
build/
*.exe
*.dll

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# 環境変数
.env
.env.local

# ログ
*.log
logs/

# 一時ファイル
tmp/
temp/
*.tmp
```

## Tips

```bash
# エイリアス設定
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.unstage 'reset HEAD --'
git config --global alias.last 'log -1 HEAD'

# カラフルな出力
git config --global color.ui auto

# 改行コード設定
git config --global core.autocrlf input  # Mac/Linux
git config --global core.autocrlf true   # Windows

# マージ時のコンフリクトマーカー形式
git config --global merge.conflictstyle diff3
```

## よくあるトラブル対処

```bash
# コミットメッセージ間違えた
git commit --amend -m "Correct message"

# 間違えてコミットした
git reset --soft HEAD~1  # コミット取り消し、変更は保持

# 間違えてプッシュした（プッシュ直後のみ）
git reset --hard HEAD~1
git push --force-with-lease

# コンフリクト解消
# 1. エディタでファイルを編集
# 2. git add <file>
# 3. git commit（マージの場合）/ git rebase --continue（リベースの場合）

# 誤って削除したコミットを復元
git reflog
git cherry-pick <commit-hash>

# 大きなファイルを履歴から削除
git filter-branch --tree-filter 'rm -f large-file.bin' HEAD
# または BFG Repo-Cleaner を使用
```
