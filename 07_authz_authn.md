# Authentication / Authorization Design

## 1. 目的
- 認証（Authentication）：ログイン状態を安全に維持し、本人として操作できる状態を提供する
- 認可（Authorization）：操作可能な対象を制限し、不正操作を防止する
- 匿名性を維持しつつ、最小限の識別情報でサービスを成立させる

## 2. 前提
- Spring Security を利用する
- 認証方式：サーバーセッション + Cookie（JSESSIONID）
- 画面遷移：Page Controller（MPA）
- VueはThymeleaf内にマウントされ、データ操作は /api/** を呼び出す
- ユーザー識別：`public_id` / ULID + username（表示用）、email（ログインID）

## 3. 認証（Authentication）

### 3.1 ログイン方式
- Spring Security 標準フォームログインを使用する
    - GET /login：ログイン画面表示（Thymeleaf）
    - POST /api/login：認証（成功時セッション発行）
    - POST /api/logout：ログアウト（セッション無効化）
- ログインパラメータ名は `loginId` を使用する
- サインアップは `POST /api/users` で受け付け、`birthDate` を必須とする
- サインアップ成功後は自動ログインする

### 3.2 セッション管理
- 認証成功時にサーバー側でセッションを作成し、Cookie（JSESSIONID）で保持する
- 未認証ユーザーは保護リソースにアクセスできない

### 3.3 Cookie属性（推奨）
- Secure（HTTPS時のみ送信）
- HttpOnly（JSから参照不可）
- SameSite（CSRF緩和）
  ※具体値はデプロイ方式確定後に調整するが、Prodでは必須とする

### 3.4 CSRF対策
- セッションCookie運用のため CSRF 対策を有効化する
- /api/** の更新系（POST/PUT/DELETE）では、フロント（Axios）がCSRFトークンを送付する
- 読み取り系（GET）はCSRF不要

> 注：実装方式（CookieにCSRFトークン、metaタグに埋め込み等）はUI実装に合わせて決定する

## 4. 認可（Authorization）

### 4.1 基本方針
- 画面・API・画像配信を原則認証必須とする
- 認可は必ずサーバー側で強制し、フロントの表示制御のみに依存しない

### 4.2 エンドポイント別ポリシー（MVP）

#### 公開（未ログイン可）
- 画面（Page Controller）
    - GET /login
- 静的リソース
    - /css/**
    - /assets/**
    - /favicon.ico
    - /favicon.svg
    - /actuator/health
- 開発環境のみ
    - /swagger-ui/**
    - /v3/api-docs/**
    - /v3/api-docs.yaml
- API（JSON）
    - POST /api/login
    - POST /api/users

#### 要ログイン
- 画面（Page Controller）
    - GET /
    - GET /timeline
    - GET /map
    - GET /search
    - GET /mypage
- API（JSON）
    - GET /api/users/me
    - GET /api/users/me/posts
    - GET /api/users/me/replies
    - GET /api/posts
    - GET /api/posts/search
    - GET /api/posts/search/map
    - GET /api/posts/{postId}
    - GET /api/posts/{postId}/replies
    - GET /api/replies/{parentReplyId}
    - POST /api/posts（投稿作成）
    - POST /api/replies/{postId}/reply（返信作成）
    - POST /api/logout

#### 画像配信
- local の `/images/**` は原則認証必須とする
- stg / prod は S3 presigned URL を暫定利用し、CloudFront 導入まで短命 URL で運用する

### 4.3 所有者制御（MVP/将来）
MVPでは「作成」のみだが、将来の編集・削除で必要となる。

- 投稿の編集/削除：投稿者（posts.user_id）のみ
- 返信の編集/削除：返信者（replies.user_id）のみ

実装は以下のいずれか（方針は実装段階で決定）：
- サービス層で userId を突合し、合致しなければ 403
- Spring Security のメソッドセキュリティ（@PreAuthorize 等）で表現

## 5. エラー応答方針
- 未認証：401 Unauthorized
- 権限不足：403 Forbidden
- 存在しないリソース：404 Not Found
- バリデーションエラー：400 Bad Request

`/api` のエラー応答は `ProblemDetail` を基本形式として統一する。

- 標準項目:
  - `type`
  - `title`
  - `status`
  - `detail`
  - `instance`（必要時のみ）
- 拡張項目:
  - `errorCode`: 機械判定用の安定識別子
  - `details`: バリデーションエラーや補足情報の一覧

クライアントは原則として `detail` の文字列解析に依存せず、`errorCode` と `details` を参照する。

補足:

- `type` は将来的に安定 URI で運用する
- `Content-Type` は `application/problem+json` を基本とする
- 詳細方針は `docs/adr/adr_024_problem_detail_error_response_policy.md` に従う

## 6. ログ方針（関連）
- ログイン成功/失敗は auth ログとして出力する
- 401/403 は auth.WARN として記録する
- 重要操作（投稿作成、返信作成）は audit.INFO として記録する
- セッションID、パスワード、CSRFトークンはログに出力しない

## 7. 今後の拡張
- メール認証（Out of scope）
- パスワード再設定（Out of scope）
- アカウント削除（Out of scope）：論理削除（status/deleted_at）を想定
- レート制限（スパム/DoS対策）：運用設計で導入を検討
