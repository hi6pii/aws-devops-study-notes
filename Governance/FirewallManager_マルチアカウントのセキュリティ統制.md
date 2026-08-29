FirewallManager/FirewallManager_マルチアカウントのセキュリティ統制.md

* AWS Firewall Manager は Organizations 配下の複数アカウントへセキュリティポリシーを一元適用するサービス。
* Organizations・Delegated Administrator・AWS Config の3つが利用の前提となる。

---

### Firewall Managerとは

Firewall Manager は、AWS Organizations 配下の複数アカウント・複数リージョンに対して、一貫したセキュリティポリシーを適用・維持するサービスです。

管理者は一度ポリシーを定義するだけで、新しく作成されたAWSアカウントやリソースにも自動的にポリシーを適用できます。

そのため

* AWS WAF
* AWS Shield Advanced
* Security Group
* AWS Network Firewall
* Route53 Resolver DNS Firewall

などを組織全体で統一して管理できます。

---

### Firewall Managerの全体像
```
Organization
      │
      ▼
Delegated Administrator
      │
      ▼
Firewall Manager
      │
      ├── AWS WAF
      ├── Shield Advanced
      ├── Security Group
      ├── Network Firewall
      └── DNS Firewall
```
Firewall Managerは各サービスを直接設定するのではなく、

「組織全体へポリシーを配布・維持する司令塔」

として動作します。

---

### AWS Configが必要な理由

Firewall Manager自身は、

EC2やALBなどの構成情報を直接収集しているわけではありません。
```
AWS Config
      │
構成情報を収集
      │
      ▼
Firewall Manager
      │
準拠状況を評価
      │
自動修復
```
つまり

AWS Config が”目”となり、Firewall Manager が”統制する”

という役割分担になっています。

AWS Config が無効だと、

* リソースを検出できない
* ポリシー違反を判定できない
* 自動修復も実行できない

ため、Firewall Managerは正常に機能しません。

---

### Organizationsが必要な理由

Firewall Managerは
```
Organization
    ├── Account A
    ├── Account B
    ├── Account C
```
全体へ同じポリシーを適用するサービスです。

Organizationsに所属していないアカウントは管理対象になりません。

新しいアカウントが追加された場合も、

Organizationsへ参加していれば自動的に管理対象になります。

---

### Delegated Administrator

運用では、
```
Management Account
        │
        ▼
Security Account
(Delegated Administrator)
        │
        ▼
Firewall Manager
```
という構成が推奨されています。

Management Accountを普段の運用で使用せず、

セキュリティ専用アカウントへ管理を委任することで、

運用と権限を分離できます。

---

### Firewall Managerが管理するもの

Firewall Manager自身が通信を制御するわけではありません。

実際に通信を制御するのは各サービスです。
```
Firewall Manager
        │
        ├── WAF Rule
        ├── Shield Protection
        ├── Security Group
        ├── Network Firewall
        └── DNS Firewall
```
Firewall Managerは

「設定を配布・維持する管理サービス」

になります。

---

### ポリシー適用の流れ
```
Organizations
        │
AWS Config
        │
Firewall Manager
        │
ポリシー作成
        │
対象
・OU
・Account
・Tag
        │
自動適用
```
対象は

* OU
* AWSアカウント
* リソースタグ

など柔軟に指定できます。

例えば
```
Environment=Production
```
というタグを付けたALBだけへ

WAFを自動適用することも可能です。

---

新しいAWSアカウントが追加されたら

例えば
```
Organization
    ├── Dev
    ├── Test
    └── Production
```
へ
```
New Account
```
を追加した場合でも、

OUのポリシー対象であれば
```
Firewall Manager
        │
        ▼
自動適用
```
されます。

これがFirewall Manager最大のメリットです。

---

よくあるトラブル

* Firewall Managerでリソースが表示されない
    * AWS Configが無効
* 新しいアカウントへポリシーが適用されない
    * Organizationsへ参加していない
* Firewall Managerが利用できない
    * Delegated Administrator未設定
* 一部リージョンだけ管理できない
    * Config Recorderがそのリージョンで無効

---

### 実務での確認手順

最初に確認するのは

