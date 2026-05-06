# 04_api_spec.yaml 実装差分レビュー

## レビュー概要

`04_api_spec.yaml` とバックエンド実装の Controller / request DTO / response DTO / Security 設定を照合した。
主な差分は、認証必須 API の OpenAPI 表現、CSRF / 認可失敗時の 403 応答、位置情報レスポンスの null 表現、入力制約の不足である。

## 指摘一覧

| No | 重要度 | 指摘内容 | 対象 | 実装との差分 | 修正方針 | ステータス |
|---:|---|---|---|---|---|---|
| 1 | High | 認証必須の read API に `security` / `401` が記載されていない | `GET /api/posts`, `/api/posts/search`, `/api/posts/search/map`, `/api/posts/{postId}`, `/api/posts/{postId}/replies`, `/api/replies/{parentReplyId}` | 実装は `POST /api/login` と `POST /api/users` 以外の `/api/**` を認証必須にしている | 該当 API に `security: SessionCookie` と `401` 応答を追加する | 対応済み |
| 2 | Medium | CSRF / 認可失敗時の `403` 応答が API 仕様に不足している | 認証必須 API、特に `POST /api/posts`, `POST /api/replies/{postId}/reply`, `POST /api/logout` | 実装は AccessDeniedHandler で `403` を返す。CSRF 拒否も `403` になる | 認証必須 API に `403 Forbidden` を追加する。特に状態変更 API は CSRF 拒否の可能性を補足する | 対応済み |
| 3 | Medium | `PostItemResponse.location` が nullable と記載されているが、実装は location オブジェクトを返す | `PostItemResponse.location` | `PostResponseMapper` は位置情報がない場合も `LocationResponse.of(null, null)` を返す | 実装どおりに `location` は object、`lat` / `lng` が nullable とする | 対応済み |
| 4 | Low | `SignUpRequest.birthDate` の未来日不可制約が OpenAPI に表現されていない | `SignUpRequest.birthDate` | 実装は `@PastOrPresent` と `BirthDate.of(..., today)` で未来日を拒否する | `birthDate` の説明に「未来日不可」を追記する。OpenAPI の静的 schema では現在日依存制約のため説明で補足する | 対応済み |
| 5 | Low | 地図検索の `min <= max` 制約が OpenAPI に表現されていない | `GET /api/posts/search/map` | 実装は `BoundingBox` で `minLat <= maxLat`、`minLng <= maxLng` を検証する | 各パラメータ説明、または endpoint description に `minLat <= maxLat` / `minLng <= maxLng` を追記する | 対応済み |
| 6 | Low | 401 / 403 のエラー本文が `ProblemDetailError` として定義されていない | Auth / Security エラー | Controller の例外は ProblemDetail だが、Security handler は `sendError` で 401 / 403 を返す | 将来的に API エラー形式を統一する。現時点では対応保留とする | 対応保留 |

## 詳細

### 1. 認証必須の read API に `security` / `401` が記載されていない

#### 設計書

`04_api_spec.yaml` では、以下の read API に `security` と `401` が記載されていない。

- `GET /api/posts`
- `GET /api/posts/search`
- `GET /api/posts/search/map`
- `GET /api/posts/{postId}`
- `GET /api/posts/{postId}/replies`
- `GET /api/replies/{parentReplyId}`

参照:

- `04_api_spec.yaml:384-408`
- `04_api_spec.yaml:435-464`
- `04_api_spec.yaml:466-513`
- `04_api_spec.yaml:515-539`
- `04_api_spec.yaml:541-571`
- `04_api_spec.yaml:573-602`

#### 実装

Security 設定では、未認証許可している API は以下のみである。

- `POST /api/login`
- `POST /api/users`

それ以外の `/api/**` は認証必須である。

参照: `../footprint/src/main/java/jp/i432kg/footprint/config/SecurityConfig.java:119-124`

#### 影響

API 仕様を読むと、投稿一覧・検索・投稿詳細・返信一覧が未認証で呼べるように見える。
実際には未ログインでは 401 になるため、フロントエンドや外部利用者が認証要否を誤解する。

#### 修正方針

該当 read API に以下を追加する。

- `security: [{ SessionCookie: [] }]`
- `401: Unauthorized`

対応内容:

- `04_api_spec.yaml` の該当 read API に `security: [{ SessionCookie: [] }]` を追加した
- `04_api_spec.yaml` の該当 read API に `401: Unauthorized` を追加した

ステータス: 対応済み

### 2. CSRF / 認可失敗時の `403` 応答が API 仕様に不足している

#### 設計書

認証必須 API の多くは `401` を記載しているが、`403` は記載していない。
特に状態変更 API は CSRF 拒否の可能性がある。

参照:

- `04_api_spec.yaml:275-285`
- `04_api_spec.yaml:409-433`
- `04_api_spec.yaml:604-634`

#### 実装

Security の `ApiAccessDeniedHandler` は認可失敗または CSRF 拒否時に `403` を返す。

参照: `../footprint/src/main/java/jp/i432kg/footprint/infrastructure/security/ApiAccessDeniedHandler.java:23-43`

