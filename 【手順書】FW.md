# SRX セキュリティ設定 手順書（Interface / Policy / NAT）

---

# 構成概要

本手順では、Juniper SRX を想定し、以下の構成を設定する。

- ge-0/0/0：外側（untrust）
- ge-0/0/1：内側（trust）
- trust ↔ untrust 間の通信制御
- Destination NAT 設定
- Proxy ARP 設定（AWS想定）

---

# 1. インターフェース設定

---

## 構成図
```bash
     [Internet]
          |
  ge-0/0/0 (untrust)
  172.31.0.10/24
          |
      [ SRX ]
          |
  ge-0/0/1 (trust)
  172.31.1.10/24
          |
    [ Internal LAN ]
```
---

## 設定コマンド
```bash
set interfaces ge-0/0/0 unit 0 family inet address 172.31.0.10/24
```
```bash
set interfaces ge-0/0/1 unit 0 family inet address 172.31.1.10/24
```
```bash
commit check
```
```bash
commit
```

---

## 各コマンドの意味

### set interfaces ge-0/0/0 unit 0 family inet address 172.31.0.10/24

| 部分 | 意味 |
|------|------|
| set | 設定追加 |
| interfaces | インターフェース設定 |
| ge-0/0/0 | 物理IF |
| unit 0 | 論理IF |
| family inet | IPv4設定 |
| address | IPアドレス設定 |

→ 外側インターフェースのIP設定

---

### commit check

- 設定文法チェック
- 実際には反映しない

---

### commit

- 設定を本番反映

---

## 確認方法
```bash
run show interfaces terse
```
### OKの目安
- ge-0/0/0 と ge-0/0/1 にIPが表示

---

# 2. セキュリティポリシー設定

---

## ゾーン作成
```bash
set security zones security-zone untrust interfaces ge-0/0/0.0
```
```bash
set security zones security-zone trust interfaces ge-0/0/1.0
```
---

## コマンドの意味

| 部分 | 意味 |
|------|------|
| security zones | セキュリティゾーン |
| security-zone | ゾーン名 |
| interfaces | ゾーンへIF割当 |

→ 外側を untrust、内側を trust と定義

---

## 内→外 通信許可
```bash
set security policies from-zone trust to-zone untrust policy trust_to_untrust match source-address any
```
```bash
set security policies from-zone trust to-zone untrust policy trust_to_untrust match destination-address any
```
```bash
set security policies from-zone trust to-zone untrust policy trust_to_untrust match application any
```
```bash
set security policies from-zone trust to-zone untrust policy trust_to_untrust then permit
```
---

## 各設定の意味

| 項目 | 意味 |
|------|------|
| from-zone trust | 内側 |
| to-zone untrust | 外側 |
| match source-address any | 全送信元 |
| match destination-address any | 全宛先 |
| match application any | 全プロトコル |
| then permit | 許可 |

---

## 外→内 通信許可
```bash
set security policies from-zone untrust to-zone trust policy untrust_to_trust match source-address any
```
```bash
set security policies from-zone untrust to-zone trust policy untrust_to_trust match destination-address any
```
```bash
set security policies from-zone untrust to-zone trust policy untrust_to_trust match application any
```
```bash
set security policies from-zone untrust to-zone trust policy untrust_to_trust then permit
```

---

## 注意点

- 本来は外→内は制限すべき
- 演習用の全許可設定

---

## コミット
```bash
commit check
```
```bash
commit
```
---

## 確認方法
```bash
run show security policies
```
### OKの目安
- trust_to_untrust / untrust_to_trust が表示

---

# 3. Destination NAT 設定

---

## 構成図
```bash
Internet → 172.31.0.150
↓
NAT変換
↓
172.31.1.48
```
---

## NATプール作成
```bash
set security nat destination pool pool_A address 172.31.1.48/32
```
### 意味
- 外部アクセスを内部172.31.1.48へ変換

---

## NATルール適用
```bash
set security nat destination rule-set 1 from interface ge-0/0/0.0
```
```bash
set security nat destination rule-set 1 rule 1A match destination-address 172.31.0.150/32
```
```bash
set security nat destination rule-set 1 rule 1A then destination-nat pool pool_A
```
---

## 各意味

| 部分 | 意味 |
|------|------|
| rule-set 1 | ルールセット作成 |
| from interface | どのIFからの通信か |
| match destination-address | マッチする外部IP |
| then destination-nat | 内部変換実行 |

---

## Proxy ARP（AWS構成）
```bash
set security nat proxy-arp interface ge-0/0/0.0 address 52.26.109.29
```
### 意味
- 外部IPに対しSRXがARP応答
- AWSでElasticIP利用時に必要

---

## コミット
```bash
commit check
```
```bash
commit
```
---

## 確認方法
```bash
run show security nat destination
```
```bash
run show security nat proxy-arp
```
### OKの目安
- pool_A表示
- proxy-arp登録確認

---

# 便利コマンド
```bash
run show configuration | display set | no-more
```
---

## 意味

| 部分 | 内容 |
|------|------|
| show configuration | 現在設定表示 |
| display set | set形式で表示 |
| no-more | ページ送り無効 |

---

## 削除方法

出力結果を元に
```bash
delete security nat destination rule-set 1
```
などで即削除可能

---

# 最終確認チェック

- インターフェースIP設定済
- ゾーン割当済
- ポリシー反映済
- NAT変換確認済
- Proxy ARP登録済
- commit成功

---

# 本質理解

| 用語 | 意味 |
|------|------|
| trust | 内側 |
| untrust | 外側 |
| Policy | 通信許可制御 |
| NAT | アドレス変換 |
| Proxy ARP | 代理応答 |
| commit | 設定確定 |

---

# 完了状態

- 内外通信可能
- NAT変換動作
- セキュリティポリシー反映
- AWS環境対応済