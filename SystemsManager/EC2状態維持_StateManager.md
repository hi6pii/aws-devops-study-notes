# EC2状態維持（SSM State Manager）

## 作成方法

CloudFormationのAWS::SSM::Associationで定義可能。

ただし目的はCloudFormation管理ではなく、
SSM State ManagerによるEC2設定維持。

```
SSM
│
├── Run Command
│   └── 任意コマンド実行
│
├── State Manager
│   └── desired state維持
│
└── Patch Manager
    └── OSパッチ管理
```

- template (ssm-state-manager-test.yaml)

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

Resources:

  EC2Role:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: [ec2.amazonaws.com]
            Action: sts:AssumeRole

      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore


  InstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref EC2Role


  MyEC2Instance:
    Type: AWS::EC2::Instance

    Properties:
      ImageId: !Ref ImageId
      InstanceType: t3.micro
      KeyName: !Ref KeyName
      SubnetId: !Ref SubnetId

      SecurityGroupIds:
        - !Ref SecurityGroupId

      IamInstanceProfile: !Ref InstanceProfile

      Tags:
        - Key: Role
          Value: ResearchMachine
        - Key: OS
          Value: AmazonLinux2023


  StateManagerAssociation:
    Type: AWS::SSM::Association

    Properties:
      Name: AWS-RunShellScript
      Parameters:
        commands:
          - |
            yum install -y amazon-cloudwatch-agent
            systemctl enable amazon-cloudwatch-agent
            systemctl restart amazon-cloudwatch-agent
      Targets:
        - Key: tag:Role
          Values:
            - ResearchMachine
        - Key: tag:OS
          Values:
            - AmazonLinux2023

Outputs:
  InstanceId:
    Value: !Ref MyEC2Instance
```
- 起動
```bash
aws cloudformation create-stack \
  --stack-name ssm-state-manager-first \
  --template-body file://ssm-state-manager-first.yaml \
  --capabilities CAPABILITY_IAM \
  --parameters \
    ParameterKey=ImageId,ParameterValue=ami-xxxxxxxx \
    ParameterKey=KeyName,ParameterValue=my-key \
    ParameterKey=SubnetId,ParameterValue=subnet-xxxxxxxx \
    ParameterKey=SecurityGroupId,ParameterValue=sg-xxxxxxxx
```
- tag例
```
Role=RStudio
OS=Ubuntu22

Role=JupyterHub
OS=AmazonLinux2023
```
- 2台目
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

  SecondEC2:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref ImageId
      InstanceType: t3.micro
      KeyName: !Ref KeyName
      SubnetId: !Ref SubnetId
      SecurityGroupIds:
        - !Ref SecurityGroupId
      IamInstanceProfile:
        !Ref InstanceProfile
      Tags:
        - Key: Role
          Value: ResearchMachine

Outputs:
  InstanceId:
    Value: !Ref SecondEC2
```
- 起動
``` sh
aws cloudformation create-stack \
  --stack-name ssm-second-ec2-test \
  --template-body file://ec2-second-test.yaml \
  --parameters \
    ParameterKey=ImageId,ParameterValue=ami-xxxxxxxx \
    ParameterKey=KeyName,ParameterValue=my-key \
    ParameterKey=SubnetId,ParameterValue=subnet-xxxxxxxx \
    ParameterKey=SecurityGroupId,ParameterValue=sg-xxxxxxxx
```