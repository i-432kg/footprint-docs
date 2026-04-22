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
- ユーザー識別：内部ID + username（表示用）、email（ログインID）

## 3. 認証（Authentication）

### 3.1 ログイン方式
- Spring Security 標準フォームログインを使用する
    - GET /login：ログイン画面表示（Thymeleaf）
    - POST /login：認証（成功時セッション発行）
    - POST /logout：ログアウト（セッション無効化）

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
- 「閲覧は公開、操作はログイン必須」を基本方針とする
- 認可は必ずサーバー側で強制し、フロントの表示制御のみに依存しない

### 4.2 エンドポイント別ポリシー（MVP）

#### 公開（未ログイン可）
- 画面（Page Controller）
    - GET /login
    - GET /timeline
    - GET /map
- API（JSON）
    - GET /api/posts（タイムライン）
    - GET /api/posts/{postId}（投稿詳細）
    - GET /api/posts/{postId}/replies（返信一覧）
    - GET /api/map/posts（bbox検索）

#### 要ログイン
- 画面（Page Controller）
    - GET /mypage
- API（JSON）
    - GET /api/users/me
    - GET /api/me/posts
    - GET /api/me/replies
    - POST /api/posts（投稿作成）
    - POST /api/posts/{postId}/replies（返信作成）

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

/ api のエラー応答は ErrorResponse 形式で統一する。

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