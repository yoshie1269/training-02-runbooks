# DNSサーバ構築 手順書（BIND / named）

---

## 前提
- root 権限が利用できること
- ネットワークグループを DNS (UDP) に設定する

---

## 1. root に切り替える

### 1-1. root になる
#### この工程でしていること
- 以降のインストール・設定ファイル編集・サービス操作に root 権限が必要なため、root に切り替える

#### 実行コマンド

```bash
sudo su -
```

#### OKの目安：

- プロンプトが `#` になる（例：`[root@xxxx ~]#`）

---

## 2. BIND（named）をインストールする

### 2-1. パッケージをインストールする
#### この工程でしていること
- DNSサーバソフト（BIND）と、DNS確認用ツール類をインストールする  
- named のサービス単位（systemd）を使えるようになる

#### 実行コマンド

```bash
dnf install bind -y
```

#### コマンドの意味
- `dnf install`：パッケージをインストールする
- `bind`：named（DNSサーバ本体）
- `-y`：確認（yesの入力）を省略

#### 確認方法
- インストール完了後にエラーが出ないこと
- サービス定義が存在することを確認する

```bash
systemctl list-unit-files | grep named
```

OKの目安：
- `named.service` が表示される

---

## 3. named サービスの起動と稼働確認

### 3-1. 起動前の状態確認をする
#### この工程でしていること
- named が起動しているかどうかを確認する（起動前の状態確認）

#### 実行コマンド

```bash
systemctl status named
```

#### 見方（OK/NGの目安）
- 起動前は `inactive (dead)` などが表示される
- `Loaded:` が表示され、サービスとして認識されていることが重要

---

### 3-2. named を起動する
#### この工程でしていること
- DNSサーバ（named）を起動する

#### 実行コマンド

```bash
systemctl start named
```

#### コマンドの意味
- `systemctl start`：サービスを起動する
- `named`：DNSサーバデーモン

---

### 3-3. 起動後の状態確認をする
#### この工程でしていること
- named が起動し、稼働中であることを確認する

#### 実行コマンド
```bash
systemctl status named
```

#### OKの目安：
- `Active: active (running)` が表示される

---

### 3-4. プロセスが存在することを確認する
#### この工程でしていること
- named のプロセスが実際に動いていることを確認する

#### 実行コマンド
```bash
ps -ef | grep named
```

#### コマンドの意味
- `ps -ef`：全プロセスを詳細表示する
- `grep named`：named を含む行だけ抽出する

#### Kの目安：
- `named` プロセスが表示される  
  （例：`/usr/sbin/named ...` のような行がある）

注意：
- `grep named` 自体の行も表示されることがある  
  （判別は `/usr/sbin/named` などの本体プロセスがあるかで行う）

---

### 3-5. 53番ポート待ち受けを確認する
#### この工程でしていること
- DNS の待ち受けポート（TCP/UDP 53）で、named が LISTEN していることを確認する

#### 実行コマンド

```bash
netstat -ln | grep 53
```

#### コマンドの意味
- `netstat -ln`：待ち受け（LISTEN/受信待ち）状態のポートを表示する
- `grep 53`：53番ポート関連のみ抽出する

#### OKな目安：
- `127.0.0.1:53`の UDP の 53 が表示される  
（ローカル限定で待ち受けている状態）

---

## 4. named.conf のバックアップと編集

### 4-1. named.conf をバックアップする
#### この工程でしていること
- 設定変更前の状態へ戻せるように、設定ファイルのバックアップをとる

#### 実行コマンド
```bash
cp /etc/named.conf{,.bak}
```

#### コマンドの意味
- `cp`：コピーする
- `/etc/named.conf{,.bak}`：
  - コピー元：`/etc/named.conf`
  - コピー先：`/etc/named.conf.bak`
  をまとめて書いている

#### 確認方法
- 一覧表示でバックアップが作成されているこを確認する

    ```bash
    ls /etc/ | grep named
    ```
OKの目安：
- `named.conf` と `named.conf.bak` が表示される

---

### 4-2. named.conf を編集する
#### この工程でしていること
- named の基本動作設定（待ち受けインタフェース・問い合わせ許可）を変更する

#### 実行コマンド
```bash
vi /etc/named.conf
```

---

## 5. named.conf の変更内容

### 5-1. listen-on / listen-on-v6 の制限を外す（コメントアウト）
#### 目的（何をしているのか）
- 初期設定では、named が **localhost（127.0.0.1 / ::1）でしか待ち受けない** ように制限されていることがある  
- その制限を外し、外部からの問い合わせを受けられるようにする

