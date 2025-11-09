# Testing ベストプラクティス

## テストピラミッド

```
        /\
       /E2E\          少ない（遅い、高コスト）
      /------\
     /  統合  \        中程度
    /----------\
   / ユニット  \      多い（速い、低コスト）
  /--------------\

推奨比率: Unit 70% : Integration 20% : E2E 10%
```

## ユニットテスト

### 基本原則

```javascript
// ❌ 悪い例：複数の責務をテスト
test('user service', () => {
  const user = createUser();
  expect(user.name).toBe('Alice');
  expect(saveUser(user)).toBe(true);
  expect(deleteUser(user.id)).toBe(true);
});

// ✅ 良い例：1テスト1責務
describe('User', () => {
  test('should create user with valid name', () => {
    const user = createUser({ name: 'Alice' });
    expect(user.name).toBe('Alice');
  });

  test('should throw error for invalid name', () => {
    expect(() => createUser({ name: '' }))
      .toThrow('Name is required');
  });
});
```

### AAA パターン（Arrange-Act-Assert）

```javascript
test('should calculate total price with tax', () => {
  // Arrange（準備）
  const items = [
    { price: 100, quantity: 2 },
    { price: 200, quantity: 1 }
  ];
  const taxRate = 0.1;

  // Act（実行）
  const total = calculateTotal(items, taxRate);

  // Assert（検証）
  expect(total).toBe(440); // (100*2 + 200) * 1.1
});
```

### テストケース設計

```javascript
describe('divide', () => {
  // 正常系
  test('should divide two positive numbers', () => {
    expect(divide(10, 2)).toBe(5);
  });

  // 境界値
  test('should handle division by one', () => {
    expect(divide(10, 1)).toBe(10);
  });

  test('should handle zero dividend', () => {
    expect(divide(0, 5)).toBe(0);
  });

  // 異常系
  test('should throw error for division by zero', () => {
    expect(() => divide(10, 0)).toThrow('Division by zero');
  });

  // エッジケース
  test('should handle negative numbers', () => {
    expect(divide(-10, 2)).toBe(-5);
  });

  test('should handle floating point', () => {
    expect(divide(10, 3)).toBeCloseTo(3.333, 2);
  });
});
```

### モック・スタブ

```javascript
// モック（振る舞いの検証）
describe('UserService', () => {
  test('should call repository save method', async () => {
    // Arrange
    const mockRepository = {
      save: jest.fn().mockResolvedValue({ id: 1 })
    };
    const userService = new UserService(mockRepository);

    // Act
    await userService.createUser({ name: 'Alice' });

    // Assert
    expect(mockRepository.save).toHaveBeenCalledWith({ name: 'Alice' });
    expect(mockRepository.save).toHaveBeenCalledTimes(1);
  });
});

// スタブ（データの置き換え）
describe('WeatherService', () => {
  test('should return weather data', async () => {
    // Arrange
    const mockApiClient = {
      get: jest.fn().mockResolvedValue({
        temp: 25,
        condition: 'sunny'
      })
    };
    const weatherService = new WeatherService(mockApiClient);

    // Act
    const weather = await weatherService.getWeather('Tokyo');

    // Assert
    expect(weather.temp).toBe(25);
    expect(weather.condition).toBe('sunny');
  });
});
```

### テストダブルの種類

```javascript
// Dummy: 引数を埋めるだけ
const dummyLogger = { log: () => {} };

// Stub: 決まった値を返す
const stubUserRepo = {
  findById: () => ({ id: 1, name: 'Alice' })
};

// Spy: 呼び出しを記録
const spyLogger = {
  log: jest.fn()
};
expect(spyLogger.log).toHaveBeenCalled();

// Mock: 振る舞いを完全に制御
const mockPaymentService = {
  charge: jest.fn()
    .mockResolvedValueOnce({ success: true })
    .mockRejectedValueOnce(new Error('Payment failed'))
};

// Fake: 動作する軽量実装
class FakeUserRepository {
  constructor() {
    this.users = new Map();
  }
  save(user) {
    this.users.set(user.id, user);
  }
  findById(id) {
    return this.users.get(id);
  }
}
```

