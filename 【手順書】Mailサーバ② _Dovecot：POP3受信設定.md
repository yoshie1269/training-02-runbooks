# メールサーバ構築 手順書②（Dovecot：POP3 受信設定）

---

## 概要

本手順書では、Postfix と連携してメールを受信できるようにするため、  
POP3 サーバソフト **Dovecot** を導入・設定する。

本工程により、以下が可能となる。

- メールクライアントから POP3 で受信できるようになる
- /var/spool/mail のメールを読み取れるようにする
- 認証設定を調整し、ログイン可能な状態にする

---

## 前提条件

- Postfix が正常に稼働していること
- メールユーザーが作成済みであること（例：yoshie）
- /var/spool/mail にメールが保存されていること
- root 権限で作業していること

---

## 1. Dovecot をインストールする

---

### この工程でしていること
- POP3 / IMAP サーバソフトである Dovecot をインストールする
- メール受信用サービスを追加する

---

### 実行コマンド
```bash
dnf install dovecot
```

---

### コマンドの意味
- dnf install  
  - パッケージをインストールする
- dovecot  
  - メール受信サーバ（POP3 / IMAP 対応）

---

### 確認方法とOKの目安
```bash
dnf list installed | grep dovecot
```

- dovecot 関連パッケージが表示される

---

## 2. SSL 設定ファイルのバックアップ（1回目）

---

### この工程でしていること
- 設定変更前に元の状態を保存する
- 失敗時に復旧できるようにする

---

### 実行コマンド
```bash
cp /etc/dovecot/dovecot.conf /etc/dovecot/dovecot.conf.`date "+%Y%m%d_%H%M%S"`.bak
```

---

### コマンドの意味
- cp  
  - ファイルをコピーする
- date  
  - 現在日時をファイル名に付与する

---

### 確認方法とOKの目安
```bash
ls -l /etc/dovecot/ | grep dovecot
```
- .bak ファイルが存在する

---

## 3. POP3 プロトコルとメール保存先の設定

---

### 3-1. 10-ssl.conf を編集する

---

### この工程でしていること
- 使用するプロトコルを POP3 のみに制限する
- メール保存場所を明示的に指定する

---

### 実行コマンド
```bash
vi /etc/dovecot/conf.d/10-ssl.conf
```

---

### 設定変更内容①（プロトコル指定）

#### 変更前（コメントアウトされている状態）
```bash
#protocols = imap pop3 lmtp
```

#### 変更後（コメント解除・POP3のみに変更）
```bash
protocols = pop3
```

---

### 設定変更内容②（メール保存場所の追記）

#### 追記
```bash
mail_location = mail:/var/spool/mail/%u/
```

---

### 各設定の意味

#### protocols = pop3
- 利用する受信プロトコルを POP3 のみに限定する
- IMAP は無効化される

#### mail_location
- メール保存先の指定
- %u はユーザー名に置換される
- 例：/var/spool/mail/yoshie

---

### 確認方法とOKの目安
- 文法ミスがない
- 設定が正しく記述されている

---

## 4. SSL 設定ファイルのバックアップ（2回目）

---

### この工程でしていること
- 再設定前の状態を再度保存する
- 作業途中の復旧ポイントを作る

---

### 実行コマンド
```bash
cp /etc/dovecot/conf.d/10-ssl.conf /etc/dovecot/conf.d/10-ssl.conf.date "+%Y%m%d_%H%M%S".bak
```

---

### 確認方法とOKの目安
- 新しい .bak ファイルが追加されている

---

## 5. SSL 必須設定の無効化

---

### 5-1. 10-ssl.conf を再編集する

---

### この工程でしていること
- SSL 接続必須設定を無効化する
- 平文通信（非SSL）でも接続できるようにする（学習用設定）

---

### 実行コマンド
```bash
vi /etc/dovecot/conf.d/10-ssl.conf
```

---

### 設定変更内容

#### 変更前
```bash
ssl = required
```

#### 変更後（コメントアウト）
```bash
#ssl = required
```

---

### 注意点（重要）
- 本番環境では SSL 必須を無効にするのは推奨されない
- 本設定は学習・検証目的向け

---

### 確認方法とOKの目安
- 行頭に # が付いていること

---

## 6. 認証設定ファイルのバックアップ

---

### この工程でしていること
- 認証設定変更前の状態を保存する

---

### 実行コマンド
```bash
cp /etc/dovecot/conf.d/10-auth.conf /etc/dovecot/conf.d/10-auth.conf.date "+%Y%m%d_%H%M%S".bak
```

---

### 確認方法とOKの目安
```bash
ls -l /etc/dovecot/conf.d/ | grep 10-auth
```

