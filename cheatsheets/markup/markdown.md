# Markdown チートシート

## 見出し

```markdown
# 見出し1
## 見出し2
### 見出し3
#### 見出し4
##### 見出し5
###### 見出し6

<!-- 別の書き方（見出し1と2のみ） -->
見出し1
=======

見出し2
-------
```

# 見出し1
## 見出し2
### 見出し3

## 段落と改行

```markdown
これは段落です。
空行で段落を分けます。

これは次の段落です。

行末に2つのスペースを入れると
改行されます。

<!-- または<br>タグ -->
改行<br>されます。
```

## 強調

```markdown
*イタリック*
_イタリック_

**太字**
__太字__

***太字イタリック***
___太字イタリック___

~~取り消し線~~
```

*イタリック*
**太字**
***太字イタリック***
~~取り消し線~~

## リスト

### 箇条書き

```markdown
- アイテム1
- アイテム2
- アイテム3
  - サブアイテム
  - サブアイテム

* アイテム1
* アイテム2

+ アイテム1
+ アイテム2
```

- アイテム1
- アイテム2
  - サブアイテム

### 番号付きリスト

```markdown
1. 最初
2. 次
3. 最後

1. 最初
1. 次（自動採番）
1. 最後
```

1. 最初
2. 次
3. 最後

### タスクリスト

```markdown
- [x] 完了したタスク
- [ ] 未完了のタスク
- [ ] もう1つのタスク
```

- [x] 完了したタスク
- [ ] 未完了のタスク

## リンク

```markdown
[リンクテキスト](https://example.com)
[リンクテキスト](https://example.com "タイトル")

<!-- 参照スタイル -->
[リンクテキスト][1]
[リンクテキスト][link-ref]

[1]: https://example.com
[link-ref]: https://example.com "タイトル"

<!-- URL自動リンク -->
<https://example.com>
<email@example.com>

<!-- ページ内リンク -->
[見出しへのリンク](#見出し)
```

