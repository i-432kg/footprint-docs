# Deployment Design

## 1. 方針
- 初期は Railway を用いて素早く公開し、運用コストを抑える
- 画像は初期段階から Amazon S3 に保存し、環境移行時のデータ移行コストを最小化する
- 利用状況・コスト・運用要件に応じて AWS（ECS or EC2 + RDS + S3）へ移行する
- アプリは Docker イメージでデプロイし、環境差分を縮小する

---

## 2. 初期構成（Railway + S3）

### 2.1 構成要素
- Hosting：Railway（Spring Boot アプリ）
- DB：Railway Managed DB（MySQL）
- Object Storage：Amazon S3（画像保存）
- Domain/TLS：Railway の提供機能 or 独自ドメイン（HTTPS）

### 2.2 目的
- 公開までのリードタイムを短縮する
- 画像をS3に置くことで、将来AWSに移行しても画像ストレージを変更不要にする

### 2.3 設定（環境変数）
- DB
    - DB_HOST / DB_PORT / DB_NAME / DB_USER / DB_PASSWORD
- Session / Security
    - （必要に応じて）COOKIE_SECURE / SAME_SITE など
- S3
    - S3_BUCKET
    - S3_REGION
    - AWS_ACCESS_KEY_ID
    - AWS_SECRET_ACCESS_KEY
    - S3_BASE_URL（署名URL or 配信URL方針に応じて）
- App
    - SPRING_PROFILES_ACTIVE（railway等）

### 2.4 注意点
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
- 画像はアプリサーバのローカルに保存しない
- 保存先は S3 に統一し、Railway/AWSで共通とする

### 4.2 オブジェクトキー例
- posts/{postId}/{uuid}.jpg

### 4.3 画像配信方式
- MVP：S3の署名付きURL（期限付き）で配信（private bucket前提）
- 将来：CloudFront + OAC/OAI 等で配信（必要に応じて）

※ どちらでも、アプリ側の「画像URL返却仕様」を揃えておく

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
- Docker image をビルドし、環境ごとにデプロイする
- 秘密情報はリポジトリに含めず、環境変数/Secretで管理する

---

## 8. 移行タイミングの判断基準（例）
- Railwayの月額費用がAWS構成の月額に近づいた
- DB/バックアップ/監視を強化したくなった
- スケールアウトや可用性が必要になった
- 運用/構成を実務寄りに寄せたくなった（ポートフォリオ強化）