- .bak ファイルが存在する

---

## 7. 平文認証を許可する設定変更

---

### 7-1. 10-auth.conf を編集する

---

### この工程でしていること
- POP3 接続時のユーザー認証を許可する
- SSL 未使用時でもログイン可能にする

---

### 実行コマンド
```bash
vi /etc/dovecot/conf.d/10-auth.conf
```

---

### 設定変更内容

#### 変更前
```bash
disable_plaintext_auth = yes
```

#### 変更後
```bash
disable_plaintext_auth = no
```

---

### 設定の意味

- disable_plaintext_auth = yes  
  - 平文パスワード認証を禁止

- disable_plaintext_auth = no  
  - 平文認証を許可（学習用）

---

### 注意点（重要）
- 本番では必ず SSL/TLS と併用する
- 平文認証のみの運用は危険

---

### 確認方法とOKの目安
- no に変更されていること

---

## 8. Dovecot を起動・自動起動設定する

---

### この工程でしていること
- Dovecot サービスを起動する
- OS 起動時に自動起動するように設定する

---

### 実行コマンド
```bash
systemctl start dovecot
systemctl enable dovecot
```

---

### コマンドの意味
- start：サービス起動
- enable：自動起動登録

---

### 確認方法とOKの目安
```bash
systemctl status dovecot
```
- Active: active (running)

---

## 9. telnet をインストールする

---

### この工程でしていること
- POP3 サーバの疎通確認ツールを導入する
- ポート通信と認証動作を直接確認できるようにする

---

### 実行コマンド
```bash
dnf install telnet
```

---

### コマンドの意味
- dnf install  
  - パッケージをインストールする
- telnet  
  - TCP 通信確認用クライアント

---

### 確認方法とOKの目安
```bash
dnf list installed | grep telnet
```

- telnet が表示される

---

## 10. POP3 サーバへ接続する

---

### この工程でしていること
- localhost の POP3（110番）へ接続する
- Dovecot が正しく待ち受けているか確認する

---

### 実行コマンド
```bash
telnet localhost 110
```

---

### コマンドの意味
- telnet：TCP 接続ツール
- localhost：自分自身
- 110：POP3 標準ポート

---

### 確認方法とOKの目安

接続後、以下のように表示される。

例：
+OK Dovecot ready.

意味：
- POP3 サーバが正常起動している

---

## 11. ユーザー認証を行う

---

### この工程でしていること
- メールユーザーでログインできるか確認する
- 認証設定が正しく動作しているか検証する

---

### 実行コマンド（telnet 接続後）
```bash
user yoshie
pass yoshie
```

---

### コマンドの意味

- user yoshie  
  - ログインユーザーを指定する

- pass yoshie
  - パスワードを送信する

---

### 確認方法とOKの目安
成功時の表示例：
+OK Logged in.
意味：
- 認証成功

---

## 12. メール一覧を取得する

---

### この工程でしていること
- サーバ上に保存されているメールを一覧表示する
- メールが正しく受信されているか確認する

---

### 実行コマンド（認証後）
```bash
list
```
---

### コマンドの意味
- list  
  - 受信メールの番号とサイズを表示

---

### 表示例
```bash
+OK 2 messages
1 1024
2 2048
.
```

---

### 確認方法とOKの目安
- メール件数が表示される
- 1 以上の件数がある

---

## 13. メール本文を取得する

---

### この工程でしていること
- 指定した番号のメール本文を取得する
- 正しく保存されているか確認する

---

### 実行コマンド（例：2番を取得）
```bash
retr 2
```
---

### コマンドの意味
- retr  
  - Retrieve（取得）の略
  - 指定番号のメールを取得

---

### 表示例
```bash
+OK 2048 octets
From: root@onishi.local
Subject: hello
...
```

---

### 確認方法とOKの目安
- From / Subject / Body が表示される
- 内容が正しい

---

## 14. テスト終了・切断

---

### この工程でしていること
- POP3 セッションを正常終了する

---

### 実行コマンド
```bash
quit
```

---

### コマンドの意味
- quit  
  - 接続終了

---

### 確認方法とOKの目安
```bash
+OK Logging out
```


が表示される

---

## 15. トラブル時の確認ポイント

---

### 接続できない場合

| 確認項目 | コマンド | 内容 |
|----------|----------|------|
| Dovecot状態 | systemctl status dovecot | 起動確認 |
| ポート | netstat -ln \| grep 110 | 待受確認 |
| FW | firewall-cmd | 110開放 |
| ログ | /var/log/maillog | エラー確認 |

---

### 認証できない場合

- パスワード再確認
- disable_plaintext_auth 設定確認
- ユーザー存在確認（id yoshie）

---



