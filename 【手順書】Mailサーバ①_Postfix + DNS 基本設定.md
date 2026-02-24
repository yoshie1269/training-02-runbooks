# メールサーバ構築 手順書①（Postfix + DNS 基本設定）

---

## 概要

本手順書では、以下の作業を行う。

- Postfix をインストールし、SMTP サーバとして起動する
- Postfix の基本設定（main.cf）を行う
- メール用ドメインの DNS（BIND）設定を行う
- ローカル DNS を参照する設定を行い、名前解決を確認する

---

## 前提条件

- root 権限が利用できること
- Amazon Linux / RHEL 系 OS であること
- サーバ IP：172.31.37.121
- ドメイン名：onishi.local
- メールホスト名：mail.onishi.local

---

## 1. root ユーザーに切り替える

---

### この工程でしていること
- 管理者権限でパッケージ管理・設定変更を行うため root に切り替える

---

### 実行コマンド
```bash
sudo su -
```

---

### コマンドの意味
- sudo：管理者権限で実行する
- su -：root ユーザーへ完全に切り替える

---

### 確認方法とOKの目安
- プロンプトが `#` になること

---

## 2. Postfix をインストールする

---

### この工程でしていること
- SMTP メールサーバソフト（Postfix）をインストールする

---

### 実行コマンド
```bash
dnf install postfix
```

---

### コマンドの意味
- dnf install：パッケージをインストールする
- postfix：メール転送エージェント

---

### 確認方法とOKの目安
```bash
systemctl status postfix
```

- サービス情報が表示されること
- Loaded: が表示されていること

---

## 3. Postfix を起動する

---

### この工程でしていること
- Postfix サービスを起動し、SMTP サーバを稼働させる

---

### 実行コマンド
```bash
systemctl start postfix
```
```bash
systemctl enable postfix
```

---

### コマンドの意味
- start：サービスを起動する

---

### 確認方法とOKの目安
```bash
systemctl status postfix
```
```bash
systemctl is-enabled postfix
```
- Active: active (running) が表示される

---

## 4. Postfix の稼働状態を確認する

---

### この工程でしていること
- プロセスとポートの状態を確認する

---

### 実行コマンド
```bash
ps -ef | grep postfix
```
```bash
netstat -ln | grep 25
```

---

### コマンドの意味
- ps -ef：全プロセス表示
- grep postfix：Postfix 関連のみ抽出
- netstat -ln：待受ポート確認
- grep 25：SMTP（25番）抽出

---

### 確認方法とOKの目安
- postfix プロセスが表示される
- 0.0.0.0:25 などで LISTEN している

---

## 5. mailx をインストールする

---

### この工程でしていること
- メール送信用ツールを導入する

---

### 実行コマンド
```bash
dnf install mailx
```

---

### コマンドの意味
- mailx：コマンドライン用メール送信ツール

---

### 確認方法とOKの目安
```bash
dnf list installed | grep mailx
```

- mailx が表示される

---

## 6. main.cf のバックアップを取得する

---

### この工程でしていること
- Postfix 設定ファイルを退避し、復旧可能にする

---

### 実行コマンド
```bash
cp /etc/postfix/main.cf /etc/postfix/main.cf.`date "+%Y%m%d_%H%M%S"`.bak
```
```bash
ls -l /etc/postfix/ | grep main
```

---

### コマンドの意味
- cp：コピー
- date：日時取得
- .bak：バックアップ識別

---

### 確認方法とOKの目安
- main.cf.bak が存在する

---

## 7. main.cf のコメント整理

---

### この工程でしていること
- コメント行を除外して設定を簡潔化する

---

### 実行コマンド
```bash
grep -v ^# /etc/postfix/main.cf | cat -s > /tmp/main.cf
```
```bash
cp /tmp/main.cf /etc/postfix/main.cf
```

---

### コマンドの意味
- grep -v ^#：コメント行除外
- cat -s：空行圧縮
- cp：上書き保存

---

### 確認方法とOKの目安
- main.cf に不要コメントがないこと

---

## 8. main.cf を編集する

---

### この工程でしていること
- Postfix の基本動作設定を行う

---

### 実行コマンド
```bash
vi /etc/postfix/main.cf
```

---

### 追記内容
```bash
myhostname = mail.onishi.local
mydomain = onishi.local
myorigin = $myhostname
mynetworks = 172.31.0.0/16, 127.0.0.1
mail_spool_directory = /var/spool/mail/
```

---

### 変更内容
```bash
inet_interfaces = all
mydestination = $mydomain, $myhostname
```


---

### 設定の意味
- myhostname：メールサーバ名
- mydomain：ドメイン名
- myorigin：送信元ドメイン
- mynetworks：中継許可IP
- inet_interfaces：待受範囲
- mydestination：受信対象

---