#### 変更前
```bash
listen-on port 53 { 127.0.0.1; };
listen-on-v6 port 53 { ::1; };
```

#### 変更後（コメントアウト）
```bash
#listen-on port 53 { 127.0.0.1; };
#listen-on-v6 port 53 { ::1; };
```

---

### 5-2. allow-query を any に変更する
#### 目的（何をしているのか）
- 初期設定では `localhost` のみ許可になっていることがある  
- 外部ホストからの DNS クエリを受けるため、許可範囲を広げる

#### 変更前
```bash
allow-query { localhost; };
```

#### 変更後
```bash
allow-query { any; };
```

#### 確認方法（OKの目安）
```bash
named-checkconf
```
- 何も表示されなければOK

#### 注意点（重要）
- `any` は「誰からの問い合わせも許可」になる  
- 実務では許可範囲（送信元IP）を必要最小限に制限するのが基本  

---

## 6. named の再起動と最終確認

### 6-1. named を再起動する
#### この工程でしていること
- named.conf の変更をサービスへ反映する

#### 実行コマンド
```bash
systemctl restart named
```

#### コマンドの意味
- `restart`：停止 → 起動を行い、設定変更を確実に反映する

#### 確認方法（OKの目安）
- 起動状態を確認する

    ```bash
    systemctl status named
    ```

OKの目安：
- `Active: active (running)`

---

### 6-2. 53番ポート待ち受けの最終確認をする
#### この工程でしていること
- 設定変更後に、named が 53番で待ち受ける範囲が想定どおりになったかを確認する

#### 実行コマンド
```bash
netstat -ln | grep 53
```

#### OKの目安
- 変更前は `127.0.0.1:53` だけだったものが、以下のように表示される  
  - `0.0.0.0:53`（全IPv4で待ち受け）
  - `:::53`（全IPv6で待ち受け）
  - もしくはサーバのIPアドレスで 53 を待ち受けている表示


## 7. named.conf にゾーン定義を追加する

### 7-1. named.conf を編集する
#### この工程でしていること
- DNS サーバに対して「hogehoge.com というドメインを管理する（master）」ことを定義する
- ゾーン情報をどのファイルから読み込むかを指定する

#### 実行コマンド

```bash
vi /etc/named.conf
```

---

### 7-2. ゾーン定義を追加する
#### 追加内容
```bash
zone "hogehoge.com" IN {
type master;
file "/var/named/hogehoge.com.zone";
};
```

#### 各行の意味
- `zone "hogehoge.com" IN`
  - 管理対象のドメイン名（ゾーン名）を指定する
  - `IN` はインターネットクラスを意味する

- `type master;`
  - この DNS サーバが **正規の管理サーバ（マスター）** であることを示す

- `file "/var/named/hogehoge.com.zone";`
  - ゾーン情報を記述したファイルのパスを指定する

---

## 8. named.conf の構文チェック

### 8-1. named-checkconf を実行する
#### この工程でしていること
- named.conf の文法が正しいかを確認する
- サービス再起動前にエラーを検出するための重要な工程

#### 実行コマンド
```bash
named-checkconf
```

#### コマンドの意味
- `named-checkconf`
  - named の設定ファイル（named.conf）をチェックする
  - デフォルトで `/etc/named.conf` を参照する

#### OKなことを示す結果
- **何も出力されないこと**
  - 出力なし ＝ 文法エラーなし

#### NGの例
- 行番号付きのエラーメッセージが表示される  
  → 設定内容を見直す必要がある

---

## 9. ゾーンファイルを作成する

### 9-1. ゾーンファイルを編集する
#### この工程でしていること
- hogehoge.com ドメインに対する DNS レコード（SOA / NS / A）を定義する

#### 実行コマンド
```bash
vi /var/named/hogehoge.com.zone
```

---

### 9-2. ゾーンファイルの内容

```bash
$TTL 3600
@ IN SOA ns.hogehoge.com. test.gmail.com. (
20220401 ; serial
3600 ; refresh
3600 ; retry
3600 ; expire
3600 ) ; minimum

 IN NS ns.hogehoge.com.
 ns IN A LinuxのIPアドレス
 www IN A LinuxのIPアドレス
```
---

