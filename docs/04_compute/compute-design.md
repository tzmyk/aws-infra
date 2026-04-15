# コンピューティング設計

> **対象システム:** shop-app（商品一覧・カート機能）
> **使用サービス:** EC2、Auto Scaling、ALB、RDS MySQL

---

## 構成概要

```
ALB
 └── Auto Scaling Group
      ├── EC2 App Server (1a)
      └── EC2 App Server (1c)

RDS MySQL
 ├── プライマリ (1a)
 └── スタンバイ (1c)  ← Multi-AZ

管理EC2 (1a, パブリックサブネット)
 └── Terraform 実行環境
```

---

## EC2 App Server

| 項目 | 値 |
|---|---|
| AMI | Amazon Linux 2023 (最新) |
| インスタンスタイプ | t3.small |
| ストレージ | gp3 20GB |
| サブネット | app-1a / app-1c（プライベート） |
| パブリック IP | **割り当てない** |
| IAM ロール | shop-app-role |

### Auto Scaling Group

| 項目 | 値 |
|---|---|
| 最小台数 | 1 |
| 最大台数 | 4 |
| 希望台数（初期） | 2 |
| スケールアウト条件 | CPU 使用率 70% が 2分間継続 |
| スケールイン条件 | CPU 使用率 30% が 5分間継続 |
| ヘルスチェック | ALB ヘルスチェック（/health） |

> **最小台数を 1 にしている理由（研修環境）:** コスト削減のため。
> 本番環境では最小 2 台にして AZ 障害に備えること。

### ユーザーデータ（起動スクリプト）

EC2 起動時に自動でアプリをセットアップする。

```bash
#!/bin/bash
yum update -y
yum install -y python3 python3-pip

# SSM からシークレットを取得して環境変数に設定
DB_PASSWORD=$(aws ssm get-parameter \
  --name /shop/db/password --with-decryption \
  --query Parameter.Value --output text)
DB_ENDPOINT=$(aws ssm get-parameter \
  --name /shop/db/endpoint \
  --query Parameter.Value --output text)

export DB_PASSWORD DB_ENDPOINT

# アプリ起動
cd /opt/shop-app
pip3 install -r requirements.txt
python3 app.py
```

---

## ALB（Application Load Balancer）

| 項目 | 値 |
|---|---|
| タイプ | Application Load Balancer |
| スキーム | internet-facing |
| サブネット | public-1a / public-1c |
| リスナー | 443/HTTPS → ターゲットグループ（8080） |
| SSL 証明書 | ACM（AWS Certificate Manager） |
| ヘルスチェックパス | /health |
| ヘルスチェック間隔 | 30秒 |
| 正常判定しきい値 | 連続2回成功 |
| 異常判定しきい値 | 連続3回失敗 |

---

## RDS MySQL

| 項目 | 値 |
|---|---|
| エンジン | MySQL 8.0 |
| インスタンスクラス | db.t3.small |
| ストレージ | gp2 20GB（Auto Scaling 有効、最大 100GB） |
| マルチAZ | 有効（スタンバイを 1c に配置） |
| サブネットグループ | db-1a / db-1c |
| パブリックアクセス | **無効** |
| 自動バックアップ | 有効（保持期間 7日） |
| 削除保護 | 有効 |
| 暗号化 | 有効 |

> **Multi-AZ の仕組み:** プライマリが障害を起こすと、自動でスタンバイに切り替わる（フェイルオーバー）。
> 切り替えには 1〜2 分かかる。スタンバイはリードレプリカではなくホットスタンバイ。

---

## 管理EC2（Terraform 実行環境）

| 項目 | 値 |
|---|---|
| AMI | Amazon Linux 2023 (最新) |
| インスタンスタイプ | t3.micro |
| サブネット | public-1a |
| パブリック IP | **割り当てる**（SSH アクセスのため） |
| IAM ロール | shop-mgt-role |

### 管理EC2 にインストールするツール

```bash
# Terraform
yum install -y yum-utils
yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
yum install -y terraform

# Git
yum install -y git

# AWS CLI（Amazon Linux 2023 はデフォルトで入っている）
```

### 研修生の作業手順

```
1. 研修生 PC から管理EC2 に SSH
   $ ssh -i <キーペア.pem> ec2-user@<管理EC2のパブリックIP>

2. リポジトリを clone
   $ git clone https://github.com/tzmyk/aws-infra.git
   $ cd aws-infra/terraform/environments/dev

3. Terraform 実行
   $ terraform init
   $ terraform plan
   $ terraform apply
```

---

## 設計上の禁止事項

- EC2 App Server にパブリック IP を割り当てない
- RDS をパブリックアクセス可能な設定にしない
- RDS の削除保護を無効にしない
- 管理EC2 のキーペアを複数人で共有しない（研修生ごとに別キーペアを作成）
- ユーザーデータにパスワードをハードコードしない（SSM から取得する）
