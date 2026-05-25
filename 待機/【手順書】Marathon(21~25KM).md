# 21KM〜25KM PM2常駐化・Cypress E2Eテスト手順書

## 1 今回の目的

### この工程でしていること

Node.jsバックエンドAPIをPM2で常駐化し、  
Cypressを利用してWebアプリのE2Eテストを実施する。

---

### 今回学習する内容

- PM2によるNode.jsプロセス管理
- バックエンド常駐化
- API疎通確認
- DB接続エラー調査
- Cypress導入
- E2Eテスト作成
- alert検知
- 更新・削除テスト
- Cypress動画取得
- scp転送

---

# 21KM PM2を利用したNode.jsバックエンド常駐化

## 2 PM2とは

### この工程でしていること

Node.jsアプリケーションをサーバ上で常駐実行できるようにする。

通常の node index.js や npm run dev は、  
SSH切断やターミナル終了で停止する可能性がある。

PM2を使うことで、

- バックグラウンド常駐
- 異常終了時の自動再起動
- ログ確認
- 状態管理

ができる。

---

## 3 PM2でバックエンド起動

### コマンド

```bash
cd /app/y-onishi/src/node
pm2 start index.js
```

---

### この工程でしていること

Node.jsのバックエンドAPIをPM2管理下で起動する。

---

## 4 PM2状態確認

### コマンド

```bash
pm2 list
```

---

### OKの目安

```bash
status online
```

---

## 5 API疎通確認

### コマンド

```bash
curl http://localhost:5631/customers
```

---

### OKの目安

```json
[]
```

または

```json
[{...}]
```

---

## 6 発生したエラー：Cannot find module 'express'

### エラー内容

```bash
Error: Cannot find module 'express'
```

---

### 原因

Node.js依存パッケージが未インストール。

---

### 対応

```bash
npm install
```

---

### OKの目安

npm install後にPM2で起動できる。

---

## 7 発生したDB接続エラー

### エラー内容

```bash
getaddrinfo EAI_AGAIN db
```

---

### 原因

Docker環境用のDB接続先になっていた。

---

### 修正前

```javascript
const pool = new Pool({
  user: "user_5631",
  host: "db",
  database: "crm_5631",
  password: "pass_5631",
  port: 5432,
});
```

---

### 修正後

```javascript
const pool = new Pool({
  user: "user_y_onishi",
  host: "localhost",
  database: "db_y_onishi",
  password: "5Rw5YDaWc5jc",
  port: 5432,
});
```

---

### この修正でしていること

Staging環境のPostgreSQLへ接続できるようにする。

---

## 8 PM2再起動

### コマンド

```bash
pm2 restart index
```

---

### OKの目安

PM2 listでonlineになる。

---

## 9 最終確認

### コマンド

```bash
curl http://localhost:5631/customers
```

---

### 正常例

```json
[
  {
    "customer_id": 2,
    "company_name": "test-company"
  }
]
```

---

# 22KM CypressによるE2Eテスト実施

## 10 今回の目的

### この工程でしていること

Cypressを利用して、  
Webアプリケーションの画面操作を自動テストする。

---

### 学習内容

- Cypress導入
- E2Eテスト作成
- 画面操作自動化
- テスト実行
- エラー解析

---

## 11 テストファイル作成

### 作成ファイル

```bash
/ci/cypress/e2e/y-onishi.cy.js
```

---

### この工程でしていること

自分用のCypressテストファイルを作成する。

---

## 12 Cypressインストール

### コマンド

```bash
cd /ci
npx cypress install
```

---

### OKの目安

Cypressがインストールされる。

---

## 13 Cypress実行

### コマンド

```bash
npx cypress run --browser chrome --spec cypress/e2e/y-onishi.cy.js
```

---

### OKの目安

テストが実行される。

---

## 14 add.html用E2Eテスト

### テスト内容

- 顧客情報入力
- 確認画面遷移
- 登録実行
- alert確認

---

## 15 発生したエラー① URL不一致

### エラー

```bash
expected URL to include /confirm.html
```

---

### 原因

実際の画面名が add-confirm.html だった。

---

### 対応

テスト側の期待URLを add-confirm.html に修正。

---

## 16 発生したエラー② alert検知失敗

### エラー

```bash
alertStub was never called
```

---

### 原因

alert が add-confirm.html 側で発火していた。

---

### 対応

確認画面へ遷移後に alert stub を設定。

