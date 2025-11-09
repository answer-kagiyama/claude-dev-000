# CI/CD ベストプラクティス

## パイプライン設計

### 基本構造

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  # ステージ1: ビルド・テスト
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Test
        run: npm test -- --coverage

      - name: Build
        run: npm run build

      - name: Upload artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build
          path: dist/

  # ステージ2: セキュリティスキャン
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Dependency check
        run: npm audit

      - name: SAST
        uses: github/codeql-action/analyze@v2

  # ステージ3: デプロイ
  deploy:
    needs: [build-and-test, security]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v3
        with:
          name: build

      - name: Deploy to production
        run: ./deploy.sh
```

### 並列実行

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [16, 18, 20]
        os: [ubuntu-latest, windows-latest, macos-latest]
    steps:
      - uses: actions/checkout@v3
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci
      - run: npm test
```

## ビルド最適化

### キャッシュ活用

```yaml
# npm キャッシュ
- name: Setup Node.js
  uses: actions/setup-node@v3
  with:
    node-version: '18'
    cache: 'npm'  # 自動的に package-lock.json をキャッシュキーに使用

# カスタムキャッシュ
- name: Cache dependencies
  uses: actions/cache@v3
  with:
    path: |
      ~/.npm
      node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

# Docker レイヤーキャッシュ
- name: Build Docker image
  uses: docker/build-push-action@v4
  with:
    context: .
    push: false
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### Docker ビルド

```yaml
# マルチステージビルド + キャッシュ
name: Build and Push Docker Image

on:
  push:
    branches: [main]

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to DockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: myapp/image:${{ github.sha }},myapp/image:latest
          cache-from: type=registry,ref=myapp/image:buildcache
          cache-to: type=registry,ref=myapp/image:buildcache,mode=max
```

### 依存関係の最適化

```yaml
# ✅ npm ci（lockfileベース、CI向け）
- run: npm ci

# ❌ npm install（lockfile更新、開発向け）
- run: npm install

# ✅ production依存関係のみ
- run: npm ci --production

# ✅ frozen lockfile
- run: npm ci --frozen-lockfile
```

## テストステージ

### 段階的テスト

```yaml
jobs:
  unit-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm ci
      - run: npm run test:unit

  integration-test:
    needs: unit-test
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v3
      - run: npm ci
      - run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test

  e2e-test:
    needs: integration-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npm run test:e2e

      - name: Upload Playwright report
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: playwright-report/
```

### カバレッジレポート

```yaml
- name: Run tests with coverage
  run: npm test -- --coverage

- name: Upload coverage to Codecov
  uses: codecov/codecov-action@v3
  with:
    files: ./coverage/lcov.info
    fail_ci_if_error: true

- name: Comment coverage on PR
  uses: romeovs/lcov-reporter-action@v0.3.1
  with:
    lcov-file: ./coverage/lcov.info
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

## セキュリティ

### 依存関係スキャン

```yaml
jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      # npm audit
      - name: Run npm audit
        run: npm audit --audit-level=high

      # Snyk
      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

      # Dependabot（自動PR作成）
      # .github/dependabot.yml で設定
```

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    reviewers:
      - "security-team"
    labels:
      - "dependencies"
      - "security"
```

### コンテナスキャン

```yaml
- name: Build image
  run: docker build -t myapp:${{ github.sha }} .

- name: Scan image with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'

- name: Upload Trivy results
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: 'trivy-results.sarif'
```

### SAST（静的解析）

```yaml
- name: Initialize CodeQL
  uses: github/codeql-action/init@v2
  with:
    languages: javascript

- name: Build
  run: npm run build

- name: Perform CodeQL Analysis
  uses: github/codeql-action/analyze@v2
```

### Secret スキャン

```yaml
- name: Scan for secrets
  uses: trufflesecurity/trufflehog@main
  with:
    path: ./
    base: ${{ github.event.repository.default_branch }}
    head: HEAD
```

## デプロイ戦略

### Blue-Green デプロイ

```yaml
jobs:
  deploy-blue-green:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to green environment
        run: |
          kubectl apply -f k8s/deployment-green.yml
          kubectl rollout status deployment/myapp-green

      - name: Run smoke tests
        run: ./smoke-test.sh https://green.myapp.com

      - name: Switch traffic to green
        run: |
          kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'

      - name: Monitor for 5 minutes
        run: sleep 300

      - name: Rollback if needed
        if: failure()
        run: |
          kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'
