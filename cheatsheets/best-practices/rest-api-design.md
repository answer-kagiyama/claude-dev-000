# REST API 設計ベストプラクティス

## 基本原則

### REST の制約
- **クライアント・サーバー分離**: UIとデータストレージを分離
- **ステートレス**: 各リクエストは独立し、必要な情報をすべて含む
- **キャッシュ可能**: レスポンスは明示的にキャッシュ可否を示す
- **統一インターフェース**: 一貫したリソース識別、操作方法
- **階層化システム**: 中間層（プロキシ、ゲートウェイ）を透過的に使用可能

## エンドポイント設計

### リソース指向
```
✅ 良い例
GET    /users              # ユーザー一覧取得
GET    /users/123          # 特定ユーザー取得
POST   /users              # ユーザー作成
PUT    /users/123          # ユーザー更新（全体）
PATCH  /users/123          # ユーザー更新（部分）
DELETE /users/123          # ユーザー削除

❌ 悪い例
GET    /getUserList
GET    /getUser?id=123
POST   /createUser
POST   /updateUser
POST   /deleteUser
```

### 名詞を使用（動詞は避ける）
```
✅ 良い例
POST /orders
GET  /orders/123/items

❌ 悪い例
POST /createOrder
GET  /getOrderItems
```

### 複数形を使用
```
✅ 良い例
/users
/products
/orders

❌ 悪い例
/user
/product
/order
```

### ネストは浅く（2-3階層まで）
```
✅ 良い例
GET /users/123/orders
GET /users/123/orders/456

⚠️ 避けるべき
GET /users/123/orders/456/items/789/details
```

### ケバブケースまたはスネークケース
```
✅ 良い例
/user-profiles
/order-items

✅ または
/user_profiles
/order_items

❌ 悪い例
/userProfiles  # キャメルケース
/UserProfiles  # パスカルケース
```

## HTTPメソッド

### 適切なメソッド選択
```
GET     # リソース取得（冪等、安全）
POST    # リソース作成、または複雑な操作
PUT     # リソース全体更新（冪等）
PATCH   # リソース部分更新
DELETE  # リソース削除（冪等）
HEAD    # GETと同じだがボディなし
OPTIONS # サポートするメソッド確認
```

### 冪等性
```
冪等: 同じリクエストを複数回実行しても同じ結果
- GET, PUT, DELETE, HEAD, OPTIONS: 冪等
- POST, PATCH: 非冪等（実装次第）
```

### 例
```http
# リスト取得
GET /api/v1/users
Response: 200 OK

# 単一リソース取得
GET /api/v1/users/123
Response: 200 OK, 404 Not Found

# リソース作成
POST /api/v1/users
Request Body: { "name": "Alice", "email": "alice@example.com" }
Response: 201 Created
Location: /api/v1/users/123

# リソース全体更新
PUT /api/v1/users/123
Request Body: { "name": "Alice", "email": "alice@example.com", "age": 30 }
Response: 200 OK, 204 No Content

# リソース部分更新
PATCH /api/v1/users/123
Request Body: { "age": 31 }
Response: 200 OK

# リソース削除
DELETE /api/v1/users/123
Response: 204 No Content, 200 OK
```

## ステータスコード

### 2xx 成功
```
200 OK              # 成功（GET, PUT, PATCH）
201 Created         # リソース作成成功（POST）
202 Accepted        # リクエスト受理、処理中
204 No Content      # 成功、ボディなし（DELETE, PUT）
```

### 3xx リダイレクト
```
301 Moved Permanently    # 恒久的な移動
302 Found               # 一時的な移動
304 Not Modified        # キャッシュ有効
```

### 4xx クライアントエラー
```
400 Bad Request         # リクエスト不正
401 Unauthorized        # 認証必要
403 Forbidden           # 権限不足
404 Not Found           # リソースが存在しない
405 Method Not Allowed  # メソッド許可されていない
409 Conflict            # リソース競合
422 Unprocessable Entity # バリデーションエラー
429 Too Many Requests   # レート制限
```

### 5xx サーバーエラー
```
500 Internal Server Error  # サーバー内部エラー
502 Bad Gateway           # ゲートウェイエラー
503 Service Unavailable   # サービス利用不可
504 Gateway Timeout       # ゲートウェイタイムアウト
```