## 統合テスト

### データベーステスト

```javascript
// テストごとにデータベースをクリーンアップ
describe('UserRepository', () => {
  beforeEach(async () => {
    await db.sync({ force: true }); // テーブル再作成
  });

  afterEach(async () => {
    await db.truncate({ cascade: true }); // データクリア
  });

  test('should save and retrieve user', async () => {
    // Arrange
    const userData = { name: 'Alice', email: 'alice@example.com' };

    // Act
    const savedUser = await userRepository.save(userData);
    const foundUser = await userRepository.findById(savedUser.id);

    // Assert
    expect(foundUser.name).toBe('Alice');
    expect(foundUser.email).toBe('alice@example.com');
  });
});
```

### テストコンテナ（推奨）

```javascript
// Testcontainers を使用
const { GenericContainer } = require('testcontainers');

describe('PostgreSQL integration', () => {
  let container;
  let db;

  beforeAll(async () => {
    // PostgreSQLコンテナ起動
    container = await new GenericContainer('postgres:15')
      .withEnvironment({
        POSTGRES_USER: 'test',
        POSTGRES_PASSWORD: 'test',
        POSTGRES_DB: 'testdb'
      })
      .withExposedPorts(5432)
      .start();

    // 接続
    const port = container.getMappedPort(5432);
    db = await connectToDatabase({
      host: 'localhost',
      port,
      database: 'testdb',
      user: 'test',
      password: 'test'
    });
  });

  afterAll(async () => {
    await db.close();
    await container.stop();
  });

  test('should perform database operations', async () => {
    const result = await db.query('SELECT 1 as value');
    expect(result.rows[0].value).toBe(1);
  });
});
```

### API統合テスト

```javascript
// Supertest を使用
const request = require('supertest');
const app = require('../app');

describe('POST /api/users', () => {
  test('should create new user', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ name: 'Alice', email: 'alice@example.com' })
      .expect(201)
      .expect('Content-Type', /json/);

    expect(response.body).toMatchObject({
      name: 'Alice',
      email: 'alice@example.com'
    });
    expect(response.body.id).toBeDefined();
  });

  test('should return 400 for invalid email', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ name: 'Alice', email: 'invalid' })
      .expect(400);

    expect(response.body.error).toContain('Invalid email');
  });
});
```

## E2Eテスト

### Playwright（推奨）

```javascript
import { test, expect } from '@playwright/test';

test.describe('User authentication', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('http://localhost:3000');
  });

  test('should login successfully', async ({ page }) => {
    // ログインフォームに入力
    await page.fill('input[name="email"]', 'alice@example.com');
    await page.fill('input[name="password"]', 'password123');
    await page.click('button[type="submit"]');

    // ダッシュボードにリダイレクト
    await expect(page).toHaveURL(/.*dashboard/);
    await expect(page.locator('h1')).toContainText('Welcome, Alice');
  });

  test('should show error for invalid credentials', async ({ page }) => {
    await page.fill('input[name="email"]', 'alice@example.com');
    await page.fill('input[name="password"]', 'wrong');
    await page.click('button[type="submit"]');

    await expect(page.locator('.error')).toContainText('Invalid credentials');
  });

  test('should handle network errors gracefully', async ({ page }) => {
    // ネットワークをオフラインに
    await page.context().setOffline(true);

    await page.fill('input[name="email"]', 'alice@example.com');
    await page.fill('input[name="password"]', 'password123');
    await page.click('button[type="submit"]');

    await expect(page.locator('.error')).toContainText('Network error');
  });
});
```

### ページオブジェクトパターン

```javascript
// pages/LoginPage.js
export class LoginPage {
  constructor(page) {
    this.page = page;
    this.emailInput = page.locator('input[name="email"]');
    this.passwordInput = page.locator('input[name="password"]');
    this.submitButton = page.locator('button[type="submit"]');
    this.errorMessage = page.locator('.error');
  }

  async goto() {
    await this.page.goto('http://localhost:3000/login');
  }

  async login(email, password) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async getErrorMessage() {
    return await this.errorMessage.textContent();
  }
}

// test/login.spec.js
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';

test('should login successfully', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login('alice@example.com', 'password123');
  await expect(page).toHaveURL(/.*dashboard/);
});
```

