# Zabbix 7.4 構築手順書（Amazon Linux 2023 / Apache / MySQL）

---

## 概要

本手順書では、Amazon Linux 2023 上に Zabbix 7.4 を構築する。

構成：
- OS：Amazon Linux 2023
- DB：MariaDB (MySQL)
- Web：Apache
- Zabbix：7.4
- DB 同一サーバ構成

---

## ① タイムゾーン設定

---

#### この工程でしていること
- ログ時刻を日本時間に統一する
- 監視ログ・障害時刻の整合性を保つ

---

#### 実行コマンド
```bash
timedatectl set-timezone Asia/Tokyo
```
---

#### コマンドの意味
- timedatectl：時刻設定管理コマンド
- set-timezone：タイムゾーン変更

---

#### 確認方法とOKの目安
```bash
timedatectl
```
Time zone: Asia/Tokyo と表示されること

---

## ② Zabbix ダウンロード（公式リポジトリ追加）

---

#### この工程でしていること
- Zabbix 7.4 の公式パッケージを利用可能にする
- OS とバージョンを一致させる

---

#### 公式ページ確認
```bash
https://www.zabbix.com/download?zabbix=7.4&os_distribution=amazon_linux&os_version=2023&components=server_frontend_agent&db=mysql&ws=apache
```
---

##### 実行コマンド
```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.4/release/amazonlinux/2023/noarch/zabbix-release-latest-7.4.amzn2023.noarch.rpm
dnf clean all
```
---

#### コマンドの意味

| コマンド | 意味 |
|-----------|--------|
| rpm -Uvh | リポジトリ追加 |
| dnf clean all | キャッシュ削除 |

---

#### 確認方法
```bash
dnf repolist | grep zabbix
```
zabbix が表示されること

---

## ③ Zabbix インストール

---

#### この工程でしていること
- Zabbix Server / Web / Agent をインストールする

---

#### 実行コマンド
```bash
dnf install zabbix-server-mysql zabbix-web-mysql zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```
---

#### 確認方法
```bash
dnf list installed | grep zabbix
```
必要パッケージが表示されること

---

## ④ DB 初期化

---

#### この工程でしていること
- Zabbix 用データベースを作成
- ユーザー作成
- SQL スキーマ投入

---

### 1. DB 作成
```bash
mysql -uroot -p
```

```bash
create database zabbix character set utf8mb4 collate utf8mb4_bin;
create user zabbix@localhost identified by ‘zabbix’;
grant all privileges on zabbix.* to zabbix@localhost;
set global log_bin_trust_function_creators = 1;
quit;
```
---

#### 各コマンドの意味

| コマンド | 意味 |
|-----------|--------|
| create database | DB 作成 |
| create user | DB ユーザー作成 |
| grant | 権限付与 |
| log_bin_trust... | 関数作成許可 |

---

### 2. SQL スキーマ投入
```bash
zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mysql –default-character-set=utf8mb4 -uzabbix -p zabbix
```
---

#### 意味
- server.sql.gz：初期テーブル定義
- zcat：解凍しながら実行
- mysql パイプ：DBへ流し込み

---

### 3. log_bin_trust を戻す
```bash
mysql -uroot -p
set global log_bin_trust_function_creators = 0;
quit;
```
---

### 4. DB 初期化確認
```bash
mysql -uzabbix -p zabbix -e “show tables like ‘users’;”
```
users が表示されれば成功

---

## ⑤ Zabbix Server 設定

---

#### この工程でしていること
- DB 接続情報を設定する

---

#### 実行コマンド
```bash
cp /etc/zabbix/zabbix_server.conf /etc/zabbix/zabbix_server.conf.bak
```
```bash
vi /etc/zabbix/zabbix_server.conf
```

---

#### 設定内容
```bash
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=zabbix
```
---

#### 意味
- DBHost：同一サーバのため localhost
- DBPassword：DBで設定した値

---

## ■ ⑥ サービス起動

---

#### 実行コマンド
```bash
systemctl restart zabbix-server zabbix-agent httpd php-fpm
```
```bash
systemctl enable zabbix-server zabbix-agent httpd php-fpm
```
---

#### 確認方法
```bash
systemctl status zabbix-server
```
active (running) であること

---

## ⑧ Zabbix Agent 設定（同一サーバ）

---

#### 実行コマンド
```bash
cp /etc/zabbix/zabbix_agentd.conf /etc/zabbix/zabbix_agentd.conf.org
```
```bash
vi /etc/zabbix/zabbix_agentd.conf
```
---

#### 設定内容
```bash
Server=127.0.0.1
ServerActive=127.0.0.1
Hostname=ip-172-31-30-185.us-west-2.compute.internal
```
---

#### 意味

| 項目 | 意味 |
|--------|--------|
| Server | 接続許可 |
| ServerActive | Active監視 |
| Hostname | ホスト識別名 |

---

## ⑩ 動作確認

---

#### 実行コマンド
```bash
systemctl status zabbix-server
ls /run/zabbix/
```
---

#### OK目安
- active (running)
- zabbix_server.pid が存在

---

## ⑪ Web UI 確認

---

#### アクセスURL

http://<IPアドレス>/zabbix

---

#### 初期ログイン

User : Admin  
Pass : zabbix  

---

## 注意事項（重要）

---

| 項目 | 理由 |
|------|------|
| 6.0 と 7.4 混在禁止 | SQL不一致で起動失敗 |
| SQL投入は1回のみ | 重複投入でエラー |
| DBHost=localhost | 同一構成 |
| DB未初期化は起動失敗 | テーブル不足 |
| バージョンとSQL一致 | 互換性問題回避 |

---

## 完成状態

✔ Zabbix Server 起動  
✔ DB 初期化完了  
✔ Agent 接続完了  
✔ Web UI 表示可能  

＝ Zabbix 7.4 構築完了


⸻
