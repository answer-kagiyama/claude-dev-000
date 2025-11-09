# GitHub Actions チートシート

## 基本構文

### ワークフローファイルの配置

```
.github/workflows/ci.yml
```

### 基本構造

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        run: npm test
```

## トリガー（on）

### プッシュ・プルリクエスト

```yaml
on:
  push:
    branches:
      - main
      - develop
    paths:
      - 'src/**'
      - '!src/docs/**'  # 除外
    tags:
      - v*

  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]
```

### スケジュール実行

```yaml
on:
  schedule:
    # 毎日午前9時（UTC）
    - cron: '0 9 * * *'
    # 毎週月曜日午前0時
    - cron: '0 0 * * 1'
```

### 手動トリガー

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
      dry_run:
        description: 'Run in dry-run mode'
        required: false
        type: boolean
```

### その他のイベント

```yaml
on:
  issues:
    types: [opened, labeled]

  release:
    types: [published]

  workflow_call:  # 他のワークフローから呼び出し可能
    inputs:
      config-path:
        required: true
        type: string
```

## ジョブ設定

### 基本設定

```yaml
jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - uses: actions/checkout@v4
      - run: npm test
```

### 実行環境（runs-on）

```yaml
runs-on: ubuntu-latest        # Ubuntu最新
runs-on: ubuntu-22.04         # Ubuntu 22.04
runs-on: macos-latest         # macOS最新
runs-on: windows-latest       # Windows最新
runs-on: [self-hosted, linux] # セルフホスト
```

### マトリックス戦略

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node-version: [16, 18, 20]
        exclude:
          - os: macos-latest
            node-version: 16
      fail-fast: false  # 失敗しても他を続行
      max-parallel: 2   # 並列実行数制限

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm test
```

### ジョブの依存関係

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building..."

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Testing..."

  deploy:
    needs: [build, test]  # 複数のジョブ待ち
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying..."
```

### 条件付き実行

```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to production"

  notify:
    if: failure()  # 前のジョブが失敗時
    runs-on: ubuntu-latest
    steps:
      - run: echo "Sending notification"
```

## ステップ

### 基本ステップ

```yaml
steps:
  # アクション使用
  - uses: actions/checkout@v4
    with:
      fetch-depth: 0

  # コマンド実行
  - name: Run script
    run: |
      echo "Multi-line script"
      npm install
      npm test

  # シェル指定
  - name: Run bash
    run: echo "Hello"
    shell: bash

  # 作業ディレクトリ指定
  - name: Run in subdirectory
    run: npm test
    working-directory: ./packages/app
```

### 環境変数

```yaml
env:
  GLOBAL_VAR: global-value

jobs:
  build:
    env:
      JOB_VAR: job-value

    steps:
      - name: Use environment variables
        env:
          STEP_VAR: step-value
        run: |
          echo "Global: $GLOBAL_VAR"
          echo "Job: $JOB_VAR"
          echo "Step: $STEP_VAR"
          echo "GitHub: ${{ github.repository }}"

      - name: Set dynamic env var
        run: echo "BUILD_TIME=$(date -u +'%Y-%m-%dT%H:%M:%SZ')" >> $GITHUB_ENV

      - name: Use dynamic env var
        run: echo "Built at $BUILD_TIME"
```

### シークレット

```yaml
steps:
  - name: Use secrets
    env:
      API_KEY: ${{ secrets.API_KEY }}
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
    run: |
      # シークレットは自動的にマスクされる
      echo "Deploying with API key"
```

### 成果物（artifacts）

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: npm run build

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
          retention-days: 7

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/

      - name: Deploy
        run: ./deploy.sh
```

### キャッシュ

```yaml
steps:
  - uses: actions/checkout@v4

  # Node.js依存関係のキャッシュ
  - uses: actions/cache@v4
    with:
      path: ~/.npm
      key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-node-

  # Python依存関係のキャッシュ
  - uses: actions/cache@v4
    with:
      path: ~/.cache/pip
      key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
      restore-keys: |
        ${{ runner.os }}-pip-
```

## よく使うアクション

### Checkout

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0        # 全履歴取得
    submodules: recursive # サブモジュール含む
    token: ${{ secrets.GITHUB_TOKEN }}
```

### セットアップ系

```yaml
# Node.js
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'

# Python
- uses: actions/setup-python@v5
  with:
    python-version: '3.11'
    cache: 'pip'

# Java
- uses: actions/setup-java@v4
  with:
    distribution: 'temurin'
    java-version: '17'
    cache: 'maven'

# Go
- uses: actions/setup-go@v5
  with:
    go-version: '1.21'
    cache: true
```

