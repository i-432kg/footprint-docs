# Deployment Design

## 1. 方針
- local / stg / prod で構成差分を持ちつつ、Docker build により実行環境差分を縮小する
- stg デプロイは GitHub Actions から Railway を用いて行う
- frontend は別リポジトリで管理し、Docker build 時に成果物を backend へ取り込む
- 画像は stg / prod で Amazon S3 に保存し、将来の AWS 移行コストを抑える
- 利用状況・コスト・運用要件に応じて AWS（ECS or EC2 + RDS + S3）へ移行する

---

## 2. 現在の構成

### 2.1 local

- Backend: `./gradlew bootRun`
- DB: `docker compose` で起動する MySQL
- Object Storage: local filesystem (`storage/local`)
- Frontend: Vite dev server を参照

### 2.2 stg

- Hosting: Railway
- Deploy Trigger: GitHub Actions (`develop` push / repository_dispatch / manual)
- Build Input:
  - backend repository
  - frontend repository
- Frontend Integration:
  - workflow で frontend repository を checkout
  - Docker build 内で frontend を build
  - build 成果物を Spring Boot の `static` 配下へ取り込む
- Object Storage: S3

### 2.3 prod

- アプリ設定上は stg と同様に S3 前提
- 実運用構成は stg と同系統を想定する
- 詳細な本番デプロイ方式は今後確定する

### 2.4 現在の STG デプロイフロー

1. backend repository を checkout
2. repository variables / secrets を検証
3. frontend repository を指定 ref で checkout
4. Railway CLI をインストール
5. `railway up --service ...` を実行

関連実装:

- `.github/workflows/deploy-stg.yml`
- `Dockerfile`

### 2.5 設定値の責務

実装上は、設定値を次の 3 系統で扱う。

- local:
  - `.env`
  - `compose.yaml`
  - `application-local.yml`
- CI/CD:
  - GitHub Actions Variables / Secrets
- デプロイ先:
  - Railway のサービス変数

主な設定値例:

- DB 接続情報
- frontend repository / ref
- Railway service 名
- S3 接続情報
- Spring profile

### 2.6 注意点
- frontend repository 取得に失敗すると STG デプロイ全体が失敗する
- backend と frontend は単一成果物へ同梱されるため、ロールバックも原則セットで考える
- Flyway migration はアプリ起動時に走る前提であり、migration 失敗は起動失敗に直結する
- サーバセッション（JSESSIONID）は単一インスタンス運用では問題ない
- 将来スケールアウトする場合はセッション共有（Redis等）を検討する

---

## 3. 目標構成（AWS：ECS or EC2 + RDS + S3）

### 3.1 構成要素（ECS案）
- ALB（HTTPS終端、ルーティング）
- ECS/Fargate（Spring Boot コンテナ）
- RDS MySQL（DB）
- S3（画像）
- CloudWatch（ログ/メトリクス）
- Secrets Manager / SSM Parameter Store（機密情報）

### 3.2 構成要素（EC2案）
- EC2（Spring Boot コンテナ or jar）
- RDS MySQL（DB）
- S3（画像）
- CloudWatch（ログ/メトリクス）
- （必要に応じて）Nginx/Caddy（HTTPS終端）

### 3.3 ECSとEC2の選定基準（目安）
- ECS（おすすめ寄り）
    - デプロイ/ロールバックや運用の再現性を高めたい
    - コンテナ運用を前提にしたい
- EC2
    - まずはシンプルなVM運用でコスト/学習を抑えたい
    - 低負荷で固定構成のまま運用したい

---

## 4. 画像保存（S3）設計

### 4.1 基本方針
- local はローカルファイルシステムへ保存する
- stg / prod は S3 へ保存する
- 将来的なクラウド移行を見据え、stg / prod は S3 前提で統一する

### 4.2 オブジェクトキー例
- posts/{postId}/{uuid}.jpg

### 4.3 画像配信方式
- 現在: S3 の presigned URL（期限付き）で配信（private bucket 前提）
- 将来: CloudFront + private content で配信

補足:

- presigned URL は CloudFront 対応までの暫定策とする
- 詳細方針は `docs/adr/adr_021_auth_required_and_temporary_presigned_image_url.md` に従う

---

## 5. DB移行設計（Railway MySQL → RDS MySQL）

### 5.1 移行方針
- dump/restore により移行する（mysqldump）
- ダウンタイム許容のMVP移行とする（初期は短時間停止で対応）

### 5.2 手順（例）
1. Railway側をメンテナンスモード（書き込み停止）
2. Railway DB から dump を取得
3. RDS に restore
4. アプリの接続先を RDS に切り替え
5. 動作確認後、メンテナンス解除

---

## 6. セッション/認証の移行影響

- 認証はセッションCookie（JSESSIONID）であり、環境切替時は既存セッションは無効化される
- 移行時はユーザーに再ログインを求める前提とする（MVPでは許容）

将来スケールアウトが必要になった場合：
- Spring Session + Redis 等でセッション共有を検討する

---

## 7. CI/CD（推奨）
- STG は GitHub Actions から Railway へデプロイする
- Docker build の中で frontend を build し、backend へ取り込む
- 秘密情報はリポジトリに含めず、GitHub Secrets / Railway 変数等で管理する
- `develop` ブランチ push を基本トリガーとする

## 8. 監視とヘルスチェック

- `/actuator/health` をヘルスチェック用に公開する
- ログはデプロイ先のログ基盤で収集する
- 将来的な構造化ログ / 可観測性方針は `06_log_design.md` と整合を取る

---

## 9. 移行タイミングの判断基準（例）
- Railwayの月額費用がAWS構成の月額に近づいた
- DB/バックアップ/監視を強化したくなった
- スケールアウトや可用性が必要になった
- 運用/構成を実務寄りに寄せたくなった（ポートフォリオ強化）