* Organizationsへ参加しているか
* Delegated Administratorが設定されているか
* AWS Configが全リージョンで有効か
* Config Recorderが動作しているか
* Firewall Manager Policyの対象(OU・Account・Tag)

特に
```
Firewall Managerが効かない
```
場合は、

ほとんどがConfigまたはOrganizations周りから切り分けを始めます。

---

### ハンズオン：Firewall ManagerでWAFを一元管理する

#### Step1 Organizationsを作成

Organizationsを作成し、

管理対象となるAWSアカウントを参加させます。
```
Management Account
      │
Organizations
      │
Member Accounts
```
---

### Step2 Delegated Administratorを設定

セキュリティ専用アカウントを

Firewall Manager管理者として委任します。
```
Management
      │
Delegate
      ▼
Security Account
```
---

### Step3 AWS Configを有効化

全アカウント・全リージョンで

AWS Configを有効化します。
```
AWS Config
    │
Configuration Recorder
    │
Snapshot
```
Recorderが有効になっていることも確認します。

---

### Step4 Firewall Managerを有効化

Security Accountから

Firewall Managerを開きます。

Organizationsが認識されていることを確認します。

---

### Step5 WAFポリシーを作成

例えば
```
対象
Environment=Production
```
のALBへ

AWS Managed Rulesを適用します。
```
Firewall Manager
        │
AWS Managed Rules
        │
Production ALB
```
---

### Step6 自動適用を確認

Productionタグを持つALBを新しく作成します。

数分後に
```
WAF
```
が自動的に関連付けられることを確認します。

タグベースで自動適用される動作を体験できます。

---

### Step7 Configを停止して確認（検証環境のみ）

Config Recorderを停止すると、

Firewall Managerが準拠状況を評価できなくなります。

この状態を確認すると、

**「Configが目であり、Firewall Managerはその情報を利用して統制している」**ことを実感できます。

---

### 実務で活かすポイント

Firewall Managerは、各サービスの通信制御を行うものではなく、組織全体のセキュリティポリシーを配布・維持する管理レイヤーです。
```
AWS Config
（構成情報を収集）
        │
        ▼
Firewall Manager
（評価・統制）
        │
        ▼
WAF / Shield / Security Group
（実際に通信を制御）
```
この役割分担を理解しておくと、

* 「なぜAWS Configが必須なのか」
* 「なぜOrganizationsが必要なのか」
* 「Firewall Managerは何をしていて、各セキュリティサービスは何をしているのか」
* 「どこからトラブルシューティングを始めるべきか」

を体系的に説明できるようになります。

---

#### 本質（2行で覚える）

* Firewall Managerは、Organizations全体へセキュリティポリシーを一元配布・維持する管理サービスであり、通信を直接制御するのはWAFやSecurity Groupなど各サービス。
* AWS Configが「目」となって構成情報を収集し、Firewall Managerがその情報を使って準拠状況の評価・自動修復を行う。この役割分担が最も重要なポイント。

---

### チープにできるミニハンズオン

Firewall Managerそのものを触ることよりも、土台となるサービスを理解することを重視します。

```
Organizations（無料）
      │
      ▼
2アカウント作成（無料）
      │
      ▼
AWS Config有効化（Configuration Item数課金。EC2・ALB・SG程度ならかなり安い）
        - Config 数円-数十円
        - FireWall Manager policy -10円
        - plus:
                - EC2 t3.micro 10-30円/時間
                - ALB 30-50円/時間
                - WAF ACL 10-30円
        - Shield Advanced 超高額
      │
      ▼
ALB + WAF作成
      │
      ▼
Configで構成変更を確認
      │
      ▼
（最後に）
Firewall Managerの画面で
「このWAFを全体へ配布できる」
という流れを理解する
      │
      ▼
終わったら：
ALB削除
EC2削除
WAF削除
Firewall Manager Policy削除
Config停止（必要なら）
```

この順番なら、Firewall Managerを単独で覚えるのではなく、「Organizations・Config・WAFを組み合わせて組織全体を統制するサービス」という本質が自然に身につきます。これは試験対策だけでなく、実務でもそのまま役立つ理解になります。