# 6KM〜12KM CRMアプリ構築手順書

## 1 今回の目的

### この工程でしていること

Docker上で動作するCRMアプリを構築し、

- Web画面
- Node.js API
- PostgreSQL
- HTML / JavaScript
- CRUD処理

を連携させながら、
実務に近いWebアプリ開発を学習する。

また、

- エラー調査
- ログ確認
- DB接続確認
- トラブルシュート

も実践する。

---

## 2 今回実施した内容

### 6KM

- PostgreSQLテーブル作成
- 顧客登録画面作成
- API登録処理

---

### 7KM

- エラー調査
- DB接続修正
- 顧客一覧画面作成

---

### 8KM

- ホーム画面作成
- 画面遷移追加

---

### 9KM

- 登録確認画面追加

---

### 10KM

- 顧客詳細画面追加

---

### 11KM

- 顧客削除機能追加

---

### 12KM

- 顧客更新機能追加

---

## 3 DBテーブル作成（6KM）

### この工程でしていること

PostgreSQLへCRM用テーブルを作成する。

---

### DBコンテナへログイン

#### コマンド

```bash
docker compose exec db bash
```

---

### PostgreSQLへログイン

#### コマンド

```bash
psql -U user_5631 -d crm_5631
```

---

### SQLファイル実行

#### コマンド

```bash
\i /tmp/database_setup.sql
```

---

### テーブル確認

#### コマンド

```bash
\dt
```

---

### OKの目安

以下のテーブルが表示される。

- customers
- negotiations
- cases
- employees

---

## 4 顧客登録画面作成（6KM）

### この工程でしていること

顧客情報を入力し、
APIへ送信する登録画面を作成する。

---

### 作成ファイル

```bash
src/web/customer/add.html
```

---

### 実装内容

入力フォームを作成。

- 会社名
- 業種
- 連絡先
- 所在地

---

### API接続先設定

#### 編集ファイル

```bash
src/web/config.js
```

---

### 修正内容

```javascript
const config = {
  apiUrl: 'http://localhost:5631'
};

export default config;
```

---

### この設定の意味

- フロント側から接続するAPIサーバURLを指定
- localhost:5631 の Node.js APIへ通信

---

### 登録API作成

#### 編集ファイル

```bash
src/node/index.js
```

---

### 実装内容

```javascript
app.post("/add-customer", async (req, res) => {
  try {

    const {
      companyName,
      industry,
      contact,
      location
    } = req.body;

    const newCustomer = await pool.query(
      "INSERT INTO customers (company_name, industry, contact, location) VALUES ($1, $2, $3, $4) RETURNING *",
      [companyName, industry, contact, location]
    );

    res.json({
      success: true,
      customer: newCustomer.rows[0]
    });

  } catch (err) {
    console.error(err);
    res.json({ success: false });
  }
});
```

---

### このAPIでしていること

- POSTリクエスト受信
- customersテーブルへINSERT
- 登録成功時JSON返却

---

### OKの目安

登録後、

- success: true
- DBへデータ登録

される。

---

## 5 エラー調査（7KM）

### この工程でしていること

実際に発生したエラーを調査し、
原因切り分けと修正を行う。

---

### ① ENOTFOUND x

#### 原因

DB接続先 host が誤っていた。

---

#### 修正前

```javascript
host: "x"
```

---

#### 修正後

```javascript
host: "db"
```

---

### このエラーの意味

名前解決できない。

---

### OKの目安

DB接続成功。

---

### ② ECONNREFUSED

#### 原因

PostgreSQL接続ポート誤り。

---

#### 修正

```javascript
port: 5432
```

---

### このエラーの意味

接続先ポートへアクセスできない。

---

### OKの目安

API起動時にDB接続エラーが出ない。

---

### ③ company_nam does not exist

#### 原因

カラム名 typo。

---

#### 修正前

```javascript
company_nam
```

---

#### 修正後

```javascript
company_name
```

---

### OKの目安

INSERT成功。

---

### ④ duplicate key value violates unique constraint

#### 原因

contact値重複。

---

### 対応

別の連絡先で登録。

---

### OKの目安

登録成功。

---

## 6 顧客一覧画面作成（7KM）

### この工程でしていること

登録済み顧客一覧を表示する。

---

### 作成ファイル

```bash
src/web/customer/list.html
```

---

### API取得

```javascript
fetch(`${config.apiUrl}/customers`)
```

---

### 一覧表示処理

```javascript
data.forEach((customer, index) => {

  const row = list.insertRow();

  const cell1 = row.insertCell(0);
  const cell2 = row.insertCell(1);
  const cell3 = row.insertCell(2);

  cell1.textContent = index + 1;

  cell2.innerHTML =
    `<a href="./detail.html?id=${customer.customer_id}">
      ${customer.company_name}
    </a>`;

  cell3.textContent = customer.contact;
});
```

---

### この処理でしていること

- API取得
- tableへ動的追加
- 詳細画面リンク生成

---

### OKの目安

一覧表示される。

---

## 7 ホーム画面作成（8KM）

### この工程でしていること

画面遷移用トップページ作成。

---

