# AWS Config による継続的なコンプライアンス管理（タグ管理）

## 概要

AWSでは、リソースのタグや設定が組織のルールに準拠しているかを**継続的に監査**し、違反が見つかった場合に**自動修正**する仕組みを構築できます。

このような要件では、**AWS Config + SSM Automation** の組み合わせが定番です。

---

# ガバナンスの3つの役割

| 目的 | 代表サービス | 役割 |
|------|-------------|------|
| **Prevent（予防）** | IAM Policy / SCP / Tag Policy | ルール違反の作成・変更を防ぐ |
| **Detect（検出）** | AWS Config | リソースを継続的に評価し、非準拠を検出する |
| **Remediate（修正）** | SSM Automation | 非準拠リソースを自動修正する |

---

# 定番構成

AWSでは次の構成がよく利用されます。

```text
AWS Config
（継続的に評価）
        │
        ▼
Remediation
        │
        ▼
SSM Automation
（自動修正）
```

例えばタグ管理では、AWS Config の **required-tags** マネージドルールを利用して、必要なタグが存在するかを継続的に評価できます。

タグ値まで厳密に制御したい場合は、AWS Config のカスタムルール（Lambda や Guard）などを利用します。

非準拠と判定された場合は、SSM Automation の Remediation を実行することで、自動的にタグを付与・修正できます。

## Config Ruleの評価結果

- COMPLIANT
    - ルールに準拠している

- NON_COMPLIANT
    - ルール違反

- NOT_APPLICABLE
    - 評価対象外

例：

- EC2向けのルールをS3バケットが評価した場合
- 対象リージョンにリソースが存在しない場合
→ NOT_APPLICABLE

---
### AWS Config

AWS Config はユーザーレベルで設定するものではない
基本的には AWSアカウント単位（リージョン単位）で有効化するサービス

```
AWS Account
│
├── AWS Config（有効化）
│      │
│      ├── 記録対象リソース
│      │      └ EC2, S3, IAM Role など
│      │
│      ├── Config Rule
│      │      └ required-tags など
│      │
│      └── Remediation
│             └ SSM Automation
│
└── IAM User / Role
       └ AWS Configを操作する権限を持つ
```

管理者が必要に応じて、有効化する

```
AWS Account
   ↓
AWS Config
   ↓
required-tags Rule
   ↓
EC2インスタンスを評価
   ↓
非準拠ならSSM Automation実行
```

### 個人・小規模環境で代替する場合：

実際、ユーザーレベルでは AWSConfig を使用できないので、別方法を考える
- 方法A：IaC（CloudFormation / Terraform）で強制する
    - メリット
        - 作成時点で正しい状態になる
        - レビュー可能
        - Git管理できる
        - DevOps文化と相性が良い
    - デメリット
        - 手動作成されたリソースは漏れる
- 方法B：AWS CLI / Lambdaで定期チェック
    ```
    EventBridge Scheduler
            ↓
    Lambda
            ↓
    DescribeInstances
            ↓
    タグチェック
            ↓
    SNS通知
    ```
    - イメージ
    ```
    if "env" not in tags:
        notify()
    ```
    - または自動修正：
    ```
    create_tags(
        Resources=[instance_id],
        Tags=[
            {
            "Key": "env",
            "Value": "dev"
            }
        ]
    )
    ```
    - メリット
        - 自分でも作れる
        - AWS Configより自由度が高い
    - デメリット
        - AWS Configほど標準化されていない
- 方法C：IAM Policy 作成時点で禁止
    - 既存リソースを見ることはできない
    - 修正できない

### 実務おすすめ構成

- 小規模
```
CloudFormation/CDK
        +
IAM Policy
        +
定期Lambdaチェック
```
- 大規模
```
Organizations
        +
AWS Config
        +
SSM Automation
```
---

# 他サービスとの役割の違い

## EventBridge

イベント駆動で処理を実行するサービスです。

リソース作成イベントなどを契機に Lambda を実行する用途には適していますが、**既存リソース全体を継続的に監査する用途には向いていません。**

---

## IAM Policy / SCP / Tag Policy

これらは**Prevent（予防）**の仕組みです。

- タグなしでは作成させない
- 指定したタグ値以外を許可しない

といった制御はできますが、

- 既存リソースの監査
- 非準拠リソースの自動修正

は行えません。

---

# 試験で覚えるポイント

## ガバナンスは3段階で整理する

```text
Prevent（予防）
    ↓
Detect（検出）
    ↓
Remediate（修正）
```

代表サービスは次のように整理できます。

|役割|サービス|
|---|---|
|Prevent|IAM Policy / SCP / Tag Policy|
|Detect|AWS Config|
|Remediate|SSM Automation|

---

# 見分け方

次のような要件があれば **AWS Config + SSM Automation** を考えます。

- 継続的な監査
- コンプライアンス管理
- 非準拠リソースの検出
- 自動修正
- Remediation

一方、

