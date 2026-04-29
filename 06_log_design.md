# Log Design

## 1. 目的
ログは以下を目的として出力する。

- 障害解析（原因特定・再現性向上）
- 不正利用/悪用の検知（スパム、DoS的挙動）
- 運用状況の把握（エラー率、遅延、画像アップロード失敗など）

## 2. 前提（本プロジェクトの構成）
- 画面遷移は MPA（Page Controller）
- 画面描画は Thymeleaf + Vue（mounted）で実施（Vue Routerなし）
- データ操作は /api/** の JSON API を標準 `fetch` で呼び出す
- 認証は Spring Security セッションCookie（JSESSIONID）
- タイムライン/一覧はシーク法（`lastId` / `size`）
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

#### 例外的に詳細出力を許容する
- local / stg の seed 実行ログ
  - 対象は fixed seed scenario の投入・cleanup に限る
  - 出力対象は seed 用の固定ダミーデータ（seed email, caption, message, image entry など）に限る
  - 実ユーザー由来の入力値や、本体リクエスト処理で扱う機微情報には適用しない

## 4. 相関ID（traceId）設計

### 4.1 サーバ側
- リクエスト単位で traceId を採番し、MDCに格納して全ログに付与する
- レスポンスヘッダに `X-Trace-Id` を付与する（フロントが障害時に参照可能）

### 4.2 クライアント側（Vue/fetch）
- レスポンスヘッダ `X-Trace-Id` を受け取り、フロントのエラーログに付与する（補助）
- traceIdを画面に常時表示しない（必要時にコピーできる程度は可）

## 5. ログ分類（カテゴリ）

### 5.1 サーバログ
- access：HTTPアクセス（共通）
- auth：認証/認可（ログイン成功/失敗、401/403、CSRFなど）
- app：業務処理（投稿作成、返信作成、EXIF解析、bbox検索など）
- audit：重要操作（投稿作成、返信作成）

### 5.1.1 サーバログの責務分担
ログは「どこでも出してよい」とは扱わず、カテゴリごとに出力責務を固定する。

- access：HTTP リクエスト全体の観測ログ。`Filter` / `Interceptor` でのみ出力する
- auth：認証/認可結果のログ。Spring Security の handler / entry point / denied handler でのみ出力する
- app：業務上の警告、想定内異常、例外整形、ユースケース進行上の補助ログ
- audit：重要操作の成功記録。use case の成功確定点でのみ出力する

レイヤごとの原則:

- presentation 層：`access` の補助情報設定、`app` のうち HTTP 入出力境界に近いログを扱う
- application 層：`audit` の成功イベント、およびユースケース内部でしか分からない `app` ログを扱う
- domain 層：原則としてログ出力しない
- infrastructure 層：`auth` と技術障害の補助的な `app` ログだけを扱う

補足:

- `audit` は controller ではなく application 層の use case 完了点に寄せる
- `GlobalExceptionHandler` のような HTTP 境界での例外整形ログは presentation 層の `app` として扱う
- 同一事象を複数レイヤで重複記録しない

read 系成功イベントの扱い:

- `POST_TIMELINE_FETCH`, `POST_DETAIL_FETCH`, `POST_MAP_BBOX_FETCH`, `REPLY_LIST_FETCH`, `ME_FETCH`, `ME_POSTS_FETCH`, `ME_REPLIES_FETCH` などの read 成功イベントは `access` の責務として扱う
- read 成功イベントは controller が event 名と補助項目を決め、最終的なログ出力は `Filter` / `Interceptor` 側で 1 リクエスト 1 本へ集約する
- `items`, `size`, `lastIdPresent`, `postId`, `parentReplyId`, bbox 値など、HTTP リクエスト結果の観測に近い項目は `access` に載せる
- read 処理中の業務警告、想定内異常、補助的な診断ログは `app` として扱う
- 例: bbox の業務制約違反、seek ページング境界の異常兆候、read 中の想定内例外整形は `app` に出す
- read 成功の主記録を `app` に重複出力しない
- failure / warning 系 event 解決は path ではなく request 文脈の `operation` を基準に行う
- `operation` は全 endpoint に付与する運用を前提とし、`Filter` / `Interceptor` / `GlobalExceptionHandler` から共通参照できるよう request 文脈へ保持する
- ただし静的リソースや annotation 付与漏れなどの特殊経路でログ基盤自体が壊れないよう、実装上は `operation` 未設定を許容し、未設定時は汎用 event へフォールバックする

### 5.2 フロントログ（補助）
- ui：UIイベント（モーダル開閉、bbox変更）
- api：HTTP API 送受信（成功/失敗、duration）

## 6. ログレベル基準
- INFO：正常系の主要イベント（投稿作成成功、返信作成成功、bbox検索実行など）
- WARN：想定内の異常（バリデーションエラー、認可エラー、`lastId` 不正、EXIF取得不可など）
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
- POST_LAST_ID_INVALID

### 7.3 返信
- REPLY_LIST_FETCH
- REPLY_CREATE_SUCCESS
- REPLY_CREATE_VALIDATION_FAIL
- REPLY_LAST_ID_INVALID

### 7.4 マイページ
- ME_FETCH
- ME_POSTS_FETCH
- ME_REPLIES_FETCH

### 7.5 validation / warning event 命名ルール
- 入力値やリクエスト形式の不正は `*_VALIDATION_FAIL` を使う
- multipart の必須不足、サイズ超過、形式不正などアップロード拒否は `*_UPLOAD_REJECTED` を使う
- シーク法の cursor / `lastId` 不正は `*_LAST_ID_INVALID` を使う
- 個別 operation へ解決できない validation fallback は `REQUEST_VALIDATION_FAIL` を使う
- read 系 success event は operation 名と同じ event 名を使い、validation / warning event だけ結果つき接尾辞で区別する

## 8. 画面・ユースケース別 ログ要件（サーバ中心）

### SCR-01 ログイン（登録含む）
- auth.INFO：AUTH_LOGIN_SUCCESS（userId, username）
- auth.WARN：AUTH_LOGIN_FAILURE（理由は抽象化：INVALID_CREDENTIALS等）
- app.INFO：登録成功（userId, username）
- app.WARN：登録バリデーション失敗（フィールド名/コード）
- app.WARN：username/email重複（UNIQUE制約）

### SCR-02 タイムライン（無限スクロール）
- access.INFO：/api/posts（size, lastId有無, 件数, durationMs）
- app.WARN：POST_LAST_ID_INVALID（lastIdが不正）
- audit.INFO：POST_CREATE_SUCCESS（postId, userId, imageSizeBytes, hasLocation）
- app.WARN：POST_CREATE_UPLOAD_REJECTED（サイズ超過、形式不正）
- app.WARN：EXIF位置情報なし（exifAvailable=true/false, hasLocation=false）
- app.ERROR：画像保存失敗、DB保存失敗（traceId必須）
- app.WARN：同一 `created_at` 境界の取りこぼし懸念がある場合はページング異常として記録する

#### 投稿詳細モーダル（SCR-02内）
- access.INFO：/api/posts/{id}
- access.INFO：/api/posts/{id}/replies（件数）
- audit.INFO：REPLY_CREATE_SUCCESS（replyId, postId, parentReplyId, userId）
- app.WARN：REPLY_CREATE_VALIDATION_FAIL
- auth.WARN：AUTH_UNAUTHORIZED（未ログインで返信POST）

### SCR-03 地図表示（bbox）
- access.INFO：/api/posts/search/map（bbox, 件数, durationMs）
- app.WARN：bboxパラメータ不正（範囲逆転、過大範囲など）
- フロントui.INFO：bbox変更は高頻度のためサンプリング/デバウンス前提

### SCR-04 マイページ
- access.INFO：/api/users/me
- access.INFO：/api/users/me/posts（lastId, size, 件数, durationMs）
- access.INFO：/api/users/me/replies（lastId, size, 件数, durationMs）
- auth.WARN：未ログインアクセス → 401

## 9. 監視に使うログ指標（MVP）
以下はログ集計（あるいはAPM/メトリクス）で観測できるようにする。

- /api の 4xx/5xx 比率
- 画像アップロード失敗率（UPLOAD_REJECTED / 例外）
- EXIF位置情報取得率（hasLocation率）
- bbox検索の件数、処理時間
- タイムライン取得（`lastId` ページング）の処理時間
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
  "lastIdPresent": true,
  "size": 20,
  "items": 20
}
