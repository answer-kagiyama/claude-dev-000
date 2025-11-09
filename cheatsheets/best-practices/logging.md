# Logging ベストプラクティス

## 構造化ログ（JSON）

### 基本原則

```javascript
// ❌ 悪い例：非構造化ログ
console.log('User Alice logged in from 192.168.1.1');
console.log('Error: Failed to connect to database');

// ✅ 良い例：構造化ログ（JSON）
logger.info({
  event: 'user_login',
  userId: '123',
  username: 'Alice',
  ipAddress: '192.168.1.1',
  timestamp: '2024-01-01T12:00:00Z'
});

logger.error({
  event: 'database_connection_error',
  error: {
    message: 'Connection timeout',
    code: 'ETIMEDOUT',
    stack: err.stack
  },
  database: 'postgres',
  timestamp: '2024-01-01T12:00:00Z'
});
```

### Winston（Node.js）

```javascript
const winston = require('winston');

// ✅ 構造化ロガー設定
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'user-service',
    environment: process.env.NODE_ENV
  },
  transports: [
    // Console（開発環境）
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      )
    }),
    // File（本番環境）
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error'
    }),
    new winston.transports.File({
      filename: 'logs/combined.log'
    })
  ]
});

// 使用例
logger.info('User login', {
  userId: '123',
  ipAddress: req.ip,
  userAgent: req.headers['user-agent']
});

logger.error('Database query failed', {
  query: 'SELECT * FROM users',
  error: err.message,
  stack: err.stack,
  executionTime: 5000
});
```

### Pino（高速ロガー）

```javascript
const pino = require('pino');

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => {
      return { level: label };
    }
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  serializers: {
    req: pino.stdSerializers.req,
    res: pino.stdSerializers.res,
    err: pino.stdSerializers.err
  }
});

// 使用例
logger.info({ userId: '123', action: 'login' }, 'User logged in');
```

## ログレベル

### レベル定義

```javascript
// 標準的なログレベル
const LOG_LEVELS = {
  ERROR: 0,   // エラー（即座に対応が必要）
  WARN: 1,    // 警告（注意が必要だが動作は継続）
  INFO: 2,    // 情報（重要なイベント）
  HTTP: 3,    // HTTPリクエスト
  DEBUG: 4,   // デバッグ情報（開発時のみ）
  TRACE: 5    // 詳細なトレース（最も詳細）
};

// ✅ 適切な使用例
logger.error('Payment processing failed', { orderId, error });
// → アラート通知、即座に対応

logger.warn('API rate limit approaching', { current: 950, limit: 1000 });
// → 監視、必要に応じて対応

logger.info('User registered', { userId, email });
// → 通常のビジネスイベント

logger.debug('Cache lookup', { key, hit: true, ttl: 3600 });
// → 開発時のデバッグ情報

logger.trace('Function entry', { args: [1, 2, 3] });
// → パフォーマンス調査時のみ
```

### 環境ごとの設定

```javascript
const getLogLevel = () => {
  switch (process.env.NODE_ENV) {
    case 'production':
      return 'info';    // 本番は INFO 以上
    case 'staging':
      return 'debug';   // ステージングは DEBUG 以上
    case 'development':
      return 'trace';   // 開発は全て
    default:
      return 'info';
  }
};

const logger = winston.createLogger({
  level: getLogLevel()
});
```

## コンテキストと相関ID

### Request ID

