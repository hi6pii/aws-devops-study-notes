- CodeDeployはEC2へPushしているのではなく、AgentがAWSへ取りに行く（Pull方式）

AWS CodeDeploy のデプロイが開始されない原因（Agent・IAM Role）

概要

Amazon EC2へデプロイする場合、AWS CodeDeployはCodeDeploy Agentを利用してデプロイを実行します。
CodeDeployはEC2へ直接接続して命令を送るのではなく、
EC2上のCodeDeploy AgentがAWS側を定期的にポーリング（Pull）し、デプロイ命令を取得する仕組みです。

そのため、

* Agentが存在しない
* Agentが停止している
* IAMロールが不足している

場合は、デプロイ自体が開始できません。

---

CodeDeployの通信方式

重要なのは PushではなくPull であることです。

```
        AWS CodeDeploy
               │
       Deployment作成
               │
               │
      （待っているだけ）
               │
──────── Internet ────────
               ▲
               │ HTTPS (443)
               │
CodeDeploy Agent
(EC2上)
      │
定期的に問い合わせ
「新しいDeploymentある？」
```

つまり、

AgentがAWSへ通信を開始する

という構成になっています。

そのため、

* インバウンド443は不要
* アウトバウンド443が必要

となります。

---

デプロイの流れ

```
Developer
      │
      ▼
CodeDeploy
      │
Deployment作成
      │
      ▼
CodeDeploy Agent
(EC2)
      │
Deployment取得
      │
S3からRevision取得
      │
appspec.yml読み込み
      │
Hooks実行
      │
デプロイ完了
```

---

CodeDeploy Agent の役割

Agentが担当する仕事

* CodeDeployをポーリング
* Deployment取得
* S3からRevision取得
* appspec.yml読み込み
* Hook実行
* デプロイ結果報告

つまり、

EC2側で実際の作業をしているのはAgentです。

---

IAMインスタンスプロファイルの役割

CodeDeploy AgentはIAM Roleを利用してAWSへアクセスします。

代表的な権限
```
CodeDeployへのアクセス
S3からRevision取得
CloudWatch Logs出力（必要なら）
```
例えば
```
s3:GetObject
s3:ListBucket
```

などが必要になります。

権限不足だと

* Revision取得失敗
* Deployment失敗

になります。

---

ライフサイクルイベント

AgentがDeploymentを取得すると、

appspec.ymlに従って各Hookを実行します。

```
ApplicationStop
↓
DownloadBundle
↓
BeforeInstall
↓
Install
↓
AfterInstall
↓
ApplicationStart
↓
ValidateService
```

Blue/Greenでは

```
BeforeBlockTraffic

AfterBlockTraffic

BeforeAllowTraffic

AfterAllowTraffic
```

などもあります。

---

今回のケース

```
ライフサイクルイベントのログが生成されていない
```

つまり以下のように考えると良い

- ⭕️ Agentが動いていない
    - Agent未インストール・停止 
        - 最も典型的な原因
        - Agentが動かなければDeploymentを取得できない
- ⭕️ Deploymentが取得できない他パターン
    -  IAM Role不足
        - AgentはIAM Roleを利用して
            * CodeDeploy
            * S3
        へアクセス
        - 必要権限が無いとDeploymentを進められない
- ❌：セキュリティグループの設定によるインバウンド通信 443ができない
    - EC2(Agent) -> AWS 形式のため　Outbound HTTPS(443)が必要
    - AWS -> EC2 の　インバウンド443は不要
- ❌：CodeDeploy サービスに割り当てられたサービスロールに iam:PassRole 権限が付与されていない可能性
    - サービスへIAM Roleを渡す権限
    - Agentが動いていない という可能性とは関係ない
- ❌：appspec.yml ファイルの hooks セクションで定義されたスクリプトの実行権限が不足している可能性
    - Hookが失敗したとしても
    ```
    BeforeInstall
    AfterInstall
    ```
    などのログが残る

---
### CodeDeployでよくあるトラブル

- Pendingのまま -> Agent停止・未インストール
- Failed（開始直後）-> IAM Role不足・Agent異常
- Hook失敗 -> appspec.yml・スクリプト
- Revision取得失敗 -> S3権限不足
- 通信不可 -> Outbound443不可・NAT未設定

---
### 実務での確認手順

- まず最初確認すべきは AGENTS
```
sudo service codedeploy-agent status
または
systemctl status codedeploy-agent
```
- 起動していない場合
```
sudo service codedeploy-agent start
```
続いて
- IAM Role
- ↓ S3アクセス
- ↓ Agentログ
```
/var/log/aws/codedeploy-agent/
```
を確認する

---
### ハンズオン：CodeDeploy Agent の動作確認

目的

CodeDeployがPull方式で動作することと、AgentやIAM Roleがデプロイ開始に必須であることを体験する。

#### Step1 EC2を作成

IAMロールには少なくとも以下を付与します。

* AmazonEC2RoleforAWSCodeDeploy（または同等権限）
* S3読み取り権限（必要に応じて）

#### Step2 CodeDeploy Agentをインストール

``` sh
sudo yum update -y
sudo yum install ruby wget -y

cd /home/ec2-user

wget https://aws-codedeploy-ap-northeast-1.s3.ap-northeast-1.amazonaws.com/latest/install

chmod +x install

sudo ./install auto
```

起動確認

``` sh
sudo systemctl status codedeploy-agent
```

#### Step3 デプロイ成功を確認

CodeDeployからデプロイを実行します。

成功すると

```
Deployment
↓
Agent取得
↓
DownloadBundle
↓
BeforeInstall
↓
AfterInstall
↓
ApplicationStart
↓
ValidateService
```
の順に進みます。

#### Step4 Agentを停止して再実行

```
sudo systemctl stop codedeploy-agent
```
再度デプロイすると、
```
Pending
```
または
```
Failed
```
となり、ライフサイクルイベントのログが生成されないことを確認できます。

#### Step5 IAMロールを変更して再実行

EC2からS3読み取り権限を外します。

すると、

* Agentは動作している
* Deploymentは取得できる

ものの、

```
DownloadBundle
```

付近でS3アクセスエラーとなり、デプロイが失敗します。

Agent停止時との違い（ライフサイクルイベントがどこまで進むか）を比較すると理解が深まります。

#### 試験で覚えるポイント

* CodeDeployはPushではなくPull方式で動作する。
* CodeDeploy AgentがEC2上でデプロイ処理を実行する。
* Agentが停止・未インストールなら、デプロイは開始されず Pending / Failed になる。
* EC2にはIAMインスタンスプロファイルが必要で、Agentはその権限でS3などへアクセスする。
* ライフサイクルイベントのログが無い場合は、Hook以前（Agentや通信、権限）の問題を疑う。
* Hookのログがあるなら、appspec.yml やスクリプトの問題を疑う。

#### 理解しておきたいこと

CodeDeployでは、「デプロイを管理するサービス」と「実際にデプロイを実行するAgent」が明確に役割分担されています。
```
CodeDeploy
（デプロイ管理）
        │
        ▼
CodeDeploy Agent
（EC2上で実行）
        │
        ▼
S3から取得
        │
        ▼
appspec.yml
        │
        ▼
Hooks実行
```
このアーキテクチャを理解しておくと、「Pendingのまま」「ログが出ない」「Hookで失敗する」といったトラブルシューティングを、症状から切り分けられるようになります。