### 確認方法とOKの目安
```bash
postfix check
```
- 文法ミスがない
- 設定が反映されている

---

## 9. Postfix を再起動する

```bash
systemctl restart postfix
```

### この工程でしていること
- 設定変更を反映する

---

### 実行コマンド
```bash
dnf install bind
```

---

### 確認方法とOKの目安
- named.service が存在

---

## 11. named.conf をバックアップする

---

### 実行コマンド
```bash
cp /etc/named.conf /etc/named.conf.date.`date "+%Y%m%d_%H%M%S"`.bak
```
```bash
ls -l /etc | grep named
```

---

### 確認方法
- .bak ファイルが存在

---

## 12. named.conf を編集する

---

### 実行コマンド
```bash
vi /etc/named.conf
```

### 修正内容
コメントアウト
```bash
#listen-on port 53 { 127.0.0.1; };
#listen-on-v6 port 53 { ::1; };
```
変更
```bash
allow-query { any; };
```
追記
```bash
zone "onishi.local" IN{
type master;
file "/var/named/onishi.local.zone";
};
```

---

### 確認
```bash
named-checkconf
```

- 出力なし＝正常

---

## 13. ゾーンファイルを作成する

---

### 実行コマンド
```bash
vi /var/named/onishi.local.zone
```

---

### 設定内容
```bash
$TTL 3600
@ IN SOA ns.onishi.local. test.gmail.com. (
20210401 ; serial
3600 ; refresh
3600 ; retry
3600 ; expire
3600 ) ; minimum
IN NS ns.onishi.local.
IN MX 10 mail.onishi.local.
ns IN A 172.31.37.121
mail IN A 172.31.37.121
```

---

### 意味
- NS：DNSサーバ指定
- MX：メールサーバ指定
- A：IP対応

---

## 14. ゾーンチェックと起動

---

### 実行コマンド
```bash
named-checkzone onishi.local /var/named/onishi.local.zone
systemctl start named
```

---

### 確認方法
- OK が表示
- named が active

---

## 15. DNS 動作確認（localhost）

---

### 実行コマンド
```bash
dig @localhost ns.onishi.local
dig @localhost mail.onishi.local
```

---

### OKの目安
- A レコードが返る
- status: NOERROR

---

## 16. systemd-resolved の設定

---

### この工程でしていること
- ローカル DNS を自分のサーバに向ける

---

### 実行コマンド
```bash
vi /etc/systemd/resolved.conf
```

---

### 設定
```bash
DNS=<プライベートIPアドレス>
```

---

### 再起動

```bash
systemctl restart systemd-resolved.service
systemctl status systemd-resolved.service
```

---

### 確認
```bash
dig ns.onishi.local
dig mail.onishi.loca
```

---

### OKの目安
- localhost 指定なしで解決できる

---

## 16. rsyslog をインストールする

---

### この工程でしていること
- メール送受信ログ（maillog）を記録するためのログ管理サービスを導入する
- Postfix の動作履歴を確認できるようにする

---

### 実行コマンド
```bash
dnf install rsyslog
```

---

### コマンドの意味
- dnf install
  - パッケージをインストールする
- rsyslog
  - Linux の標準ログ管理デーモン

---

### 確認方法とOKの目安
```bash
dnf list installed | grep rsyslog
```

- rsyslog が表示されること

---

## 17. rsyslog を起動する

---

### この工程でしていること
- ログ収集サービスを起動し、メールログを記録可能にする

---

### 実行コマンド
```bash
systemctl start rsyslog
systemctl status rsyslog
```

---

### コマンドの意味
- systemctl start
  - サービスを起動する
- systemctl status
  - 状態を確認する

---

### 確認方法とOKの目安
- Active: active (running) と表示されること

---

## 18. mail コマンドによるメール送信テスト（自分宛）

---

### この工程でしていること
- Postfix が正しくメールを送信できるか確認する
- まずは自分自身のドメイン宛に送信して動作確認する

---

### 実行コマンド
```bash
mail -s hello root@onishi.local
```

---

### 入力内容（本文）
```bash
Hello
.
```

※「.」のみの行を入力すると送信完了となる

---

### コマンドの意味
- mail
  - メール送信・閲覧コマンド
- -s hello
  - 件名を「hello」に設定
- root@onishi.local
  - 送信先アドレス

---

### 確認方法とOKの目安
- エラーメッセージが表示されないこと
- プロンプトに戻ること

---

## 19. メールログの確認

---

### この工程でしていること
- メール送信処理が正常に行われたかログで確認する
- Postfix の内部動作を検証する

---

### 実行コマンド
```bash
less /var/log/maillog
```

---

### コマンドの意味
- less
  - ファイル内容をページ表示する
- /var/log/maillog
  - メール関連ログファイル

---

### 確認方法とOKの目安

