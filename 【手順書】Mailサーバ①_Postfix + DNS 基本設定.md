# メールサーバ構築 手順書①（Postfix + DNS 基本設定）

---

## 構成図
```bash
[クライアント]
   ↓ SMTP(25)
[Postfix（172.31.37.121）]
   ↓
[ローカル配送 /var/spool/mail]
```
```bash
[DNS(BIND)]
mail.onishi.local → 172.31.37.121
ns.onishi.local   → 172.31.37.121
```
---

## 前提条件

| 項目 | 内容 |
|------|------|
| OS | Amazon Linux / RHEL系 |
| サーバIP | 172.31.37.121 |
| ドメイン | onishi.local |
| メールホスト | mail.onishi.local |

---

# 1. rootへ切り替え

## この工程でしていること
- システム設定変更を行うため管理者権限を取得

## コマンド
```bash
sudo su -
```
## 確認方法
- プロンプトが `#` になる

---

# 2. Postfix インストール

## この工程でしていること
- SMTPサーバ(Postfix)を導入

## コマンド
```bash
dnf install postfix -y
```
## 確認
```bash
systemctl status postfix
```
OK目安：
- Loaded 表示

---

# 3. Postfix 起動

## この工程でしていること
- SMTPサービスを稼働させる
```bash
systemctl start postfix
```
```bash
systemctl enable postfix
```

## 確認
```bash
systemctl status postfix
```
```bash
systemctl is-enabled postfix
```
OK：
- active (running)
- enabled

---

# 4. ポート・プロセス確認

## この工程でしていること
- 25番ポートで待受しているか確認
```bash
ps -ef | grep postfix
```
```bash
netstat -ln | grep 25
```

OK：
- postfixプロセス表示
- 0.0.0.0:25 LISTEN

---

# 5. mailx インストール

## この工程でしていること
- テスト送信用ツール導入
```bash
dnf install mailx -y
```
確認：
```bash
dnf list installed | grep mailx
```
---

# 6. main.cf バックアップ

## この工程でしていること
- 設定変更前に退避
```bash
cp /etc/postfix/main.cf /etc/postfix/main.cf.$(date “+%Y%m%d_%H%M%S”).bak
```
---

# 7. main.cf 整理

## この工程でしていること
- コメント削除し見やすくする
```bash
grep -v ^# /etc/postfix/main.cf | cat -s > /tmp/main.cf
```
```bash
cp /tmp/main.cf /etc/postfix/main.cf
```
---

# 8. main.cf 編集

## この工程でしていること
- メールサーバの基本定義を行う
```bash
vi /etc/postfix/main.cf
```
### 追記
```bash
myhostname = mail.onishi.local
mydomain = onishi.local
myorigin = $myhostname
mynetworks = 172.31.0.0/16, 127.0.0.1
mail_spool_directory = /var/spool/mail/
```
### 変更
```bash
inet_interfaces = all
mydestination = $mydomain, $myhostname
```
---

## 各設定の意味

| パラメータ | 意味 |
|------------|------|
| myhostname | このメールサーバの正式名 |
| mydomain | 管理ドメイン |
| myorigin | 送信時のドメイン付与 |
| mynetworks | 中継許可ネットワーク |
| inet_interfaces | どのIPで待受するか |
| mydestination | 自分が受信対象となるドメイン |

---

## 設定確認
```bash
postfix check
```
OK：
- エラーなし

---

# 9. Postfix 再起動

## この工程でしていること
- 設定反映
```bash
systemctl restart postfix
```
---

# 10. BIND インストール

## この工程でしていること
- 独自DNSサーバ構築
```bash
dnf install bind -y
```
```bash
systemctl start named
```
```bash
systemctl enable named
```

確認：
```bash
systemctl status named
```
---

# 11. named.conf バックアップ
```bash
cp /etc/named.conf /etc/named.conf.$(date “+%Y%m%d_%H%M%S”).bak
```
---

# 12. named.conf 編集

