
---

## 【実践記】Terraform × GitHubでAWS移行を完全IaC化した記録

〜2人チームで3ヶ月、Oracle RDS含む60台本番構築〜

---

### 🔧 プロジェクト概要

* **目的**：オンプレ環境からAWSへ移行（全リソースをIaC化）
* **規模**：サーバー台数60台（EC2＋ECS/Fargate）
* **方式**：Terraform + GitHub（CI/CD自動化）
* **構築対象リージョン**：東京（ap-northeast-1）、大阪（ap-northeast-3）
* **体制**：エンジニア2名（IaC専任1名、AWS設計/レビュー1名）
* **期間**：全体で約3ヶ月（要件定義〜本番環境applyまで）

---

## 🚀 フェーズ別展開と内容

### フェーズ1：Terraform最小構成の実装（1週間）

#### ✅ 構成要素と設定

| 項目             | 内容                         |
| -------------- | -------------------------- |
| VPC            | 10.0.0.0/16                |
| Public Subnet  | 10.0.1.0/24（Web/AP系、NAT設置） |
| Private Subnet | 10.0.2.0/24（DB/ECS系）       |
| IGW/NATGW      | パブリック側にIGW/NATを配置          |
| EC2            | Amazon Linux 2、テスト用SSH許可   |
| SG             | SSH (22)、HTTP(80)のみ許可      |

#### Terraformサンプル（抜粋）

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  map_public_ip_on_launch = true
  availability_zone = "ap-northeast-1a"
}

resource "aws_subnet" "private" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.2.0/24"
  map_public_ip_on_launch = false
  availability_zone = "ap-northeast-1a"
}
```

---

### フェーズ2：監査・セキュリティ対応（2週間）

#### ✅ 実装内容

| 項目         | 内容                                   |
| ---------- | ------------------------------------ |
| CloudTrail | グローバル有効、S3にログ集約                      |
| CloudWatch | アラーム設定（EC2停止/CPU高負荷）                 |
| SNS通知      | CloudWatchアラーム → Lambda経由通知          |
| KMS        | RDS/EBS/S3全て暗号化対応                    |
| IAM        | ECS Task用Role、EC2 Instance Profile設定 |

#### Terraformサンプル（CloudTrail）

```hcl
resource "aws_cloudtrail" "main" {
  name                          = "org-trail"
  s3_bucket_name                = aws_s3_bucket.trail_logs.bucket
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true
  event_selector {
    read_write_type           = "All"
    include_management_events = true
  }
}
```

---

### フェーズ3：本番環境の全体設計・構築（2ヶ月）

#### ✅ サービス別構成内容

| サービス            | 配置サブネット | 内容                       |
| --------------- | ------- | ------------------------ |
| EC2（API/Batch系） | パブリック   | 固定IP、NLB背後で運用            |
| ECS/Fargate     | プライベート  | ALB + ECSでコンテナ運用、自動スケール  |
| RDS（Oracle）     | プライベート  | Multi-AZ構成、KMS暗号化、監査ログ有効 |
| Route53         | ー       | プライベートゾーン＋パブリックゾーン       |
| ALB/NLB         | パブリック   | ECS・EC2を背後に持つLB構成        |
| S3（ログ保管）        | プライベート  | バケットポリシーで制限、KMS暗号化       |
| Secrets Manager | プライベート  | RDS・ECSパスワード等を管理         |

#### モジュール構成（Terraform）

```
modules/
├── vpc/
├── subnet/
├── rds/
├── ec2/
├── ecs-fargate/
├── security-group/
├── cloudtrail/
├── monitoring/
├── kms/
```

#### RDS（Oracle）構成詳細

| 項目        | 内容                                    |
| --------- | ------------------------------------- |
| インスタンスタイプ | db.m6g.large                          |
| エンジン      | oracle-se2                            |
| バージョン     | 19.0.0                                |
| マルチAZ     | 有効（ap-northeast-1a/1c）                |
| 暗号化       | KMS適用（カスタムキー）                         |
| モニタリング    | Enhanced Monitoring + CloudWatch Logs |
| バックアップ保持  | 7日間、自動スナップショットあり                      |

#### Terraformサンプル（RDS Oracle）

```hcl
resource "aws_db_instance" "oracle" {
  engine            = "oracle-se2"
  engine_version    = "19.0.0.0.ru-2023-10.rur-2023-10.r1"
  instance_class    = "db.m6g.large"
  allocated_storage = 100
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn
  name              = "appdb"
  username          = "admin"
  password          = var.db_password
  multi_az          = true
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name
  backup_retention_period = 7
  monitoring_interval     = 60
}
```

---

## 🧪 テストとCI/CD

* **GitHub Actions**でPRベースのCIを構成：

  * `terraform fmt`, `validate`, `plan`, `apply`
  * S3バックエンド + DynamoDBロック
* **Terraform Workspace**で`dev`, `staging`, `prod`を完全分離

---

## 🏁 成果とメリット

| 項目     | 内容                               |
| ------ | -------------------------------- |
| 再現性    | dev→stg→prodがコードで完全同期            |
| 保守性    | モジュール化により影響範囲の明確化                |
| 可視性    | GitHub上でコードレビュー → チーム全体の品質担保     |
| セキュリティ | CloudTrailで全アクション監査、IAM最小権限原則を徹底 |

---

## 今後の展望
* インフラ詳細設計書⇒IaC自動生成
* Terraform CloudによるState管理の中央集約
* Sentinelによるポリシー制御（Policy as Code）
* OPA (Open Policy Agent) との連携
* AWS Control Tower + IaCの統合

---

## 👥 体制・役割分担

| メンバー      | 役割                                |
| --------- | --------------------------------- |
| メンバーA | Terraform全体実装、CI/CD構築、レビュー主導      |
| メンバーB     | AWSアーキテクト、Terraformレビュー・検証、RDS管理者 |

---