```javascript
const { v4: uuidv4 } = require('uuid');

// ✅ ミドルウェアで Request ID を付与
app.use((req, res, next) => {
  req.id = req.headers['x-request-id'] || uuidv4();
  res.setHeader('X-Request-ID', req.id);
  next();
});

// ✅ すべてのログに Request ID を含める
app.use((req, res, next) => {
  req.logger = logger.child({
    requestId: req.id,
    method: req.method,
    path: req.path,
    userAgent: req.headers['user-agent']
  });
  next();
});

// 使用例
app.get('/users/:id', async (req, res) => {
  req.logger.info('Fetching user', { userId: req.params.id });

  try {
    const user = await getUser(req.params.id);
    req.logger.info('User fetched successfully');
    res.json(user);
  } catch (err) {
    req.logger.error('Failed to fetch user', { error: err.message });
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

### Correlation ID（マイクロサービス）

```javascript
// ✅ サービス間で Correlation ID を伝播
async function callExternalService(url, data) {
  const correlationId = req.id;

  const response = await fetch(url, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-Correlation-ID': correlationId  // 伝播
    },
    body: JSON.stringify(data)
  });

  logger.info('External service called', {
    correlationId,
    url,
    statusCode: response.status
  });

  return response.json();
}
```

## セキュリティとプライバシー

### 機密情報のマスキング

```javascript
// ✅ 機密情報を自動的にマスキング
const sensitiveFields = ['password', 'ssn', 'creditCard', 'apiKey', 'token'];

const maskSensitiveData = (obj) => {
  if (typeof obj !== 'object' || obj === null) return obj;

  const masked = { ...obj };

  for (const key in masked) {
    if (sensitiveFields.some(field => key.toLowerCase().includes(field))) {
      masked[key] = '***REDACTED***';
    } else if (typeof masked[key] === 'object') {
      masked[key] = maskSensitiveData(masked[key]);
    }
  }

  return masked;
};

// Winston formatter として使用
const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format((info) => {
      return maskSensitiveData(info);
    })(),
    winston.format.json()
  )
});

// 使用例
logger.info('User data', {
  username: 'alice',
  email: 'alice@example.com',
  password: 'secret123',  // → '***REDACTED***'
  apiKey: 'sk-1234567890'  // → '***REDACTED***'
});
```

### PII（個人識別情報）の扱い

```javascript
// ✅ 最小限の情報のみログ出力
logger.info('User action', {
  userId: hash(user.email),  // ハッシュ化
  action: 'purchase',
  amount: 1000
  // ❌ email, name, address は含めない
});

// ✅ GDPR 対応のログ保持期間
const logger = winston.createLogger({
  transports: [
    new winston.transports.File({
      filename: 'logs/app.log',
      maxsize: 10 * 1024 * 1024,  // 10MB
      maxFiles: 30,                // 30日間
      tailable: true
    })
  ]
});
```

## HTTPリクエストログ

### Express ミドルウェア

```javascript
const morgan = require('morgan');

// ✅ JSON形式でログ出力
morgan.token('user-id', (req) => req.user?.id || 'anonymous');

app.use(morgan((tokens, req, res) => {
  return JSON.stringify({
    method: tokens.method(req, res),
    url: tokens.url(req, res),
    status: tokens.status(req, res),
    responseTime: parseFloat(tokens['response-time'](req, res)),
    contentLength: tokens.res(req, res, 'content-length'),
    userAgent: tokens['user-agent'](req, res),
    userId: tokens['user-id'](req, res),
    requestId: req.id,
    timestamp: new Date().toISOString()
  });
}));
```

### カスタムHTTPロガー

```javascript
app.use((req, res, next) => {
  const startTime = Date.now();

  // レスポンス完了時
  res.on('finish', () => {
    const duration = Date.now() - startTime;

    req.logger.info('HTTP request', {
      method: req.method,
      url: req.originalUrl,
      statusCode: res.statusCode,
      duration,
      contentLength: res.get('content-length'),
      userAgent: req.headers['user-agent'],
      ip: req.ip
    });

    // 遅いリクエストを警告
    if (duration > 5000) {
      req.logger.warn('Slow request detected', {
        method: req.method,
        url: req.originalUrl,
        duration
      });
    }
  });

  next();
});
```

## エラーログ

### エラーコンテキスト

```javascript
// ✅ 豊富なコンテキスト
try {
  await processPayment(orderId, amount);
} catch (err) {
  logger.error('Payment processing failed', {
    error: {
      name: err.name,
      message: err.message,
      stack: err.stack,
      code: err.code
    },
    context: {
      orderId,
      amount,
      userId: req.user.id,
      paymentMethod: 'credit_card'
    },
    timestamp: new Date().toISOString()
  });

  // エラー通知
  notifyErrorTracking(err, { orderId, userId: req.user.id });

  throw err;
}
```

### グローバルエラーハンドラ

```javascript
// ✅ 未処理エラーのキャッチ
process.on('uncaughtException', (err) => {
  logger.error('Uncaught exception', {
    error: {
      message: err.message,
      stack: err.stack
    }
  });

  // Graceful shutdown
  process.exit(1);
});

