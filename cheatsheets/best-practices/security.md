# セキュリティベストプラクティス

## OWASP Top 10（2021）

### 1. アクセス制御の不備（Broken Access Control）

#### 問題
```javascript
// ❌ 悪い例：ユーザーIDをクライアントから受け取る
app.get('/user/:id', (req, res) => {
  const userId = req.params.id;
  const user = db.getUser(userId);  // どのユーザーでもアクセス可能
  res.json(user);
});
```

#### 対策
```javascript
// ✅ 良い例：認証されたユーザー自身の情報のみアクセス可能
app.get('/user/me', authenticateToken, (req, res) => {
  const userId = req.user.id;  // トークンから取得
  const user = db.getUser(userId);
  res.json(user);
});

// 管理者のみアクセス可能
app.delete('/user/:id', authenticateToken, requireAdmin, (req, res) => {
  // 削除処理
});
```

### 2. 暗号化の失敗（Cryptographic Failures）

#### パスワードのハッシュ化
```javascript
// ❌ 悪い例
const password = req.body.password;
user.password = md5(password);  // MD5は脆弱

// ✅ 良い例：bcrypt使用
const bcrypt = require('bcrypt');
const saltRounds = 10;
const hashedPassword = await bcrypt.hash(password, saltRounds);
```

#### 機密データの暗号化
```javascript
// ✅ 良い例
const crypto = require('crypto');

// 暗号化
function encrypt(text, key) {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);
  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  const authTag = cipher.getAuthTag();
  return { encrypted, iv, authTag };
}

// 復号化
function decrypt(encrypted, key, iv, authTag) {
  const decipher = crypto.createDecipheriv('aes-256-gcm', key, iv);
  decipher.setAuthTag(authTag);
  let decrypted = decipher.update(encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  return decrypted;
}
```

### 3. インジェクション（Injection）

#### SQLインジェクション対策
```javascript
// ❌ 悪い例
const query = `SELECT * FROM users WHERE username = '${username}'`;
db.execute(query);

// ✅ 良い例：プリペアドステートメント
const query = 'SELECT * FROM users WHERE username = ?';
db.execute(query, [username]);

// ✅ ORMを使用
const user = await User.findOne({ where: { username } });
```

#### NoSQLインジェクション対策
```javascript
// ❌ 悪い例
const user = await User.findOne({ username: req.body.username });

// ✅ 良い例：入力検証
const username = req.body.username;
if (typeof username !== 'string') {
  throw new Error('Invalid username');
}
const user = await User.findOne({ username });
```

#### コマンドインジェクション対策
```javascript
// ❌ 悪い例
const { exec } = require('child_process');
exec(`ping ${req.body.host}`);  // 危険

// ✅ 良い例：入力検証とホワイトリスト
const { execFile } = require('child_process');
const host = req.body.host;

if (!/^[\w.-]+$/.test(host)) {
  throw new Error('Invalid host');
}

execFile('ping', ['-c', '1', host]);
```

### 4. 安全でない設計（Insecure Design）

#### レート制限
```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15分
  max: 100,  // 最大100リクエスト
  message: 'Too many requests from this IP'
});

app.use('/api/', limiter);
```

#### アカウントロックアウト
```python
# ログイン失敗回数を記録
def login(username, password):
    user = get_user(username)

    # アカウントロック確認
    if user.failed_attempts >= 5:
        if datetime.now() < user.locked_until:
            raise AccountLockedException()
        else:
            user.failed_attempts = 0

    if not verify_password(user, password):
        user.failed_attempts += 1
        if user.failed_attempts >= 5:
            user.locked_until = datetime.now() + timedelta(minutes=30)
        save_user(user)
        raise InvalidCredentialsException()

    user.failed_attempts = 0
    save_user(user)
    return create_session(user)
```

### 5. セキュリティ設定ミス（Security Misconfiguration）

#### HTTPセキュリティヘッダー
```javascript
const helmet = require('helmet');

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
    },
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  },
}));

// または手動設定
app.use((req, res, next) => {
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '1; mode=block');
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
  next();
});
```

### 6. 脆弱で古いコンポーネント（Vulnerable and Outdated Components）

#### 依存関係の監査
```bash
# npm
npm audit
npm audit fix

# yarn
yarn audit

# 自動更新
npm install -g npm-check-updates
ncu -u
npm install

# Dependabot（GitHub）を有効化
```

### 7. 識別と認証の失敗（Identification and Authentication Failures）

#### JWT認証
```javascript
const jwt = require('jsonwebtoken');

// トークン生成
function generateToken(user) {
  return jwt.sign(
    { id: user.id, email: user.email },
    process.env.JWT_SECRET,
    { expiresIn: '1h', algorithm: 'HS256' }
  );
}

// トークン検証
function authenticateToken(req, res, next) {
  const token = req.headers['authorization']?.split(' ')[1];

  if (!token) {
    return res.sendStatus(401);
  }

  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) {
      return res.sendStatus(403);
    }
    req.user = user;
    next();
  });
}

app.get('/protected', authenticateToken, (req, res) => {
  res.json({ message: 'Protected resource' });
});
```

