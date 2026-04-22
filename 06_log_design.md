# Log Design

## 1. 目的
ログは以下を目的として出力する。

- 障害解析（原因特定・再現性向上）
- 不正利用/悪用の検知（スパム、DoS的挙動）
- 運用状況の把握（エラー率、遅延、画像アップロード失敗など）

## 2. 前提（本プロジェクトの構成）
- 画面遷移は MPA（Page Controller）
- 画面描画は Thymeleaf + Vue（mounted）で実施（Vue Routerなし）
- データ操作は /api/** のJSON APIをAxiosで呼び出す
- 認証は Spring Security セッションCookie（JSESSIONID）
- タイムライン/一覧はシーク法（opaque cursor）
- 地図は bbox 検索（minLat/maxLat/minLng/maxLng）

## 3. 基本方針

### 3.1 サーバログを正とする
ブラウザ側（Vue）のログは補助とし、監査・障害解析の根拠はサーバログを正とする。

### 3.2 構造化ログ（JSON）を基本とする
最低限、以下の共通キーを揃える。

- timestamp
- level
- logger
- message
- traceId（リクエスト相関ID）
- method / path
- status（HTTPステータス）
- durationMs
- userId（ログイン時のみ）
- username（ログイン時のみ）
- event（イベント名）
- errorCode（アプリ定義コード）
- client（必要に応じて：ua、ipなど）

### 3.3 個人情報・機密情報の取り扱い

#### 絶対に出力しない
- パスワード
- セッションID（JSESSIONID）
- CSRFトークン
- 画像のバイナリ（multipart中身）

#### 原則マスキングする
- email（例：a***@example.com）
- IPアドレス（必要なら部分マスク）

## 4. 相関ID（traceId）設計

### 4.1 サーバ側
- リクエスト単位で traceId を採番し、MDCに格納して全ログに付与する
- レスポンスヘッダに `X-Trace-Id` を付与する（フロントが障害時に参照可能）

### 4.2 クライアント側（Vue/Axios）
- レスポンスヘッダ `X-Trace-Id` を受け取り、フロントのエラーログに付与する（補助）
- traceIdを画面に常時表示しない（必要時にコピーできる程度は可）

## 5. ログ分類（カテゴリ）

### 5.1 サーバログ
- access：HTTPアクセス（共通）
- auth：認証/認可（ログイン成功/失敗、401/403、CSRFなど）
- app：業務処理（投稿作成、返信作成、EXIF解析、bbox検索など）
- audit：重要操作（投稿作成、返信作成）

### 5.2 フロントログ（補助）
- ui：UIイベント（モーダル開閉、bbox変更）
- api：Axios送受信（成功/失敗、duration）

## 6. ログレベル基準
- INFO：正常系の主要イベント（投稿作成成功、返信作成成功、bbox検索実行など）
- WARN：想定内の異常（バリデーションエラー、認可エラー、cursor不正、EXIF取得不可など）
- ERROR：想定外例外、処理継続困難、保存失敗、整合性問題
- DEBUG：開発環境のみ（詳細デバッグ）

## 7. イベント定義（最低限）
イベント名（event）は下記のいずれかを使用する（拡張可）。

### 7.1 認証/認可
- AUTH_LOGIN_SUCCESS
- AUTH_LOGIN_FAILURE
- AUTH_UNAUTHORIZED   （401）
- AUTH_FORBIDDEN      （403）
- AUTH_CSRF_REJECTED  （CSRF関連）

### 7.2 投稿
- POST_TIMELINE_FETCH
- POST_CREATE_SUCCESS
- POST_CREATE_VALIDATION_FAIL
- POST_CREATE_UPLOAD_REJECTED
- POST_DETAIL_FETCH
- POST_MAP_BBOX_FETCH
- POST_CURSOR_INVALID

### 7.3 返信
- REPLY_LIST_FETCH
- REPLY_CREATE_SUCCESS
- REPLY_CREATE_VALIDATION_FAIL
- REPLY_CURSOR_INVALID

### 7.4 マイページ
- ME_FETCH
- ME_POSTS_FETCH
- ME_REPLIES_FETCH

## 8. 画面・ユースケース別 ログ要件（サーバ中心）

### SCR-01 ログイン（登録含む）
- auth.INFO：AUTH_LOGIN_SUCCESS（userId, username）
- auth.WARN：AUTH_LOGIN_FAILURE（理由は抽象化：INVALID_CREDENTIALS等）
- app.INFO：登録成功（userId, username）
- app.WARN：登録バリデーション失敗（フィールド名/コード）
- app.WARN：username/email重複（UNIQUE制約）

### SCR-02 タイムライン（無限スクロール）
- access.INFO：/api/posts（limit, cursor有無, 件数, durationMs）
- app.WARN：POST_CURSOR_INVALID（cursorが不正/期限切れ）
- audit.INFO：POST_CREATE_SUCCESS（postId, userId, imageSizeBytes, hasLocation）
- app.WARN：POST_CREATE_UPLOAD_REJECTED（サイズ超過、形式不正）
- app.WARN：EXIF位置情報なし（exifAvailable=true/false, hasLocation=false）
- app.ERROR：画像保存失敗、DB保存失敗（traceId必須）

#### 投稿詳細モーダル（SCR-02内）
- access.INFO：/api/posts/{id}
- access.INFO：/api/posts/{id}/replies（limit, cursor, 件数）
- audit.INFO：REPLY_CREATE_SUCCESS（replyId, postId, parentReplyId, userId）
- app.WARN：REPLY_CREATE_VALIDATION_FAIL
- auth.WARN：AUTH_UNAUTHORIZED（未ログインで返信POST）

### SCR-03 地図表示（bbox）
- access.INFO：/api/map/posts（bbox, limit, 件数, durationMs）
- app.WARN：bboxパラメータ不正（範囲逆転、過大範囲など）
- フロントui.INFO：bbox変更は高頻度のためサンプリング/デバウンス前提

### SCR-04 マイページ
- access.INFO：/api/users/me
- access.INFO：/api/me/posts（cursor, 件数, durationMs）
- access.INFO：/api/me/replies（cursor, 件数, durationMs）
- auth.WARN：未ログインアクセス → 401

## 9. 監視に使うログ指標（MVP）
以下はログ集計（あるいはAPM/メトリクス）で観測できるようにする。

- /api の 4xx/5xx 比率
- 画像アップロード失敗率（UPLOAD_REJECTED / 例外）
- EXIF位置情報取得率（hasLocation率）
- bbox検索の件数、処理時間
- タイムライン取得（cursorページング）の処理時間
- 返信作成失敗率（validation / 401 / 例外）

## 10. JSONログ例

### 10.1 アクセスログ例（タイムライン取得）
```json
{
  "timestamp": "2026-02-22T12:34:56.789+09:00",
  "level": "INFO",
  "logger": "access",
  "event": "POST_TIMELINE_FETCH",
  "traceId": "8c3e0f5a7c4b4c1a",
  "method": "GET",
  "path": "/api/posts",
  "status": 200,
  "durationMs": 128,
  "userId": 12,
  "username": "anon_fox",
  "cursorPresent": true,
  "limit": 20,
  "items": 20
}