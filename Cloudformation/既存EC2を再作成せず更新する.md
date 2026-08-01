# CloudFormationで既存EC2を再作成せず設定を更新する

## 概要

Amazon EC2 Auto Scaling環境では、新しく起動するインスタンスだけでなく、稼働中のインスタンスにも設定変更を反映したい場合があります。

CloudFormationでは、`AWS::CloudFormation::Init`、`cfn-init`、`cfn-hup` を組み合わせることで、インスタンスを再作成することなく設定を更新できます。

---

## ユースケース

- ミドルウェアのバージョン更新
- 設定ファイルの変更
- サービスの起動・再起動
- パッケージの追加

要件

- 新規EC2は最新設定で起動
- 既存EC2もIn-placeで更新
- CloudFormationのみで管理

---

## 更新アイデア

1. cfn-init + cfn-hup（今回）
2. Instance Refresh（DevOps頻出）
3. Rolling Update（CloudFormation更新制御）
4. SSM Association（別管理思想）

---

## アーキテクチャ

```text
CloudFormation Stack
        │
        ▼
AWS::CloudFormation::Init (Metadata)
        │
        ▼
     cfn-init
        │
        ├── Package Install
        ├── File Update
        └── Service Restart

        ▲
        │
     cfn-hup
        │
 Metadata Change Detection
```

---
## リソース

### Metadataとは

CloudFormationのリソースには、主に以下の2種類の情報があります。

|項目|役割|
|--|--|
|Properties|AWSリソースそのものの設定|
|Metadata|リソースに付随する追加情報・構成管理情報|

- Properties例：これはEC2自体の設定
```yaml
Properties:
  InstanceType: t3.micro
```
- Metadata例: これはEC2内部をどのような状態にするかの定義
```yaml
Metadata:
  AWS::CloudFormation::Init:
    config:
      files:
        /tmp/version.txt:
          content: "version 1"
```

---

## コンポーネント

### AWS::CloudFormation::Init

EC2のあるべき状態を定義する。
Metadata配下に記述し、以下の設定を管理できる。

- Packages
- Files
- Sources
- Commands
- Services

---

### cfn-init

インスタンス起動時にMetadataを取得し、定義された状態を適用する。

実行タイミング

- UserDataから呼び出す
- 初回起動時

---

### cfn-hup

CloudFormation Metadataを監視するデーモン。

Metadataが更新されると、自動で`cfn-init`を再実行する。

これにより既存EC2も最新状態へ更新できる。

---

## 更新フロー

```text
CloudFormation Update
        │
Metadata変更
        │
cfn-hup検知
        │
cfn-init再実行
        │
Package更新
File更新
Service再起動
```

---

## メリット

- インスタンスを再作成しない
- CloudFormationのみで管理できる
- Infrastructure as Codeを維持できる

---

## 他方式との比較

|方法|既存EC2更新|EC2再作成|
|------|-----------|---------|
|cfn-init + cfn-hup|○|不要|
|Instance Refresh|○|必要|
|Rolling Update|○|必要|
|SSM State Manager|○|不要（CloudFormation外）|

---

## ポイント

- `AWS::CloudFormation::Init` は「あるべき状態」を定義する
- `cfn-init` は初回構築を担当する
- `cfn-hup` は変更検知と再適用を担当する
- **「既存EC2を再作成せず更新」がキーワードなら、この組み合わせを検討する**

---

## テスト

### 1. template用意

```yaml
AWSTemplateFormatVersion: "2010-09-09"

Parameters:
  ImageId:
    Type: AWS::EC2::Image::Id

  KeyName:
    Type: AWS::EC2::KeyPair::KeyName

  SubnetId:
    Type: AWS::EC2::Subnet::Id

  SecurityGroupId:
    Type: AWS::EC2::SecurityGroup::Id

  Version:
    Type: String
    Default: "version 1"

Resources:
  MyEC2Instance:
    Type: AWS::EC2::Instance

    Metadata:
      AWS::CloudFormation::Init:
        config:
          files:
            /tmp/version.txt:
              content: !Sub "${Version}\n"

    Properties:
      ImageId: !Ref ImageId
      InstanceType: t3.micro
      KeyName: !Ref KeyName
      SubnetId: !Ref SubnetId
      SecurityGroupIds:
        - !Ref SecurityGroupId

      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash -xe

          /opt/aws/bin/cfn-init \
            -v \
            --stack ${AWS::StackName} \
            --resource MyEC2Instance \
            --region ${AWS::Region}

Outputs:
  InstanceId:
    Value: !Ref MyEC2Instance
```

### 2. 起動

