# メールサーバ構築 手順書②（Dovecot：POP3 受信設定）完全版

---

## 概要

本手順書では、Postfix と連携し、POP3 によるメール受信を可能にするため  
Dovecot を導入・設定する。

本構成では：

- Postfix → /var/spool/mail に配送
- Dovecot → /var/spool/mail を参照してPOP3配信
- telnet で動作確認

を行う。

---

## 構成図
```bash
[Postfix]
   ↓ ローカル配送
/var/spool/mail/yoshie
   ↑ 参照
[Dovecot (POP3:110)]
   ↑
telnet / メールクライアント
```

---

## 前提条件

- Postfix が稼働中
- メールユーザー作成済（例：yoshie）
- /var/spool/mail にメールが存在
- root で作業

---

# 1. Dovecot をインストールする

## この工程でしていること
- POP3/IMAP サーバを導入する
```bash
dnf install dovecot
```

確認：
```bash
dnf list installed | grep dovecot
```

OK目安：
- dovecot が表示される

---

# 2. dovecot.conf をバックアップし基本設定を行う

## この工程でしていること
- メイン設定ファイルを退避
- POP3のみ有効化
- メール保存先を /var/spool/mail に指定
```bash
cp /etc/dovecot/dovecot.conf /etc/dovecot/dovecot.conf.date "+%Y%m%d_%H%M%S".bak
```

確認：
```bash
ls -l /etc/dovecot/ | grep dovecot
```

編集：
```bash
vi /etc/dovecot/dovecot.conf
```

### 変更内容①（プロトコル指定）

変更前：
```bash
#protocols = imap pop3 lmtp
```

変更後：
```bash
protocols = pop3
```

### 変更内容②（メール保存先追記）

追記：
```bash
mail_location = maildir:/var/spool/mail/%u/
```
---

## 各設定の意味

### protocols = pop3
- Dovecotが提供する受信方式をPOP3のみに制限
- IMAPは無効化

### mail_location = mail:/var/spool/mail/%u/
- Dovecotが参照する受信箱の場所を指定
- %u はユーザー名に置換
- 例：/var/spool/mail/yoshie

---

## 確認方法
```bash
grep -nE '^(protocols|mail_location)' /etc/dovecot/dovecot.conf
```
OK目安：
- protocols = pop3
- mail_location が表示される

---

# 3. 10-ssl.conf をバックアップ

## この工程でしていること
- SSL設定変更前に退避

```bash
cp /etc/dovecot/conf.d/10-ssl.conf \
/etc/dovecot/conf.d/10-ssl.conf.$(date +%Y%m%d_%H%M%S).bak
```

確認：
```bash
ls -l /etc/dovecot/conf.d/ | grep 10-ssl
```

---

# 4. SSL必須設定を無効化（学習用）

## この工程でしていること
- 平文POP3で接続できるようにする
```bash
vi /etc/dovecot/conf.d/10-ssl.conf
```
変更前：
```bash
ssl = required
```
変更後：
```bash
#ssl = required
```
確認：
```bash
grep -n '^#ssl = required' /etc/dovecot/conf.d/10-ssl.conf
```
---

# 5. 10-auth.conf をバックアップ
```bash
cp /etc/dovecot/conf.d/10-auth.conf \
/etc/dovecot/conf.d/10-auth.conf.$(date +%Y%m%d_%H%M%S).bak
```

確認：
```bash
ls -l /etc/dovecot/conf.d/10-auth.conf*
```
---

# 6. 平文認証を許可する（学習用）

## この工程でしていること
- SSL無しでUSER/PASS認証を可能にする
```bash
vi /etc/dovecot/conf.d/10-auth.conf
```

変更前：
```bash
disable_plaintext_auth = yes
```

変更後：
```bash
disable_plaintext_auth = no
```

確認：
```bash
grep -n '^disable_plaintext_auth' /etc/dovecot/conf.d/10-auth.conf
```
---

# 7. Dovecot を起動・自動起動設定

## この工程でしていること
- POP3待受開始（110番）
```bash
systemctl start dovecot
```
```bash
systemctl enable dovecot
```

確認：
```bash
systemctl status dovecot
```

OK目安：
- active (running)

---

# 8. ポート確認

## この工程でしていること
- 110番で待受しているか確認
```bash
netstat -ln | grep 110
```

OK目安：
- 0.0.0.0:110 LISTEN

---

# 9. telnet をインストール
```bash
dnf install telnet
```
---

# 10. POP3 接続確認

## この工程でしていること
- Dovecotが応答するか確認
```bash
telnet localhost 110
```

OK表示例：

+OK Dovecot ready.

---

# 11. ユーザー認証確認

（telnet接続後）
```bash
user <ユーザ名>
```
```bash
pass <パスワード>
```

OK表示：

+OK Logged in.

---

# 12. メール一覧確認
```bash
list
```
OK例：

+OK 2 messages

1 1024

2 2048

.

---

# 13. メール本文取得
```bash
retr 2
```
OK目安：
- From / Subject / 本文表示

---

# 14. セッション終了
```bash
quit
```
OK表示：

+OK Logging out

---

# トラブルシュート

## 110番に接続できない
```bash
systemctl status dovecot
```
```bash
netstat -ln | grep 110
```
---

## 認証失敗
```bash
id yoshie
```
```bash
grep disable_plaintext_auth /etc/dovecot/conf.d/10-auth.conf
```
---

## メール0件
```bash
ls -l /var/spool/mail/
```
```bash
ls -l /var/spool/mail/yoshie
```
---

# 完了状態

- Postfix → /var/spool/mail に配送
- Dovecot → POP3(110)で待受
- USER/PASS 認証成功
- list / retr 成功
- ローカルメール受信確認完了

