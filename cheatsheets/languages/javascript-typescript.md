# JavaScript/TypeScript チートシート

## 変数・定数

```javascript
// var（使用非推奨）
var x = 10;

// let（再代入可能）
let count = 0;
count = 1;

// const（再代入不可）
const PI = 3.14159;
const user = { name: "Alice" };
user.name = "Bob";  // OK（オブジェクトの中身は変更可能）

// 分割代入
const { name, age } = user;
const [first, second] = [1, 2, 3];
const { name: userName, age = 30 } = user;  // リネーム、デフォルト値

// スプレッド演算子
const arr1 = [1, 2, 3];
const arr2 = [...arr1, 4, 5];
const obj1 = { a: 1, b: 2 };
const obj2 = { ...obj1, c: 3 };

// レスト演算子
const [first, ...rest] = [1, 2, 3, 4];
const { name, ...others } = { name: "Alice", age: 30, city: "Tokyo" };
```

## データ型

```javascript
// プリミティブ型
const str = "string";
const num = 42;
const bool = true;
const nul = null;
const undef = undefined;
const sym = Symbol("id");
const bigint = 123n;

// 型確認
typeof "hello"  // "string"
typeof 42       // "number"
typeof true     // "boolean"
typeof undefined // "undefined"
typeof null     // "object"（歴史的なバグ）
Array.isArray([])  // true

// 型変換
String(123)     // "123"
Number("123")   // 123
Boolean(1)      // true
parseInt("123") // 123
parseFloat("3.14")  // 3.14
```

## 文字列

```javascript
// テンプレートリテラル
const name = "Alice";
const message = `Hello, ${name}!`;
const multiLine = `
  Line 1
  Line 2
`;

// メソッド
str.length
str.toUpperCase()
str.toLowerCase()
str.trim()
str.split(",")
str.substring(0, 5)
str.slice(0, 5)
str.slice(-3)  // 末尾3文字
str.indexOf("text")
str.includes("text")
str.startsWith("prefix")
str.endsWith("suffix")
str.replace("old", "new")
str.replaceAll("old", "new")
str.padStart(10, "0")
str.padEnd(10, "0")
str.repeat(3)
```

## 配列

```javascript
// 作成
const arr = [1, 2, 3, 4, 5];
const arr2 = new Array(10);  // 長さ10
const arr3 = Array.from({ length: 5 }, (_, i) => i);  // [0,1,2,3,4]

// 基本操作
arr.push(6)         // 末尾追加
arr.pop()           // 末尾削除
arr.unshift(0)      // 先頭追加
arr.shift()         // 先頭削除
arr.splice(2, 1)    // インデックス2から1つ削除
arr.splice(2, 0, 99)  // インデックス2に挿入
arr.slice(1, 3)     // 部分配列（元配列は変更なし）
arr.concat([6, 7])  // 結合

// 検索
arr.indexOf(3)
arr.includes(3)
arr.find(x => x > 3)         // 条件に合う最初の要素
arr.findIndex(x => x > 3)    // インデックス

// 変換
arr.map(x => x * 2)          // [2, 4, 6, 8, 10]
arr.filter(x => x % 2 === 0) // [2, 4]
arr.reduce((sum, x) => sum + x, 0)  // 15
arr.join(", ")               // "1, 2, 3, 4, 5"
arr.reverse()                // [5, 4, 3, 2, 1]（破壊的）
arr.sort()                   // ソート（破壊的）
arr.sort((a, b) => a - b)    // 数値ソート

// その他
arr.forEach(x => console.log(x))
arr.some(x => x > 3)         // いずれかが条件満たす
arr.every(x => x > 0)        // すべてが条件満たす
arr.flat()                   // 平坦化
arr.flatMap(x => [x, x * 2]) // map + flat

// スプレッド
[...arr]                     // コピー
Math.max(...arr)             // 最大値
```

## オブジェクト

