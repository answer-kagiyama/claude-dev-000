# Shell (Bash) チートシート

## 基本コマンド

### ファイル・ディレクトリ操作

```bash
# ディレクトリ移動
cd /path/to/directory
cd ~                  # ホームディレクトリ
cd -                  # 前のディレクトリ
cd ..                 # 親ディレクトリ

# ファイル一覧
ls
ls -la                # 詳細表示（隠しファイル含む）
ls -lh                # 人間が読みやすいサイズ表示
ls -lt                # 更新日時順
ls -lS                # サイズ順

# ディレクトリ作成
mkdir directory
mkdir -p parent/child/grandchild  # 親ディレクトリも作成

# ファイル作成
touch file.txt
touch file{1..5}.txt  # file1.txt, file2.txt, ...

# コピー
cp source.txt dest.txt
cp -r source_dir/ dest_dir/  # ディレクトリごと
cp -p file.txt backup.txt    # 属性保持

# 移動・名前変更
mv old.txt new.txt
mv file.txt /path/to/destination/

# 削除
rm file.txt
rm -r directory/      # ディレクトリごと
rm -f file.txt        # 確認なし
rm -rf directory/     # 強制削除（危険）

# シンボリックリンク
ln -s /path/to/target link_name
```

### ファイル内容表示

```bash
# 全体表示
cat file.txt
cat file1.txt file2.txt  # 複数ファイル連結

# ページング
less file.txt
more file.txt

# 先頭・末尾
head file.txt         # 先頭10行
head -n 20 file.txt   # 先頭20行
tail file.txt         # 末尾10行
tail -n 20 file.txt   # 末尾20行
tail -f log.txt       # リアルタイム監視

# 行数・文字数
wc file.txt           # 行数、単語数、バイト数
wc -l file.txt        # 行数のみ
wc -w file.txt        # 単語数のみ
```

### 検索

```bash
# ファイル検索
find . -name "*.txt"
find . -type f -name "*.log"
find . -type d -name "config"
find . -mtime -7      # 7日以内に更新
find . -size +100M    # 100MB以上
find . -name "*.tmp" -delete  # 削除

# テキスト検索
grep "pattern" file.txt
grep -r "pattern" .   # 再帰的検索
grep -i "pattern" file.txt  # 大文字小文字無視
grep -n "pattern" file.txt  # 行番号表示
grep -v "pattern" file.txt  # マッチしない行
grep -E "regex" file.txt    # 正規表現
grep -A 3 "pattern" file.txt  # マッチ後3行
grep -B 3 "pattern" file.txt  # マッチ前3行
grep -C 3 "pattern" file.txt  # 前後3行

# 高速検索（ripgrep）
rg "pattern"
rg -i "pattern"       # 大文字小文字無視
rg -t py "pattern"    # Pythonファイルのみ
```

### パーミッション

```bash
# 表示
ls -l file.txt

# 変更
chmod 644 file.txt    # rw-r--r--
chmod 755 file.txt    # rwxr-xr-x
chmod +x script.sh    # 実行権限追加
chmod -R 755 dir/     # 再帰的

# 所有者変更
chown user:group file.txt
chown -R user:group dir/
```

## パイプとリダイレクト

```bash
# リダイレクト
command > file.txt    # 上書き
command >> file.txt   # 追記
command 2> error.log  # エラーのみ
command &> all.log    # 標準出力とエラー両方
command 2>&1          # エラーを標準出力にマージ

# パイプ
cat file.txt | grep "pattern"
ls -l | wc -l
cat access.log | grep "ERROR" | wc -l

# tee（表示＆ファイル出力）
command | tee output.txt
command | tee -a output.txt  # 追記
```

## テキスト処理

### sed（置換・編集）

```bash
# 置換
sed 's/old/new/' file.txt           # 各行の最初のみ
sed 's/old/new/g' file.txt          # 全て置換
sed -i 's/old/new/g' file.txt       # ファイルを直接編集
sed -i.bak 's/old/new/g' file.txt   # バックアップ作成

# 行削除
sed '3d' file.txt                   # 3行目削除
sed '/pattern/d' file.txt           # パターンマッチ行削除

# 行抽出
sed -n '10,20p' file.txt            # 10〜20行目のみ表示
```

### awk（列処理）

```bash
# 列抽出
awk '{print $1}' file.txt           # 1列目
awk '{print $1, $3}' file.txt       # 1列目と3列目
awk -F: '{print $1}' /etc/passwd    # 区切り文字指定

# 条件抽出
awk '$3 > 100' file.txt             # 3列目が100より大きい
awk '/pattern/ {print $1}' file.txt # パターンマッチ

# 集計
awk '{sum += $1} END {print sum}' file.txt  # 1列目の合計
awk '{print NR, $0}' file.txt       # 行番号付き
```

### sort・uniq