#### 影響

CSRF token 欠落・不正、または認可失敗時の応答が API 仕様から読み取れない。
フロントエンド側で 403 を想定しない実装になりやすい。

#### 修正方針

認証必須 API に `403 Forbidden` を追加する。
状態変更 API では「CSRF 拒否時も 403」と補足する。

対応内容:

- `04_api_spec.yaml` の認証必須 API に `403: Forbidden` を追加した
- `POST /api/logout`、`POST /api/posts`、`POST /api/replies/{postId}/reply` を含む状態変更 API も 403 応答を明記した

ステータス: 対応済み

### 3. `PostItemResponse.location` が nullable と記載されているが、実装は location オブジェクトを返す

#### 設計書

`PostItemResponse.location` は nullable と定義されている。

参照: `04_api_spec.yaml:216-218`

#### 実装

`PostResponseMapper` は、`LocationSummary` が null の場合も `LocationResponse.of(null, null)` を返す。
つまり `location` 自体は null ではなく、`lat` / `lng` が null の object になる。

参照: `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/response/mapper/PostResponseMapper.java:86-90`

#### 影響

API 仕様上は `location: null` を許容しているが、実装レスポンスは `{ "lat": null, "lng": null }` になる。
クライアント側の null 判定ロジックと実レスポンスがずれる可能性がある。

#### 修正方針

実装を正とし、`location` は常に object として返す。
位置情報がない場合は `LocationResponse` の `lat` / `lng` が null になる。

対応内容:

- `04_api_spec.yaml` の `PostItemResponse.location` から `nullable: true` を削除した
- `LocationResponse.lat` / `lng` の nullable は維持した

ステータス: 対応済み

### 4. `SignUpRequest.birthDate` の未来日不可制約が OpenAPI に表現されていない

#### 設計書

`birthDate` は `format: date` のみで、未来日不可の説明がない。

参照: `04_api_spec.yaml:135-137`

#### 実装

presentation 層では `@PastOrPresent` を指定し、domain 層でも `BirthDate.of(..., today)` で未来日を拒否している。

参照:

- `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/request/SignUpRequest.java:54-61`
- `../footprint/src/main/java/jp/i432kg/footprint/domain/value/BirthDate.java:38-45`

#### 影響

API 利用者は未来日を送れるように見えるが、実際には 400 になる。

#### 修正方針

`birthDate` の description に「未来日は不可。過去日または当日を指定する」と追記する。
現在日に依存する制約のため、OpenAPI schema の固定 `maximum` ではなく説明で表現する。

対応内容:

- `04_api_spec.yaml` の `SignUpRequest.birthDate` に「未来日は不可。過去日または当日を指定する。」と追記した

ステータス: 対応済み

### 5. 地図検索の `min <= max` 制約が OpenAPI に表現されていない

#### 設計書

`minLat` / `maxLat` / `minLng` / `maxLng` は個別の範囲制約のみ記載している。

参照: `04_api_spec.yaml:466-498`

#### 実装

`BoundingBox` は以下の相関制約を検証する。

- `minLat <= maxLat`
- `minLng <= maxLng`

参照: `../footprint/src/main/java/jp/i432kg/footprint/domain/model/BoundingBox.java:61-75`

#### 影響

各値が範囲内でも、`minLat > maxLat` や `minLng > maxLng` の場合は 400 になる。
API 仕様だけではこの入力制約が分からない。

#### 修正方針

`GET /api/posts/search/map` の description または各 parameter description に、相関制約を追記する。

対応内容:

- `04_api_spec.yaml` の `GET /api/posts/search/map` description に `minLat <= maxLat`、`minLng <= maxLng` を満たす必要がある旨を追記した

ステータス: 対応済み

### 6. 401 / 403 のエラー本文が `ProblemDetailError` として定義されていない

#### 設計書

多くの validation / resource not found は `application/problem+json` を定義しているが、401 / 403 の本文仕様は明確でない。

参照:

- `04_api_spec.yaml:269-285`
- `04_api_spec.yaml:314-382`
- `04_api_spec.yaml:409-433`
- `04_api_spec.yaml:604-634`

#### 実装

Controller / GlobalExceptionHandler 経由の例外は `ProblemDetail` を返す。
一方、Security handler は `response.sendError(...)` により 401 / 403 を返す。

参照:

- `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/GlobalExceptionHandler.java:333-342`
- `../footprint/src/main/java/jp/i432kg/footprint/infrastructure/security/ApiAuthenticationEntryPoint.java:22-37`
- `../footprint/src/main/java/jp/i432kg/footprint/infrastructure/security/ApiAuthenticationFailureHandler.java:22-38`
- `../footprint/src/main/java/jp/i432kg/footprint/infrastructure/security/ApiAccessDeniedHandler.java:23-43`

#### 影響

API 利用者が 401 / 403 も `ProblemDetailError` と同じ形式で処理できるのか判断しづらい。

#### 修正方針

将来的に API エラー形式を統一する。
現時点では Security handler が `sendError` で返す 401 / 403 を維持し、仕様・実装の変更は行わない。

ステータス: 対応保留