process.on('unhandledRejection', (reason, promise) => {
  logger.error('Unhandled rejection', {
    reason,
    promise
  });
});

// Express エラーハンドラ
app.use((err, req, res, next) => {
  req.logger.error('Express error handler', {
    error: {
      message: err.message,
      stack: err.stack
    },
    url: req.originalUrl,
    method: req.method
  });

  res.status(err.status || 500).json({
    error: {
      message: process.env.NODE_ENV === 'production'
        ? 'Internal server error'
        : err.message,
      requestId: req.id
    }
  });
});
```

## パフォーマンス

### ログの非同期化

```javascript
// ✅ 非同期トランスポート
const logger = winston.createLogger({
  transports: [
    new winston.transports.Stream({
      stream: fs.createWriteStream('logs/app.log', { flags: 'a' })
    })
  ]
});

// ✅ Pino（デフォルトで非同期）
const logger = pino(
  pino.destination({
    dest: 'logs/app.log',
    sync: false  // 非同期
  })
);
```

### サンプリング

```javascript
// ✅ 高頻度ログのサンプリング
let requestCount = 0;

app.use((req, res, next) => {
  requestCount++;

  // 100リクエストに1回だけログ
  if (requestCount % 100 === 0) {
    logger.debug('Request sample', {
      count: requestCount,
      method: req.method,
      url: req.url
    });
  }

  next();
});
```

### ログローテーション

```javascript
// ✅ ファイルローテーション
const DailyRotateFile = require('winston-daily-rotate-file');

const logger = winston.createLogger({
  transports: [
    new DailyRotateFile({
      filename: 'logs/application-%DATE%.log',
      datePattern: 'YYYY-MM-DD',
      maxSize: '20m',
      maxFiles: '14d',  // 14日間保持
      zippedArchive: true
    })
  ]
});
```

## ログ集約

### ELK Stack（Elasticsearch + Logstash + Kibana）

```javascript
// ✅ Elasticsearch トランスポート
const { ElasticsearchTransport } = require('winston-elasticsearch');

const esTransport = new ElasticsearchTransport({
  level: 'info',
  clientOpts: {
    node: 'http://localhost:9200',
    auth: {
      username: 'elastic',
      password: process.env.ELASTIC_PASSWORD
    }
  },
  index: 'logs-myapp'
});

const logger = winston.createLogger({
  transports: [esTransport]
});
```

### Fluentd

```javascript
// ✅ Fluentd 形式で出力
const logger = pino({
  formatters: {
    log: (obj) => {
      return {
        time: obj.time,
        level: obj.level,
        message: obj.msg,
        ...obj
      };
    }
  }
});

// Fluentd 設定例（td-agent.conf）
/*
<source>
  @type tail
  path /var/log/myapp/*.log
  pos_file /var/log/td-agent/myapp.pos
  tag myapp.logs
  <parse>
    @type json
    time_key time
    time_format %Y-%m-%dT%H:%M:%S.%L%z
  </parse>
</source>

<match myapp.logs>
  @type elasticsearch
  host elasticsearch
  port 9200
  index_name myapp
  type_name _doc
</match>
*/
```

### CloudWatch Logs（AWS）

```javascript
const WinstonCloudWatch = require('winston-cloudwatch');

const logger = winston.createLogger({
  transports: [
    new WinstonCloudWatch({
      logGroupName: '/aws/lambda/myapp',
      logStreamName: process.env.AWS_LAMBDA_LOG_STREAM_NAME,
      awsRegion: 'us-east-1',
      jsonMessage: true
    })
  ]
});
```

## 分散トレーシング

### OpenTelemetry

```javascript
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');