## リクエスト・レスポンス形式

### JSONを使用
```json
// ✅ 良い例
{
  "id": 123,
  "name": "Alice",
  "email": "alice@example.com",
  "createdAt": "2024-01-01T00:00:00Z"
}

// ❌ 悪い例（XMLは避ける）
<user>
  <id>123</id>
  <name>Alice</name>
</user>
```

### キャメルケース
```json
// ✅ 良い例
{
  "firstName": "Alice",
  "lastName": "Smith",
  "emailAddress": "alice@example.com"
}

// ❌ 悪い例
{
  "first_name": "Alice",  // スネークケース
  "FirstName": "Smith"    // パスカルケース
}
```

### 日付はISO 8601形式
```json
{
  "createdAt": "2024-01-01T12:00:00Z",
  "updatedAt": "2024-01-01T12:00:00+09:00"
}
```

### 列挙型は文字列
```json
// ✅ 良い例
{
  "status": "active",
  "role": "admin"
}

// ❌ 悪い例
{
  "status": 1,  // マジックナンバー
  "role": 0
}
```

## エラーレスポンス

### 一貫した形式
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      },
      {
        "field": "age",
        "message": "Must be greater than 0"
      }
    ],
    "timestamp": "2024-01-01T12:00:00Z",
    "path": "/api/v1/users",
    "requestId": "abc123"
  }
}
```

### エラーコード例
```
VALIDATION_ERROR      # バリデーションエラー
AUTHENTICATION_ERROR  # 認証エラー
AUTHORIZATION_ERROR   # 認可エラー
NOT_FOUND            # リソースが見つからない
CONFLICT             # リソース競合
RATE_LIMIT_EXCEEDED  # レート制限超過
INTERNAL_ERROR       # 内部エラー
SERVICE_UNAVAILABLE  # サービス利用不可
```

## ページネーション

### オフセットベース
```http
GET /api/v1/users?page=2&size=20

Response:
{
  "data": [...],
  "pagination": {
    "page": 2,
    "size": 20,
    "total": 100,
    "totalPages": 5
  }
}
```

### カーソルベース（大規模データ向け）
```http
GET /api/v1/users?cursor=eyJpZCI6MTIzfQ&limit=20

Response:
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJpZCI6MTQzfQ",
    "hasMore": true
  }
}
```

### リンクヘッダー（推奨）
```http
Link: <https://api.example.com/users?page=3>; rel="next",
      <https://api.example.com/users?page=1>; rel="prev",
      <https://api.example.com/users?page=1>; rel="first",
      <https://api.example.com/users?page=10>; rel="last"
```

## フィルタリング・ソート

### クエリパラメータ
```http
# フィルタリング
GET /api/v1/users?status=active&role=admin

# ソート
GET /api/v1/users?sort=createdAt:desc

# 複数条件
GET /api/v1/users?sort=name:asc,createdAt:desc

# フィールド選択
GET /api/v1/users?fields=id,name,email

# 検索
GET /api/v1/users?q=alice
```

### 複雑な検索
```http
# フィルタ式
GET /api/v1/users?filter=age>30 AND status='active'

# または専用エンドポイント
POST /api/v1/users/search
{
  "filters": {
    "age": { "gt": 30 },
    "status": "active"
  },
  "sort": [{ "field": "createdAt", "order": "desc" }],
  "page": 1,
  "size": 20
}
```

## バージョニング

### URLパス（推奨）
```http
https://api.example.com/v1/users
https://api.example.com/v2/users
```

### リクエストヘッダー
```http
GET /api/users
Accept: application/vnd.example.v1+json
```

### クエリパラメータ（非推奨）
```http
GET /api/users?version=1
```

### セマンティックバージョニング
```
v1.0.0  # メジャー.マイナー.パッチ

メジャー: 互換性のない変更
マイナー: 後方互換性のある機能追加
パッチ: 後方互換性のあるバグ修正
```

## 認証・認可

### Bearer Token（JWT）
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### API Key
```http
X-API-Key: your-api-key
```

### OAuth 2.0
```http
Authorization: Bearer {access_token}
```

### レスポンス
```http
# 認証エラー
401 Unauthorized
WWW-Authenticate: Bearer realm="example"

