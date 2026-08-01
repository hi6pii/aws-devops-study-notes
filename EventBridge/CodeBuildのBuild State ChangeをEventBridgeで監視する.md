# CodeBuild × EventBridge（Build State Change）

## Why

CodeBuild のビルド結果（成功・失敗・停止）をトリガーとして、
SNS・Lambda・Step Functions などへ連携し、自動通知・自動復旧を実現するため。

---

# Flow

Git Push
↓
CodePipeline（任意）
↓
CodeBuild
↓
Build State Change
↓
EventBridge Rule
↓
SNS / Lambda / Step Functions

---

# EventBridge Event Pattern

```json
{
  "source": ["aws.codebuild"],
  "detail-type": ["CodeBuild Build State Change"],
  "detail": {
    "project-name": ["Payment-Gateway-Build"],
    "build-status": ["FAILED", "STOPPED"]
  }
}
```

- source：`aws.codebuild`
- detail-type：`CodeBuild Build State Change`
- project-name：対象プロジェクトを指定
- build-status：監視したい状態を指定

---

# Build State Change と Build Phase Change

## Build State Change（重要）

ビルド全体の状態変化を通知する。

用途

- ビルド成功通知
- ビルド失敗通知
- 後続処理開始
- 自動Rollback
- 運用監視

代表的な状態

- IN_PROGRESS
- SUCCEEDED
- FAILED
- STOPPED

**試験・実務ともにこちらが基本。**

---

## Build Phase Change

ビルド内部のフェーズ変化を通知する。

例

- DOWNLOAD_SOURCE
- INSTALL
- PRE_BUILD
- BUILD
- POST_BUILD

用途

- 詳細なデバッグ
- ボトルネック調査
- フェーズ単位の監視

通常の運用通知ではあまり利用しない。

---

# CodeBuild の重要な Status

| Status | 意味 |
|---------|------|
| IN_PROGRESS | 実行中 |
| SUCCEEDED | 成功 |
| FAILED | 失敗 |
| STOPPED | 停止 |

※ `FAILURE` や `ABORTED` は CodeBuild の Build State Change のステータスではない。

---

# State を持つ代表的な AWS サービス

| サービス | 覚えるべき State |
|-----------|------------------|
| CodeBuild | IN_PROGRESS / SUCCEEDED / FAILED / STOPPED |
| CodePipeline | InProgress / Succeeded / Failed / Stopped |
| CodeDeploy | Created / InProgress / Succeeded / Failed / Stopped |
| Step Functions | RUNNING / SUCCEEDED / FAILED / TIMED_OUT / ABORTED |
| CloudFormation | CREATE_COMPLETE / UPDATE_COMPLETE / FAILED / ROLLBACK_COMPLETE |
| ECS Task | PENDING / RUNNING / STOPPED |
| EC2 | pending / running / stopping / stopped / terminated |

---

# CloudFormation の重要な状態

すべて暗記する必要はない。

ライフサイクルを理解することが重要。

```
CREATE_IN_PROGRESS
        ↓
CREATE_COMPLETE
        ↓
CREATE_FAILED
```

更新時

```
UPDATE_IN_PROGRESS
        ↓
UPDATE_COMPLETE
        ↓
UPDATE_FAILED
        ↓
ROLLBACK
```

特に

- UPDATE_FAILED
- UPDATE_ROLLBACK_FAILED

が大切。

---

# DevOps で重要な考え方

AWS の多くのサービスは **State（状態）** を持つ。

その状態変化を

```
State Change
      ↓
EventBridge
      ↓
SNS / Lambda / Step Functions
```

へ連携する **イベント駆動アーキテクチャ** が AWS の基本設計パターンである。

---

# 覚えるポイント

- Build 全体を監視するなら **Build State Change**
- Build 内部の工程を監視するなら **Build Phase Change**
- EventBridge は **State Change を起点に他サービスと連携するための中心サービス**
- 「State → EventBridge → 自動処理」は AWS DevOps の頻出パターン

---

# ハンズオン：CodeBuild → EventBridge → SNS 通知

## 目的

CodeBuild の Build State Change を EventBridge で検知し、
FAILED / STOPPED 時に SNS 通知する。

```
CodeBuild
    │
    ▼
Build State Change
    │
    ▼
EventBridge Rule
    │
    ▼
SNS
    │
    ▼
Email
```

---

# Step0 共通変数

```bash
Region=$(aws configure get region)
AccountId=$(aws sts get-caller-identity \
  --query Account \
  --output text)
MailAddress="your@example.com"
```

---

# Step1 SNS Topic 作成

