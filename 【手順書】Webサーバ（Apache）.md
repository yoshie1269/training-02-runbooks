# Webサーバ構築 手順書（Apache / httpd インストール・動作確認）

---

## 概要

本手順書では、Linux サーバ上に Apache（httpd）を導入し、  
Web サーバとして稼働させるまでの基本設定と動作確認を行う。

本工程により、以下が可能となる。

- Apache を用いた Web サービスの提供
- HTTP（80番ポート）での通信受付
- アクセスログの取得と確認

---

## 前提条件

- root 権限で作業できること
- インターネット接続が可能であること
- ファイアウォールおよびセキュリティグループで 80/TCP が許可されていること

---

## 1. root ユーザーへ切り替える

---

### この工程でしていること
- パッケージのインストールやサービス管理を行うため、管理者権限へ切り替える

---

### 実行コマンド
```bash
sudo su -
```

---

### コマンドの意味
- sudo：管理者権限で実行
- su -：root ユーザーへ完全切替

---

### 確認方法とOKの目安
- プロンプトが `#` になること

---

## 2. Apache（httpd）をインストールする

---

### この工程でしていること
- Web サーバソフト Apache を導入する

---

### 実行コマンド
```bash
dnf install httpd
```

---

### コマンドの意味
- dnf install：パッケージ導入
- httpd：Apache 本体

---

### 確認方法とOKの目安
```bash
dnf list installed | grep httpd
```

- httpd が表示されること

---

## 3. Apache サービスを起動する

---

### この工程でしていること
- Apache を起動し、Web サービスを開始する

---

### 実行コマンド
```bash
systemctl start httpd
```
```bash
systemctl status httpd
```

---

### コマンドの意味
- start：サービス起動
- status：状態確認

---

### 確認方法とOKの目安
- Active: active (running) と表示されること

---

## 4. Apache の自動起動を設定する

---

### この工程でしていること
- OS 起動時に Apache が自動で起動するよう設定する

---

### 実行コマンド
```bash
systemctl enable httpd
```
```bash
systemctl list-unit-files -t service | grep httpd
```

---

### コマンドの意味
- enable：自動起動登録
- list-unit-files：登録状態一覧表示

---

### 確認方法とOKの目安
```bash
httpd.service enabled
```
と表示されること

---

## 5. Apache の待受ポートを確認する

---

### この工程でしていること
- Apache が HTTP（80番ポート）で通信待受しているか確認する

---

### 実行コマンド
```bash
netstat -ln | grep 80 | grep -v grep
```

---

### コマンドの意味
- netstat -ln：待受ポート表示
- grep 80：80番抽出
- grep -v grep：不要行除外

---

### 確認方法とOKの目安

例：tcp 0 0 0.0.0.0:80 0.0.0.0:* LISTEN

- LISTEN 状態で表示されること

---

## 6. トップページ（index.html）を作成する

---

### この工程でしていること
- Web ブラウザで表示されるトップページを作成する
- Apache の動作確認用ページを用意する

---

### 実行コマンド
```bash
vi /var/www/html/index.html
```

---

### コマンドの意味
- vi：テキストエディタ
- /var/www/html：ドキュメントルート

---
### 確認方法とOKの目安

ブラウザで以下にアクセス：
`http://サーバIP/`
- 作成したページが表示されること

---

## 7. アクセスログを確認する

---

### この工程でしていること
- Web サイトへのアクセス履歴をリアルタイムで監視する
- 通信状況や障害調査に活用する

---

### 実行コマンド
```bash
tail -f /var/log/httpd/access_log
```

---

### コマンドの意味
- tail：ファイル末尾表示
- -f：リアルタイム監視
- access_log：アクセス履歴

---



