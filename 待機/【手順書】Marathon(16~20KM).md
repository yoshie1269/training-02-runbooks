# 16KM〜20KM GitHub Actions 自動デプロイ手順書

## 1 今回の目的

### この工程でしていること

StagingサーバからGitHubへSSH認証で接続し、  
GitHub Actions を利用した自動デプロイ環境を構築する。

---

### 今回学習する内容

- SSH鍵認証
- 公開鍵 / 秘密鍵
- GitHub SSH接続
- git clone
- GitHub Actions
- workflow.yml
- GitHub Secrets
- CI/CD
- 自動デプロイ
- ログ確認
- deploy.sh

---

## 2 全体構成

### 構成イメージ

```text
ローカルPC
↓ git push
GitHub
↓ GitHub Actions
Stagingサーバ
↓ deploy.sh
Nginx
```

---

### この構成でしていること

- GitHubへpush
- Actions自動実行
- SSH接続
- deploy.sh実行
- 自動デプロイ

---

# 16KM Linux上でSSH認証鍵を作成しGitHubへ登録

## 3 SSH鍵認証とは

### この工程でしていること

SSH鍵認証では、

- 秘密鍵（サーバ側保持）
- 公開鍵（GitHub登録）

を利用して安全に認証する。

---

### SSH鍵認証のメリット

- パスワード不要
- セキュア接続
- GitHub Actions利用可能

---

## 4 Stagingサーバへログイン

### SSHログイン

#### コマンド

```bash
ssh ユーザ名@dev.marathon.rplearn.net
```

---

### OKの目安

- サーバへログイン成功

---

## 5 SSH鍵作成

### この工程でしていること

SSH認証用鍵を作成する。

---

### コマンド

```bash
ssh-keygen -t ed25519
```

---

### 実行時

保存先やパスフレーズは  
すべて Enter でOK。

---

### 作成されるファイル

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

---

### OKの目安

- 鍵ファイル生成される

---

## 6 .sshディレクトリへ移動

### コマンド

```bash
cd ~/.ssh
```

---

### OKの目安

- ~/.ssh 配下へ移動できる

---

## 7 authorized_keys設定

### この工程でしていること

SSH接続時に利用する公開鍵を登録する。

---

### 公開鍵登録

#### コマンド

```bash
cat id_ed25519.pub > authorized_keys
```

---

### 権限変更

#### コマンド

```bash
chmod 600 authorized_keys
```

---

### OKの目安

- authorized_keys 作成される
- 権限600

---

## 8 公開鍵確認

### コマンド

```bash
cat id_ed25519.pub
```

---

### この工程の意味

表示された公開鍵をGitHubへ登録する。

---

### OKの目安

- ssh-ed25519 から始まる文字列表示

---

## 9 GitHubへSSH鍵登録

### GitHub画面

```text
https://github.com/settings/keys
```

---

### 登録内容

|項目|内容|
|---|---|
Title|staging-y-onishi|
Key|id_ed25519.pub の内容|

---

### OKの目安

- GitHubへ鍵登録完了

---

## 10 GitHub SSH接続確認

### コマンド

```bash
ssh -T git@github.com
```

---

### 成功例

```bash
Hi yoshie1269! You've successfully authenticated
```

---

### OKの目安

- successfully authenticated 表示

---

### 学べるポイント

- SSH鍵認証
- GitHub SSH接続
- 公開鍵 / 秘密鍵
- セキュア認証

---

# 17KM git cloneする

## 11 今回の目的

### この工程でしていること

GitHub上のソースコードを  
Stagingサーバへcloneする。

---

### 今回学習する内容

- git clone
- GitHub連携
- サーバへのソース配置

---

## 12 /app配下へ移動

### コマンド

```bash
cd /app/自分のユーザ名
```

---

### OKの目安

- app配下へ移動できる

---

## 13 不要ファイル削除

### コマンド

```bash
rm -r ./*
```

---

### 注意

```bash
rm ./.*
```

は .git まで削除するため注意。

---

### OKの目安

- 不要ファイル削除される

---

## 14 GitHubからclone

### この工程でしていること

GitHub上のソースコードを取得する。

---

### コマンド

```bash
git clone git@github.com:GitHubアカウント/リポジトリ名.git .
```

---

### この工程の意味

現在ディレクトリへcloneする。

---

### OKの目安

- clone成功
- エラーなし

---

## 15 clone確認

### コマンド

```bash
ls
```

---

### 実行例

```bash
README.md
compose.yml
src
test
tmp
```

---

### OKの目安

- ソースコード確認できる

---

### 学べるポイント

- git clone
- GitHub SSH認証
- サーバ配置

---

# 18KM GitHub Actions workflow作成

## 16 今回の目的

### この工程でしていること

GitHubへpushした際に、
自動デプロイを実行するworkflowを作成する。

---

### 今回学習する内容

- GitHub Actions
- workflow.yml
- CI/CD
- 自動デプロイ

---

## 17 ローカルMacでリポジトリへ移動

### コマンド

```bash
cd ~/dev-marathon
```

---