## テストカバレッジ

### カバレッジ目標

```bash
# 推奨カバレッジ
Statements   : 80% 以上
Branches     : 75% 以上
Functions    : 80% 以上
Lines        : 80% 以上

# ⚠️ 100% を目指さない（コスパが悪い）
# ✅ 重要なビジネスロジックは必ずカバー
```

### Jest カバレッジ

```json
// package.json
{
  "jest": {
    "collectCoverage": true,
    "collectCoverageFrom": [
      "src/**/*.{js,ts}",
      "!src/**/*.test.{js,ts}",
      "!src/**/*.spec.{js,ts}",
      "!src/**/index.{js,ts}"
    ],
    "coverageThreshold": {
      "global": {
        "branches": 75,
        "functions": 80,
        "lines": 80,
        "statements": 80
      },
      "./src/services/**/*.js": {
        "branches": 90,
        "functions": 90,
        "lines": 90,
        "statements": 90
      }
    }
  }
}
```

```bash
# カバレッジレポート生成
npm test -- --coverage

# HTML レポート
npm test -- --coverage --coverageReporters=html
open coverage/index.html
```

## TDD（Test-Driven Development）

### Red-Green-Refactor

```javascript
// 1. Red: 失敗するテストを書く
describe('Calculator', () => {
  test('should add two numbers', () => {
    const calc = new Calculator();
    expect(calc.add(2, 3)).toBe(5);  // まだ実装していないので失敗
  });
});

// 2. Green: 最小限の実装でテストを通す
class Calculator {
  add(a, b) {
    return a + b;
  }
}

// 3. Refactor: コードを改善
class Calculator {
  add(...numbers) {
    return numbers.reduce((sum, num) => sum + num, 0);
  }
}

// テストは引き続き通る
test('should add multiple numbers', () => {
  const calc = new Calculator();
  expect(calc.add(1, 2, 3, 4)).toBe(10);
});
```

## BDD（Behavior-Driven Development）

```javascript
// Cucumber/Gherkin 形式
Feature: User Login
  As a user
  I want to login to the application
  So that I can access my account

  Scenario: Successful login
    Given I am on the login page
    When I enter valid credentials
    And I click the login button
    Then I should be redirected to the dashboard
    And I should see a welcome message

  Scenario: Failed login with invalid password
    Given I am on the login page
    When I enter an invalid password
    And I click the login button
    Then I should see an error message
    And I should remain on the login page
```

```javascript
// Jest BDD スタイル
describe('User Login', () => {
  describe('when user enters valid credentials', () => {
    it('should redirect to dashboard', async () => {
      // Test implementation
    });

    it('should display welcome message', async () => {
      // Test implementation
    });
  });

  describe('when user enters invalid password', () => {
    it('should show error message', async () => {
      // Test implementation
    });

    it('should not redirect', async () => {
      // Test implementation
    });
  });
});
```

## テストデータ管理

### Factory Pattern

```javascript
// factories/userFactory.js
export const userFactory = {
  build: (overrides = {}) => ({
    id: faker.datatype.uuid(),
    name: faker.name.fullName(),
    email: faker.internet.email(),
    createdAt: new Date(),
    ...overrides
  }),

  buildMany: (count, overrides = {}) => {
    return Array.from({ length: count }, () =>
      userFactory.build(overrides)
    );
  }
};

// テストでの使用
test('should display user list', () => {
  const users = userFactory.buildMany(5);
  const result = renderUserList(users);
  expect(result).toHaveLength(5);
});

test('should handle admin user', () => {
  const admin = userFactory.build({ role: 'admin' });
  expect(isAdmin(admin)).toBe(true);
});
```

### Fixture

```javascript
// fixtures/users.json
{
  "regularUser": {
    "id": "123",
    "name": "Alice",
    "email": "alice@example.com",
    "role": "user"
  },
  "adminUser": {
    "id": "456",
    "name": "Admin",
    "email": "admin@example.com",
    "role": "admin"
  }
}

// テストでの使用
const fixtures = require('./fixtures/users.json');

test('should handle regular user', () => {
  const user = fixtures.regularUser;
  expect(user.role).toBe('user');
});
```

