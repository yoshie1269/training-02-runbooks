# LAMP + WordPress 構築 手順書②（WordPress 導入・公開設定）

---

## 概要

本手順書では、構築済みの LAMP 環境上に WordPress を導入し、  
ブログとして公開できる状態まで設定する。

本工程により、以下を実現する。

- WordPress 本体の導入
- MariaDB への専用データベース作成
- WordPress 設定ファイル作成
- Apache 設定変更
- ファイル権限調整
- Web インストール準備完了

---

## 前提条件

- LAMP 環境が構築済みであること
- Apache / MariaDB が正常稼働していること
- /var/www/html が利用可能であること
- root 権限または sudo 権限があること
- ポート 80 が開放されていること

---

## 1. WordPress 用パッケージをインストールする

---

### この工程でしていること
- WordPress に必要な PHP 拡張・DB 連携機能を追加する
- 不足パッケージを補完する

---

### 実行コマンド
```bash
dnf install wget php-mysqlnd httpd php-fpm php-mysqli mariadb105-server php-json php php-devel -y
```

---

### コマンドの意味

| 項目 | 内容 |
|------|------|
| php-mysqlnd | PHP-DB連携 |
| php-fpm | PHP管理 |
| mariadb | DB |
| wget | DL |
| php-devel | 拡張用 |

---

### 確認方法とOKの目安
```bash
dnf list installed | grep php
```
```bash
dnf list installed | grep mariadb
```
- 必要パッケージが表示される

---

## 2. WordPress 本体をダウンロード・展開する

---

### この工程でしていること
- WordPress 最新版を取得する
- 利用可能な状態に展開する

---

### 実行コマンド
```bash
wget https://wordpress.org/latest.tar.gz
```
```bash
tar -xzf latest.tar.gz
```

---

### コマンドの意味
- wget：ダウンロード
- tar -xzf：gzip展開

---

### 確認方法とOKの目安
```bash
ls
```
- wordpress ディレクトリが存在

---

## 3. Apache / MariaDB を起動する

---

### この工程でしていること
- WordPress 利用に必要なサービスを起動する

---

### 実行コマンド
```bash
sudo systemctl start mariadb httpd
```

---

### 確認方法とOKの目安
```bash
systemctl status mariadb
```
```bash
systemctl status httpd
```
- active (running)

---

## 4. WordPress 用データベースを作成する

---

### この工程でしていること
- WordPress 専用 DB とユーザーを作成する
- アクセス権限を付与する

---

### MariaDB へ接続
```bash
mysql -u root -p
```
---

### SQL 設定
```bash
CREATE USER ‘wordpress-user’@‘localhost’ IDENTIFIED BY ‘your_strong_password’;
```
```bash
CREATE DATABASE wordpress-db;
```
```bash
GRANT ALL PRIVILEGES ON wordpress-db.* TO “wordpress-user”@“localhost”;
```
```bash
FLUSH PRIVILEGES;
```
```bash
exit
```
---

### 設定の意味

| 項目 | 内容 |
|------|------|
| CREATE USER | DBユーザー作成 |
| CREATE DATABASE | DB作成 |
| GRANT | 権限付与 |
| FLUSH | 反映 |

---

### 確認方法とOKの目安
```bash
SHOW DATABASES;
```
```bash
SELECT user,host FROM mysql.user;
```
- wordpress-db が存在
- wordpress-user が存在

---

## 5. wp-config.php を作成する

---

### この工程でしていること
- WordPress と DB を接続する設定を行う

---

### 実行コマンド
```bash
cp wordpress/wp-config-sample.php wordpress/wp-config.php
```
```bash
vi wordpress/wp-config.php
```
---

### 設定内容
```bash
define(‘DB_NAME’, ‘wordpress-db’);
define(‘DB_USER’, ‘wordpress-user’);
define(‘DB_PASSWORD’, ‘your_strong_password’);
```
---

### セキュリティキー設定

以下 URL で生成した値を貼り付ける：
`https://api.wordpress.org/secret-key/1.1/salt/`

---

### 意味
- DB 接続情報設定
- Cookie 暗号化強化

---

### 確認方法とOKの目安
- 設定ミスなし
- 構文エラーなし

---

## 6. WordPress ファイルを配置する

---

### この工程でしていること
- Apache 公開領域に WordPress を設置する

---

### 実行コマンド
```bash
cp -r wordpress/* /var/www/html/
```
```bash
mkdir /var/www/html/blog
```
```bash
cp -r wordpress/* /var/www/html/blog/
```

---

### 確認方法とOKの目安
```bash
ls /var/www/html
```
- wp-config.php などが存在

---

## 7. Apache 設定を変更する

---

### この工程でしていること
- .htaccess を有効化する
- WordPress のリライトルールを有効にする

---

### 実行コマンド
```bash
sudo vi /etc/httpd/conf/httpd.conf
```

---

### 設定変更
```bash
<Directory “/var/www/html”>
AllowOverride All
```

---

## 8. 画像処理用 PHP モジュールを導入する

---

### この工程でしていること
- WordPress の画像処理機能を有効化する

---

### 実行コマンド
```bash
sudo dnf install php-gd
```
```bash
sudo dnf list installed | grep php-gd
```
```bash
dnf list | grep php
```
```bash
sudo dnf install -y php8.1-gd
```
---

### 確認方法とOKの目安
- php-gd が表示される

---

## 9. Apache 公開領域の権限を設定する

---

### この工程でしていること
- Apache が WordPress を操作できるようにする
- セキュリティと運用性を両立する

---

### 実行コマンド
```bash
sudo chown -R apache /var/www
```
```bash
sudo chgrp -R apache /var/www
```
```bash
sudo chmod 2775 /var/www
```
```bash
find /var/www -type d -exec sudo chmod 2775 {} ;
```
```bash
find /var/www -type f -exec sudo chmod 0644 {} ;
```
---

### 確認方法とOKの目安
```bash
ls -ld /var/www/html
```
- apache apache になっている

---

## 10. Apache を再起動する

---

### この工程でしていること
- 設定変更を反映する

---

### 実行コマンド
```bash
sudo systemctl restart httpd
```

---

### 確認方法とOKの目安
- エラーなく起動

---

## 11. 自動起動設定・最終確認

---

### この工程でしていること
- OS 起動時にサービスを自動起動させる
- 本番運用準備を完了する

---

### 実行コマンド
```bash
sudo systemctl enable httpd
```
```bash
sudo systemctl enable mariadb
```
```bash
sudo systemctl status mariadb
```
```bash
sudo systemctl status httpd
```

---

### 確認方法とOKの目安
- 両方 active (running)
- enabled

---

## 12. Web インストール画面の確認

---

### この工程でしていること
- WordPress 初期設定画面を表示する

---

### 確認方法（ブラウザ）
`http://サーバのパブリックDNS/`

または

`http://サーバのパブリックDNS/blog/`

---

### OKの目安
- WordPress セットアップ画面が表示される
- サイト情報入力画面が表示される

---

## 13. ここまでで完了していること

- WordPress 本体導入完了
- DB 接続設定完了
- Apache 連携完了
- 権限設定完了
- 初期セットアップ準備完了

---

## 重要ポイントまとめ

- wp-config.php は最重要
- DB 権限設定は必須
- AllowOverride がないと URL が壊れる
- php-gd がないと画像不具合
- 権限ミスは最大のトラブル要因