```bash
aws cloudformation create-stack \
  --stack-name cfn-init-test \
  --template-body file://cfn-init-test.yaml \
  --parameters \
    ParameterKey=ImageId,ParameterValue=ami-xxxxxxxx \
    ParameterKey=SubnetId,ParameterValue=subnet-xxxxxxxx \
    ParameterKey=SecurityGroupId,ParameterValue=sg-xxxxxxxx

aws cloudformation wait stack-create-complete \
  --stack-name cfn-init-test

aws cloudformation describe-stacks \
  --stack-name cfn-init-test \
  --query "Stacks[0].Outputs"
```

---

### 3. 起動確認
- ssh接続
```
cat /tmp/version.txt
```
- 結果
```
version 1
```

---

### 4. 更新処理（反映されない）

- コマンド：
```
aws cloudformation update-stack \
  --stack-name cfn-init-test \
  --template-body file://cfn-init-test.yaml \
  --parameters \
    ParameterKey=ImageId,UsePreviousValue=true \
    ParameterKey=SubnetId,UsePreviousValue=true \
    ParameterKey=SecurityGroupId,UsePreviousValue=true \
    ParameterKey=Version,ParameterValue="version 2"
```
- 結果
    - この状態ではEC2内部のファイルは更新されない。
- 理由：
    - この構成ではcfn-initはUserDataから初回起動時に実行される
    - Stack Updateだけでは自動実行されない
- 一回削除：
```
aws cloudformation delete-stack \
  --stack-name cfn-init-test
```

---

### 5. 正しい版

``` yaml
AWSTemplateFormatVersion: "2010-09-09"

Parameters:

  ImageId:
    Type: AWS::EC2::Image::Id

  KeyName:
    Type: AWS::EC2::KeyPair::KeyName

  SubnetId:
    Type: AWS::EC2::Subnet::Id

  SecurityGroupId:
    Type: AWS::EC2::SecurityGroup::Id

  Version:
    Type: String
    Default: "version 1"

Resources:
  MyEC2Instance:
    Type: AWS::EC2::Instance

    Metadata:
      AWS::CloudFormation::Init:
        config:
          files:
            /tmp/version.txt:
              content: !Sub "${Version}\n"

            /etc/cfn/cfn-hup.conf:
              content: !Sub |
                [main]
                stack=${AWS::StackId}
                region=${AWS::Region}
                interval=1

            /etc/cfn/hooks.d/cfn-auto-reloader.conf:
              content: !Sub |
                [cfn-auto-reloader-hook]
                triggers=post.update
                path=Resources.MyEC2Instance.Metadata.AWS::CloudFormation::Init
                action=/opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource MyEC2Instance --region ${AWS::Region}
                runas=root

          services:
            sysvinit:
              cfn-hup:
                enabled: true
                ensureRunning: true

    Properties:
      ImageId: !Ref ImageId
      InstanceType: t3.micro
      KeyName: !Ref KeyName
      SubnetId: !Ref SubnetId
      SecurityGroupIds:
        - !Ref SecurityGroupId

      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash -xe

          /opt/aws/bin/cfn-init \
            -v \
            --stack ${AWS::StackName} \
            --resource MyEC2Instance \
            --region ${AWS::Region}

Outputs:
  InstanceId:
    Value: !Ref MyEC2Instance
```

- このテンプレートを用いやりなおすことで自動アップデートされる


### 6. cfn-hup版の確認

1. Stack作成後、初期状態確認

```bash
cat /tmp/version.txt
```

結果：

```text
version 1
```

---

2. Template変更

```yaml
Default: "version 1"
```

↓

```yaml
Default: "version 2"
```

---

3. Stack Update実行

```bash
aws cloudformation update-stack \
  --stack-name cfn-init-test \
  --template-body file://cfn-init-test.yaml \
  --parameters \
    ParameterKey=ImageId,UsePreviousValue=true \
    ParameterKey=SubnetId,UsePreviousValue=true \
    ParameterKey=SecurityGroupId,UsePreviousValue=true \
    ParameterKey=Version,ParameterValue="version 2"
```

---

4. 更新確認

```bash
cat /tmp/version.txt
```

結果：

```text
version 2
```

確認ポイント：

- EC2 Instance IDは変更されない
- Stack Updateだけで設定変更が反映される
- EC2の再作成は発生しない

---

### 注意

`services` の定義方法は、利用するOSのinit systemに依存する。

例：

- Amazon Linux系 → `sysvinit`
- systemd系 → `systemd`

---

## 7. ASG版