// ✅ トレーシング設定
const provider = new NodeTracerProvider({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'user-service',
  }),
});

provider.register();

const tracer = provider.getTracer('user-service');

// 使用例
app.get('/users/:id', async (req, res) => {
  const span = tracer.startSpan('get-user');

  try {
    logger.info('Fetching user', {
      traceId: span.spanContext().traceId,
      spanId: span.spanContext().spanId,
      userId: req.params.id
    });

    const user = await getUser(req.params.id);
    span.setStatus({ code: 0 });
    res.json(user);
  } catch (err) {
    span.setStatus({ code: 2, message: err.message });
    throw err;
  } finally {
    span.end();
  }
});
```

## アラート設定

### ログベースアラート

```javascript
// ✅ エラー率が閾値を超えたらアラート
const errorCount = new Map();

logger.on('data', (log) => {
  if (log.level === 'error') {
    const minute = Math.floor(Date.now() / 60000);
    const count = errorCount.get(minute) || 0;
    errorCount.set(minute, count + 1);

    // 1分間に10件以上のエラー
    if (count + 1 >= 10) {
      notifyAlert({
        severity: 'high',
        message: 'High error rate detected',
        errorCount: count + 1,
        minute
      });
    }
  }
});
```

## ベストプラクティスまとめ

### ✅ やるべきこと

```javascript
// 1. 構造化ログ（JSON）
logger.info({ event: 'user_login', userId: '123' });

// 2. 適切なログレベル
logger.error('Critical error');  // 即座に対応
logger.info('Business event');    // 記録

// 3. コンテキスト情報
logger.info({ requestId, userId, action: 'purchase' });

// 4. 機密情報のマスキング
logger.info({ email: 'alice@example.com', password: '***' });

// 5. タイムスタンプ（ISO 8601）
logger.info({ timestamp: '2024-01-01T12:00:00Z' });

// 6. 環境ごとの設定
const level = process.env.LOG_LEVEL || 'info';

// 7. ログローテーション
maxFiles: '14d'

// 8. パフォーマンス測定
logger.info({ duration: 123, query: 'SELECT...' });
```

### ❌ 避けるべきこと

```javascript
// ❌ 非構造化ログ
console.log('User 123 logged in');

// ❌ 不適切なレベル
logger.error('User logged in');  // これは INFO

// ❌ 機密情報の漏洩
logger.info({ password: 'secret123', apiKey: 'sk-...' });

// ❌ コンテキスト不足
logger.error('Error occurred');  // 何のエラー？

// ❌ 過剰なログ
for (let i = 0; i < 1000000; i++) {
  logger.debug(`Iteration ${i}`);  // パフォーマンス低下
}

// ❌ 同期ログ（本番環境）
logger.add(new winston.transports.File({ sync: true }));
```

## チェックリスト

### 基本設定
- [ ] 構造化ログ（JSON形式）
- [ ] 適切なログレベル設定
- [ ] タイムスタンプ（ISO 8601形式）
- [ ] Request ID / Correlation ID
- [ ] 環境ごとのログレベル設定

### セキュリティ
- [ ] 機密情報のマスキング
- [ ] PII の最小化
- [ ] ログ保持期間の設定
- [ ] アクセス制御

### パフォーマンス
- [ ] 非同期ログ出力
- [ ] ログローテーション
- [ ] サンプリング（高頻度ログ）
- [ ] 適切なログレベル（本番は INFO 以上）

### 運用
- [ ] ログ集約（ELK, Fluentd等）
- [ ] アラート設定
- [ ] ダッシュボード
- [ ] ログ検索・分析

## 参考リソース

- [12 Factor App - Logs](https://12factor.net/logs)
- [Structured Logging](https://www.thoughtworks.com/insights/blog/structured-logging)
- [OpenTelemetry](https://opentelemetry.io/)
- [ELK Stack](https://www.elastic.co/elastic-stack)
