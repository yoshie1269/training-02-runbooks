# SRXファイアウォール構築 手順書（Interface / Policy / NAT 設定）

---

# 概要

本手順書では、Juniper SRX を用いて以下の設定を実施する。

1. インターフェース設定
2. セキュリティゾーン・ポリシー設定
3. Destination NAT 設定
4. Proxy ARP 設定
5. 動作確認
6. 便利コマンド

構成は以下を想定する。

- ge-0/0/0：外側（untrust）
- ge-0/0/1：内側（trust）

---

# 構成図
```bash
        Internet
            |
    ge-0/0/0 (untrust)
    172.31.0.10/24
            |
          SRX
            |
    ge-0/0/1 (trust)
    172.31.1.10/24
            |
       内部サーバ
    172.31.1.48
```

---

# 1. インターフェース設定

---

## この工程でしていること

- SRXの物理インターフェースにIPアドレスを設定
- 外部側と内部側ネットワークを定義
- ルーティングの基盤を作成

---

## 操作モード
```bash
configure
```
---

## 外部インターフェース設定
```bash
set interfaces ge-0/0/0 unit 0 family inet address 172.31.0.10/24
```

### 意味

| 項目 | 説明 |
|------|------|
| ge-0/0/0 | 外部ポート |
| unit 0 | 論理インターフェース |
| family inet | IPv4 |
| address | IP設定 |

---

## 内部インターフェース設定
```bash
set interfaces ge-0/0/1 unit 0 family inet address 172.31.1.10/24
```
---

## 設定確認
```bash
commit check
```
```bash
commit
```
### 意味

| コマンド | 意味 |
|----------|------|
| commit check | 構文チェック |
| commit | 設定反映 |

---

## OKの目安
```bash
show interfaces terse
```
- ge-0/0/0 と ge-0/0/1 が up

---

# 2. セキュリティポリシー設定

---

# ゾーン作成

## この工程でしていること

- インターフェースを信頼ゾーン・非信頼ゾーンへ分類
- 通信制御の基準を作る

---

## 設定
```bash
set security zones security-zone untrust interfaces ge-0/0/0.0
```
```bash
set security zones security-zone trust interfaces ge-0/0/1.0
```
---

## 確認
```bash
show security zones
```
OK：
- 各インターフェースがゾーンに属している

---

# トラフィック制御（ポリシー）

---

## 内 → 外 通信許可

### この工程でしていること

- trust → untrust への通信を許可
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

## 外 → 内 通信許可
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

## ポリシー確認
```bash
show security policies
```
OK：
- 2方向ポリシー表示

---

## コミット
```bash
commit check
```
```bash
commit
```
---

# 3. NAT設定（Destination NAT）

---

# NAT構成図
```bash
Internet
   |
52.26.109.29
   |
SRX ge-0/0/0
   |
172.31.1.48（内部サーバ）
```
---

## 3-1 NATプール設定

### この工程でしていること

- 変換先アドレスを定義
```bash
set security nat destination pool pool_A address 172.31.1.48/32
```
---

## 3-2 NATルール適用

### 外部IFからの通信を対象
```bash
set security nat destination rule-set 1 from interface ge-0/0/0.0
```
### マッチ条件
```bash
set security nat destination rule-set 1 rule 1A match destination-address 172.31.0.150/32
```

### NAT適用
```bash
set security nat destination rule-set 1 rule 1A then destination-nat pool pool_A
```
---

## 意味

| 項目 | 内容 |
|------|------|
| rule-set | ルール集合 |
| match | 変換前アドレス |
| pool | 変換後内部IP |

---

## 3-3 Proxy ARP設定（AWS必須）

### この工程でしていること

- SRXがEIPに対してARP応答する
```bash
set security nat proxy-arp interface ge-0/0/0.0 address 52.26.109.29
```
---

## コミット
```bash
commit check
```
```bash
commit
```
---

## 確認
```bash
show security nat destination
```
```bash
show security nat proxy-arp
```
---

## OKの目安

- NATルール表示
- proxy-arp登録済み

---

# 4. 動作確認

---

## 内部から外部

- ping外部
- インターネット通信可能

---

## 外部から内部

- 52.26.109.29へアクセス
- 内部172.31.1.48へ転送

---

# 5. 便利コマンド

---

## 設定をset形式で表示
```bash
run show configuration | display set | no-more
```
---

## 意味

| コマンド | 内容 |
|----------|------|
| display set | set形式表示 |
| no-more | ページ送り停止 |

---

## 削除時

表示結果を元に
```bash
delete security nat destination rule-set 1
```
などで削除可能

---

# トラブルシュート

---

## 通信できない場合

- ゾーン未設定
- ポリシー未設定
- NAT未設定
- Proxy-ARP未設定
- commit忘れ

---

# 本質理解

| 概念 | 意味 |
|------|------|
| Interface | ネットワーク境界 |
| Zone | 信頼レベル分類 |
| Policy | 通信許可条件 |
| NAT | アドレス変換 |
| Proxy ARP | AWS特有必須設定 |

---

# 完了状態

- インターフェース設定完了
- trust/untrustゾーン設定完了
- 双方向ポリシー設定完了
- Destination NAT動作確認
- Proxy ARP設定完了
- commit反映済み
