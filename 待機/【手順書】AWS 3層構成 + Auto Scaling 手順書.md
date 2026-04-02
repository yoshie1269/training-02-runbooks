# AWS 3層構成 + Auto Scaling 手順書（WordPress構築）

## 概要

### この構成でしていること

- AWSで3層構成（Web / DB）を構築
- Auto Scaling による冗長化
- ALBで負荷分散
- RDSでDB管理

---

### 構成図
```bash
Internet  
↓  
ALB  
↓  
EC2（Auto Scaling）  
↓  
RDS（MySQL）  
```

---

## 1 VPC構築

### この工程でしていること

ネットワーク基盤作成

---

### VPC

- 名前：SAAExamBook-VPC  
- CIDR：10.0.0.0/20  

---

### サブネット

|名前|CIDR|
|---|---|
|Public-a|10.0.0.0/24|
|Public-c|10.0.1.0/24|
|Private-a|10.0.2.0/24|
|Private-c|10.0.3.0/24|

---

### IGW

- 作成 → VPCにアタッチ  

---

### ルートテーブル

#### Public-RT

0.0.0.0/0 → IGW  

関連付け：  
- PublicSubnet-a  
- PublicSubnet-c  

---

#### Private-RT

- ローカルのみ  

関連付け：  
- PrivateSubnet-a  
- PrivateSubnet-c  

---

### OKの目安

- Publicからインターネット通信可能  
- Privateは外部非公開  

---

## 2 セキュリティグループ

### この工程でしていること

通信制御

---

### ALB-SG

|Port|Source|
|---|---|
|80|0.0.0.0/0|

---

### EC2-SG

|Port|Source|
|---|---|
|80|ALB-SG|
|22|自分IP|

---

### RDS-SG

|Port|Source|
|---|---|
|3306|EC2-SG|

---

### OKの目安

- EC2はALBからのみHTTP許可  
- RDSはEC2からのみ接続  

---

## 3 RDS構築（MySQL）

### この工程でしていること

DBサーバ構築（マネージド）

---

### サブネットグループ

- PrivateSubnet-a  
- PrivateSubnet-c  

---

### RDS設定

- エンジン：MySQL  
- DB名：wordpress  
- ユーザー：admin  
- パスワード：wppassword  

---

### ネットワーク

- VPC：SAAExamBook-VPC  
- パブリックアクセス：なし  
- SG：RDS-SG  

---

### 確認

- エンドポイント取得  

---

### OKの目安

- ステータス：available  
- PrivateIPのみ  

---

## 4 EC2構築（WordPress）

### この工程でしていること

Webサーバ準備

---

### EC2設定

- AMI：Amazon Linux 2023  
- タイプ：t2.micro / t3.micro  
- サブネット：Public  
- SG：EC2-SG  

---

### ユーザーデータ

- Apacheインストール  
- PHP導入  
- WordPress配置  

---

### 確認

http://EC2のIP  

---

### OKの目安

- WordPressファイル表示  

---

## 5 ターゲットグループ作成

### この工程でしていること

ALBの振り分け先定義

---

### 設定

- 名前：SAA-TargetGroup  
- プロトコル：HTTP  
- ポート：80  

---

### ヘルスチェック

- パス：/readme.html  
- 間隔：20秒  
- 正常：3  

---

### OKの目安

- TargetがHealthy  

---

## 6 ALB構築

### この工程でしていること

ロードバランサ構築

---

### 設定

- 名前：SAA-ALB  
- サブネット：Public-a / Public-c  
- SG：ALB-SG  

---

### リスナー

HTTP:80 → TargetGroup  

---

### OKの目安

- ALB DNSでアクセス可能  

---

## 7 WordPress初期設定

### この工程でしていること

DB連携設定

---

### アクセス

http://ALBのDNS名  

---

### DB情報

- DB名：wordpress  
- ユーザ：admin  
- パスワード：wppassword  
- ホスト：RDSエンドポイント  

---

### OKの目安

- WordPress画面表示  

---

## 8 AMI作成

### この工程でしていること

スケール用イメージ作成

---

### 手順

- EC2 → イメージ作成  
- 名前：wp-ami  

---

### OKの目安

- AMI status：available  

---

## 9 起動テンプレート

### この工程でしていること

Auto Scaling用設定

---

### 設定

- 名前：wp-template  
- AMI：wp-ami  
- SG：EC2-SG  

---

### OKの目安

- テンプレート作成完了  

---

## 10 Auto Scaling Group

### この工程でしていること

冗長化構成作成

---

### 設定

- 名前：wp-asg  
- サブネット：Public-a / Public-c  
- ターゲットグループ：SAA-TargetGroup  

---

### キャパシティ

- 最小：2  
- 最大：4  
- 希望：2  

---

### OKの目安

- EC2が2台起動  
- ALBに登録  

---

## 11 動作確認

### この工程でしていること

全体動作確認

---

### 確認項目

① EC2が2台以上  
② Target Healthy  
③ ALBアクセス成功  

---

### OKの目安

- WordPress正常表示  

---

## 12 冗長化テスト

### この工程でしていること

障害耐性確認

---

### テスト

① EC2を1台停止/削除  
② Auto Scalingで復旧  

---

### OKの目安

- 新しいEC2が自動起動  
- ALBアクセス継続  

---

## 重要ポイント

### 3層構成の役割

- ALB：負荷分散  
- EC2：Web  
- RDS：DB  

---

### Auto Scalingの価値

- 自動復旧  
- 負荷対応  
- 可用性向上  

---

## 完成状態

- ALB + EC2(AutoScaling) + RDS  
- 冗長化構成  
- WordPress稼働  