### Docker

```yaml
# Docker Buildx
- uses: docker/setup-buildx-action@v3

# Docker login
- uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

# Build and push
- uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ghcr.io/${{ github.repository }}:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

## コンテキスト変数

```yaml
steps:
  - name: Show contexts
    run: |
      echo "Event: ${{ github.event_name }}"
      echo "Ref: ${{ github.ref }}"
      echo "SHA: ${{ github.sha }}"
      echo "Repository: ${{ github.repository }}"
      echo "Actor: ${{ github.actor }}"
      echo "Run ID: ${{ github.run_id }}"
      echo "Run Number: ${{ github.run_number }}"

      # ブランチ名取得
      echo "Branch: ${{ github.ref_name }}"

      # PR番号
      echo "PR: ${{ github.event.pull_request.number }}"

      # Runner情報
      echo "OS: ${{ runner.os }}"
      echo "Arch: ${{ runner.arch }}"
```

## 条件式

```yaml
steps:
  - name: Run on main branch only
    if: github.ref == 'refs/heads/main'
    run: echo "Main branch"

  - name: Run on tag push
    if: startsWith(github.ref, 'refs/tags/')
    run: echo "Tag pushed"

  - name: Run on PR
    if: github.event_name == 'pull_request'
    run: echo "Pull request"

  - name: Run on success
    if: success()
    run: echo "Previous steps succeeded"

  - name: Run on failure
    if: failure()
    run: echo "Previous steps failed"

  - name: Always run
    if: always()
    run: echo "Cleanup"

  - name: Complex condition
    if: |
      github.event_name == 'push' &&
      github.ref == 'refs/heads/main' &&
      !contains(github.event.head_commit.message, '[skip ci]')
    run: echo "Deploy"
```

## 実践例

### Node.js CI/CD

```yaml
name: Node.js CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [18, 20]

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - run: npm ci

      - run: npm run lint

      - run: npm test

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        if: matrix.node-version == 20
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - run: npm ci

      - run: npm run build

      - name: Deploy to production
        env:
          DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}
        run: npm run deploy
```

### Docker ビルド＆プッシュ

```yaml
name: Docker Build

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### リリース自動化

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest

    permissions:
      contents: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - run: npm ci
      - run: npm run build

      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          files: |
            dist/*.zip
            dist/*.tar.gz
          generate_release_notes: true
          draft: false
          prerelease: ${{ contains(github.ref, 'alpha') || contains(github.ref, 'beta') }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### モノレポ対応

```yaml
name: Monorepo CI

on: [push, pull_request]

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      app1: ${{ steps.filter.outputs.app1 }}
      app2: ${{ steps.filter.outputs.app2 }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            app1:
              - 'packages/app1/**'
            app2:
              - 'packages/app2/**'

  test-app1:
    needs: changes
    if: needs.changes.outputs.app1 == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test
        working-directory: packages/app1

  test-app2:
    needs: changes
    if: needs.changes.outputs.app2 == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test
        working-directory: packages/app2
```

## 再利用可能なワークフロー

### 呼び出される側（.github/workflows/reusable-test.yml）

```yaml
name: Reusable Test Workflow

on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string
    secrets:
      npm-token:
        required: false

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}

      - run: npm ci
        env:
          NPM_TOKEN: ${{ secrets.npm-token }}

      - run: npm test
```

### 呼び出す側

```yaml
name: CI

on: [push]

jobs:
  call-test:
    uses: ./.github/workflows/reusable-test.yml
    with:
      node-version: '20'
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
```

## Tips

### デバッグ

```yaml
# 詳細ログを有効にする（リポジトリのSecretsに設定）
# ACTIONS_STEP_DEBUG = true
# ACTIONS_RUNNER_DEBUG = true

steps:
  - name: Debug
    run: |
      echo "GitHub context:"
      echo '${{ toJSON(github) }}'

      echo "Job context:"
      echo '${{ toJSON(job) }}'

      echo "Runner context:"
      echo '${{ toJSON(runner) }}'
```

### ステップ出力

```yaml
steps:
  - id: step1
    run: echo "result=success" >> $GITHUB_OUTPUT

  - name: Use output
    run: echo "Result was ${{ steps.step1.outputs.result }}"
```

### 並列ジョブの制限

```yaml
# 同じワークフローの並列実行を制限
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true  # 古い実行をキャンセル
```