- 作成時に制御したい
- 作成を拒否したい
- 強制したい

という要件であれば、IAM Policy・SCP・Tag Policy などの **Prevent（予防）** の仕組みが候補になります。

---

# この問題で理解しておきたいこと

AWSでは、**「防ぐ」「見つける」「直す」**をそれぞれ別のサービスが担当しています。

- **Prevent**：IAM Policy / SCP / Tag Policy
- **Detect**：AWS Config
- **Remediate**：SSM Automation

この役割分担はタグ管理だけでなく、セキュリティやコンプライアンス全般で頻出の考え方です。

---

# 個人レベルでテストする場所と手順

### 結論：

- 自分のAWSアカウントで試すのが一番良い
- AWS Configはアカウント全体を見るサービスなので、会社sandboxだと影響範囲が読みにくい

---

# AWS Config + SSM Automation ハンズオン（タグ管理）

## 目的

AWS Config と SSM Automation を利用して、

- リソースを継続的に監査する
- 非準拠リソースを検出する
- 自動修正（Remediation）する

一連の流れを体験する。

## なぜ自分のAWSアカウントで試すのか

AWS Config はアカウント全体を対象に構成情報を収集・評価するサービスです。

会社のSandbox環境では既存リソースへの影響や運用ルールを考慮する必要があるため、学習目的であれば個人AWSアカウントで試す方が安全です。

---

# 前提

- 個人AWSアカウント
- 東京リージョン（ap-northeast-1）
- AdministratorAccess 相当の権限

作成するリソース

- AWS Config
- Config Rule（required-tags）
- EC2インスタンス
- SSM Automation（Remediation）

---

# Step1 AWS Configを有効化

AWS Console

```
AWS Config
    ↓
Get started
```

設定

- Record all resources
- Include global resources（任意）
- S3バケット（自動作成でOK）

Recording を開始する。

---

# Step2 Config Ruleを作成

```
AWS Config
    ↓
Rules
    ↓
Add Rule
```

Managed Rule を選択

```
required-tags
```

設定例

```
Tag Key

env
```

許可する値

```
dev
stg
prd
```

保存する。

---

# Step3 テスト用EC2を作成

EC2を作成する。

ポイント

```
envタグを付けない
```

または

```
env=test
```

のように不正な値を設定する。

---

# Step4 Configの評価を確認

数分待つ。

```
AWS Config
    ↓
Rules
    ↓
required-tags
```

期待結果

```
NON_COMPLIANT
```

となる。

対象リソースを開き、

どのタグが不足・不正なのか確認する。

---

# Step5 Remediationを設定

```
AWS Config
    ↓
Rules
    ↓
required-tags
    ↓
Actions
    ↓
Manage remediation
```

設定

```
Automatic remediation
```

Runbook

```
AWS-CreateTags
```

タグ

```
Key

env

Value

dev
```

Remediation を保存する。

※ IAMロール作成を求められたら許可する。

---

# Step6 自動修正を確認

数分待つ。

EC2のタグを確認する。

Before

```
Tags

なし
```

↓

After

```
env=dev
```

になれば成功。

---

# Step7 再評価を確認

再度 Config Rule を開く。

期待結果

```
COMPLIANT
```

になっていることを確認する。

---

# 動作イメージ

```text
EC2作成
（タグなし）
      │
      ▼
AWS Config
(required-tags)
      │
      ▼
NON_COMPLIANT
      │
      ▼
Remediation
      │
      ▼
SSM Automation
(AWS-CreateTags)
      │
      ▼
タグ付与
      │
      ▼
COMPLIANT
```

---

# 試験との対応

|サービス|役割|
|---|---|
|AWS Config|継続的な監査（Detect）|
|required-tags|タグのコンプライアンス評価|
|Remediation|修正処理の起動|
|SSM Automation|自動修正（Remediate）|
|IAM Policy / SCP|作成時の制御（Prevent）|

---

# 学習ポイント

- AWS Config は「構成変更イベント」と「定期評価」により継続的にリソースを監査する。
- `required-tags` はマネージドルールとしてすぐ利用できる。
- Remediation を設定することで、非準拠リソースを SSM Automation で自動修正できる。
- **Detect（AWS Config）→ Remediate（SSM Automation）** は、DOP-C02で頻出の運用パターンである。

---

# 費用

学習には以下の項目でコストが発生

- Configuration Items (CI): リソースの構成変更を記録するたびに課金
- Config Rule Evaluations: ルールの評価回数に応じて課金
- さらに、間接的に以下も発生
    - Config履歴保存用の S3
    - Config配信用の SNS（使う場合）
    - SSM Automation（無料枠内ならほぼ気にならない）
    - EC2（今回のテスト用インスタンス）


学習後は必ず削除
終わったら次を削除すれば安心です。

* EC2
* AWS Config Rule
* Remediation
* Configuration Recorder
* Delivery Channel
* Config用S3バケット（不要なら）