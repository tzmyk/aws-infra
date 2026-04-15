# セキュリティ設計

> **方針:** 最小権限・暗号化・証跡の3本柱

---

## 多層防御の考え方

```
[インターネット]
     ↓ HTTPS のみ（HTTP はリダイレクト）
[ALB]
     ↓ SG でポート制限（8080 のみ）
[EC2 App Server]
     ↓ SG でポート制限（3306 のみ）
[RDS MySQL]
```

各層で独立したセキュリティコントロールを持つ。1層が破られても次の層で止める。

---

## 暗号化方針

| データの種類 | 保存時 | 転送時 |
|---|---|---|
| RDS データ | AES-256（RDS 暗号化有効） | TLS（`require_secure_transport=ON`） |
| S3 オブジェクト（ログ等） | SSE-S3 | TLS（バケットポリシーで強制） |
| EC2 EBS ボリューム | AES-256（EBS 暗号化有効） | — |

---

## シークレット管理

DB パスワードなどのシークレットは **AWS Systems Manager Parameter Store（SecureString）** で管理する。  
コードや設定ファイルにパスワードを直接書かない。

```
取得フロー:
EC2 起動 → IAM ロール経由で SSM GetParameter → 環境変数に展開 → アプリ起動
```

| パラメータ名 | 種別 | 中身 |
|---|---|---|
| `/shop/db/password` | SecureString | RDS マスターパスワード |
| `/shop/db/endpoint` | String | RDS エンドポイント |
| `/shop/app/secret_key` | SecureString | アプリケーションのセッションキー |

---

## 証跡・ログ設計

「誰が・いつ・何をしたか」を追えるように最低限のログを取得する。

| ログ種別 | 取得場所 | 保持期間 |
|---|---|---|
| CloudTrail | S3 | 90日 |
| ALB アクセスログ | S3 | 30日 |
| EC2 アプリログ | CloudWatch Logs | 30日 |
| VPC Flow Logs | CloudWatch Logs | 30日 |

---

## 設計上の禁止事項

- S3 バケットのパブリックアクセスを有効にしない（Block Public Access 4項目すべて ON）
- RDS をパブリックアクセス可能な設定にしない
- EC2 App Server にパブリック IP を割り当てない
- SSM Parameter Store のパラメータをコードにハードコードしない
- CloudTrail を無効化しない