### 作成ファイル

```bash
src/web/index.html
```

---

### 実装内容

- 顧客一覧リンク
- 顧客登録リンク

---

### アクセスURL

```bash
http://localhost:8080/
```

---

### OKの目安

トップ画面表示。

---

## 8 登録確認画面作成（9KM）

### この工程でしていること

登録前確認画面を追加する。

---

### 作成ファイル

```bash
src/web/customer/add-confirm.html
```

---

### 処理流れ

```text
add.html
↓
URLパラメータへ値格納
↓
確認画面表示
↓
登録API実行
```

---

### add.html修正

```javascript
window.location.href =
  './add-confirm.html?' + params.toString();
```

---

### この処理の意味

入力値をURLへ付与し確認画面へ渡す。

---

### OKの目安

入力値が確認画面へ表示される。

---

## 9 顧客詳細画面作成（10KM）

### この工程でしていること

選択した顧客詳細を表示する。

---

### 作成ファイル

```bash
src/web/customer/detail.html
```

---

### URLパラメータ取得

```javascript
const params = new URLSearchParams(window.location.search);

const customerId = params.get("id");
```

---

### API取得

```javascript
fetch(`http://localhost:5631/customers`)
```

---

### 詳細表示

```javascript
const customer = data.find(
  item => item.customer_id == customerId
);
```

---

### この処理でしていること

- URLからid取得
- API取得
- 対象顧客検索

---

### OKの目安

対象顧客情報が表示される。

---

## 10 顧客削除機能追加（11KM）

### この工程でしていること

顧客削除機能を追加する。

---

### detail.html修正

#### 追加内容

- 削除ボタン追加

---

### 削除API作成

#### 編集ファイル

```bash
src/node/index.js
```

---

### 実装内容

```javascript
app.delete("/delete-customer/:id", async (req, res) => {

  try {

    const customerId = req.params.id;

    await pool.query(
      "DELETE FROM customers WHERE customer_id = $1",
      [customerId]
    );

    res.json({ success: true });

  } catch (err) {

    console.error(err);
    res.json({ success: false });

  }
});
```

---

### フロント側処理

```javascript
fetch(`http://localhost:5631/delete-customer/${customerId}`, {
  method: "DELETE"
})
```

---

### この処理でしていること

- DELETE API実行
- DBデータ削除

---

### OKの目安

一覧からデータ消える。

---

## 11 顧客更新機能追加（12KM）

### この工程でしていること

顧客情報更新機能を追加する。

---

### detail.html修正

#### 変更内容

```text
textContent
↓
input value
```

---

### 更新API作成

#### 編集ファイル

```bash
src/node/index.js
```

---

### 実装内容

```javascript
app.put("/update-customer/:id", async (req, res) => {

  try {

    const customerId = req.params.id;

    const {
      companyName,
      industry,
      contact,
      location
    } = req.body;

    await pool.query(
      "UPDATE customers SET company_name = $1, industry = $2, contact = $3, location = $4 WHERE customer_id = $5",
      [
        companyName,
        industry,
        contact,
        location,
        customerId
      ]
    );

    res.json({ success: true });

  } catch (err) {

    console.error(err);
    res.json({ success: false });

  }
});
```

---

### 更新処理

```javascript
fetch(`http://localhost:5631/update-customer/${customerId}`, {

  method: "PUT",

  headers: {
    "Content-Type": "application/json"
  },

  body: JSON.stringify({

    companyName:
      document.getElementById("company_name").value,

    industry:
      document.getElementById("industry").value,

    contact:
      document.getElementById("contact").value,

    location:
      document.getElementById("location").value
  })
})
```

---

### この処理でしていること

- フォーム入力値取得
- PUT API送信
- DB更新

---

### OKの目安

更新内容が一覧・詳細へ反映される。

---

## 12 最終構成

### 画面遷移

```text
ホーム画面
↓
一覧画面
↓
詳細画面
↓
更新・削除
```

---

### 登録処理

```text
登録画面
↓
確認画面
↓
DB登録
```

---

## 13 今回学習できたこと

### フロント

- HTML
- JavaScript
- Bootstrap
- fetch
- URLパラメータ

---

### バックエンド

- Node.js
- Express
- REST API
- CRUD処理

---

### DB

- PostgreSQL
- INSERT
- UPDATE
- DELETE
- SELECT

---

### インフラ

- Docker
- コンテナ操作

---

### トラブルシュート

- ログ確認
- DB接続確認
- エラー調査

---

## 14 実際に発生した実務系エラー

### 発生したエラー

- ENOTFOUND
- ECONNREFUSED
- Cannot GET /
- Unexpected token <
- duplicate key
- column does not exist

---

### この工程で学習したこと

ログを確認しながら、

- 原因切り分け
- 設定修正
- 動作確認

を実施した。

---

## 完了条件

- 顧客登録可能
- 顧客一覧表示可能
- 顧客詳細表示可能
- 顧客更新可能
- 顧客削除可能
- DB連携成功
- Docker上で正常動作

---

## 完成

Docker + Node.js + PostgreSQL を利用した  
CRMアプリ構築完了。