#### ゾーンファイルの内容説明
```bash
$TTL 3600
→ DNSのキャッシュ有効時間（秒）
→ このゾーン情報は 3600秒（1時間）キャッシュされる

@ IN SOA ns.hogehoge.com. test.gmail.com. (
→ ゾーンの管理情報（Start Of Authority）
→ DNSゾーンの「責任者」と「管理サーバ」を定義

ns.hogehoge.com.
→ このゾーンの管理DNSサーバ（ネームサーバ）

test.gmail.com.
→ 管理者のメールアドレス
→「@」の代わりに「.」で表記している

20220401 ; serial
→ シリアル番号（更新番号）
→ ゾーン変更時に必ず増やす

3600 ; refresh
→ セカンダリDNSが更新確認する間隔（秒）

3600 ; retry
→ 更新失敗時の再試行間隔（秒）

3600 ; expire
→ 同期できない場合の有効期限（秒）

3600 ) ; minimum
→ 最小TTL（ネガティブキャッシュ用）

------------------------------------------------

IN NS ns.hogehoge.com.
→ このドメインのネームサーバ指定
→ 「このDNSが管理している」という宣言

------------------------------------------------

ns IN A LinuxのIPアドレス
→ ns.hogehoge.com のIPアドレス登録
→ ネームサーバ自身のIP

------------------------------------------------

www IN A LinuxのIPアドレス
→ www.hogehoge.com のIPアドレス登録
→ Webサーバへの対応付け

------------------------------------------------

# 重要ポイント

・SOAは必須レコード
・serialは更新のたびに変更する
・NSとAはセットで必要
・TTLはキャッシュ時間
・@ はドメイン自身を表す
```

---

#### 確認方法（OKの目安）
```bash
named-checkzone hogehoge.com /var/named/hogehoge.com.zone
```
- 何も表示されなければOK

---

### 9-3. named を再起動する
#### この工程でしていること
- named.conf およびゾーンファイルの設定内容を、DNS サービスに反映する
- 設定変更後は **必ず再起動** が必要

#### 実行コマンド

```bash
systemctl restart named
```

---

### 9-4. www.hogehoge.com を問い合わせる
#### この工程でしていること
- localhost（自分自身の DNS サーバ）に対して
- www.hogehoge.com の名前解決を要求する

#### 実行コマンド

```bash
dig @localhost www.hogehoge.com
```

#### コマンドの意味
- `dig`
  - DNS 問い合わせツール
- `@localhost`
  - 問い合わせ先の DNS サーバを指定する
  - 今回は **自分自身の named**
- `www.hogehoge.com`
  - 問い合わせるホスト名（FQDN）

---

### 9-5. 出力結果の見方

#### 確認すべき主なポイント

##### ① QUESTION SECTION

```bash
;; QUESTION SECTION:
;www.hogehoge.com. IN A
```

- A レコードを問い合わせていることを示す

---

##### ② ANSWER SECTION（最重要）

```bash
;; ANSWER SECTION:
www.hogehoge.com. 3600 IN A 172.31.18.221
```

#### OKなことを示す条件
- `ANSWER SECTION` が存在する
- `www.hogehoge.com` に対して
- `A 172.31.18.221` が返ってくる

→ ゾーンファイルの A レコード設定が正しく反映されている

---

##### ③ status

```bash
;; ->>HEADER<<- opcode: QUERY, status: NOERROR
```


- `status: NOERROR`
  - 問い合わせが正常に処理されたことを示す

---

### 3-3. NGパターン例
- `status: NXDOMAIN`
  - ドメイン名が存在しない
  - ゾーン定義やレコード名の誤り
- `ANSWER SECTION` が空
  - レコードが未定義
- タイムアウト
  - named が起動していない
  - allow-query / listen-on の設定不備

---

### 9-6. ns.hogehoge.com を問い合わせる
#### この工程でしていること
- DNS サーバ自身（ns.hogehoge.com）の A レコードを確認する
- NS レコードと A レコードの整合性を確認する

#### 実行コマンド
```bash
dig @localhost ns.hogehoge.com
```

---

### 9-7. 出力結果の確認ポイント

#### ANSWER SECTION の例

```bash
;; ANSWER SECTION:
ns.hogehoge.com. 3600 IN A 172.31.18.221
```


#### OKなことを示す条件
- `ns.hogehoge.com` が
- `172.31.18.221` に解決される
- status が `NOERROR`

なぜ ns レコード確認が重要か
- NS レコードで指定した DNS サーバ名は
  - **必ず A レコード（または AAAA）を持つ必要がある**
- これがないと、ゾーンは正しく動作しない

---

## 10. トラブルシュート時の確認順

1. systemctl status named  
2. named-checkconf  
3. named-checkzone  
4. dig @localhost 対象ホスト名  
5. ポート 53 の LISTEN 状態確認

---

## ポイントまとめ

- 設定変更後は必ず named を再起動する
- dig は DNS 動作確認の基本ツール
- @localhost を付けることで「どの DNS に聞いているか」が明確になる
- ANSWER SECTION が正しく返ることが最重要
- NS レコードと A レコードの整合性は必須