```javascript
// 作成
const obj = {
  name: "Alice",
  age: 30,
  greet() {
    return `Hello, ${this.name}`;
  },
  // 計算プロパティ
  ["computed_" + "key"]: "value"
};

// アクセス
obj.name
obj["name"]

// 操作
obj.city = "Tokyo";          // 追加
delete obj.age;              // 削除
"name" in obj;               // キー存在確認
obj.hasOwnProperty("name");

// メソッド
Object.keys(obj)             // キー配列
Object.values(obj)           // 値配列
Object.entries(obj)          // [key, value]配列
Object.assign({}, obj, { age: 31 })  // マージ
Object.freeze(obj)           // 変更不可に
Object.seal(obj)             // プロパティ追加削除不可に

// 分割代入
const { name, age } = obj;
const { name: userName } = obj;  // リネーム

// スプレッド
const newObj = { ...obj, city: "Tokyo" };
```

## 関数

```javascript
// 関数宣言
function add(a, b) {
  return a + b;
}

// 関数式
const add = function(a, b) {
  return a + b;
};

// アロー関数
const add = (a, b) => a + b;
const square = x => x * 2;
const greet = () => "Hello";
const complex = (a, b) => {
  const sum = a + b;
  return sum;
};

// デフォルト引数
const greet = (name = "Guest") => `Hello, ${name}`;

// レスト引数
const sum = (...numbers) => numbers.reduce((a, b) => a + b, 0);

// 即座実行関数（IIFE）
(function() {
  console.log("Immediately invoked");
})();

// 高階関数
const withLogging = (fn) => {
  return (...args) => {
    console.log("Calling function");
    return fn(...args);
  };
};
```

## 非同期処理

```javascript
// Promise
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("Success");
    // reject(new Error("Failed"));
  }, 1000);
});

promise
  .then(result => console.log(result))
  .catch(error => console.error(error))
  .finally(() => console.log("Done"));

// Promise.all（並列実行、すべて完了待ち）
Promise.all([promise1, promise2, promise3])
  .then(results => console.log(results));

// Promise.race（最初の1つ完了待ち）
Promise.race([promise1, promise2])
  .then(result => console.log(result));

// Promise.allSettled（すべて完了待ち、失敗も含む）
Promise.allSettled([promise1, promise2])
  .then(results => console.log(results));

// async/await
async function fetchData() {
  try {
    const response = await fetch("https://api.example.com/data");
    const data = await response.json();
    return data;
  } catch (error) {
    console.error("Error:", error);
    throw error;
  }
}

// 並列実行
async function fetchAll() {
  const [data1, data2] = await Promise.all([
    fetchData1(),
    fetchData2()
  ]);
  return { data1, data2 };
}
```

## クラス

```javascript
// 基本クラス
class Person {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }

  greet() {
    return `Hello, I'm ${this.name}`;
  }

  // ゲッター
  get info() {
    return `${this.name} (${this.age})`;
  }

  // セッター
  set age(value) {
    if (value < 0) throw new Error("Invalid age");
    this._age = value;
  }

  // 静的メソッド
  static create(name, age) {
    return new Person(name, age);
  }
}

// 継承
class Student extends Person {
  constructor(name, age, studentId) {
    super(name, age);
    this.studentId = studentId;
  }

  greet() {
    return `${super.greet()}, ID: ${this.studentId}`;
  }
}

// プライベートフィールド
class BankAccount {
  #balance = 0;  // プライベート

  deposit(amount) {
    this.#balance += amount;
  }

  getBalance() {
    return this.#balance;
  }
}
```

## モジュール（ES6）

```javascript
// エクスポート
export const PI = 3.14159;
export function add(a, b) {
  return a + b;
}
export class Calculator {}

export default class App {}

// インポート
import { PI, add } from "./math.js";
import { PI as pi } from "./math.js";  // リネーム
import * as math from "./math.js";
import App from "./App.js";  // デフォルトエクスポート
import App, { PI, add } from "./module.js";  // 混在
```

## エラー処理

```javascript
// try-catch
try {
  throw new Error("Something went wrong");
} catch (error) {
  console.error(error.message);
} finally {
  console.log("Cleanup");
}

