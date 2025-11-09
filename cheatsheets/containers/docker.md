# Docker チートシート

## イメージ操作

```bash
# イメージ検索
docker search nginx

# イメージ取得
docker pull nginx
docker pull nginx:1.25
docker pull nginx:latest

# イメージ一覧
docker images
docker image ls
docker images -a  # 中間イメージも表示

# イメージ削除
docker rmi nginx
docker rmi nginx:1.25
docker image rm nginx
docker rmi $(docker images -q)  # すべて削除
docker image prune  # 未使用イメージ削除
docker image prune -a  # すべての未使用イメージ削除

# イメージ情報
docker inspect nginx
docker history nginx

# イメージのタグ付け
docker tag nginx:latest myregistry.com/nginx:v1
docker tag image_id myname/myimage:tag

# イメージの保存・読み込み
docker save nginx > nginx.tar
docker save -o nginx.tar nginx
docker load < nginx.tar
docker load -i nginx.tar
```

## コンテナ操作

### 基本操作

```bash
# コンテナ起動
docker run nginx
docker run -d nginx  # バックグラウンド
docker run -it ubuntu bash  # 対話モード
docker run --name mynginx nginx  # 名前指定
docker run --rm nginx  # 終了時に自動削除

# ポートマッピング
docker run -p 8080:80 nginx  # ホスト8080 -> コンテナ80
docker run -p 127.0.0.1:8080:80 nginx  # IPアドレス指定

# ボリュームマウント
docker run -v /host/path:/container/path nginx  # バインドマウント
docker run -v myvolume:/data nginx  # ボリューム
docker run -v /data nginx  # 匿名ボリューム
docker run --mount type=bind,source=/host,target=/container nginx

# 環境変数
docker run -e "ENV_VAR=value" nginx
docker run -e ENV_VAR nginx  # ホストから継承
docker run --env-file .env nginx

# ネットワーク
docker run --network mynetwork nginx
docker run --network host nginx  # ホストネットワーク使用

# リソース制限
docker run -m 512m nginx  # メモリ制限
docker run --cpus 2 nginx  # CPU制限
docker run --memory 1g --cpus 1.5 nginx

# 再起動ポリシー
docker run --restart always nginx
docker run --restart unless-stopped nginx
docker run --restart on-failure:3 nginx
```

### コンテナ管理

```bash
# コンテナ一覧
docker ps  # 実行中
docker ps -a  # すべて
docker ps -q  # IDのみ
docker ps --filter "status=exited"
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}"

# コンテナ起動・停止
docker start container_name
docker stop container_name
docker restart container_name
docker pause container_name  # 一時停止
docker unpause container_name

# コンテナ削除
docker rm container_name
docker rm -f container_name  # 強制削除
docker rm $(docker ps -aq)  # すべて削除
docker container prune  # 停止中のコンテナ削除

# コンテナ情報
docker inspect container_name
docker logs container_name
docker logs -f container_name  # リアルタイム表示
docker logs --tail 100 container_name
docker logs --since 10m container_name
docker stats  # リソース使用状況
docker stats container_name
docker top container_name  # プロセス一覧
docker port container_name  # ポートマッピング確認
```

### コンテナ操作

```bash
# コンテナに接続
docker exec -it container_name bash
docker exec -it container_name sh
docker exec container_name ls /app

# ファイルコピー
docker cp host_file container_name:/path/
docker cp container_name:/path/file ./

# コンテナからイメージ作成
docker commit container_name myimage:tag
docker commit -m "message" container_name myimage:tag

# コンテナの変更確認
docker diff container_name

# コンテナのエクスポート・インポート
docker export container_name > container.tar
docker import container.tar myimage:tag

# コンテナの一時停止・再開
docker pause container_name
docker unpause container_name

# コンテナ名変更
docker rename old_name new_name
```

## Dockerfile

### 基本構文

```dockerfile
# ベースイメージ
FROM ubuntu:22.04
FROM node:18-alpine AS builder

# メタデータ
LABEL maintainer="your@email.com"
LABEL version="1.0"
LABEL description="My application"

# 環境変数
ENV NODE_ENV=production
ENV APP_HOME=/app

# 作業ディレクトリ
WORKDIR /app

# ファイルコピー
COPY package*.json ./
COPY . .
ADD https://example.com/file.tar.gz /tmp/

# コマンド実行
RUN apt-get update && apt-get install -y curl
RUN npm install
RUN npm run build

# ポート公開
EXPOSE 8080
EXPOSE 8080/tcp
EXPOSE 8080/udp

# ボリューム
VOLUME ["/data"]

# ユーザー切り替え
USER node
USER 1000:1000

# ヘルスチェック
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost/ || exit 1

# 起動コマンド
CMD ["npm", "start"]
CMD npm start

# エントリーポイント
ENTRYPOINT ["docker-entrypoint.sh"]
ENTRYPOINT docker-entrypoint.sh

# 引数
ARG VERSION=latest
ARG BUILD_DATE
```

