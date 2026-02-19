# DNS構築手順（BIND）

---

## この工程でしていること
- DNSサーバ（BIND）を構築する
- 独自ドメイン（例：hogehoge.com）の名前解決を可能にする
- Aレコード・NSレコードを設定する

---

## 1. BIND インストール

### 実行コマンド
```bash
dnf install bind -y
```

### コマンドの意味
- bind：DNSサーバソフトウェア（named）
- -y：自動承認

### 確認
```bash
rpm -qa | grep bind
```

OKの目安：
- bind パッケージが表示される

---

## 2. named 起動・自動化

### 実行コマンド
```bash
systemctl start named
```
```bash
systemctl enable named
```

### 確認
```bash
systemctl status named
```
```bash
systemctl is-enabled named
```

OKの目安：
- active (running)
- enabled

---

## 3. named.conf バックアップ

### 実行コマンド
```bash
cp /etc/named.conf /etc/named.conf.bak
```
```bash
ls /etc/ | grep named
```
### この工程の意味
- 設定変更前にバックアップを取得
- named.conf の存在確認

OKの目安：
- named.conf.bak が作成されている

---

## 4. named.conf 編集（外部公開設定）

### 実行コマンド
```bash
vi /etc/named.conf
```

### 設定変更内容

コメントアウト解除（外部受付有効化）
```bash
#listen-on port 53 { 127.0.0.1; };
#listen-on-v6 port 53 { ::1; };
```
→ コメントアウト（#を付ける）

変更
```bash
allow-query { any; };
```

### 設定の意味
- listen-on：127.0.0.1のみ受付 → 無効化して外部受付
- allow-query any：誰でも問い合わせ可能にする

### 確認
```bash
named-checkconf
```
```bash
systemctl restart named
```
```bash
netstat -ln | grep 53
```

OKの目安：
- エラーが出ない
- 0.0.0.0:53 がLISTEN表示される

---

## 5. ゾーン定義追加

### 実行コマンド
```bash
vi /etc/named.conf
```
追加：
```bash
zone "hogehoge.com" IN{
    type master;
    file "/var/named/hogehoge.com.zone";
};
```

### 意味
- hogehoge.com のゾーン管理を宣言
- ゾーンファイルの場所を指定

---

## 6. ゾーンファイル作成

### 実行コマンド
```bash
vi /var/named/hogehoge.com.zone
```

### 設定内容
```bash
$TTL 3600
@ IN SOA ns.hogehoge.com. test.gmail.com. (
20220401 ; serial
3600 ; refresh
3600 ; retry
3600 ; expire
3600 ) ; minimum

 IN NS ns.hogehoge.com.
 ns  IN A 172.31.18.203
 www IN A 172.31.18.203
```
---

### 設定の意味

SOA：
- ゾーンの管理情報
- serial：更新番号（修正時は増やす）

NS：
- このドメインを管理するDNSサーバ

Aレコード：
- ns → DNSサーバIP
- www → WebサーバIP

---

## 7. ゾーンファイルチェック

### 実行コマンド
```bash
named-checkzone hogehoge.com /var/named/hogehoge.com.zone
```

OKの目安：
- OK と表示される

---

## 8. 再起動
```bash
systemctl restart named
```
確認：
```bash
systemctl status named
```

OKの目安：
- active (running)

---

## 9. 動作確認（ローカル確認）
```bash
dig @localhost www.hogehoge.com
```
```bash
dig @localhost ns.hogehoge.com
```

OKの目安：
- ANSWER SECTION にIPが表示される
- status: NOERROR

---

## 完成状態

- DNSサーバ稼働
- hogehoge.com の名前解決成功
- www.hogehoge.com が指定IPへ解決される