ログ内に以下のような情報があること。

```bash
例：postfix/smtp[...] status=sent
　　または
　　postfix/local[...] status=sent
```

意味：
- status=sent → 正常送信完了

---

## 20. 受信メールの確認

---

### この工程でしていること
- 自分宛に送ったメールを実際に受信できているか確認する

---

### 実行コマンド
```bash
mail
```

---

### 表示例
```bash
1 hello root@onishi.local
```

---

### コマンドの意味
- mail
  - 受信メール一覧を表示する

---

### 確認方法とOKの目安
- 件名「hello」のメールが表示される
- 送信した本文「Hello」が確認できる

---

## 21. メール本文の閲覧

---

### この工程でしていること
- 受信したメールの内容を確認する

---

### 実行コマンド（mail画面内）
```bash
& number
```

（例：1番を読む場合 → & 1）

---

### コマンドの意味
- &
  - mail コマンドの操作モード
- number
  - 読みたいメール番号

---

### 確認方法とOKの目安
- 本文「Hello」が表示される
- 文字化けや欠損がない

---

## 22. 他サーバ宛メール送信テスト（ペア確認）

---

### この工程でしていること
- DNS 設定を変更し、別サーバのメールサーバを参照するようにする
- 外部（ペア相手）とのメール送受信が可能か検証する

---

### 作業内容（概要）

1. DNS の参照先をペア相手の DNS に変更する
2. 名前解決が正しく切り替わったことを確認する
3. 相手のドメイン宛に mail で送信する

---

### 実行例（送信）
```bash
mail -s test user@pair.local
```

---

### 確認方法とOKの目安

#### 送信側
- エラーが出ない
- maillog に status=sent が記録される

#### 受信側
- 相手がメールを受信できる

---

## 23. トラブル時の確認ポイント（重要）

---

### メールが届かない場合の確認順

1. Postfix の状態確認  
   - systemctl status postfix

2. DNS 解決確認  
   - dig mail.onishi.local

3. ポート確認  
   - netstat -ln | grep 25

4. ログ確認  
   - less /var/log/maillog

5. rsyslog 状態確認  
   - systemctl status rsyslog

---

## 23. メール専用ユーザーを作成する

---

### この工程でしていること
- メール受信専用の Linux ユーザーを作成する
- SSH やコンソールログインを禁止する
- ホームディレクトリを作成しない設定にする
- メール保存先を無効化し、システムへの影響を最小化する

---

### 実行コマンド
```bash
useradd yoshie -g mail -M -K MAIL_DIR=/dev/null -s /sbin/nologin
```


---

### コマンドの意味

- useradd  
  - 新しい Linux ユーザーを作成するコマンド

- yoshie  
  - 作成するユーザー名（メールアカウント名）

- -g mail  
  - プライマリグループを mail グループに設定する

- -M  
  - ホームディレクトリを作成しない

- -K MAIL_DIR=/dev/null  
  - メール保存ディレクトリを無効化する設定
  - /dev/null に設定することで、不要なメールディレクトリを作らない

- -s /sbin/nologin  
  - ログインシェルを無効化する
  - SSH や su によるログインを禁止する

---

### 確認方法とOKの目安

#### ユーザーの存在確認
```bash
id yoshie
```

OKの目安：
- uid / gid が表示される
- グループに mail が含まれている

例：uid=1002(yoshie) gid=12(mail) groups=12(mail)


---

#### ログイン不可の確認（任意）

```bash
su - yoshie
```


OKの目安：
- `This account is currently not available` などと表示される
- ログインできないこと

---

## 24. ユーザーのパスワードを設定する

---

### この工程でしていること
- メール送信・認証等で使用するためにパスワードを設定する
- OS ログインには使用できないが、メール用認証に利用できる

---

### 実行コマンド
```bash
passwd yoshie
```

---

### コマンドの意味

- passwd  
  - ユーザーのパスワードを設定・変更する

- yoshie  
  - 対象ユーザー名

---

### 設定時の注意点

- 強力なパスワードを設定すること
- 他人に知られないように管理すること
- メモに残す場合は厳重に管理すること

---

### 確認方法とOKの目安

#### パスワード設定結果
OKの目安：
```bash
passwd: all authentication tokens updated successfully.
```

と表示されること

---

#### パスワード状態の確認（任意）
```bash
passwd -S yoshie
```

OKの目安：
- `PS`（Password Set）と表示される

例：yoshie PS 2026-01-23 0 99999 7 -1


---

## 25. この設定の目的とメリット（重要）

---

### セキュリティ面のメリット

| 項目 | 内容 |
|------|------|
| ログイン制限 | SSH / コンソールログイン不可 |
| 権限最小化 | mail グループのみ所属 |
| ホームなし | 不要な領域を使わない |
| メール専用 | 目的外利用を防止 |

---