#### セッション管理
```javascript
const session = require('express-session');

app.use(session({
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: true,  // HTTPS必須
    httpOnly: true,  // JavaScriptからアクセス不可
    maxAge: 3600000,  // 1時間
    sameSite: 'strict'
  }
}));
```

### 8. ソフトウェアとデータの整合性の不具合

#### Subresource Integrity（SRI）
```html
<!-- CDNからのスクリプト読み込み時にSRI使用 -->
<script
  src="https://cdn.example.com/library.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
  crossorigin="anonymous">
</script>
```

### 9. セキュリティログとモニタリングの失敗

#### ログ記録
```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});

// セキュリティイベントのログ
function login(req, res) {
  const { username, password } = req.body;

  try {
    const user = authenticate(username, password);
    logger.info('Successful login', {
      username,
      ip: req.ip,
      userAgent: req.get('user-agent'),
      timestamp: new Date()
    });
    // ...
  } catch (err) {
    logger.warn('Failed login attempt', {
      username,
      ip: req.ip,
      timestamp: new Date()
    });
    // ...
  }
}
```

### 10. サーバサイドリクエストフォージェリ（SSRF）

#### 対策
```javascript
// ❌ 悪い例
app.get('/fetch', async (req, res) => {
  const url = req.query.url;
  const response = await fetch(url);  // 任意のURLにアクセス可能
  res.send(await response.text());
});

// ✅ 良い例：URLホワイトリスト
const ALLOWED_HOSTS = ['api.example.com', 'cdn.example.com'];

app.get('/fetch', async (req, res) => {
  const url = new URL(req.query.url);

  if (!ALLOWED_HOSTS.includes(url.hostname)) {
    return res.status(400).send('Invalid URL');
  }

  const response = await fetch(url.toString());
  res.send(await response.text());
});
```

## XSS（クロスサイトスクリプティング）対策

### 出力のエスケープ
```javascript
// ❌ 悪い例
app.get('/search', (req, res) => {
  const query = req.query.q;
  res.send(`<p>Search results for: ${query}</p>`);  // XSS脆弱性
});

// ✅ 良い例：テンプレートエンジン使用（自動エスケープ）
app.set('view engine', 'ejs');
app.get('/search', (req, res) => {
  res.render('search', { query: req.query.q });
});
```

### Content Security Policy
```javascript
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'", "'nonce-{random}'"],
    objectSrc: ["'none'"],
    upgradeInsecureRequests: [],
  },
}));
```

## CSRF（クロスサイトリクエストフォージェリ）対策

```javascript
const csrf = require('csurf');
const csrfProtection = csrf({ cookie: true });

app.get('/form', csrfProtection, (req, res) => {
  res.render('form', { csrfToken: req.csrfToken() });
});

app.post('/process', csrfProtection, (req, res) => {
  // CSRF トークン検証済み
  res.send('Data processed');
});
```

```html
<!-- フォームにCSRFトークン埋め込み -->
<form method="POST" action="/process">
  <input type="hidden" name="_csrf" value="<%= csrfToken %>">
  <button type="submit">Submit</button>
</form>
```

## CORS設定

```javascript
const cors = require('cors');

// ✅ 特定のオリジンのみ許可
app.use(cors({
  origin: 'https://example.com',
  credentials: true,
  optionsSuccessStatus: 200
}));

// ✅ 複数のオリジン許可
const whitelist = ['https://example.com', 'https://app.example.com'];
app.use(cors({
  origin: (origin, callback) => {
    if (whitelist.indexOf(origin) !== -1 || !origin) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  }
}));

// ❌ すべてのオリジンを許可（本番環境では避ける）
app.use(cors({ origin: '*' }));
```

## 入力検証

### バリデーションライブラリ使用
```javascript
const { body, validationResult } = require('express-validator');

app.post('/user',
  body('email').isEmail().normalizeEmail(),
  body('password').isLength({ min: 8 }).matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/),
  body('age').isInt({ min: 0, max: 150 }),
  (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }
    // 処理続行
  }
);
```

### サニタイゼーション
```javascript
const sanitizeHtml = require('sanitize-html');

const clean = sanitizeHtml(dirty, {
  allowedTags: ['b', 'i', 'em', 'strong', 'a'],
  allowedAttributes: {
    'a': ['href']
  }
});
```

## シークレット管理

### 環境変数
```bash
# .env ファイル（Gitにコミットしない）
DATABASE_URL=postgresql://user:pass@localhost/db
JWT_SECRET=your-secret-key
API_KEY=your-api-key
```