# 認可エラー
403 Forbidden
{
  "error": {
    "code": "AUTHORIZATION_ERROR",
    "message": "Insufficient permissions"
  }
}
```

## レート制限

### ヘッダー
```http
X-RateLimit-Limit: 1000      # 1時間あたりの上限
X-RateLimit-Remaining: 999   # 残り
X-RateLimit-Reset: 1672531200 # リセット時刻（Unix時間）
```

### 超過時のレスポンス
```http
HTTP/1.1 429 Too Many Requests
Retry-After: 3600

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Try again in 1 hour."
  }
}
```

## キャッシュ

### Cache-Controlヘッダー
```http
# キャッシュ可能
Cache-Control: public, max-age=3600

# キャッシュ不可
Cache-Control: no-store

# 条件付きキャッシュ
Cache-Control: private, max-age=0, must-revalidate
```

### ETag
```http
# レスポンス
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"

# リクエスト
If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"

# レスポンス（変更なし）
304 Not Modified
```

### Last-Modified
```http
# レスポンス
Last-Modified: Wed, 01 Jan 2024 12:00:00 GMT

# リクエスト
If-Modified-Since: Wed, 01 Jan 2024 12:00:00 GMT

# レスポンス（変更なし）
304 Not Modified
```

## CORS

```http
# プリフライトリクエスト
OPTIONS /api/v1/users
Origin: https://example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type

# レスポンス
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

## 非同期処理

### ロングランニングタスク
```http
# リクエスト
POST /api/v1/jobs
{
  "type": "export",
  "parameters": {...}
}

# レスポンス
HTTP/1.1 202 Accepted
Location: /api/v1/jobs/abc123

{
  "jobId": "abc123",
  "status": "pending",
  "statusUrl": "/api/v1/jobs/abc123"
}

# ステータス確認
GET /api/v1/jobs/abc123

{
  "jobId": "abc123",
  "status": "completed",
  "result": {...}
}
```

## ベストプラクティスまとめ

### ✅ やるべきこと
- RESTful な URL 設計（リソース指向、名詞、複数形）
- 適切な HTTP メソッドとステータスコードの使用
- 一貫した JSON レスポンス形式
- 適切なエラーハンドリングとエラーメッセージ
- バージョニング戦略の実装
- ページネーション、フィルタリング、ソートの提供
- 適切な認証・認可の実装
- レート制限の実装
- HTTPS の使用
- API ドキュメントの提供（OpenAPI/Swagger）

### ❌ 避けるべきこと
- 動詞を含む URL
- 深すぎるネスト
- クライアント固有の実装
- 機密情報をURLに含める
- HTTPステータスコードの誤用
- エラーメッセージに内部実装の詳細を含める
- バージョニングなしでの破壊的変更

## OpenAPI/Swagger ドキュメント例

```yaml
openapi: 3.0.0
info:
  title: User API
  version: 1.0.0
  description: User management API

servers:
  - url: https://api.example.com/v1

paths:
  /users:
    get:
      summary: Get all users
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: size
          in: query
          schema:
            type: integer
            default: 20
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/User'
                  pagination:
                    $ref: '#/components/schemas/Pagination'

    post:
      summary: Create a new user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserInput'
      responses:
        '201':
          description: User created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'

  /users/{id}:
    get:
      summary: Get user by ID
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          description: User not found

components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: integer
        name:
          type: string
        email:
          type: string
        createdAt:
          type: string
          format: date-time

    UserInput:
      type: object
      required:
        - name
        - email
      properties:
        name:
          type: string
        email:
          type: string
          format: email

    Pagination:
      type: object
      properties:
        page:
          type: integer
        size:
          type: integer
        total:
          type: integer
        totalPages:
          type: integer
```

## 参考リソース

- [RFC 7231 - HTTP/1.1 Semantics](https://tools.ietf.org/html/rfc7231)
- [REST API Tutorial](https://restfulapi.net/)
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)
- [Google API Design Guide](https://cloud.google.com/apis/design)
- [OpenAPI Specification](https://swagger.io/specification/)