[Google](https://google.com)

## 画像

```markdown
![代替テキスト](image.jpg)
![代替テキスト](image.jpg "タイトル")
![代替テキスト](https://example.com/image.jpg)

<!-- 参照スタイル -->
![代替テキスト][image-ref]

[image-ref]: image.jpg "タイトル"

<!-- リンク付き画像 -->
[![代替テキスト](image.jpg)](https://example.com)

<!-- HTMLタグでサイズ指定 -->
<img src="image.jpg" width="200" height="100">
```

## コード

### インラインコード

```markdown
`インラインコード`

`const x = 10;`
```

`インラインコード`

### コードブロック

````markdown
```
コードブロック
複数行
```

```javascript
const greeting = "Hello, World!";
console.log(greeting);
```

```python
def hello():
    print("Hello, World!")
```

<!-- インデント方式（4スペースまたはタブ） -->
    インデントでも
    コードブロックになります
````

```javascript
const greeting = "Hello, World!";
console.log(greeting);
```

## 引用

```markdown
> これは引用です。
> 複数行にわたる
> 引用もできます。

> 引用1
>
> > ネストした引用
>
> 引用1に戻る
```

> これは引用です。
> 複数行にわたる引用もできます。

## 水平線

```markdown
---

***

___

<!-- 3つ以上の記号 -->
```

---

## テーブル

```markdown
| ヘッダー1 | ヘッダー2 | ヘッダー3 |
| --------- | --------- | --------- |
| セル1     | セル2     | セル3     |
| セル4     | セル5     | セル6     |

<!-- 整列 -->
| 左寄せ | 中央寄せ | 右寄せ |
| :--- | :---: | ---: |
| Left | Center | Right |
| L | C | R |

<!-- パイプは揃えなくてもOK -->
| ヘッダー1 | ヘッダー2 |
| --- | --- |
| A | B |
| C | D |
```

| ヘッダー1 | ヘッダー2 | ヘッダー3 |
| --------- | --------- | --------- |
| セル1     | セル2     | セル3     |
| セル4     | セル5     | セル6     |

| 左寄せ | 中央寄せ | 右寄せ |
| :--- | :---: | ---: |
| Left | Center | Right |

## HTML

```markdown
<!-- Markdownの中でHTMLタグも使える -->

<div style="color: red;">
  赤い文字
</div>

<details>
<summary>クリックして展開</summary>

この部分は折りたたまれています。

</details>

<kbd>Ctrl</kbd> + <kbd>C</kbd>

<mark>ハイライト</mark>
```

<details>
<summary>クリックして展開</summary>

この部分は折りたたまれています。

</details>

## エスケープ

```markdown
<!-- バックスラッシュでエスケープ -->
\* アスタリスクをそのまま表示

\# ハッシュをそのまま表示

\[リンクにしない\](url)
```

\* アスタリスクをそのまま表示

## GitHub Flavored Markdown (GFM)

### シンタックスハイライト

````markdown
```javascript
function hello() {
  console.log("Hello");
}
```

```python
def hello():
    print("Hello")
```

```bash
echo "Hello"
```

```diff
- 削除行
+ 追加行
```
````

### タスクリスト

```markdown
- [x] 完了
- [ ] 未完了
```

### テーブル

```markdown
| Column 1 | Column 2 |
| -------- | -------- |
| Data 1   | Data 2   |
```

### 絵文字

```markdown
:smile: :heart: :+1:
:rocket: :fire: :sparkles:
```

:smile: :heart: :+1:

### メンション・参照

```markdown
@username
#123（Issue番号）
SHA: 1234567
```

### 自動リンク

```markdown
https://github.com
```

https://github.com

## 数式（一部のプラットフォーム）

```markdown
<!-- インライン数式 -->
$E = mc^2$

<!-- ブロック数式 -->
$$
\int_{a}^{b} f(x) dx
$$
```

## 脚注

```markdown
テキスト[^1]

[^1]: 脚注の内容
```

## 定義リスト

```markdown
Term 1
: Definition 1

Term 2
: Definition 2a
: Definition 2b
```

## よく使うパターン

### README.mdテンプレート

```markdown
# プロジェクト名

プロジェクトの簡単な説明

## 特徴

- 特徴1
- 特徴2
- 特徴3

## インストール

\```bash
npm install project-name
\```

## 使い方

\```javascript
const lib = require('project-name');
lib.doSomething();
\```

## API

### `function()`

関数の説明

**引数:**
- `param1` (string): 説明
- `param2` (number): 説明

**戻り値:**
- (boolean): 説明

## ライセンス

MIT
```

### バッジ

```markdown
![Build Status](https://img.shields.io/badge/build-passing-brightgreen)
![Version](https://img.shields.io/badge/version-1.0.0-blue)
![License](https://img.shields.io/badge/license-MIT-green)

<!-- shields.ioで生成 -->
[![Build Status](https://travis-ci.org/user/repo.svg?branch=master)](https://travis-ci.org/user/repo)
```

### 折りたたみ

```markdown
<details>
<summary>詳細を見る</summary>

ここに詳細な内容を書きます。

\```javascript
const code = "example";
\```

</details>
```

### 警告・注意

```markdown
> **Warning**
> 警告メッセージ

> **Note**
> 注意メッセージ

> [!WARNING]
> GitHub形式の警告

> [!NOTE]
> GitHub形式のノート

> [!IMPORTANT]
> 重要な情報
```

### 目次

```markdown
## 目次

- [セクション1](#セクション1)
- [セクション2](#セクション2)
  - [サブセクション2.1](#サブセクション21)
- [セクション3](#セクション3)

## セクション1

内容

## セクション2

内容

### サブセクション2.1

内容

## セクション3

内容
```

## Tips

```markdown
<!-- コメント（表示されない） -->

<!-- 改行したい場合は行末に2スペース、またはHTMLの<br>タグ -->
テキスト
次の行

テキスト<br>次の行

<!-- URLやメールアドレスは<>で囲むと自動リンク -->
<https://example.com>
<email@example.com>

<!-- バックスラッシュでエスケープ -->
\* \[ \] \# \> \- \\

<!-- ネストしたリスト（2スペースまたは4スペースでインデント） -->
- アイテム1
  - サブアイテム1
    - サブサブアイテム1

<!-- コードブロック内でMarkdownを表示 -->
\```markdown
# 見出し
\```

<!-- テーブルのセル内で改行 -->
| Column |
| ------ |
| Line1<br>Line2 |
```

## プラットフォーム別の違い

### GitHub

- タスクリスト対応
- 絵文字対応
- テーブル対応
- シンタックスハイライト対応
- 警告ブロック対応

### GitLab

- GitHubとほぼ同様
- 数式対応（LaTeX）
- Mermaid図対応

### VSCode

- プレビュー機能
- プラグインで拡張可能

### Slack

- 簡易版Markdown
- コードブロック対応
- リスト対応
- 強調対応（一部制限あり）

## オンラインエディタ・ツール

- [Dillinger](https://dillinger.io/) - オンラインMarkdownエディタ
- [StackEdit](https://stackedit.io/) - オンラインMarkdownエディタ
- [Markdownlint](https://github.com/DavidAnson/markdownlint) - リンター
- [Prettier](https://prettier.io/) - フォーマッター
- [Shields.io](https://shields.io/) - バッジ生成

## エディタプラグイン

### VSCode

- Markdown All in One
- Markdown Preview Enhanced
- markdownlint

### Vim

- vim-markdown
- markdown-preview.nvim

### Atom

- markdown-preview-plus
- markdown-writer