```bash
# ソート
sort file.txt
sort -r file.txt      # 逆順
sort -n file.txt      # 数値順
sort -k2 file.txt     # 2列目でソート
sort -u file.txt      # 重複削除

# 重複処理
uniq file.txt         # 連続する重複削除（要事前sort）
uniq -c file.txt      # カウント付き
uniq -d file.txt      # 重複行のみ
```

### cut・paste

```bash
# 列抽出
cut -d: -f1 /etc/passwd             # :区切りで1列目
cut -c1-10 file.txt                 # 1〜10文字目

# 結合
paste file1.txt file2.txt           # 横に結合
paste -d, file1.txt file2.txt       # カンマ区切り
```

## プロセス管理

```bash
# プロセス一覧
ps aux
ps aux | grep process_name
ps -ef

# リアルタイム監視
top
htop

# プロセス終了
kill PID
kill -9 PID           # 強制終了
killall process_name
pkill -f pattern

# バックグラウンド実行
command &
nohup command &       # ログアウト後も継続

# ジョブ管理
jobs                  # ジョブ一覧
fg %1                 # フォアグラウンドに
bg %1                 # バックグラウンドで再開
```

## ネットワーク

```bash
# 接続確認
ping google.com
ping -c 4 google.com  # 4回のみ

# HTTP通信
curl https://api.example.com
curl -X POST -H "Content-Type: application/json" -d '{"key":"value"}' https://api.example.com
curl -o file.zip https://example.com/file.zip  # ダウンロード
curl -I https://example.com  # ヘッダーのみ

wget https://example.com/file.zip
wget -c https://example.com/file.zip  # 再開

# ポート確認
netstat -tuln
ss -tuln
lsof -i :8080         # 8080番ポート使用プロセス

# DNS
nslookup example.com
dig example.com
host example.com

# 接続テスト
telnet host 80
nc -zv host 80        # ポート開放確認
```

## 圧縮・解凍

```bash
# tar.gz
tar -czf archive.tar.gz directory/    # 圧縮
tar -xzf archive.tar.gz               # 解凍
tar -xzf archive.tar.gz -C /path/to/  # 展開先指定
tar -tzf archive.tar.gz               # 内容確認

# tar.bz2
tar -cjf archive.tar.bz2 directory/
tar -xjf archive.tar.bz2

# zip
zip -r archive.zip directory/
unzip archive.zip
unzip -l archive.zip  # 内容確認

# gzip
gzip file.txt         # file.txt.gz作成（元ファイル削除）
gzip -k file.txt      # 元ファイル保持
gunzip file.txt.gz
```

## 環境変数

```bash
# 表示
echo $PATH
echo $HOME
env                   # すべて表示
printenv

# 設定
export VAR_NAME="value"
export PATH=$PATH:/new/path

# 永続化（.bashrc または .zshrc）
echo 'export VAR_NAME="value"' >> ~/.bashrc
source ~/.bashrc
```

## シェルスクリプト

### 基本構造

```bash
#!/bin/bash

# コメント

# 変数
name="World"
echo "Hello, $name"
echo "Hello, ${name}!"

# コマンド実行結果を変数に
current_date=$(date)
file_count=$(ls | wc -l)

# 引数
echo "Script: $0"
echo "First arg: $1"
echo "All args: $@"
echo "Arg count: $#"
```

### 条件分岐

```bash
# if文
if [ -f "file.txt" ]; then
    echo "File exists"
elif [ -d "directory" ]; then
    echo "Directory exists"
else
    echo "Not found"
fi

# 文字列比較
if [ "$var" = "value" ]; then
    echo "Match"
fi

if [ "$var" != "value" ]; then
    echo "Not match"
fi

# 数値比較
if [ $num -eq 10 ]; then echo "Equal"; fi
if [ $num -ne 10 ]; then echo "Not equal"; fi
if [ $num -gt 10 ]; then echo "Greater"; fi
if [ $num -lt 10 ]; then echo "Less"; fi

# ファイル・ディレクトリチェック
if [ -f "file.txt" ]; then echo "File exists"; fi
if [ -d "dir" ]; then echo "Directory exists"; fi
if [ -r "file.txt" ]; then echo "Readable"; fi
if [ -w "file.txt" ]; then echo "Writable"; fi
if [ -x "script.sh" ]; then echo "Executable"; fi

# 論理演算
if [ -f "file.txt" ] && [ -r "file.txt" ]; then
    echo "File exists and readable"
fi

if [ "$var" = "a" ] || [ "$var" = "b" ]; then
    echo "var is a or b"
fi

# case文
case "$1" in
    start)
        echo "Starting..."
        ;;
    stop)
        echo "Stopping..."
        ;;
    restart)
        echo "Restarting..."
        ;;
    *)
        echo "Usage: $0 {start|stop|restart}"
        exit 1
        ;;
esac
```

### ループ