### マルチステージビルド

```dockerfile
# ビルドステージ
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# 実行ステージ
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./
RUN npm ci --production
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### ベストプラクティス

```dockerfile
FROM node:18-alpine

# レイヤーキャッシュを活用
COPY package*.json ./
RUN npm ci

# 変更頻度が高いファイルは後で
COPY . .

# 1つのRUNで複数コマンド（レイヤー削減）
RUN apt-get update && \
    apt-get install -y curl vim && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# .dockerignoreを使用
# node_modules
# .git
# *.log

# 非rootユーザーで実行
RUN addgroup -g 1001 appgroup && \
    adduser -D -u 1001 -G appgroup appuser
USER appuser

# ヘルスチェック
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s \
  CMD node healthcheck.js

CMD ["node", "index.js"]
```

## イメージビルド

```bash
# 基本ビルド
docker build -t myapp:latest .
docker build -t myapp:v1.0 -f Dockerfile.prod .

# ビルド引数
docker build --build-arg VERSION=1.0 -t myapp .

# キャッシュ無効化
docker build --no-cache -t myapp .

# 特定のステージまでビルド
docker build --target builder -t myapp:builder .

# ビルドコンテキスト指定
docker build -t myapp https://github.com/user/repo.git
docker build -t myapp - < Dockerfile

# BuildKit使用（高速化）
DOCKER_BUILDKIT=1 docker build -t myapp .

# プラットフォーム指定
docker build --platform linux/amd64 -t myapp .
docker build --platform linux/arm64 -t myapp .

# プログレス表示
docker build --progress=plain -t myapp .
```

## Docker Compose

### docker-compose.yml

```yaml
version: '3.8'

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        - NODE_ENV=production
    image: myapp:latest
    container_name: myapp-web
    ports:
      - "8080:80"
    volumes:
      - ./app:/app
      - node_modules:/app/node_modules
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgres://db/mydb
    env_file:
      - .env
    depends_on:
      - db
      - redis
    networks:
      - mynetwork
    restart: unless-stopped
    command: npm start
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 3s
      retries: 3
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M

  db:
    image: postgres:15
    container_name: myapp-db
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    networks:
      - mynetwork

  redis:
    image: redis:7-alpine
    container_name: myapp-redis
    ports:
      - "6379:6379"
    networks:
      - mynetwork

volumes:
  postgres_data:
  node_modules:

networks:
  mynetwork:
    driver: bridge
```

### Composeコマンド

```bash
# 起動
docker-compose up
docker-compose up -d  # バックグラウンド
docker-compose up --build  # ビルドしてから起動
docker-compose up --force-recreate  # 強制再作成

# 停止
docker-compose down
docker-compose down -v  # ボリュームも削除
docker-compose down --rmi all  # イメージも削除

# サービス管理
docker-compose start
docker-compose stop
docker-compose restart
docker-compose pause
docker-compose unpause

# ログ
docker-compose logs
docker-compose logs -f  # リアルタイム
docker-compose logs web  # 特定サービス

# 実行中のサービス確認
docker-compose ps
docker-compose ps -a

# コマンド実行
docker-compose exec web bash
docker-compose exec db psql -U user mydb
docker-compose run --rm web npm test

# ビルド
docker-compose build
docker-compose build --no-cache

# 設定確認
docker-compose config
docker-compose config --services

# スケール
docker-compose up -d --scale web=3

# トップ
docker-compose top
```

## ボリューム

```bash
# ボリューム作成
docker volume create myvolume

# ボリューム一覧
docker volume ls

# ボリューム詳細
docker volume inspect myvolume

# ボリューム削除
docker volume rm myvolume
docker volume prune  # 未使用ボリューム削除

# ボリューム使用
docker run -v myvolume:/data nginx
docker run --mount source=myvolume,target=/data nginx

# 読み取り専用マウント
docker run -v myvolume:/data:ro nginx

# バインドマウント
docker run -v /host/path:/container/path nginx
docker run -v $(pwd):/app nginx

# 匿名ボリューム
docker run -v /data nginx
```

## ネットワーク

```bash
# ネットワーク作成
docker network create mynetwork
docker network create --driver bridge mynetwork
docker network create --subnet 172.20.0.0/16 mynetwork

# ネットワーク一覧
docker network ls

