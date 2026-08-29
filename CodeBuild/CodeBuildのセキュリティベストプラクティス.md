
### サービス関係性まとめ

#### 共通理解：

```
           sts:AssumeRole
                 ▲
                 │
        Trust Policy評価
                 │
          Temporary Credential
                 │
     ┌───────────┼────────────┐
     │           │            │
   EC2        CodeBuild     Lambda
     │           │            │
   IMDS      環境変数注入   環境変数注入
     │           │            │
     └───────────┼────────────┘
                 │
              AWS SDK
                 │
          s3:GetObject
                 │
        IAM Policy評価
                 │
                 ▼
                S3
                 │
         Bucket Policy評価
                 ▼
              Access
```

#### EC2の場合：

```
Application
      │
      │ aws s3 cp ...
      ▼
AWS SDK
      │
      │① Credentialある？
      ▼
169.254.169.254 (IMDS)
      │
      │② AWS内部では
      ▼
sts:AssumeRole
      │
      │ Trust Policy評価
      ▼
Temporary Credentials
      │
      │③ IMDSからSDKへ返す
      ▼
AWS SDK
      │
      │④ s3:GetObject
      ▼
IAM Policy評価
      │
      ▼
S3
      │
      │⑤ Bucket Policy評価
      ▼
アクセス許可
```

#### CodeBuildの場合：

```
CodeBuild起動
      │
      │① AWS内部
      ▼
STS: AssumeRole を要求
      │
      │ Trust Policyを評価「CodeBuildはこのRoleを使って良いか」 Yes/No
      ▼
STS が Temporary Credentialsを発行
(AccessKeyId, SecretAccessKey, SessionToken: 有効期限付き)
      │
      │② AWSが環境変数へ設定
      ▼
AWS SDK
      │
      │③ s3:GetObject
      ▼
IAM Policy評価
      │
      ▼
S3
      │
      │④ Bucket Policy評価
      ▼
アクセス許可
```

---


#### 用語理解

- IMDS:
    - Instance Metadata Service
    - EC2がIAMロールの一時認証情報などを取得するためのローカルサービス（169.254.169.254）
    - EC2がその認証情報を受け取る窓口
- STS:
    - Security Token Service
    - Trust Policy=「このユーザー・サービスはログインして良いか」を使
    - IAMロールから一時的な認証情報（Temporary Credentials）を発行するサービス
    - 認証情報を作る工場
- SDK:
    - Software Development Kit
    - AWS APIを簡単・安全に呼び出すためのライブラリ（認証・署名も自動化）
    - 受け取った認証情報を使ってAWSサービスへアクセスする運転手