```javascript
cy.window().then((win) => {
  cy.stub(win, 'alert').as('alertStub');
});
```

---

## 17 テスト成功確認

### 成功例

```bash
1 passing
```

---

# 23KM Cypress動画ダウンロード

## 18 今回の目的

### この工程でしていること

Cypress実行結果の動画を  
サーバからローカルPCへ取得する。

---

### 学習内容

- Cypress動画保存場所
- scpファイル転送
- テスト証跡管理

---

## 19 動画確認

### コマンド

```bash
cd /ci
ls cypress/videos
```

---

### 確認例

```bash
y-onishi.cy.js.mp4
```

---

### OKの目安

動画ファイルが存在する。

---

## 20 Macへ動画ダウンロード

### コマンド

```bash
scp y-onishi@dev.marathon.rplearn.net:/ci/cypress/videos/y-onishi.cy.js.mp4 ~/Downloads/
```

---

### この工程でしていること

Stagingサーバ上の動画をMacへコピーする。

---

## 21 ダウンロード確認

### コマンド

```bash
ls ~/Downloads | grep y-onishi
```

---

### OKの目安

y-onishi.cy.js.mp4 が表示される。

---

# 24KM add.html追加テスト作成

## 22 今回の目的

### この工程でしていること

QA観点で追加E2Eテストを作成する。

---

### 学習内容

- 異常系テスト
- 画面仕様確認
- QA観点でのテスト設計

---

## 23 実施した追加テスト

### テスト内容

未入力状態で送信した場合の挙動を確認する。

---

## 24 初回テスト結果

### 想定

add.html に留まる。

---

### 実際

add-confirm.html に遷移した。

---

## 25 判明した仕様

### 内容

未入力でも確認画面へ進める仕様。

---

## 26 修正後テスト

### テスト内容

空欄のまま確認画面へ遷移することを確認する。

---

### 成功結果

```bash
2 passing
```

---

# 25KM detail.html更新・削除テスト

## 27 今回の目的

### この工程でしていること

顧客詳細画面に対する  
更新・削除テストを実装する。

---

### 学習内容

- 詳細画面テスト
- 更新確認
- 削除確認
- DB反映確認

---

## 28 更新テスト内容

### 実施内容

- 一覧画面から詳細画面へ遷移
- 会社名を変更
- 更新ボタン押下
- alert確認
- 再読み込み後に変更内容確認

---

### OKの目安

更新後の会社名が画面に反映される。

---

## 29 削除テスト内容

### 実施内容

- API経由でテストデータ作成
- 詳細画面表示
- 削除ボタン押下
- alert確認
- 一覧画面から削除確認

---

## 30 削除テストで利用したAPI

### Cypressコード例

```javascript
cy.request({
  method: 'POST',
  url: '/api_y-onishi/add-customer',
})
```

---

### この工程でしていること

削除テスト用の顧客データを事前に作成する。

---

## 31 最終実行

### コマンド

```bash
npx cypress run --browser chrome --spec cypress/e2e/y-onishi.cy.js
```

---

### 結果

```bash
4 passing
```

---

## 32 今回学習した内容まとめ

### PM2 / Node.js

- PM2常駐化
- Node.js運用
- API起動確認
- DB接続修正

---

### Cypress

- Cypress導入
- E2Eテスト作成
- 画面操作自動化
- alert検知
- 更新テスト
- 削除テスト

---

### 証跡管理

- Cypress動画取得
- scp転送

---

### QA観点

- 正常系テスト
- 異常系テスト
- 実装仕様の確認

---

## 33 実際に発生したエラーと対応

### Cannot find module 'express'

#### 原因

npm install 未実施。

#### 対応

```bash
npm install
```

---

### getaddrinfo EAI_AGAIN db

#### 原因

DB接続先がDocker用の db のまま。

#### 対応

host を localhost に修正。

---

### expected URL to include /confirm.html

#### 原因

実際の画面名が add-confirm.html。

#### 対応

期待URLを修正。

---

### alertStub was never called

#### 原因

alert設定タイミングが違った。

#### 対応

確認画面遷移後にstub設定。

---

## 完了条件

- PM2でNode.jsがonline
- APIが正常応答
- Cypressテストが4 passing
- テスト動画取得済み
- 更新・削除テスト成功

---

## 完成

PM2によるバックエンド常駐化と、  
CypressによるE2Eテスト環境構築完了。