# ネットワーク詳細
docker network inspect mynetwork

# ネットワーク削除
docker network rm mynetwork
docker network prune  # 未使用ネットワーク削除

# コンテナをネットワークに接続
docker network connect mynetwork container_name
docker network disconnect mynetwork container_name

# ネットワーク指定で起動
docker run --network mynetwork nginx

# ホストネットワーク
docker run --network host nginx

# コンテナ名で通信
# 同じネットワーク内ならコンテナ名で名前解決可能
docker run --network mynetwork --name web nginx
docker run --network mynetwork alpine ping web
```

## レジストリ

```bash
# Docker Hubへログイン
docker login
docker login -u username -p password

# イメージのプッシュ
docker tag myapp username/myapp:latest
docker push username/myapp:latest

# プライベートレジストリ
docker tag myapp registry.example.com/myapp:latest
docker push registry.example.com/myapp:latest
docker pull registry.example.com/myapp:latest

# ローカルレジストリ起動
docker run -d -p 5000:5000 --name registry registry:2
docker tag myapp localhost:5000/myapp
docker push localhost:5000/myapp
```

## システム管理

```bash
# ディスク使用量
docker system df
docker system df -v

# クリーンアップ
docker system prune  # 未使用のすべて削除
docker system prune -a  # すべての未使用リソース削除
docker system prune --volumes  # ボリュームも削除

# イベント監視
docker events
docker events --filter 'type=container'

# バージョン情報
docker version
docker info

# Docker Engineの設定
# /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2"
}
```

## デバッグ・トラブルシューティング

```bash
# ログ確認
docker logs container_name
docker logs -f --tail 100 container_name

# コンテナに入る
docker exec -it container_name bash
docker exec -it container_name sh

# プロセス確認
docker top container_name

# リソース使用状況
docker stats
docker stats container_name

# ポート確認
docker port container_name

# ネットワーク接続確認
docker network inspect mynetwork

# コンテナの詳細情報
docker inspect container_name
docker inspect --format '{{.State.Status}}' container_name
docker inspect --format '{{.NetworkSettings.IPAddress}}' container_name

# 実行中のプロセス
docker exec container_name ps aux

# ファイルシステムの変更
docker diff container_name

# ヘルスチェック状態
docker inspect --format '{{.State.Health.Status}}' container_name
```

## セキュリティ

```bash
# 非rootユーザーで実行
docker run --user 1000:1000 nginx

# 読み取り専用ルートファイルシステム
docker run --read-only nginx

# Capability制限
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx

# SELinux/AppArmor
docker run --security-opt label=level:s0:c100,c200 nginx

# Secrets管理（Swarm）
docker secret create my_secret secret.txt
docker service create --secret my_secret nginx

# イメージスキャン
docker scan myapp:latest

# Content Trust（署名検証）
export DOCKER_CONTENT_TRUST=1
docker pull nginx
```

## よく使うパターン

### 開発環境

```bash
# ホットリロード開発
docker run -v $(pwd):/app -p 3000:3000 node:18 npm run dev

# データベース起動
docker run -d -p 5432:5432 -e POSTGRES_PASSWORD=password postgres:15

# 一時的なテスト
docker run --rm -it python:3.11 python
docker run --rm -v $(pwd):/app -w /app node:18 npm test
```

### docker-compose.override.yml（開発用）

```yaml
version: '3.8'

services:
  web:
    volumes:
      - ./src:/app/src
    environment:
      - DEBUG=true
    command: npm run dev
    ports:
      - "3000:3000"
      - "9229:9229"  # デバッグポート
```

### .dockerignore

```
node_modules
npm-debug.log
.git
.gitignore
.env
.DS_Store
*.md
Dockerfile
docker-compose.yml
.vscode
.idea
dist
coverage
.pytest_cache
__pycache__
*.pyc
.next
out
```

## Tips

```bash
# コンテナIDのみ取得
docker ps -q

# すべてのコンテナ停止
docker stop $(docker ps -q)

# すべてのコンテナ削除
docker rm $(docker ps -aq)

# すべてのイメージ削除
docker rmi $(docker images -q)

# 特定のイメージを使用しているコンテナ
docker ps -a --filter ancestor=nginx

# エイリアス設定（~/.bashrc）
alias dps='docker ps'
alias dpa='docker ps -a'
alias di='docker images'
alias drm='docker rm'
alias drmi='docker rmi'
alias dexec='docker exec -it'
alias dlogs='docker logs -f'

# コンテナ内のIPアドレス取得
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name

# 特定のポートでリッスンしているコンテナ
docker ps --filter "expose=80"
```
