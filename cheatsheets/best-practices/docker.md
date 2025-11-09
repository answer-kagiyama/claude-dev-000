# Docker ベストプラクティス

## Dockerfileの最適化

### マルチステージビルド

```dockerfile
# ❌ 悪い例：ビルドツールが本番イメージに含まれる
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
CMD ["node", "dist/index.js"]

# ✅ 良い例：マルチステージビルド
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./
RUN npm ci --production
CMD ["node", "dist/index.js"]
```

### レイヤーキャッシュの活用

```dockerfile
# ❌ 悪い例：変更頻度の高いものが先
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
CMD ["npm", "start"]

# ✅ 良い例：変更頻度の低いものから順に
FROM node:18
WORKDIR /app
COPY package*.json ./  # 依存関係ファイルのみ先にコピー
RUN npm ci            # キャッシュが効きやすい
COPY . .              # ソースコードは最後
CMD ["npm", "start"]
```

### 軽量ベースイメージ

```dockerfile
# ❌ 1GB以上
FROM ubuntu:22.04

# ⚠️ 300-400MB
FROM node:18

# ✅ 100-200MB
FROM node:18-slim

# ✅ 50-100MB（推奨）
FROM node:18-alpine

# ✅ Distroless（最小限）
FROM gcr.io/distroless/nodejs18
```

### 不要なファイルを除外

```.dockerignore
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
.env
.env.local
README.md
*.md
.vscode
.idea
.DS_Store
dist
coverage
.next
out
```

### 単一のRUNで複数コマンド

```dockerfile
# ❌ 悪い例：レイヤーが増える
FROM ubuntu:22.04
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN apt-get clean

# ✅ 良い例：1つのRUNにまとめる
FROM ubuntu:22.04
RUN apt-get update && \
    apt-get install -y \
        curl \
        vim \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

## セキュリティ

### 非rootユーザーで実行

```dockerfile
# ❌ 悪い例：rootで実行
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "index.js"]

# ✅ 良い例：非rootユーザー
FROM node:18-alpine
WORKDIR /app
COPY --chown=node:node . .
RUN npm ci --production
USER node
CMD ["node", "index.js"]

# ✅ カスタムユーザー作成
FROM alpine:3.18
RUN addgroup -g 1000 appgroup && \
    adduser -D -u 1000 -G appgroup appuser
USER appuser
WORKDIR /home/appuser
COPY --chown=appuser:appgroup . .
CMD ["./app"]
```

### シークレットの管理

```dockerfile
# ❌ 悪い例：ARGにシークレット
ARG DATABASE_PASSWORD=secret
ENV DATABASE_PASSWORD=$DATABASE_PASSWORD

# ✅ 良い例：Buildkit Secret使用
# docker build --secret id=db_pass,src=db_password.txt .
RUN --mount=type=secret,id=db_pass \
    export DB_PASS=$(cat /run/secrets/db_pass) && \
    # 処理

# ✅ 実行時に環境変数で渡す
docker run -e DATABASE_PASSWORD=$DB_PASSWORD myapp
```

### イメージスキャン

```bash
# Trivy
trivy image myapp:latest

# Snyk
snyk container test myapp:latest

# Docker Scout
docker scout cves myapp:latest
```

### 最小限のパッケージ

```dockerfile
# ✅ 必要なパッケージのみインストール
FROM alpine:3.18
RUN apk add --no-cache \
    ca-certificates \
    tzdata

# 不要なパッケージは削除
RUN apk add --no-cache --virtual .build-deps \
        gcc \
        musl-dev \
    && # ビルド処理 \
    && apk del .build-deps
```

## イメージサイズ削減

### ビルド成果物のみコピー

```dockerfile
# Go アプリケーション
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.* ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

FROM alpine:3.18
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/main .
CMD ["./main"]
```

### Distroless イメージ

```dockerfile
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM gcr.io/distroless/nodejs18-debian11
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
CMD ["dist/index.js"]
```

### Alpine Linux活用

```dockerfile
FROM python:3.11-alpine
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

## パフォーマンス最適化

### BuildKitの活用