### Database Seeding

```javascript
// seeds/test.js
export async function seedTestData(db) {
  // ユーザー作成
  const users = await db.users.bulkCreate([
    { name: 'Alice', email: 'alice@example.com' },
    { name: 'Bob', email: 'bob@example.com' }
  ]);

  // 投稿作成
  await db.posts.bulkCreate([
    { title: 'Post 1', userId: users[0].id },
    { title: 'Post 2', userId: users[1].id }
  ]);

  return { users };
}

// テストでの使用
beforeEach(async () => {
  await db.truncate();
  await seedTestData(db);
});
```

## パフォーマンステスト

### 負荷テスト（k6）

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 20 },  // 30秒で20ユーザーまで増加
    { duration: '1m', target: 20 },   // 1分間20ユーザー維持
    { duration: '30s', target: 0 },   // 30秒で0に減少
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95%のリクエストが500ms以内
    http_req_failed: ['rate<0.01'],   // エラー率1%未満
  },
};

export default function () {
  const response = http.get('http://localhost:3000/api/users');

  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });

  sleep(1);
}
```

### ベンチマーク

```javascript
import { performance } from 'perf_hooks';

describe('Performance', () => {
  test('should process 10000 items in less than 1 second', () => {
    const items = Array.from({ length: 10000 }, (_, i) => i);

    const start = performance.now();
    const result = processItems(items);
    const end = performance.now();

    const duration = end - start;
    expect(duration).toBeLessThan(1000);
    expect(result).toHaveLength(10000);
  });
});
```

## 言語別ベストプラクティス

### JavaScript/TypeScript

```typescript
// Jest + TypeScript
describe('UserService', () => {
  let userService: UserService;
  let mockRepository: jest.Mocked<UserRepository>;

  beforeEach(() => {
    mockRepository = {
      findById: jest.fn(),
      save: jest.fn(),
    } as jest.Mocked<UserRepository>;

    userService = new UserService(mockRepository);
  });

  test('should get user by id', async () => {
    const mockUser = { id: '1', name: 'Alice' };
    mockRepository.findById.mockResolvedValue(mockUser);

    const user = await userService.getUserById('1');

    expect(user).toEqual(mockUser);
    expect(mockRepository.findById).toHaveBeenCalledWith('1');
  });
});
```

### Python

```python
# pytest
import pytest
from unittest.mock import Mock, patch

class TestUserService:
    @pytest.fixture
    def user_service(self):
        repository = Mock()
        return UserService(repository)

    def test_get_user_by_id(self, user_service):
        # Arrange
        mock_user = {"id": "1", "name": "Alice"}
        user_service.repository.find_by_id.return_value = mock_user

        # Act
        user = user_service.get_user_by_id("1")

        # Assert
        assert user == mock_user
        user_service.repository.find_by_id.assert_called_once_with("1")

    @patch('requests.get')
    def test_fetch_external_data(self, mock_get):
        # Mock external API
        mock_get.return_value.json.return_value = {"data": "value"}

        result = fetch_data()

        assert result == {"data": "value"}
```

### Go

```go
// Go testing
func TestUserService_GetUser(t *testing.T) {
    // Arrange
    mockRepo := &MockUserRepository{
        FindByIDFunc: func(id string) (*User, error) {
            return &User{ID: "1", Name: "Alice"}, nil
        },
    }
    service := NewUserService(mockRepo)

    // Act
    user, err := service.GetUser("1")

    // Assert
    assert.NoError(t, err)
    assert.Equal(t, "Alice", user.Name)
}