// カスタムエラー
class CustomError extends Error {
  constructor(message) {
    super(message);
    this.name = "CustomError";
  }
}

throw new CustomError("Custom error message");
```

## よく使うメソッド

```javascript
// setTimeout/setInterval
const timeoutId = setTimeout(() => {
  console.log("Delayed");
}, 1000);
clearTimeout(timeoutId);

const intervalId = setInterval(() => {
  console.log("Repeated");
}, 1000);
clearInterval(intervalId);

// JSON
JSON.stringify({ name: "Alice", age: 30 });
JSON.parse('{"name":"Alice","age":30}');

// Math
Math.random()           // 0〜1
Math.floor(3.7)         // 3
Math.ceil(3.2)          // 4
Math.round(3.5)         // 4
Math.max(1, 2, 3)       // 3
Math.min(1, 2, 3)       // 1
Math.abs(-5)            // 5
Math.pow(2, 3)          // 8
Math.sqrt(16)           // 4

// Date
new Date()
new Date("2024-01-01")
new Date(2024, 0, 1)  // 月は0始まり
date.getFullYear()
date.getMonth()       // 0-11
date.getDate()        // 1-31
date.getDay()         // 0(日)〜6(土)
date.getHours()
date.getMinutes()
date.getSeconds()
date.toISOString()
date.toLocaleDateString()
```

## TypeScript 基本

```typescript
// 型アノテーション
let name: string = "Alice";
let age: number = 30;
let isActive: boolean = true;
let numbers: number[] = [1, 2, 3];
let tuple: [string, number] = ["Alice", 30];

// オブジェクト型
let user: { name: string; age: number } = {
  name: "Alice",
  age: 30
};

// 配列型
let numbers: Array<number> = [1, 2, 3];
let matrix: number[][] = [[1, 2], [3, 4]];

// 関数型
function add(a: number, b: number): number {
  return a + b;
}

const greet = (name: string): string => `Hello, ${name}`;

// オプショナル・デフォルト
function greet(name: string, greeting?: string): string {
  return `${greeting || "Hello"}, ${name}`;
}

// Union型
let id: string | number = "123";
id = 123;

// 型エイリアス
type Point = { x: number; y: number };
type ID = string | number;

// Interface
interface User {
  name: string;
  age: number;
  email?: string;  // オプショナル
  readonly id: string;  // 読み取り専用
}

interface Admin extends User {
  role: string;
}

// Generics
function identity<T>(arg: T): T {
  return arg;
}

const numbers = identity<number>(42);
const text = identity<string>("hello");

// 配列のGenerics
function getFirst<T>(arr: T[]): T | undefined {
  return arr[0];
}

// ジェネリック制約
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// Utility Types
type Partial<T> = { [P in keyof T]?: T[P] };  // すべてオプショナル
type Required<T> = { [P in keyof T]-?: T[P] };  // すべて必須
type Readonly<T> = { readonly [P in keyof T]: T[P] };  // すべて読み取り専用
type Pick<T, K extends keyof T> = { [P in K]: T[P] };  // 特定プロパティ抽出
type Omit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;  // 特定プロパティ除外

// よく使うUtility Types
interface User {
  name: string;
  age: number;
  email: string;
}

type PartialUser = Partial<User>;  // すべてオプショナル
type UserNameAge = Pick<User, "name" | "age">;
type UserWithoutEmail = Omit<User, "email">;
type ReadonlyUser = Readonly<User>;

// 型ガード
function isString(value: any): value is string {
  return typeof value === "string";
}

if (isString(value)) {
  console.log(value.toUpperCase());  // stringとして扱える
}

// as const
const colors = ["red", "green", "blue"] as const;
type Color = typeof colors[number];  // "red" | "green" | "blue"

// Enum
enum Status {
  Pending,
  Active,
  Inactive
}

enum Role {
  Admin = "ADMIN",
  User = "USER",
  Guest = "GUEST"
}

