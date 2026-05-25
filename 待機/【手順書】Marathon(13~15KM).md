# 13KM〜15KM AWS手動デプロイ手順書（CRMアプリ）

## 1 今回の目的

### この工程でしていること

ローカル環境で作成したCRMアプリを  
AWS EC2上へ手動デプロイし、

- Web画面
- Node.js API
- PostgreSQL

を連携させながら、
インターネット経由でアクセスできるようにする。

---

### 今回学習する内容

- AWS EC2ログイン
- Linux操作
- Nginx公開ディレクトリ
- zip圧縮 / 解凍
- scpファイル転送
- Node.js起動
- PostgreSQL接続
- API疎通確認
- AWS上でのトラブルシュート

---

## 2 全体構成

### 構成イメージ

```text
ブラウザ
↓
Nginx（Webサーバ）
↓
Node.js API
↓
PostgreSQL
```

---

### この構成でしていること

- Nginx：HTML公開
- Node.js：API処理
- PostgreSQL：データ保存

---

## 3 13KM：AWS EC2へログイン

### この工程でしていること

AWS上のLinuxサーバへログインし、
Web公開ディレクトリを確認する。

---

### SSHログイン

#### 実行コマンド

```bash
ssh ユーザ名@dev.marathon.rplearn.net
```

---

### ログイン確認

#### コマンド

```bash
pwd
```

---

### 実行例

```bash
/home/y-onishi
```

---

### OKの目安

- Linuxへログインできる
- ホームディレクトリ表示

---

## 4 Nginx公開ディレクトリ確認

### この工程でしていること

Web画面を配置するディレクトリを確認する。

---

### Nginx配下へ移動

#### コマンド

```bash
cd /usr/share/nginx/html
```

---

### 自分のディレクトリ確認

#### コマンド

```bash
ls | grep y-
```

---

### 実行例

```bash
y-onishi
```

---

### ディレクトリ移動

#### コマンド

```bash
cd /usr/share/nginx/html/y-onishi
```

---

### OKの目安

- 自分用ディレクトリが存在
- 移動できる

---

## 5 index.html作成確認

### この工程でしていること

Nginx配下へHTMLを配置し、
ブラウザ公開確認を行う。

---

### index.html作成

#### コマンド

```bash
vi index.html
```

---

### 記載内容例

```html
<h1>Hello Marathon</h1>
```

---

### ブラウザ確認

```text
http://dev.marathon.rplearn.net/y-onishi
```

---

### この工程の意味

Nginx配下へHTMLを配置すると、
インターネット公開できる。

---

### OKの目安

- ブラウザで Hello Marathon 表示

---

## 6 14KM：ローカルsrcをzip化

### この工程でしていること

ローカルアプリを圧縮し、
AWSへアップロード準備する。

---

### zip作成

#### Mac側コマンド

```bash
zip -r dev-marathon.zip src
```

---

### 確認

#### コマンド

```bash
ls
```

---

### 実行例

```bash
dev-marathon.zip
```

---

### OKの目安

- zipファイル生成される

---

## 7 zipファイルをAWSへアップロード

### この工程でしていること

scpでローカルPCからAWSへファイル転送する。

---

### scpアップロード

#### Mac側コマンド

```bash
scp dev-marathon.zip y-onishi@dev.marathon.rplearn.net:/app/y-onishi
```

---

### この工程の意味

ローカルPC → AWSサーバへ転送。

---

### OKの目安

- 転送成功
- エラーなし

---

## 8 AWS側でzip解凍

### この工程でしていること

アップロードしたアプリをサーバ上へ展開する。

---

### app配下へ移動

#### コマンド

```bash
cd /app/y-onishi
```

---

### ファイル確認

#### コマンド

```bash
ls
```

---

### zip解凍

#### コマンド

```bash
unzip dev-marathon.zip
```

---

### OKの目安

- srcディレクトリ生成される

---

## 9 15KM：WebファイルをNginx配下へ配置

### この工程でしていること

HTML / JavaScript をNginx公開ディレクトリへ配置する。

---

### コピー

#### コマンド

```bash
cp -r /app/y-onishi/src/web/* /usr/share/nginx/html/y-onishi/
```

---

### 確認

#### コマンド

```bash
ls /usr/share/nginx/html/y-onishi
```

---

### 実行例

```bash
case
config.js
customer
index.html
negotiation
```

---

### OKの目安

- Webファイルが配置される

---

## 10 config.js修正

### この工程でしていること

ローカル用API設定から、
AWS用API設定へ変更する。