```bash
# BuildKit有効化
export DOCKER_BUILDKIT=1
docker build .

# キャッシュマウント
```

```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci
COPY . .
RUN npm run build
```

### 並列ビルド

```dockerfile
FROM node:18 AS base
WORKDIR /app
COPY package*.json ./

FROM base AS dependencies
RUN npm ci

FROM base AS build
COPY --from=dependencies /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM base AS test
COPY --from=dependencies /app/node_modules ./node_modules
COPY . .
RUN npm test

FROM node:18-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=dependencies /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
```

## ヘルスチェック

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .

HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
    CMD node healthcheck.js

CMD ["node", "index.js"]
```

```javascript
// healthcheck.js
const http = require('http');

const options = {
  host: 'localhost',
  port: 3000,
  path: '/health',
  timeout: 2000
};

const request = http.request(options, (res) => {
  if (res.statusCode === 200) {
    process.exit(0);
  } else {
    process.exit(1);
  }
});

request.on('error', () => process.exit(1));
request.end();
```

## ベストプラクティスDockerfile例

### Node.js

```dockerfile
# Node.js アプリケーション
FROM node:18-alpine AS base

# 依存関係インストール
FROM base AS dependencies
WORKDIR /app
COPY package*.json ./
RUN npm ci --production && \
    npm cache clean --force

# ビルド
FROM base AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# 本番
FROM base AS production
WORKDIR /app

# 非rootユーザー
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

COPY --from=dependencies --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=build --chown=nodejs:nodejs /app/dist ./dist
COPY --chown=nodejs:nodejs package*.json ./

USER nodejs

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD node healthcheck.js

CMD ["node", "dist/index.js"]
```

### Python

```dockerfile
FROM python:3.11-slim AS base

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app

# 依存関係
FROM base AS dependencies
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# 本番
FROM base AS production

# 非rootユーザー
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

COPY --from=dependencies /root/.local /home/appuser/.local
COPY --chown=appuser:appuser . .

USER appuser

ENV PATH=/home/appuser/.local/bin:$PATH

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD python healthcheck.py

CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0"]
```

### Go

```dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app

# 依存関係
COPY go.mod go.sum ./
RUN go mod download

# ビルド
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o main .

# 本番
FROM alpine:3.18

RUN apk --no-cache add ca-certificates tzdata && \
    addgroup -g 1000 appgroup && \
    adduser -D -u 1000 -G appgroup appuser

WORKDIR /home/appuser

COPY --from=builder --chown=appuser:appgroup /app/main .

USER appuser

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD ./main -healthcheck

CMD ["./main"]
```

## Docker Compose

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        NODE_ENV: production
    image: myapp:latest
    container_name: myapp
    restart: unless-stopped
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://db/mydb
    env_file:
      - .env
    ports:
      - "3000:3000"
    volumes:
      - ./logs:/app/logs
    networks:
      - app-network
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "node", "healthcheck.js"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 40s
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M

  db:
    image: postgres:15-alpine
    container_name: myapp-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
    driver: local

networks:
  app-network:
    driver: bridge
```

## チェックリスト

### Dockerfile
- [ ] 軽量ベースイメージ（Alpine、Slim、Distroless）
- [ ] マルチステージビルド
- [ ] .dockerignoreファイル
- [ ] 非rootユーザーで実行
- [ ] レイヤーキャッシュ最適化
- [ ] 不要なパッケージ削除
- [ ] ヘルスチェック定義
- [ ] シークレットを含めない

### セキュリティ
- [ ] 脆弱性スキャン実施
- [ ] 最新バージョンのベースイメージ
- [ ] 署名済みイメージ使用
- [ ] 読み取り専用ファイルシステム（可能な場合）
- [ ] Capabilities削減

### パフォーマンス
- [ ] イメージサイズ最小化
- [ ] BuildKitキャッシュ活用
- [ ] 並列ビルド
- [ ] 不要なファイル除外

### 運用
- [ ] バージョンタグ付け
- [ ] ヘルスチェック
- [ ] ログ出力（stdout/stderr）
- [ ] グレースフルシャットダウン
- [ ] リソース制限

## 参考リソース

- [Dockerfile Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [OWASP Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