### OKの目安

- Gitリポジトリへ移動できる

---

## 18 workflowディレクトリ作成

### コマンド

```bash
mkdir -p .github/workflows
```

---

### OKの目安

- workflowsディレクトリ作成される

---

## 19 develop.yml作成

### コマンド

```bash
vi .github/workflows/develop.yml
```

---

## 20 develop.yml内容

### 設定内容

```yaml
name: ssh command

on:
  push:
    branches:
      - develop

jobs:
  test:
    runs-on: ubuntu-latest

    steps:

      - id: ssh
        uses: invi5H/ssh-action@v1

        with:
          SSH_HOST: ${{ secrets.DEV_SSH_HOST }}
          SSH_PORT: ${{ secrets.DEV_SSH_PORT }}
          SSH_USER: ${{ secrets.DEV_SSH_USER }}
          SSH_KEY: ${{ secrets.DEV_SSH_KEY }}

      - run: ssh ${{ steps.ssh.outputs.SERVER }} bash /app/y-onishi/deploy.sh
```

---

### このworkflowでしていること

GitHubへpushされたタイミングで、

- StagingサーバへSSH接続
- deploy.sh実行

を自動化している。

---

### 保存方法

```bash
Esc
:wq
Enter
```

---

### OKの目安

- YAML保存成功

---

### 学べるポイント

- workflow.yml
- CI/CD
- 自動デプロイ

---

# 19KM GitHub Actions Secrets設定

## 21 今回の目的

### この工程でしていること

GitHub Actionsで利用する秘密情報を、
GitHub Secretsへ登録する。

---

### 今回学習する内容

- GitHub Secrets
- 秘密鍵管理
- セキュリティ管理

---

## 22 GitHub Secrets画面

### URL

```text
https://github.com/自分のGitHubアカウント/リポジトリ名/settings/secrets/actions
```

---

### OKの目安

- Actions Secrets画面開ける

---

## 23 New repository secret 作成

### この工程でしていること

Secretsを1つずつ登録する。

---

## 24 Secrets登録内容

|Name|Value|
|---|---|
DEV_SSH_HOST|dev.marathon.rplearn.net|
DEV_SSH_PORT|22|
DEV_SSH_USER|自分のユーザ名|
DEV_SSH_KEY|id_ed25519 の中身|

---

## 25 id_ed25519確認

### コマンド

```bash
cat ~/.ssh/id_ed25519
```

---

### この工程の意味

秘密鍵をGitHub Secretsへ登録する。

---

### OKの目安

- Secrets登録完了

---

### 学べるポイント

- Secrets管理
- SSH秘密鍵
- セキュリティ

---

# 20KM GitHub Actions実行

## 26 今回の目的

### この工程でしていること

GitHubへpushし、
Actions自動実行確認を行う。

---

### 今回学習する内容

- Actionsログ確認
- deploy.sh
- エラー解析

---

## 27 workflowをGitHubへpush

### コマンド

```bash
git add .
```

```bash
git commit -m "add github actions workflow"
```

```bash
git push origin develop
```

---

### OKの目安

- push成功

---

## 28 Actions画面確認

### URL

```text
https://github.com/自分のGitHubアカウント/リポジトリ名/actions
```

---

### 実行状態

|色|意味|
|---|---|
黄色|実行中|
緑|成功|
赤|エラー|

---

### OKの目安

- workflow実行される

---

## 29 エラー確認

### この工程でしていること

Actionsログを確認し、
原因調査する。

---

### SSH接続先エラー

#### エラー

```bash
getaddrinfo: Name or service not known
```

---

#### 原因

DEV_SSH_HOST誤り。

---

### deploy.sh不存在

#### エラー

```bash
No such file or directory
```

---

#### 原因

deploy.sh未作成。

---

## 30 deploy.sh作成

### 作成場所

```bash
cd /app/y-onishi
```

---

### 作成

```bash
vi deploy.sh
```

---

### deploy.sh内容

```bash
#!/bin/bash

cd /app/y-onishi

git pull origin develop

cp -r src/web/* /usr/share/nginx/html/y-onishi/
```

---

### 実行権限付与

```bash
chmod 755 deploy.sh
```

---

### OKの目安

- deploy.sh実行可能

---

## 31 Actions再実行

### 実行方法

Actions画面右上の

```text
Re-run jobs
```

を押下。

---

### OKの目安

- workflow緑色
- Success表示

---

## 32 最終的にできたこと

### CI/CD構成

```text
ローカルでpush
↓
GitHub Actions起動
↓
StagingサーバへSSH接続
↓
deploy.sh実行
↓
自動デプロイ
```

---

### 学べるポイント

- GitHub Actions
- CI/CD
- 自動デプロイ
- SSH接続
- Secrets管理
- ログ解析
- deploy.sh

---

## 完了条件

- GitHub SSH接続成功
- git clone成功
- workflow実行成功
- Secrets設定成功
- deploy.sh実行成功
- 自動デプロイ成功

---

## 完成

GitHub Actions を利用した  
CI/CD自動デプロイ環境構築完了。