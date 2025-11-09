# Python チートシート

## 基本構文

### 変数・型

```python
# 変数宣言（型推論）
name = "Alice"
age = 30
height = 1.65
is_active = True

# 型ヒント（Python 3.5+）
name: str = "Alice"
age: int = 30
numbers: list[int] = [1, 2, 3]
data: dict[str, int] = {"a": 1, "b": 2}

# 型確認
type(name)  # <class 'str'>
isinstance(age, int)  # True

# 型変換
int("123")
str(123)
float("3.14")
list("abc")  # ['a', 'b', 'c']
```

### 文字列

```python
# 文字列操作
s = "Hello, World!"
s.lower()           # 小文字化
s.upper()           # 大文字化
s.strip()           # 前後の空白削除
s.split(",")        # 分割
",".join(["a", "b"])  # 結合
s.replace("World", "Python")
s.startswith("Hello")
s.endswith("!")
s.find("World")     # 位置検索（-1: 見つからず）
s.count("l")        # 出現回数

# フォーマット
name = "Alice"
age = 30
f"Name: {name}, Age: {age}"  # f-string（推奨）
"Name: {}, Age: {}".format(name, age)
"Name: {name}, Age: {age}".format(name=name, age=age)

# 複数行
text = """
Line 1
Line 2
Line 3
"""

# raw文字列（エスケープなし）
path = r"C:\Users\name\file.txt"
```

### リスト

```python
# 作成
numbers = [1, 2, 3, 4, 5]
mixed = [1, "two", 3.0, True]
empty = []

# 操作
numbers.append(6)       # 末尾追加
numbers.insert(0, 0)    # 位置指定追加
numbers.extend([7, 8])  # 複数追加
numbers.remove(3)       # 値で削除
numbers.pop()           # 末尾削除して返す
numbers.pop(0)          # 位置指定削除
numbers.clear()         # すべて削除
numbers.sort()          # ソート
numbers.reverse()       # 反転
numbers.index(3)        # 検索
numbers.count(3)        # 出現回数
len(numbers)            # 長さ

# スライス
numbers[1:4]    # インデックス1〜3
numbers[:3]     # 先頭から3つ
numbers[3:]     # インデックス3以降
numbers[-1]     # 末尾
numbers[-3:]    # 末尾3つ
numbers[::2]    # 2つおき
numbers[::-1]   # 反転

# リスト内包表記
squares = [x**2 for x in range(10)]
evens = [x for x in range(10) if x % 2 == 0]
matrix = [[i+j for j in range(3)] for i in range(3)]
```

### 辞書

```python
# 作成
person = {"name": "Alice", "age": 30}
empty = {}
from_keys = dict.fromkeys(["a", "b"], 0)  # {"a": 0, "b": 0}

# 操作
person["name"]          # 取得
person.get("name")      # 取得（存在しない場合None）
person.get("city", "Unknown")  # デフォルト値
person["city"] = "Tokyo"  # 追加・更新
person.update({"age": 31, "city": "Osaka"})
del person["age"]       # 削除
person.pop("name")      # 削除して返す
person.keys()           # キー一覧
person.values()         # 値一覧
person.items()          # (key, value)一覧
"name" in person        # キー存在チェック

# 辞書内包表記
squares = {x: x**2 for x in range(5)}
filtered = {k: v for k, v in person.items() if v > 30}
```

### セット

```python
# 作成
numbers = {1, 2, 3, 4, 5}
empty = set()

# 操作
numbers.add(6)
numbers.remove(3)       # KeyError if not exists
numbers.discard(3)      # エラーにならない
numbers.pop()           # ランダムに削除
numbers.clear()

# 集合演算
a = {1, 2, 3}
b = {3, 4, 5}
a | b           # 和集合
a & b           # 積集合
a - b           # 差集合
a ^ b           # 対称差

# セット内包表記
squares = {x**2 for x in range(10)}
```

### タプル

```python
# 作成（不変）
point = (10, 20)
single = (1,)  # カンマ必須
empty = ()

# アンパック
x, y = point
first, *rest = [1, 2, 3, 4]  # first=1, rest=[2,3,4]

# 名前付きタプル
from collections import namedtuple
Point = namedtuple('Point', ['x', 'y'])
p = Point(10, 20)
p.x, p.y
```

## 制御構文

### 条件分岐

```python
# if-elif-else
if age < 18:
    print("Minor")
elif age < 65:
    print("Adult")
else:
    print("Senior")

# 三項演算子
status = "Adult" if age >= 18 else "Minor"

# match-case（Python 3.10+）
match status:
    case "active":
        print("Active user")
    case "inactive":
        print("Inactive user")
    case _:
        print("Unknown status")
```

### ループ