// Table-driven tests
func TestDivide(t *testing.T) {
    tests := []struct {
        name      string
        a, b      int
        want      int
        wantError bool
    }{
        {"positive numbers", 10, 2, 5, false},
        {"divide by zero", 10, 0, 0, true},
        {"negative result", -10, 2, -5, false},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := divide(tt.a, tt.b)
            if tt.wantError {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
                assert.Equal(t, tt.want, got)
            }
        })
    }
}
```

### Java

```java
// JUnit 5 + Mockito
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    void shouldGetUserById() {
        // Arrange
        User mockUser = new User("1", "Alice");
        when(userRepository.findById("1"))
            .thenReturn(Optional.of(mockUser));

        // Act
        User user = userService.getUserById("1");

        // Assert
        assertEquals("Alice", user.getName());
        verify(userRepository).findById("1");
    }

    @ParameterizedTest
    @CsvSource({
        "10, 2, 5",
        "-10, 2, -5",
        "0, 5, 0"
    })
    void shouldDivideNumbers(int a, int b, int expected) {
        assertEquals(expected, divide(a, b));
    }
}
```

## CI/CD統合

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
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

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run unit tests
        run: npm test -- --coverage

      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test

      - name: Run E2E tests
        run: npm run test:e2e

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info

      - name: Comment coverage on PR
        uses: romeovs/lcov-reporter-action@v0.3.1
        with:
          lcov-file: ./coverage/lcov.info
```

## チェックリスト

### テスト作成時
- [ ] AAA パターンに従っているか
- [ ] テスト名が明確か（何をテストしているか）
- [ ] 1テスト1責務
- [ ] エッジケース・異常系をカバーしているか
- [ ] テストが独立しているか（実行順序に依存しない）
- [ ] モック・スタブを適切に使用しているか

### テストコード品質
- [ ] テストコードも本番コードと同等の品質
- [ ] 重複を避ける（DRY原則）
- [ ] マジックナンバーを避ける
- [ ] 適切なアサーションメソッドを使用

### カバレッジ
- [ ] 重要なビジネスロジックをカバー
- [ ] 境界値テストを含む
- [ ] エラーハンドリングをカバー
- [ ] 目標カバレッジ達成（80%以上推奨）

### CI/CD
- [ ] すべてのテストがCIで実行される
- [ ] テスト失敗時はデプロイ中止
- [ ] カバレッジレポートが自動生成される

## よくあるアンチパターン

### ❌ 避けるべきこと

```javascript
// ❌ テスト間で状態を共有
let sharedUser; // グローバル変数

test('create user', () => {
  sharedUser = createUser();
});

test('update user', () => {
  updateUser(sharedUser); // 前のテストに依存
});

// ❌ 実装の詳細をテスト
test('should update internal state', () => {
  const component = new Component();
  component.handleClick();
  expect(component._internalState).toBe(true); // private実装に依存
});

// ❌ 過度なモック
test('should process data', () => {
  const mock1 = jest.fn();
  const mock2 = jest.fn();
  const mock3 = jest.fn();
  // すべてをモックするとテストの価値が低い
});

// ❌ 曖昧なテスト名
test('test 1', () => { /* ... */ });
test('it works', () => { /* ... */ });

// ❌ 遅いテスト
test('should process', async () => {
  await sleep(5000); // 不必要な待機
});
```

### ✅ 推奨事項

```javascript
// ✅ 各テストで準備
test('create user', () => {
  const user = createUser();
  expect(user.name).toBe('Alice');
});

test('update user', () => {
  const user = createUser();
  updateUser(user);
  expect(user.name).toBe('Updated');
});

// ✅ 公開APIをテスト
test('should toggle visibility', () => {
  const component = new Component();
  component.toggle();
  expect(component.isVisible()).toBe(true);
});

// ✅ 適度なモック
test('should save user to database', async () => {
  const mockDb = { save: jest.fn() }; // 外部依存のみモック
  await userService.createUser({ name: 'Alice' }, mockDb);
  expect(mockDb.save).toHaveBeenCalled();
});

// ✅ 明確なテスト名
test('should throw error when email is invalid', () => {
  expect(() => validateEmail('invalid')).toThrow();
});
```

## 参考リソース

- [Jest Documentation](https://jestjs.io/)
- [Playwright Documentation](https://playwright.dev/)
- [Testing Best Practices](https://github.com/goldbergyoni/javascript-testing-best-practices)
- [Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)
