# インフラテスト方針

## テストの目的

**設計書に書かれた期待値と、実際にデプロイされた AWS リソースが一致していることを確認する。**

設計書がいくら正確でも、実装がその通りになっていなければ意味がない。
このテストは「設計書 = 仕様書」として機能させるための手段。

---

## テストの考え方

```
設計書（期待値）
    ↓
テストコード（期待値を boto3 で検証するコードに変換）
    ↓
実際の AWS リソース（テスト対象）
    ↓
差分レポート（設計と乖離している箇所を報告）
```

設計書の中で **具体的な値が明記されている箇所** がすべてテスト対象になる。
「適切に設定する」のような曖昧な記述はテストできない。これが設計書に具体的な値を書く理由でもある。

---

## テストツール

| ツール | 用途 |
|---|---|
| pytest | テストフレームワーク |
| boto3 | AWS リソースへの問い合わせ |

```bash
# セットアップ（管理EC2 上で実行）
cd ~/aws-infra
python3 -m venv .venv
.venv/bin/pip install pytest boto3

# 実行
.venv/bin/pytest tests/infra/ -v
```

管理EC2 のインスタンスプロファイル（shop-mgt-role）で認証されるため、
アクセスキーの設定は不要。

---

## テストカテゴリと対応する設計書

| カテゴリ | テストファイル | 参照する設計書 |
|---|---|---|
| ネットワーク | `infra/test_network.py` | `docs/01_network/vpc-design.md` |
| セキュリティ | `infra/test_security.py` | `docs/02_security/security-design.md` |
| IAM | `infra/test_iam.py` | `docs/03_iam/roles.md` |
| コンピューティング | `infra/test_compute.py` | `docs/04_compute/compute-design.md` |

---

## テストケース一覧

### test_network.py

設計書 `docs/01_network/vpc-design.md` の内容を検証する。

| テストケース | 検証内容 | 設計書の根拠 |
|---|---|---|
| `test_vpc_cidr` | VPC の CIDR が `10.0.0.0/16` であること | サブネット一覧 |
| `test_subnets_exist` | 6つのサブネットが正しい CIDR・AZ で存在すること | サブネット一覧 |
| `test_db_subnet_no_internet_route` | DB サブネットのルートテーブルに IGW・NAT の経路がないこと | 「インターネット経路: なし」 |
| `test_app_subnet_uses_nat` | アプリサブネットに NAT Gateway 経由の経路があること | 「NAT Gateway 経由」 |
| `test_sg_db_no_outbound` | `shop-db-sg` のアウトバウンドルールが空であること | SG 一覧「なし」 |
| `test_sg_alb_inbound_443_only` | `shop-alb-sg` のインバウンドが 443/tcp のみであること | SG 一覧 |
| `test_sg_app_inbound_from_alb_only` | `shop-app-sg` のインバウンドが `shop-alb-sg` からのみであること | SG 一覧 |
| `test_mgt_sg_no_public_ssh` | `shop-mgt-sg` の SSH が `0.0.0.0/0` を許可していないこと | 禁止事項 |

### test_security.py

設計書 `docs/02_security/security-design.md` の内容を検証する。

| テストケース | 検証内容 | 設計書の根拠 |
|---|---|---|
| `test_s3_block_public_access` | S3 バケットの BlockPublicAccess が4項目すべて有効であること | 禁止事項 |
| `test_rds_not_publicly_accessible` | RDS インスタンスの `PubliclyAccessible` が False であること | 禁止事項 |
| `test_rds_encryption_enabled` | RDS インスタンスの暗号化が有効であること | 暗号化方針 |
| `test_ebs_encryption_enabled` | EC2 の EBS ボリュームの暗号化が有効であること | 暗号化方針 |
| `test_cloudtrail_enabled` | CloudTrail が有効であること | 証跡・ログ設計 |
| `test_ssm_parameters_exist` | `/shop/db/password` など必要なパラメータが存在すること | シークレット管理 |

### test_iam.py

設計書 `docs/03_iam/roles.md` の内容を検証する。

| テストケース | 検証内容 | 設計書の根拠 |
|---|---|---|
| `test_roles_exist` | `shop-app-role` と `shop-mgt-role` が存在すること | ロール一覧 |
| `test_app_role_no_admin_policy` | `shop-app-role` に AdministratorAccess がアタッチされていないこと | 禁止事項 |
| `test_app_role_ssm_scoped` | `shop-app-role` の SSM 権限が `/shop/*` に限定されていること | ロール詳細 |
| `test_mgt_role_pass_role_scoped` | `shop-mgt-role` の `iam:PassRole` が `shop-app-role` のみを許可していること | 禁止事項 |
| `test_ec2_has_instance_profile` | EC2 App Server にインスタンスプロファイルがアタッチされていること | ロール詳細 |

### test_compute.py

設計書 `docs/04_compute/compute-design.md` の内容を検証する。

| テストケース | 検証内容 | 設計書の根拠 |
|---|---|---|
| `test_app_ec2_no_public_ip` | EC2 App Server にパブリック IP が割り当てられていないこと | 禁止事項 |
| `test_asg_min_max` | Auto Scaling Group の min=1, max=4 であること | Auto Scaling Group |
| `test_alb_health_check_path` | ALB のヘルスチェックパスが `/health` であること | ALB |
| `test_rds_multi_az` | RDS の Multi-AZ が有効であること | RDS MySQL |
| `test_rds_backup_enabled` | RDS の自動バックアップが有効（保持7日）であること | RDS MySQL |
| `test_rds_deletion_protection` | RDS の削除保護が有効であること | 禁止事項 |

---

## テストを書く際のガイドライン

### 1. テスト名と設計書を対応させる

```python
def test_db_subnet_no_internet_route():
    """
    [設計書参照] docs/01_network/vpc-design.md > サブネット一覧
    DB サブネットのインターネット経路: なし（完全プライベート）
    """
```

テストが失敗したとき、どの設計書のどの記述に違反しているかをすぐに追えるようにする。

### 2. テストは読み取り専用にする

`describe_*` / `get_*` / `list_*` のみ使うこと。
リソースを変更・削除する操作は絶対に書かない。

### 3. リソース名は定数化する

```python
# conftest.py に定義する
REGION = "ap-northeast-1"
VPC_NAME = "shop-vpc"
APP_SG_NAME = "shop-app-sg"
DB_INSTANCE_ID = "shop-db"
```

### 4. 1テスト = 1つの設計上の制約

複数の検証を1テストに詰め込まない。失敗時にどの制約が破られているか明確にするため。

---

## よくある詰まりポイント（ヒント）

| 詰まりポイント | ヒント |
|---|---|
| SG のアウトバウンドが「空」のはずなのに通らない | AWS はデフォルトで全許可のアウトバウンドを作る。Terraform で明示的に削除しているか確認 |
| SG 参照かどうかの判定 | `IpRanges` が空で `UserIdGroupPairs` に値があれば SG 参照 |
| EC2 のパブリック IP 確認 | `describe_instances` の `PublicIpAddress` が存在しないかを確認 |
| RDS の Multi-AZ 確認 | `describe_db_instances` の `MultiAZ` フィールドを確認 |
| IAM ポリシーのリソース制限確認 | インラインポリシーと管理ポリシーの両方をチェックすること |
