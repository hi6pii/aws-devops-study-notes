
* AMIは後から暗号化できない。Copy AMI時のみ暗号化できる。
* 暗号化AMIをクロスアカウント共有するには、Launch Permission + KMS(CMK)共有 の両方が必要。

---

### 暗号化AMIのクロスアカウント共有（AMI Copy + KMS）

#### 概要

Amazon Machine Image (AMI) は作成後に暗号化状態を変更できません。

そのため、既存の非暗号化AMIを暗号化して利用したい場合は、AMI Copy を実行し、その際に暗号化を有効化して新しいAMIを作成します。

また、暗号化されたAMIを別AWSアカウントへ共有する場合は、AMIの共有だけでは不十分です。

AMIの実体となるEBSスナップショットはKMSキーで暗号化されているため、共有先アカウントがそのKMSキーを利用できるよう権限を付与する必要があります。

---

### AMIの暗号化

AMIは作成後に暗号化状態を変更できません。

```
非暗号化AMI
      │
      │ Copy AMI
      │ Encrypt = ON
      ▼
暗号化AMI（新規作成）
```

つまり

「暗号化したいならCopyする」

という設計になっています。

既存AMIを直接暗号化することはできません。

---

### 暗号化AMI共有の仕組み

#### 暗号化AMIを共有するときは

```
AMI
 ↓
EBS Snapshot
 ↓
KMS(CMK)
```

という構成になります。

インスタンス起動時は

```
Snapshot
    │
Decrypt
    │
EBS Volume作成
    │
EC2起動
```

という処理が内部で実行されます。

そのため、

AMIだけ共有しても

```
Decryptできない
```

ため起動できません。

---

### クロスアカウント共有で必要なもの

#### 暗号化AMIを共有する場合は

```
Account A
暗号化AMI
      │
      ├── Launch Permission
      │
      └── KMS Key Policy
              │
              ▼
Account B
```

最低限必要なのは

* AMIのLaunch Permission
* KMSキー(CMK)へのアクセス権限

の2つです。

どちらか一方だけでは利用できません。

---

### KMSキーの役割

暗号化AMIはKMSキーで保護されたEBS Snapshotを利用します。

そのため、

KMSには例えば

```
kms:Decrypt
kms:DescribeKey
kms:CreateGrant
```

などの権限が必要になります。

実際には

* Account A
    * KMS Key Policy
* Account B
    * IAM Policy

の両方で許可されて初めて利用できます。

---

### AWS管理キーと顧客管理キー

クロスアカウント共有では
```
aws/ebs
```
は利用できません。

共有可能なのは
```
Customer Managed Key
(CMK)
```
になります。

そのため

クロスアカウント利用を想定するAMIは、

最初からCMKで暗号化しておくことが推奨されます。

---

### AMIコピーの流れ

```
非暗号化AMI
      │
Copy AMI
(Encrypt=ON)
      │
      ▼
暗号化AMI(CMK)
      │
Launch Permission共有
      │
KMS共有
      │
      ▼
別Accountで利用
```
---

### 今回のケース

```
Account A
非暗号化AMI
```

これを

```
Account B
CMKで利用したい
```

という要件でした。

そのため

* ⭕ Copy AMI時に暗号化する
    * AMIは後から暗号化できないため
* ⭕ Launch PermissionとKMSキーを共有する
    * AMIだけでは復号できない

一方で

* ❌ S3へExportしてImport
    * VM Import/Exportの用途であり、AWSアカウント間AMI共有には不要
* ❌ RunInstances時に暗号化指定
    * 起動時に元AMIを暗号化することはできない
* ❌ Snapshotを共有してAMIを作り直す
    * 実現は可能だがAMI Copy機能の方がAMI属性も保持され、自動化・運用性に優れる

---

### よくあるトラブル

* AMIを共有したのに起動できない
    * KMSキー共有漏れ
* Launch PermissionはあるのにAccessDenied
    * KMS Key Policy不足
* aws/ebsを利用している
    * クロスアカウント共有不可
* AMIを暗号化したい
    * Copy AMIが必要

---

### 実務での確認手順

最初に確認するのは

* AMIが暗号化されているか
* 使用しているKMSキー
* Launch Permission
* KMS Key Policy
* Account BのIAM Policy

特に

```
AMIは見えるのに起動できない
```

場合は、

ほぼKMS権限を疑うと切り分けができます。

---

### ハンズオン：暗号化AMIのクロスアカウント共有

Step1 非暗号化AMIを作成

EC2からAMIを作成します。

```
EC2
   │
Create Image
   │
非暗号化AMI
```

---

Step2 Copy AMIで暗号化

AMI Copyを実行し、

```
Encrypt = ON
KMS Key = CMK
```

を指定します。

暗号化された新しいAMIが作成されることを確認します。

---

### Step3 Launch Permissionを共有

Account Bへ

```
Launch Permission
```

を追加します。

---

### Step4 KMSキー共有

CMKのKey Policyへ

Account Bを追加します。

必要に応じて

Account B側にもIAM Policyを設定します。

---

### Step5 起動確認

Account Bから

暗号化AMIを利用してEC2を起動します。

正常なら

```
AMI
↓
Snapshot復号
↓
EBS作成
↓
EC2起動
```

まで進みます。

---

Step6 KMS共有を外して確認

Key PolicyからAccount Bを削除します。

すると

* AMIは見える
* Launch Permissionもある

にもかかわらず

```
AccessDenied
```

となり起動できません。

Launch Permissionだけでは不足することを確認できます。

---

実務で活かすポイント

暗号化AMIの共有は、AMIそのものではなく、その背後にあるEBS SnapshotとKMSの仕組みを理解しているとトラブルシューティングが容易になります。

```
AMI
   │
EBS Snapshot
   │
KMS(CMK)
   │
Decrypt
   │
EC2起動
```

この構造を理解しておけば、

* 「AMIは見えるのに起動できない」
* 「AccessDeniedになる」
* 「なぜLaunch Permissionだけでは足りないのか」
* 「なぜCMKが必要なのか」

を、KMS権限という観点から切り分けられるようになります。

---

このシリーズ全体で統一するなら、冒頭の**2行サマリー（本質）**を毎回置くのがおすすめです。

* AMIは後から暗号化できない。Copy AMI時のみ暗号化できる。
* 暗号化AMIのクロスアカウント共有では、AMI共有だけでなくKMS(CMK)共有も必須。

この2行だけでも、後から見返したときに重要ポイントをすぐ思い出せます。