```

### Canary デプロイ

```yaml
- name: Deploy canary (10%)
  run: |
    kubectl apply -f k8s/deployment-canary.yml
    kubectl scale deployment myapp-canary --replicas=1
    kubectl scale deployment myapp-stable --replicas=9

- name: Monitor metrics
  run: ./monitor-canary.sh

- name: Gradually increase traffic
  run: |
    kubectl scale deployment myapp-canary --replicas=5
    kubectl scale deployment myapp-stable --replicas=5
    sleep 300

    kubectl scale deployment myapp-canary --replicas=10
    kubectl scale deployment myapp-stable --replicas=0
```

### ローリングアップデート

```yaml
- name: Update deployment
  run: |
    kubectl set image deployment/myapp \
      app=myapp:${{ github.sha }}

    kubectl rollout status deployment/myapp

    # Rollback on failure
    kubectl rollout undo deployment/myapp
```

## 環境管理

### 環境分離

```yaml
jobs:
  deploy-dev:
    if: github.ref == 'refs/heads/develop'
    environment:
      name: development
      url: https://dev.myapp.com
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to dev
        run: ./deploy.sh dev
        env:
          API_URL: ${{ secrets.DEV_API_URL }}
          DATABASE_URL: ${{ secrets.DEV_DATABASE_URL }}

  deploy-staging:
    if: github.ref == 'refs/heads/main'
    needs: deploy-dev
    environment:
      name: staging
      url: https://staging.myapp.com
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to staging
        run: ./deploy.sh staging
        env:
          API_URL: ${{ secrets.STAGING_API_URL }}

  deploy-production:
    needs: deploy-staging
    environment:
      name: production
      url: https://myapp.com
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: ./deploy.sh prod
        env:
          API_URL: ${{ secrets.PROD_API_URL }}
```

### 承認プロセス

```yaml
jobs:
  deploy-prod:
    environment:
      name: production
      url: https://myapp.com
    runs-on: ubuntu-latest
    steps:
      # GitHub Environment Protection Rules で承認者を設定
      # Settings > Environments > production > Required reviewers

      - name: Deploy
        run: ./deploy.sh
```

## アーティファクト管理

### ビルド成果物

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build

      - name: Upload artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build-${{ github.sha }}
          path: |
            dist/
            package.json
          retention-days: 30

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-${{ github.sha }}

      - name: Deploy
        run: ./deploy.sh
```

### Docker イメージタグ戦略

```yaml
- name: Docker meta
  id: meta
  uses: docker/metadata-action@v4
  with:
    images: myapp/image
    tags: |
      type=ref,event=branch
      type=ref,event=pr
      type=semver,pattern={{version}}
      type=semver,pattern={{major}}.{{minor}}
      type=sha,prefix={{branch}}-

# 出力例
# - myapp/image:main
# - myapp/image:pr-123
# - myapp/image:1.2.3
# - myapp/image:1.2
# - myapp/image:main-abc1234

- name: Build and push
  uses: docker/build-push-action@v4
  with:
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}
```

## モニタリング・通知

### デプロイ通知

```yaml
- name: Notify Slack on success
  if: success()
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": "✅ Deploy successful",
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "*Deploy to Production*\n✅ Success\nCommit: ${{ github.sha }}\nBy: ${{ github.actor }}"
            }
          }
        ]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

- name: Notify on failure
  if: failure()
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": "❌ Deploy failed",
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "*Deploy to Production*\n❌ Failed\nCommit: ${{ github.sha }}\nWorkflow: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }
          }
        ]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### デプロイメトリクス

```yaml
- name: Record deployment
  run: |
    curl -X POST https://api.datadoghq.com/api/v1/events \
      -H "Content-Type: application/json" \
      -H "DD-API-KEY: ${{ secrets.DD_API_KEY }}" \
      -d '{
        "title": "Deployment to Production",
        "text": "Version ${{ github.sha }} deployed",
        "tags": ["env:production", "service:myapp"]
      }'