// Type assertion
const input = document.getElementById("input") as HTMLInputElement;
const value = someValue as string;
```

## DOM操作

```javascript
// 要素取得
document.getElementById("id")
document.querySelector(".class")
document.querySelectorAll("div")
document.getElementsByClassName("class")
document.getElementsByTagName("div")

// 要素作成・操作
const div = document.createElement("div");
div.textContent = "Hello";
div.innerHTML = "<strong>Hello</strong>";
div.className = "container";
div.classList.add("active");
div.classList.remove("hidden");
div.classList.toggle("visible");
div.setAttribute("data-id", "123");
div.getAttribute("data-id");
parent.appendChild(div);
parent.removeChild(div);
element.remove();

// イベント
element.addEventListener("click", (event) => {
  console.log(event.target);
});

element.removeEventListener("click", handler);

// よく使うイベント
// click, dblclick, mousedown, mouseup, mousemove
// keydown, keyup, keypress
// submit, change, input, focus, blur
// load, DOMContentLoaded, resize, scroll
```

## Fetch API

```javascript
// GET
fetch("https://api.example.com/data")
  .then(response => response.json())
  .then(data => console.log(data))
  .catch(error => console.error(error));

// async/await
async function fetchData() {
  try {
    const response = await fetch("https://api.example.com/data");
    if (!response.ok) throw new Error("HTTP error");
    const data = await response.json();
    return data;
  } catch (error) {
    console.error(error);
  }
}

// POST
fetch("https://api.example.com/data", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": "Bearer token"
  },
  body: JSON.stringify({ name: "Alice", age: 30 })
})
  .then(response => response.json())
  .then(data => console.log(data));

// その他のメソッド
fetch(url, { method: "PUT", body: JSON.stringify(data) });
fetch(url, { method: "DELETE" });
fetch(url, { method: "PATCH", body: JSON.stringify(data) });
```

## よく使うパターン

```javascript
// デバウンス
function debounce(func, delay) {
  let timeoutId;
  return (...args) => {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => func(...args), delay);
  };
}

// スロットル
function throttle(func, limit) {
  let inThrottle;
  return (...args) => {
    if (!inThrottle) {
      func(...args);
      inThrottle = true;
      setTimeout(() => inThrottle = false, limit);
    }
  };
}

// ディープコピー
const deepCopy = obj => JSON.parse(JSON.stringify(obj));
const deepCopy2 = obj => structuredClone(obj);  // モダンブラウザ

// 配列の重複削除
const unique = arr => [...new Set(arr)];

// オブジェクトの配列から特定フィールド抽出
const names = users.map(user => user.name);

// グルーピング
const grouped = items.reduce((acc, item) => {
  const key = item.category;
  if (!acc[key]) acc[key] = [];
  acc[key].push(item);
  return acc;
}, {});

// null/undefined チェック
const value = obj?.property?.nested;  // Optional chaining
const name = user?.name ?? "Guest";   // Nullish coalescing

// 条件付きオブジェクトプロパティ
const obj = {
  name: "Alice",
  ...(age && { age }),
  ...(city && { city })
};
```

## Node.js特有

```javascript
// モジュール（CommonJS）
const fs = require("fs");
const path = require("path");

module.exports = { add, subtract };
module.exports.multiply = (a, b) => a * b;

// ES Modules（package.jsonで"type": "module"が必要）
import fs from "fs/promises";
import path from "path";

export { add, subtract };
export const multiply = (a, b) => a * b;

// ファイル操作
const fs = require("fs/promises");

await fs.readFile("file.txt", "utf-8");
await fs.writeFile("file.txt", "content");
await fs.appendFile("file.txt", "more content");
await fs.unlink("file.txt");
await fs.mkdir("dir", { recursive: true });

// パス
const path = require("path");
path.join(__dirname, "file.txt");
path.resolve("file.txt");
path.basename("/path/to/file.txt");
path.dirname("/path/to/file.txt");
path.extname("file.txt");

// 環境変数
process.env.NODE_ENV
process.env.API_KEY

// プロセス
process.argv  // コマンドライン引数
process.cwd() // 現在のディレクトリ
process.exit(0)
```