```yaml
AWSTemplateFormatVersion: "2010-09-09"

Parameters:

  ImageId:
    Type: AWS::EC2::Image::Id

  KeyName:
    Type: AWS::EC2::KeyPair::KeyName

  SubnetId:
    Type: AWS::EC2::Subnet::Id

  SecurityGroupId:
    Type: AWS::EC2::SecurityGroup::Id

  Version:
    Type: String
    Default: "version 1"

  DesiredCapacity:
    Type: Number
    Default: 2


Resources:

  EC2Role:

    Type: AWS::IAM::Role

    Properties:

      AssumeRolePolicyDocument:

        Version: "2012-10-17"

        Statement:

          - Effect: Allow

            Principal:

              Service:

                - ec2.amazonaws.com

            Action:

              - sts:AssumeRole


      ManagedPolicyArns:

        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore


  InstanceProfile:

    Type: AWS::IAM::InstanceProfile

    Properties:

      Roles:

        - !Ref EC2Role


  LaunchTemplate:

    Type: AWS::EC2::LaunchTemplate

    Properties:

      LaunchTemplateData:

        ImageId: !Ref ImageId

        InstanceType: t3.micro

        KeyName: !Ref KeyName

        SecurityGroupIds:

          - !Ref SecurityGroupId


        UserData:

          Fn::Base64: !Sub |

            #!/bin/bash -xe

            /opt/aws/bin/cfn-init \
              -v \
              --stack ${AWS::StackName} \
              --resource LaunchTemplate \
              --region ${AWS::Region}


            /opt/aws/bin/cfn-hup


        IamInstanceProfile:

          Name: !Ref InstanceProfile


    Metadata:

      AWS::CloudFormation::Init:

        config:

          files:


            /tmp/version.txt:

              content: !Sub "${Version}\n"


            /etc/cfn/cfn-hup.conf:

              content: !Sub |

                [main]

                stack=${AWS::StackId}

                region=${AWS::Region}

                interval=1


            /etc/cfn/hooks.d/cfn-auto-reloader.conf:

              content: !Sub |

                [cfn-auto-reloader-hook]

                triggers=post.update

                path=LaunchTemplate.Metadata.AWS::CloudFormation::Init

                action=/opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource LaunchTemplate --region ${AWS::Region}

                runas=root


          services:

            sysvinit:

              cfn-hup:

                enabled: true

                ensureRunning: true



  AutoScalingGroup:

    Type: AWS::AutoScaling::AutoScalingGroup

    Properties:

      VPCZoneIdentifier:

        - !Ref SubnetId


      MinSize: 1

      MaxSize: 3

      DesiredCapacity: !Ref DesiredCapacity


      LaunchTemplate:

        LaunchTemplateId:

          !Ref LaunchTemplate

        Version:

          !GetAtt LaunchTemplate.LatestVersionNumber



Outputs:

  AutoScalingGroupName:

    Value: !Ref AutoScalingGroup
```

- 想定
```
create-stack
 ↓
2台起動(version1)
 ↓
update-stack(version2)
 ↓
既存2台がversion2へ
 ↓
update-stack(desired=3)
 ↓
3台目がversion2で起動
```
- 初回起動
```sh
aws cloudformation create-stack \
  --stack-name cfn-init-asg-test \
  --template-body file://cfn-init-asg-test.yaml \
  --capabilities CAPABILITY_IAM \
  --parameters \
    ParameterKey=ImageId,ParameterValue=ami-xxxxxxxx \
    ParameterKey=KeyName,ParameterValue=my-key \
    ParameterKey=SubnetId,ParameterValue=subnet-xxxxxxxx \
    ParameterKey=SecurityGroupId,ParameterValue=sg-xxxxxxxx \
    ParameterKey=Version,ParameterValue="version 1" \
    ParameterKey=DesiredCapacity,ParameterValue=2
```
- 更新1
```
aws cloudformation update-stack \
  --stack-name cfn-init-asg-test \
  --template-body file://cfn-init-asg-test.yaml \
  --capabilities CAPABILITY_IAM \
  --parameters \
    ParameterKey=ImageId,UsePreviousValue=true \
    ParameterKey=KeyName,UsePreviousValue=true \
    ParameterKey=SubnetId,UsePreviousValue=true \
    ParameterKey=SecurityGroupId,UsePreviousValue=true \
    ParameterKey=Version,ParameterValue="version 2" \
    ParameterKey=DesiredCapacity,UsePreviousValue=true
```
- 更新2
```sh
aws cloudformation update-stack \
  --stack-name cfn-init-asg-test \
  --template-body file://cfn-init-asg-test.yaml \
  --capabilities CAPABILITY_IAM \
  --parameters \
    ParameterKey=ImageId,UsePreviousValue=true \
    ParameterKey=KeyName,UsePreviousValue=true \
    ParameterKey=SubnetId,UsePreviousValue=true \
    ParameterKey=SecurityGroupId,UsePreviousValue=true \
    ParameterKey=Version,UsePreviousValue=true \
    ParameterKey=DesiredCapacity,ParameterValue=3
```