## この工程でしていること
- 権威DNS設定
- 外部から問い合わせ可能にする
```bash
vi /etc/named.conf
```
### コメントアウト
```bash
#listen-on port 53 { 127.0.0.1; };
#listen-on-v6 port 53 { ::1; };
```
### 変更
```bash
allow-query { any; };
```

### 追記
```bash
zone “onishi.local” IN {
type master;
file “/var/named/onishi.local.zone”;
};
```

確認：
```bash
named-checkconf
```
OK：
- 出力なし

---

# 13. ゾーンファイル作成

## この工程でしていること
- ドメインとIPの対応を定義
```bash
vi /var/named/onishi.local.zone
```

### 内容
```bash
$TTL 3600
@ IN SOA ns.onishi.local. test.gmail.com. (
20210401
3600
3600
3600
3600 )

IN NS ns.onishi.local.
IN MX 10 mail.onishi.local.

ns   IN A 172.31.37.121
mail IN A 172.31.37.121
```

---

## 各レコードの意味

| レコード | 意味 |
|----------|------|
| SOA | ゾーン管理情報 |
| NS | 権威DNS指定 |
| MX | メール配送先指定 |
| A | 名前→IP変換 |

---

## Aレコード重要ポイント

mail IN A 172.31.37.121

意味：
- mail.onishi.local → 172.31.37.121 に変換
- SMTP接続時にIP解決に使われる

ns IN A 172.31.37.121

意味：
- DNSサーバ自身のIP定義

---

## 権限設定
```bash
chown root:named /var/named/onishi.local.zone
```
```bash
chmod 640 /var/named/onishi.local.zone
```

OK例：

-rw-r—– 1 root named

---

# 14. ゾーンチェック
```bash
named-checkzone onishi.local /var/named/onishi.local.zone
```
```bash
systemctl restart named
```

OK：
- OK表示

---

# 15. DNS動作確認
```bash
dig @localhost ns.onishi.local
```
```bash
dig @localhost mail.onishi.local
```

OK：
- status: NOERROR
- ANSWER SECTIONにAレコード表示

---

# 16. systemd-resolved 設定

## この工程でしていること
- 自サーバをDNS参照先にする
```bash
vi /etc/systemd/resolved.conf
```

追加：
```bash
DNS=172.31.37.121
```

```bash
systemctl restart systemd-resolved
```

確認：
```bash
dig mail.onishi.local
```
OK：
- localhost指定なしで解決

---

# 17. rsyslog インストール

## この工程でしていること
- メールログ記録有効化
```bash
dnf install rsyslog -y
```
```bash
systemctl start rsyslog
```

確認：
```bash
systemctl status rsyslog
```

---

# 18. 自分宛メール送信テスト
```bash
mail -s hello root@onishi.local
```
```bash
Hello
.
```
---

# 19. ログ確認
```bash
less /var/log/maillog
```

OK例：
```bash
status=sent
```

---

# 20. 受信確認
```bash
mail
```
メール閲覧：
```bash
& 1
```

---

# 21. メール専用ユーザー作成

## この工程でしていること
- ログイン不可のメール専用アカウント作成
```bash
useradd yoshie -g mail -M -K MAIL_DIR=/dev/null -s /sbin/nologin
```
```bash
passwd yoshie
```

確認：
```bash
id yoshie
```
```bash
passwd -S yoshie
```
OK：
- PS表示
- ログイン不可

---

# 完了状態

- Postfix稼働
- 25番ポートLISTEN
- DNS名前解決成功
- MX解決成功
- ローカルメール送受信成功
- メールログ確認可能
- メール専用ユーザー作成済

---

# 本質理解まとめ

| 要素 | 役割 |
|------|------|
| Postfix | SMTP配送 |
| DNS A | 名前→IP |
| MX | メール配送先指定 |
| mydestination | 自分が受信するドメイン |
| rsyslog | ログ記録 |
| mail | テスト送受信 |