```python
# for文
for i in range(5):
    print(i)

for i in range(1, 10, 2):  # start, stop, step
    print(i)

for item in [1, 2, 3]:
    print(item)

for key, value in {"a": 1, "b": 2}.items():
    print(f"{key}: {value}")

for i, value in enumerate(["a", "b", "c"]):
    print(f"{i}: {value}")

# while文
count = 0
while count < 5:
    print(count)
    count += 1

# break, continue
for i in range(10):
    if i == 3:
        continue
    if i == 7:
        break
    print(i)

# else句（breakしなかった場合）
for i in range(5):
    if i == 10:
        break
else:
    print("Completed")
```

## 関数

```python
# 基本
def greet(name):
    return f"Hello, {name}!"

# デフォルト引数
def greet(name, greeting="Hello"):
    return f"{greeting}, {name}!"

# 可変長引数
def sum_all(*args):
    return sum(args)

def print_info(**kwargs):
    for key, value in kwargs.items():
        print(f"{key}: {value}")

# 型ヒント
def add(a: int, b: int) -> int:
    return a + b

# ラムダ式
square = lambda x: x ** 2
add = lambda x, y: x + y

# デコレータ
def timer(func):
    import time
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"Time: {time.time() - start:.2f}s")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)
```

## クラス

```python
# 基本クラス
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def greet(self):
        return f"Hello, I'm {self.name}"

    def __str__(self):
        return f"Person({self.name}, {self.age})"

    def __repr__(self):
        return f"Person(name='{self.name}', age={self.age})"

# 使用
person = Person("Alice", 30)
person.greet()

# プロパティ
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("Radius must be positive")
        self._radius = value

    @property
    def area(self):
        return 3.14159 * self._radius ** 2

# 継承
class Student(Person):
    def __init__(self, name, age, student_id):
        super().__init__(name, age)
        self.student_id = student_id

    def greet(self):
        return f"Hi, I'm {self.name}, student ID: {self.student_id}"

# データクラス（Python 3.7+）
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

    def distance(self):
        return (self.x**2 + self.y**2)**0.5
```

## ファイル操作

```python
# ファイル読み込み
with open("file.txt", "r") as f:
    content = f.read()

with open("file.txt", "r") as f:
    lines = f.readlines()

with open("file.txt", "r") as f:
    for line in f:
        print(line.strip())

# ファイル書き込み
with open("file.txt", "w") as f:
    f.write("Hello\n")

with open("file.txt", "a") as f:  # 追記
    f.write("World\n")

# JSON
import json

data = {"name": "Alice", "age": 30}
with open("data.json", "w") as f:
    json.dump(data, f, indent=2)

with open("data.json", "r") as f:
    data = json.load(f)

json.dumps(data)  # 文字列化
json.loads(json_string)  # パース

# CSV
import csv

with open("data.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["Name", "Age"])
    writer.writerows([["Alice", 30], ["Bob", 25]])

with open("data.csv", "r") as f:
    reader = csv.reader(f)
    for row in reader:
        print(row)

# DictWriter/DictReader
with open("data.csv", "w", newline="") as f:
    fieldnames = ["name", "age"]
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerow({"name": "Alice", "age": 30})

# パス操作
from pathlib import Path

path = Path("folder/file.txt")
path.exists()
path.is_file()
path.is_dir()
path.parent
path.name
path.stem  # 拡張子なし
path.suffix  # 拡張子
path.mkdir(parents=True, exist_ok=True)
path.read_text()
path.write_text("content")
list(path.glob("*.txt"))
list(path.rglob("*.py"))  # 再帰的
```

## 例外処理

```python
# 基本
try:
    result = 10 / 0
except ZeroDivisionError:
    print("Division by zero!")
except Exception as e:
    print(f"Error: {e}")
else:
    print("Success")
finally:
    print("Cleanup")

# 複数の例外
try:
    # code
except (ValueError, TypeError) as e:
    print(f"Error: {e}")

# 例外発生
raise ValueError("Invalid value")
raise Exception("Something went wrong")

# カスタム例外
class CustomError(Exception):
    pass

raise CustomError("Custom error message")
```

## 標準ライブラリ

### datetime

```python
from datetime import datetime, date, time, timedelta

# 現在
now = datetime.now()
today = date.today()

# 作成
dt = datetime(2024, 1, 1, 12, 30, 0)
d = date(2024, 1, 1)

# フォーマット
now.strftime("%Y-%m-%d %H:%M:%S")
datetime.strptime("2024-01-01", "%Y-%m-%d")

# 計算
tomorrow = today + timedelta(days=1)
week_ago = now - timedelta(weeks=1)

# タイムゾーン
from datetime import timezone
utc_now = datetime.now(timezone.utc)
```

### collections

