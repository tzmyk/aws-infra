# IAM 設計

> **原則:** 最小権限（Least Privilege）。必要なアクション・リソースのみを許可する。

---

## ロール一覧

| ロール名 | アタッチ先 | 目的 |
|---|---|---|
| shop-app-role | EC2 App Server | アプリが AWS サービスを呼ぶための権限 |
| shop-mgt-role | 管理EC2 | Terraform を実行するための権限 |

---

## ロール詳細

### shop-app-role（EC2 App Server 用）

アプリケーションが実行中に必要とする権限のみを付与する。

```
許可するアクション:

  SSM Parameter Store（シークレット取得）:
    - ssm:GetParameter
    - ssm:GetParameters
    リソース: arn:aws:ssm:ap-northeast-1:*:parameter/shop/*

  S3（静的ファイル・ログ）:
    - s3:GetObject
    リソース: arn:aws:s3:::shop-static-*/assets/*

  CloudWatch Logs（アプリログ送信）:
    - logs:CreateLogGroup
    - logs:CreateLogStream
    - logs:PutLogEvents
    リソース: arn:aws:logs:ap-northeast-1:*:log-group:/shop/app:*
```

### shop-mgt-role（管理EC2 用）

Terraform が AWS リソースを作成・変更・削除するための権限。  
研修用途のため広めの権限を付与するが、本番運用では絞ること。

```
許可するアクション:
  EC2: *（インスタンス・SG・VPC・サブネット・ALB の操作）
  RDS: *（クラスター・インスタンス・サブネットグループの操作）
  IAM: PassRole（shop-app-role のみ）
  SSM: *（パラメータの作成・取得・削除）
  S3: *（Terraform state バケットの操作）
  DynamoDB: *（Terraform state ロックテーブルの操作）
  CloudWatch: *（ロググループ・アラームの操作）
  AutoScaling: *
  ElasticLoadBalancing: *

  ※ 本番環境では操作対象リソースを ARN で絞ること
```

> **IAM PassRole について:** Terraform が EC2 を起動する際に `shop-app-role` を  
> インスタンスプロファイルとして渡す必要がある。`iam:PassRole` はその許可。  
> `"Resource": "*"` にすると任意のロールを渡せてしまうため、対象ロールの ARN を指定する。

---

## インスタンスプロファイルとロールの関係

EC2 に IAM ロールを付与するには、ロールを**インスタンスプロファイル**でラップする必要がある。

```
IAM ロール（shop-app-role）
    ↓ ラップ
インスタンスプロファイル（shop-app-profile）
    ↓ アタッチ
EC2 インスタンス
```

Terraform では `aws_iam_instance_profile` リソースで作成する。

---

## よくある間違い（研修用）

| 間違い | 正しい設計 |
|---|---|
| EC2 に AdministratorAccess を付ける | 必要なアクションのみのカスタムポリシー |
| DB パスワードを環境変数に直書き | SSM Parameter Store から取得 |
| IAM ロールなしで EC2 を起動し、アクセスキーをコードに書く | インスタンスプロファイルを使う |
| `iam:PassRole` に `"Resource": "*"` を指定 | 渡すロールの ARN を指定 |

---

## 設計上の禁止事項

- EC2 に AdministratorAccess ポリシーをアタッチしない
- アクセスキー・シークレットキーをコードやファイルに書かない（インスタンスプロファイルを使う）
- `iam:PassRole` のリソースに `*` を使わない