```bash

TopicArn=$(aws sns create-topic \
  --name codebuild-notify \
  --query 'TopicArn' \
  --output text)

echo $TopicArn
```

メール登録

```bash
aws sns subscribe \
  --topic-arn "$TopicArn" \
  --protocol email \
  --notification-endpoint "$MailAddress"
```

メールの **Confirm Subscription** をクリック。


# Step2 IAM Role

Role ARN取得

```bash
ServiceRoleArn=$(aws iam get-role \
  --role-name CodeBuildServiceRole \
  --query 'Role.Arn' \
  --output text)

echo $ServiceRoleArn
```

※存在しない場合は Console または CloudFormation で作成する。

必要最低限の権限

- CloudWatchLogsFullAccess
- AmazonS3ReadOnlyAccess

（学習用なら AWSCodeBuildDeveloperAccess でも可）

---

# Step3 CloudFormation

## build.yml

```yaml
AWSTemplateFormatVersion: '2010-09-09'

Parameters:

  ServiceRole:
    Type: String

Resources:

  CodeBuildProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: Demo-CodeBuild
      ServiceRole: !Ref ServiceRole
      Artifacts:
        Type: NO_ARTIFACTS
      Environment:
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/amazonlinux2-x86_64-standard:5.0
        Type: LINUX_CONTAINER
      Source:
        Type: NO_SOURCE
        BuildSpec: |
          version: 0.2
          phases:
            build:
              commands:
                - echo "Hello"
```

---

作成

```bash
ServiceRoleArn=$(aws iam get-role \
  --role-name CodeBuildServiceRole \
  --query 'Role.Arn' \
  --output text)

aws cloudformation deploy \
  --stack-name codebuild-demo \
  --template-file build.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
  ServiceRole="$ServiceRoleArn"
```

Build Project
```
Demo-CodeBuild
```
が作成される。

---

# Step4 EventBridge Rule

event-pattern.json

```json
{
  "source": [
    "aws.codebuild"
  ],
  "detail-type": [
    "CodeBuild Build State Change"
  ],
  "detail": {
    "project-name": [
      "Demo-CodeBuild"
    ],
    "build-status": [
      "FAILED",
      "STOPPED"
    ]
  }
}
```

作成

```bash
aws events put-rule \
  --name CodeBuildFailed \
  --event-pattern file://event-pattern.json
```

---

SNS をターゲットに追加

```bash
aws events put-targets \
  --rule CodeBuildFailed \
  --targets \
Id=1,Arn="$TopicArn"
```

---

SNS Publish 権限

```bash
aws sns add-permission \
  --topic-arn "$TopicArn" \
  --label EventBridge \
  --aws-account-id "$AccountId" \
  --action-name Publish
```

---

# Step5 成功確認

Build 実行

```bash
aws codebuild start-build \
  --project-name Demo-CodeBuild
```

成功する。

↓

SNS通知は来ない。

---

# Step6 FAILED を発生させる

CloudFormation の BuildSpec を変更

```yaml
commands:
  - echo Hello
  - exit 1
```

更新

```bash
aws cloudformation deploy \
  --stack-name codebuild-demo \
  --template-file build.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
  ServiceRole="$ServiceRoleArn"
```

---

再度 Build

```bash
aws codebuild start-build \
  --project-name Demo-CodeBuild
```

↓

Build

```
FAILED
```

↓

EventBridge

↓

SNS

↓

メール受信

---

# Step7 STOPPED を確認

Build 開始

```bash
BuildId=$(aws codebuild start-build \
  --project-name Demo-CodeBuild \
  --query 'build.id' \
  --output text)

echo $BuildId
```

停止

```bash
aws codebuild stop-build \
  --id "$BuildId"
```

↓

Build

```
STOPPED
```

↓

EventBridge

↓

SNS通知

---

# 確認

Build一覧

```bash
aws codebuild list-builds
```

Build詳細

```bash
aws codebuild batch-get-builds \
  --ids "$BuildId"
```

Rule確認

```bash
aws events list-rules
```

Target確認

```bash
aws events list-targets-by-rule \
  --rule CodeBuildFailed
```

CloudFormation

```bash
aws cloudformation describe-stacks \
  --stack-name codebuild-demo
```

---

# 学べること

- CloudFormation による CodeBuild 作成
- CodeBuild Build State Change
- EventBridge Event Pattern
- SNS 通知
- FAILED の検知
- STOPPED の検知
- イベント駆動アーキテクチャの基本パターン

```
CodeBuild
      │
      ▼
Build State Change
      ▼
EventBridge
      ▼
SNS
      ▼
Email
```

この構成は、Lambda・Step Functions・ECS・CloudFormation など他の AWS サービスでも同じ考え方で応用できる。