```python
from collections import Counter, defaultdict, deque, OrderedDict

# Counter
counter = Counter([1, 2, 2, 3, 3, 3])
counter.most_common(2)  # [(3, 3), (2, 2)]

# defaultdict
dd = defaultdict(list)
dd['key'].append('value')  # KeyErrorにならない

# deque（両端キュー）
dq = deque([1, 2, 3])
dq.append(4)      # 右端追加
dq.appendleft(0)  # 左端追加
dq.pop()          # 右端削除
dq.popleft()      # 左端削除
```

### itertools

```python
from itertools import (
    count, cycle, repeat,
    chain, combinations, permutations,
    groupby, islice
)

# 無限イテレータ
count(10)  # 10, 11, 12, ...
cycle([1, 2, 3])  # 1, 2, 3, 1, 2, 3, ...
repeat(10, 3)  # 10, 10, 10

# 組み合わせ
list(combinations([1, 2, 3], 2))  # [(1,2), (1,3), (2,3)]
list(permutations([1, 2, 3], 2))  # [(1,2), (1,3), (2,1), ...]

# chain（連結）
list(chain([1, 2], [3, 4]))  # [1, 2, 3, 4]

# groupby
data = [1, 1, 2, 2, 3, 3]
for key, group in groupby(data):
    print(key, list(group))
```

### os / pathlib

```python
import os
from pathlib import Path

# 環境変数
os.environ.get("HOME")
os.getenv("PATH")

# ディレクトリ操作
os.getcwd()
os.chdir("/path/to/dir")
os.listdir(".")
os.mkdir("new_dir")
os.makedirs("path/to/dir", exist_ok=True)
os.remove("file.txt")
os.rmdir("dir")

# Path（推奨）
cwd = Path.cwd()
home = Path.home()
for p in Path(".").iterdir():
    print(p)
```

### subprocess

```python
import subprocess

# 実行
result = subprocess.run(["ls", "-la"], capture_output=True, text=True)
print(result.stdout)
print(result.returncode)

# シェル実行（セキュリティ注意）
subprocess.run("ls -la", shell=True)

# パイプ
result = subprocess.run(
    ["grep", "pattern", "file.txt"],
    capture_output=True,
    text=True
)
```

### re（正規表現）

```python
import re

# マッチ
re.match(r"^Hello", "Hello World")  # 先頭のみ
re.search(r"World", "Hello World")  # 全体から検索
re.findall(r"\d+", "abc123def456")  # ['123', '456']
re.finditer(r"\d+", "abc123def456")  # イテレータ

# 置換
re.sub(r"\d+", "X", "abc123def456")  # 'abcXdefX'

# 分割
re.split(r"[,;]", "a,b;c")  # ['a', 'b', 'c']

# グループ
match = re.search(r"(\w+)@(\w+\.\w+)", "user@example.com")
match.group(0)  # 全体
match.group(1)  # 'user'
match.group(2)  # 'example.com'

# コンパイル（繰り返し使う場合）
pattern = re.compile(r"\d+")
pattern.findall("abc123def456")
```

## よく使うイディオム

```python
# リスト連結
list1 + list2
[*list1, *list2]

# 辞書マージ
{**dict1, **dict2}  # Python 3.5+
dict1 | dict2       # Python 3.9+

# 条件付きリスト追加
items = []
if condition:
    items.append(value)
# または
items = [value] if condition else []

# デフォルト辞書値
value = dict.get(key, default)
value = dict.setdefault(key, default)  # なければ設定

# ファイル存在チェック
from pathlib import Path
if Path("file.txt").exists():
    # ...

# リスト平坦化
nested = [[1, 2], [3, 4]]
flat = [item for sublist in nested for item in sublist]

# 辞書の値でソート
sorted(dict.items(), key=lambda x: x[1])

# 複数変数のスワップ
a, b = b, a

# enumerate with start
for i, value in enumerate(items, start=1):
    print(f"{i}. {value}")

# zip（複数リストを並列処理）
for name, age in zip(names, ages):
    print(f"{name}: {age}")

# any/all
any([False, True, False])  # True
all([True, True, True])    # True

# filter/map
list(filter(lambda x: x > 0, [-1, 0, 1, 2]))  # [1, 2]
list(map(lambda x: x**2, [1, 2, 3]))          # [1, 4, 9]
```

## 仮想環境・パッケージ管理

```bash
# venv
python -m venv venv
source venv/bin/activate  # Linux/Mac
venv\Scripts\activate     # Windows
deactivate

# pip
pip install package
pip install -r requirements.txt
pip freeze > requirements.txt
pip list
pip show package
pip uninstall package

# requirements.txt例
requests==2.31.0
pandas>=1.5.0
numpy~=1.24.0
```

## デバッグ

```python
# print デバッグ
print(f"{variable=}")  # Python 3.8+

# pdb（デバッガ）
import pdb; pdb.set_trace()  # ブレークポイント
breakpoint()  # Python 3.7+

# ロギング
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

logger.debug("Debug message")
logger.info("Info message")
logger.warning("Warning message")
logger.error("Error message")
logger.critical("Critical message")
```