```bash
# for文
for i in 1 2 3 4 5; do
    echo "Number: $i"
done

for file in *.txt; do
    echo "Processing $file"
done

for i in {1..10}; do
    echo $i
done

for ((i=1; i<=10; i++)); do
    echo $i
done

# while文
count=1
while [ $count -le 5 ]; do
    echo "Count: $count"
    ((count++))
done

# ファイル読み込み
while IFS= read -r line; do
    echo "Line: $line"
done < file.txt
```

### 関数

```bash
# 関数定義
function greet() {
    echo "Hello, $1!"
}

# または
greet() {
    local name=$1
    echo "Hello, $name!"
}

# 呼び出し
greet "World"

# 返り値
add() {
    local result=$(($1 + $2))
    echo $result
}

sum=$(add 5 3)
echo "Sum: $sum"
```

### エラーハンドリング

```bash
# コマンド成功チェック
if ! command -v docker &> /dev/null; then
    echo "Docker not found"
    exit 1
fi

# エラー時に即終了
set -e

# 未定義変数でエラー
set -u

# パイプの途中でエラー時も検出
set -o pipefail

# 組み合わせ
set -euo pipefail

# trap（エラー時の処理）
cleanup() {
    echo "Cleaning up..."
    rm -f /tmp/tempfile
}

trap cleanup EXIT
```

## 便利なワンライナー

```bash
# 大きいファイル・ディレクトリ検索
du -sh */ | sort -h
du -ah . | sort -rh | head -20

# プロセスのメモリ使用量
ps aux --sort=-%mem | head

# ファイル内の重複行をカウント
sort file.txt | uniq -c | sort -rn

# ログから特定時間帯を抽出
sed -n '/2024-01-01 10:00/,/2024-01-01 11:00/p' access.log

# JSONの整形（jq）
echo '{"name":"John","age":30}' | jq .
curl -s https://api.example.com/data | jq '.items[] | .name'

# 複数ファイルの一括置換
find . -name "*.txt" -exec sed -i 's/old/new/g' {} \;

# ディレクトリ構造表示
tree -L 2

# ファイルの差分
diff file1.txt file2.txt
diff -u file1.txt file2.txt  # unified形式

# 並列実行
cat urls.txt | xargs -P 4 -I {} curl -O {}

# CSVの列抽出
awk -F, '{print $1,$3}' file.csv

# IPアドレス一覧
ip addr show | grep 'inet ' | awk '{print $2}'
ifconfig | grep 'inet ' | awk '{print $2}'

# ディスク使用率
df -h
df -h | grep -v tmpfs

# 最近変更されたファイル
find . -type f -mtime -1
ls -lt | head

# ファイル内の特定パターンを含む行数
grep -c "pattern" file.txt

# 複数の拡張子を検索
find . -type f \( -name "*.jpg" -o -name "*.png" \)

# バックアップ作成
cp file.txt{,.bak}
tar -czf backup_$(date +%Y%m%d).tar.gz directory/
```

## シェルショートカット

```bash
# カーソル移動
Ctrl+A    # 行頭へ
Ctrl+E    # 行末へ
Ctrl+B    # 1文字戻る
Ctrl+F    # 1文字進む
Alt+B     # 1単語戻る
Alt+F     # 1単語進む

# 編集
Ctrl+U    # カーソルから行頭まで削除
Ctrl+K    # カーソルから行末まで削除
Ctrl+W    # カーソル前の単語を削除
Ctrl+Y    # ペースト

# 履歴
Ctrl+R    # 履歴検索
Ctrl+P    # 前のコマンド（↑）
Ctrl+N    # 次のコマンド（↓）
!!        # 直前のコマンド
!$        # 直前のコマンドの最後の引数
!*        # 直前のコマンドのすべての引数

# その他
Ctrl+L    # 画面クリア
Ctrl+C    # 実行中断
Ctrl+D    # EOF（終了）
Ctrl+Z    # 一時停止
```

## Tips

```bash
# コマンド履歴
history
history | grep "command"
!123      # 履歴番号123のコマンド実行

# エイリアス
alias ll='ls -la'
alias gs='git status'
alias ..='cd ..'

# 永続化（~/.bashrc または ~/.zshrc）
echo "alias ll='ls -la'" >> ~/.bashrc

# ブレース展開
echo {A,B,C}          # A B C
echo {1..5}           # 1 2 3 4 5
mkdir -p test/{a,b,c}/{1,2,3}

# コマンド置換
echo "Today is $(date)"
files=$(ls *.txt)

# 算術展開
echo $((5 + 3))
((i++))

# デフォルト値
echo ${VAR:-default}  # VARが未設定ならdefault使用

# 文字列操作
str="hello world"
echo ${str^^}         # 大文字化
echo ${str,,}         # 小文字化
echo ${#str}          # 文字数
echo ${str:0:5}       # 部分文字列

# サブシェル
(cd /tmp && ls)       # 元のディレクトリに戻る

# 複数コマンド
command1 && command2  # command1成功時のみcommand2実行
command1 || command2  # command1失敗時にcommand2実行
command1 ; command2   # 順次実行
```
