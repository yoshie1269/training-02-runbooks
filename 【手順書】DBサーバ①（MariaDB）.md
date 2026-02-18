# LAMP サーバー構築 手順書①（Apache / PHP / MariaDB 導入）

---

## 概要

本手順書では、Amazon Linux 2023 上に LAMP 環境を構築するため、  
以下の作業を実施する。

- OS・パッケージの更新
- Apache / PHP / MariaDB の導入
- Apache の起動・自動起動設定
- Web ディレクトリ権限の設定
- PHP 動作確認
- MariaDB 初期セキュリティ設定

---

## 前提条件

- Amazon Linux 2023 を使用していること
- EC2 にパブリックアクセス可能であること
- セキュリティグループで 80/TCP が開放されていること
- root 権限または sudo 権限を持っていること

---

## 1. OS・パッケージを最新化する

---

### この工程でしていること
- OS の更新を行い、脆弱性を解消する
- 依存関係の問題を防止する

---

### 実行コマンド
```bash
sudo dnf upgrade -y
```

---

### コマンドの意味
- dnf upgrade：パッケージ更新
- -y：確認なしで実行

---

### 確認方法とOKの目安
- エラーなく処理が完了する
- Complete! が表示される

---

## 2. Apache / PHP / 関連モジュールをインストールする

---

### この工程でしていること
- Web サーバ（Apache）を導入する
- PHP 実行環境を構築する
- WordPress 等に必要な拡張を導入する

---

### 実行コマンド
```bash
sudo dnf install -y httpd wget php-fpm php-mysqli php-json php php-devel
```

---

### コマンドの意味

| 項目 | 内容 |
|------|------|
| httpd | Apache 本体 |
| wget | ダウンロード用 |
| php-fpm | PHP 実行管理 |
| php-mysqli | DB連携 |
| php-json | JSON処理 |
| php-devel | 開発用 |

---

### 確認方法とOKの目安
```bash
dnf list installed | grep httpd
```
```bash
dnf list installed | grep php
```

- パッケージが表示される

---

## 3. MariaDB をインストールする

---

### この工程でしていること
- データベースサーバを導入する
- WordPress 等のデータ保存先を準備する

---

### 実行コマンド
```bash
sudo dnf install mariadb105-server
```
```bash
sudo dnf info mariadb105-server
```

---

### コマンドの意味
- mariadb105-server：MariaDB 本体
- info：詳細情報表示

---

### 確認方法とOKの目安
- バージョン情報が表示される
- インストール済みであること

---

## 4. Apache を起動・自動起動設定する

---

### この工程でしていること
- Web サーバを起動する
- OS 起動時に自動起動させる

---

### 実行コマンド
```bash
sudo systemctl start httpd
```
```bash
sudo systemctl enable httpd
```

---

### コマンドの意味
- start：起動
- enable：自動起動登録

---

### 確認方法とOKの目安
```bash
sudo systemctl status httpd
```
```bash
sudo systemctl is-enabled httpd
```
- active (running)
- enabled と表示される

---

## 5. ec2-user を Apache グループへ追加する

---

### この工程でしていること
- ec2-user が Web ファイルを編集できるようにする
- 権限管理を適切にする

---

### 実行コマンド
```bash
sudo usermod -a -G apache ec2-user
```
```bash
exit
```
 - sshでログイン

---

### コマンドの意味
- usermod -a -G：グループ追加
- exit：再ログイン
- groups：所属グループ表示

---

### 確認方法とOKの目安
```bash
groups
```
- apache が表示される

---

## 6. Web ディレクトリの権限を設定する

---

### この工程でしていること
- Apache と ec2-user が共同管理できるように設定する
- セキュリティを維持しつつ編集可能にする

---

### 実行コマンド
```bash
sudo chown -R ec2-user:apache /var/www
```
```bash
sudo find /var/www -type d -exec chmod 2775 {} \;
```
```bash
sudo find /var/www -type f -exec chmod 0664 {} \;
```

---

### コマンドの意味

| 項目 | 内容 |
|------|------|
| chown | 所有者変更 |
| chmod 2775 | SGID設定 |
| find | 再帰適用 |

---

### 確認方法とOKの目安
```bash
ls -ld /var/www
```
- ec2-user apache になっている

---

## 7. PHP 動作確認を行う

---

### この工程でしていること
- PHP が正常に動作するか確認する
- LAMP 構成の正常性を検証する

---

### 実行コマンド
```bash
echo '<?php phpinfo(); ?>' | sudo tee /var/www/html/phpinfo.php
```

---

### 確認方法

ブラウザでアクセス
`http://<パブリックIPアドレス>/phpinfo.php`

---

### OKの目安
- PHP 情報ページが表示される

---

### テストファイル削除
```bash
rm /var/www/html/phpinfo.php
```

---

## 8. MariaDB を起動する

---

### この工程でしていること
- データベースサービスを開始する

---

### 実行コマンド
```bash
sudo systemctl start mariadb
```

---

### 確認方法とOKの目安
```bash
systemctl status mariadb
```
- active (running)

---

## 9. MariaDB 初期セキュリティ設定を行う

---

### この工程でしていること
- DB を安全に運用できる状態にする
- 不正アクセスを防止する

---

### 実行コマンド
```bash
sudo mysql_secure_installation
```

---

### 設定内容（対話形式）

1. 現在の root パスワード  
   → Enter

2. root パスワード設定  
   → Y → 設定

3. 匿名ユーザー削除  
   → Y

4. リモート root 無効化  
   → Y

5. テストDB削除  
   → Y

6. 権限再読込  
   → Y

---

### 確認方法とOKの目安
- エラーなく完了
- All done! と表示される

---

## 10. MariaDB の自動起動設定

---

### この工程でしていること
- OS 起動時に DB を自動起動させる

---

### 実行コマンド
```bash
sudo systemctl enable mariadb
```

---

### 確認方法とOKの目安
```bash
systemctl is-enabled mariadb
```

- enabled

---

## 11. ここまでで完了していること

- Apache が稼働している
- PHP が動作している
- MariaDB が稼働している
- Web 権限が適切に設定されている
- 基本セキュリティが確保されている

---