```

## 再利用可能なワークフロー

### Composite Action

```yaml
# .github/actions/setup-node/action.yml
name: 'Setup Node.js with cache'
description: 'Setup Node.js and restore cache'
inputs:
  node-version:
    description: 'Node.js version'
    required: true
    default: '18'
runs:
  using: 'composite'
  steps:
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'

    - name: Install dependencies
      shell: bash
      run: npm ci

    - name: Cache node_modules
      uses: actions/cache@v3
      with:
        path: node_modules
        key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}

# 使用例
- uses: ./.github/actions/setup-node
  with:
    node-version: '18'
```

### Reusable Workflow

```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
    secrets:
      deploy-key:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v3
      - name: Deploy
        run: ./deploy.sh ${{ inputs.environment }}
        env:
          DEPLOY_KEY: ${{ secrets.deploy-key }}

# 使用例（別ファイル）
jobs:
  deploy-prod:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
    secrets:
      deploy-key: ${{ secrets.PROD_DEPLOY_KEY }}
```

## ベストプラクティス集

### ✅ やるべきこと

```yaml
# 1. fail-fast を適切に使う
strategy:
  fail-fast: true  # 1つ失敗したら全体停止
  matrix:
    node-version: [16, 18, 20]

# 2. timeout を設定
jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 10  # 無限ループ防止

# 3. 条件付き実行
- name: Deploy
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  run: ./deploy.sh

# 4. 明示的なシェル指定
- name: Run script
  shell: bash
  run: |
    set -euo pipefail
    ./script.sh

# 5. secrets の安全な使用
- name: Deploy
  run: ./deploy.sh
  env:
    API_KEY: ${{ secrets.API_KEY }}  # ✅ 環境変数として渡す

# ❌ ログに出力される
- run: echo ${{ secrets.API_KEY }}
```

### ❌ 避けるべきこと

```yaml
# ❌ Hardcoded secrets
- name: Deploy
  run: ./deploy.sh
  env:
    API_KEY: "abc123"  # 絶対にやらない

# ❌ sudo の使用
- run: sudo apt-get install something  # 可能な限り避ける

# ❌ 非決定的なバージョン
- uses: actions/checkout@master  # ❌ タグを使う
- uses: actions/checkout@v3      # ✅

# ❌ キャッシュなし
- run: npm install  # 毎回フルインストール

# ✅ キャッシュあり
- uses: actions/setup-node@v3
  with:
    cache: 'npm'
```

## パフォーマンス最適化

### 並列実行の最大化

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: npm run lint

  test-unit:
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:unit

  test-integration:
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:integration

  # lint, test-unit, test-integration は並列実行

  deploy:
    needs: [lint, test-unit, test-integration]
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

### Self-hosted Runner

```yaml
jobs:
  build:
    runs-on: self-hosted  # 独自のランナー使用
    steps:
      - uses: actions/checkout@v3
      - run: npm ci
      - run: npm run build

# メリット
# - より高速なネットワーク
# - カスタムな環境
# - コスト削減（大規模な場合）
```

## チェックリスト

### パイプライン設計
- [ ] ステージが明確に分離されているか
- [ ] 失敗時の通知設定があるか
- [ ] タイムアウト設定があるか
- [ ] 並列実行を最大化しているか

### セキュリティ
- [ ] Secrets を環境変数で渡しているか
- [ ] 依存関係スキャンを実行しているか
- [ ] コンテナイメージスキャンを実行しているか
- [ ] SAST を実行しているか

### テスト
- [ ] ユニットテストを実行しているか
- [ ] 統合テストを実行しているか
- [ ] カバレッジレポートを生成しているか
- [ ] テスト失敗時はデプロイを中止しているか

### デプロイ
- [ ] 環境ごとに分離されているか
- [ ] 本番デプロイに承認プロセスがあるか
- [ ] ロールバック手順があるか
- [ ] デプロイ後のモニタリングがあるか

### パフォーマンス
- [ ] キャッシュを活用しているか
- [ ] 並列実行を最大化しているか
- [ ] 不要な処理を削減しているか

## 参考リソース

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [The Twelve-Factor App](https://12factor.net/)
- [DORA Metrics](https://cloud.google.com/blog/products/devops-sre/using-the-four-keys-to-measure-your-devops-performance)