---

### 修正前

```javascript
const config = {
  apiUrl: 'http://localhost:3000'
};
```

---

### 修正後

```javascript
const config = {
  apiUrl: '/api_y-onishi'
};
```

---

### この設定の意味

ブラウザからAWS上のNode.js APIへアクセスする。

---

### OKの目安

- localhost が消えている
- /api_y-onishi へ変更済み

---

## 11 Node.js(index.js)修正

### この工程でしていること

Docker用DB接続設定から、
AWS用PostgreSQL設定へ変更する。

---

### Pool設定確認

#### コマンド

```bash
grep -A5 "new Pool" index.js
```

---

### 修正前

```javascript
host: "db"
```

---

### 修正後

```javascript
host: "localhost"
```

---

### DB接続情報修正例

```javascript
user: "user_y_onishi"
database: "db_y_onishi"
password: "5Rw5YDaWc5jc"
```

---

### この設定の意味

Node.jsからAWS上のPostgreSQLへ接続する。

---

### OKの目安

- DB接続情報修正済み
- host が localhost

---

## 12 PostgreSQL接続確認

### この工程でしていること

DBへログインし、
テーブル存在確認を行う。

---

### PostgreSQLログイン

#### コマンド

```bash
psql -U user_y_onishi -d db_y_onishi
```

---

### パスワード入力

```text
5Rw5YDaWc5jc
```

---

### テーブル確認

#### コマンド

```bash
\dt
```

---

### 実行例

```bash
customers
cases
negotiations
```

---

### PostgreSQL終了

#### コマンド

```bash
\q
```

---

### OKの目安

- DBログイン成功
- テーブル表示される

---

## 13 Node.jsバックエンド起動

### この工程でしていること

Node.js APIサーバを起動する。

---

### nodeディレクトリ移動

#### コマンド

```bash
cd /app/y-onishi/src/node
```

---

### npm install

#### コマンド

```bash
npm install
```

---

### Node.js起動

#### コマンド

```bash
npm run dev
```

---

### 正常時

```bash
Server running on port 5631
```

---

### OKの目安

- エラーなし
- API起動成功

---

## 14 API確認

### この工程でしていること

Node.js APIが利用できるか確認する。

---

### ブラウザ確認

```text
http://dev.marathon.rplearn.net/api_y-onishi/customers
```

---

### 正常時

```json
[]
```

または

```json
[{...}]
```

---

### この工程の意味

Node.js APIがブラウザから利用可能か確認している。

---

### OKの目安

- JSON表示される
- エラーなし

---

## 15 顧客登録確認

### この工程でしていること

Web画面からCRUD動作確認を行う。

---

### 顧客登録画面

```text
http://dev.marathon.rplearn.net/y-onishi/customer/add.html
```

---

### 顧客一覧画面

```text
http://dev.marathon.rplearn.net/y-onishi/customer/list.html
```

---

### 確認内容

- 顧客登録できる
- 一覧表示される
- 詳細画面開ける
- 更新できる
- 削除できる

---

### OKの目安

- CRUD全て成功

---

## 16 今回発生した主なエラー

### password authentication failed

#### 原因

DB接続情報誤り。

---

#### 対応

index.js修正。

---

### EADDRINUSE

#### 原因

Node.js重複起動。

---

#### 対応

重複起動しない。

---

### localhostのままAPIアクセス

#### 原因

config.jsがローカル設定。

---

#### 対応

```javascript
/api_y-onishi
```

へ修正。

---

### config.jsコメントミス

#### 原因

```javascript
//};
```

---

#### 対応

```javascript
};
```

へ修正。

---

### No space left on device

#### 原因

容量不足。

---

#### 対応

不要ファイル削除。

```bash
rm -rf node_modules
rm -f dev-marathon.zip
```

---

## 17 今回学習できたこと

### Linux / AWS

- AWS EC2ログイン
- Linux基本操作
- Nginx公開ディレクトリ

---

### デプロイ

- zip圧縮
- scp転送
- unzip展開

---

### Node.js

- npm install
- Node.js起動
- API疎通確認

---

### PostgreSQL

- DB接続
- テーブル確認

---

### 実務スキル

- フロント/バックエンド連携
- AWSトラブルシュート
- 実務に近いデプロイ作業

---

## 完了条件

- Web画面表示成功
- API疎通成功
- PostgreSQL接続成功
- CRUD成功
- AWS上で正常動作

---

## 完成

AWS EC2上へ  
CRMアプリの手動デプロイ完了。