```javascript
require('dotenv').config();

const dbUrl = process.env.DATABASE_URL;
const jwtSecret = process.env.JWT_SECRET;
```

### AWS Secrets Manager
```javascript
const AWS = require('aws-sdk');
const secretsManager = new AWS.SecretsManager();

async function getSecret(secretName) {
  const data = await secretsManager.getSecretValue({
    SecretId: secretName
  }).promise();

  return JSON.parse(data.SecretString);
}

const dbCredentials = await getSecret('prod/db/credentials');
```

## HTTPS強制

```javascript
// HTTPSリダイレクト
app.use((req, res, next) => {
  if (req.header('x-forwarded-proto') !== 'https') {
    res.redirect(`https://${req.header('host')}${req.url}`);
  } else {
    next();
  }
});

// HSTSヘッダー
app.use((req, res, next) => {
  res.setHeader('Strict-Transport-Security',
    'max-age=31536000; includeSubDomains; preload');
  next();
});
```

## セキュアなファイルアップロード

```javascript
const multer = require('multer');
const path = require('path');

const storage = multer.diskStorage({
  destination: './uploads/',
  filename: (req, file, cb) => {
    // ランダムなファイル名生成
    const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1E9);
    cb(null, uniqueSuffix + path.extname(file.originalname));
  }
});

const upload = multer({
  storage: storage,
  limits: {
    fileSize: 5 * 1024 * 1024  // 5MB
  },
  fileFilter: (req, file, cb) => {
    // 許可する拡張子
    const allowedTypes = /jpeg|jpg|png|pdf/;
    const extname = allowedTypes.test(path.extname(file.originalname).toLowerCase());
    const mimetype = allowedTypes.test(file.mimetype);

    if (extname && mimetype) {
      return cb(null, true);
    } else {
      cb(new Error('Invalid file type'));
    }
  }
});

app.post('/upload', upload.single('file'), (req, res) => {
  res.json({ filename: req.file.filename });
});
```

## API キーのローテーション

```javascript
class ApiKeyManager {
  constructor() {
    this.keys = new Map();
  }

  generateKey(userId) {
    const key = crypto.randomBytes(32).toString('hex');
    const hashedKey = crypto.createHash('sha256').update(key).digest('hex');

    this.keys.set(hashedKey, {
      userId,
      createdAt: new Date(),
      expiresAt: new Date(Date.now() + 90 * 24 * 60 * 60 * 1000)  // 90日
    });

    return key;
  }

  validateKey(key) {
    const hashedKey = crypto.createHash('sha256').update(key).digest('hex');
    const keyData = this.keys.get(hashedKey);

    if (!keyData || keyData.expiresAt < new Date()) {
      return null;
    }

    return keyData.userId;
  }
}
```

## セキュリティチェックリスト

### 認証・認可
- [ ] パスワードは適切にハッシュ化（bcrypt、scrypt、Argon2）
- [ ] セッションIDはランダムで推測不可能
- [ ] セッションタイムアウトを実装
- [ ] 多要素認証（MFA）を検討
- [ ] パスワード要件を設定（最小長、複雑さ）
- [ ] アカウントロックアウトを実装

### データ保護
- [ ] HTTPS を強制
- [ ] 機密データを暗号化
- [ ] 個人情報を適切に扱う（GDPR、個人情報保護法）
- [ ] バックアップを暗号化

### 入力検証
- [ ] すべての入力を検証
- [ ] ホワイトリストベースの検証
- [ ] 出力をエスケープ
- [ ] SQLインジェクション対策
- [ ] XSS対策
- [ ] CSRF対策

### インフラ
- [ ] ファイアウォールを設定
- [ ] 最小権限の原則
- [ ] 不要なサービスを無効化
- [ ] 定期的なセキュリティパッチ適用
- [ ] ログとモニタリング

### API
- [ ] レート制限を実装
- [ ] CORS を適切に設定
- [ ] API キーを安全に管理
- [ ] 適切なHTTPステータスコード
- [ ] エラーメッセージに機密情報を含めない

### コード
- [ ] 依存関係の脆弱性スキャン
- [ ] コードレビュー
- [ ] 静的解析ツール使用
- [ ] セキュリティテスト

## ツール

### 脆弱性スキャン
```bash
# npm audit
npm audit

# Snyk
npm install -g snyk
snyk test

# OWASP Dependency-Check
dependency-check --project myapp --scan .
```

### 静的解析
```bash
# ESLint セキュリティプラグイン
npm install --save-dev eslint-plugin-security
```

### ペネトレーションテスト
- OWASP ZAP
- Burp Suite
- Nikto

## 参考リソース

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [CWE Top 25](https://cwe.mitre.org/top25/)
- [Mozilla Web Security Guidelines](https://infosec.mozilla.org/guidelines/web_security)
