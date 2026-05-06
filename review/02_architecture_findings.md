# `02_architecture.md` 設計実装差分レビュー

## 概要

- 対象設計書: `02_architecture.md`
- 対象実装:
  - バックエンド: `../footprint`
  - フロントエンド: `../footprint-front`
- 確認日: 2026-05-02
- 目的: `02_architecture.md` のアーキテクチャ構成と実装を突き合わせ、設計と実装が一致していない箇所を洗い出す

## 結論

`02_architecture.md` は、Spring MVC / Thymeleaf の MPA、各画面に mount する Vue UI、Leaflet、Spring Security セッション認証、MySQL、画像ストレージ、フロント別リポジトリ取り込みという大枠では実装と整合している。

修正した方がよい差分は、投稿作成シーケンスの処理順、ログインシーケンスの責務表現、API Controller 例の抜けである。その他の差分は詳細設計・実装説明レベルであり、基本設計としては対応不要でよい。

## 指摘一覧

| No | 重要度 | 指摘内容 | 対象箇所 | 修正方針 | 対応状況 |
| --- | --- | --- | --- | --- | --- |
| 1 | Medium | 投稿作成シーケンスの画像保存と EXIF 抽出の順序が実装と逆 | シーケンス図 Create post | `save image -> extract metadata/EXIF -> save DB` の順に修正する | 対応済み |
| 2 | Medium | ログインシーケンスが Page Controller 経由に見えるが、実装は Spring Security filter が処理する | シーケンス図 Login | `/api/login` は Spring Security の loginProcessingUrl として表現し、成功後は 200 + Set-Cookie、画面遷移はフロント側 redirect として表現する | 対応済み |
| 3 | Low | API Controller 例に投稿詳細取得 API がない | Controller例 API Controller | `GET /api/posts/{postId}` を追加する | 対応済み |
| 4 | Low | 画像配信の local / S3 差分が図では単一の Image Storage に丸められている | アーキテクチャ図 / 図の補足 | 基本設計では現行表現で十分。必要なら補足に local は `/images/**`、S3 は presigned URL と追記する | 対応不要 |
| 5 | Low | ログ出力箇所が Application Services 起点に見えるが、実装は filter / interceptor / security / storage などにもまたがる | アーキテクチャ図 / 図の補足 | 基本設計ではログ分類の方針が示されていれば十分。詳細な出力点はログ設計・実装で管理する | 対応不要 |

## 指摘事項

### 1. Medium: 投稿作成シーケンスの画像保存と EXIF 抽出の順序が実装と逆

#### 設計

`02_architecture.md` の Create post シーケンスでは、Application Services が EXIF 抽出を行った後に画像を保存し、最後に DB 保存する流れになっている。

- `02_architecture.md:124-137`
- 特に以下の順序:
  - `SVC->>EX: extract GPS (optional)`
  - `SVC->>IMG: save image`
  - `SVC->>DB: save post + location`

#### 実装

実装では、まず画像を物理ストレージへ保存し、その保存済み実ファイルまたは S3 オブジェクトからメタデータを抽出している。

- `../footprint/src/main/java/jp/i432kg/footprint/application/command/service/PostCommandService.java:68-85`
  - `imageStorage.store(...)`
  - `imageMetadataExtractor.extract(storageObject)`
- local 保存実装は一時ファイル保存、形式判定、最終保存先への移動を行う
  - `../footprint/src/main/java/jp/i432kg/footprint/infrastructure/storage/repository/LocalImageRepositoryImpl.java:132-174`
- S3 保存実装はバイト列を S3 に put した後、保存済み S3 オブジェクトをメタデータ抽出対象にする
  - `../footprint/src/main/java/jp/i432kg/footprint/infrastructure/storage/repository/S3ImageRepositoryImpl.java:167-203`

#### 差分

設計図では `EXIF 抽出 -> 画像保存` に見えるが、実装は `画像保存 -> EXIF/メタデータ抽出` である。

#### 影響

- 障害時の補償処理の理解がずれる
- メタデータ抽出が保存済み `StorageObject` に依存している実装意図が図から読み取れない
- S3 利用時に「アップロード前に EXIF を読んでいる」と誤解される余地がある

#### 修正方針

Create post シーケンスを以下の順序に更新する。

1. `AC->>SVC: create post`
2. `SVC->>IMG: save image`
3. `IMG-->>SVC: storage object`
4. `SVC->>EX: extract metadata/GPS from stored image`
5. `EX-->>SVC: metadata/location`
6. `SVC->>DB: save post + image metadata + location`

`02_architecture.md` は上記順序で更新済み。

### 2. Medium: ログインシーケンスが Page Controller 経由に見えるが、実装は Spring Security filter が処理する

#### 設計

Login シーケンスでは、`POST /api/login` が Page Controller に送られ、Page Controller から Spring Security へ認証を委譲するように見える。

- `02_architecture.md:116-122`
- `U->>PC: POST /api/login (form)`
- `PC->>SEC: authenticate`
- `PC-->>U: 200/302 + Set-Cookie`

#### 実装

`POST /api/login` は Controller メソッドではなく、Spring Security の `loginProcessingUrl` として処理される。

- `../footprint/src/main/java/jp/i432kg/footprint/config/SecurityConfig.java:130-137`
  - `loginPage("/login")`
  - `loginProcessingUrl("/api/login")`
  - `successHandler(authenticationSuccessHandler)`
  - `failureHandler(authenticationFailureHandler)`
- 認証成功時は `ApiAuthenticationSuccessHandler` が 200 OK を返す
  - `../footprint/src/main/java/jp/i432kg/footprint/infrastructure/security/ApiAuthenticationSuccessHandler.java:28-47`
- フロントエンドはログイン API 成功後に `window.location.href = '/timeline'` で遷移する
  - `../footprint-front/src/components/login/LoginForm.vue:62-68`
- API client は `/api` prefix を付与するため、フロントの `userService.login()` は `/api/login` を呼び出す
  - `../footprint-front/src/services/apiClient.js:3`
  - `../footprint-front/src/services/userService.js:34-47`

#### 差分

設計図では Page Controller がログイン処理に関与するように見えるが、実装では Spring Security filter chain が直接 `POST /api/login` を処理する。また、成功時の画面遷移はサーバー 302 ではなく、フロント側の明示的な redirect である。

#### 影響

- Controller 責務を誤解しやすい
- ログイン成功時のレスポンス契約を 302 と誤認する可能性がある
- API ログインと画面遷移の境界が曖昧になる

#### 修正方針

Login シーケンスを以下の表現に更新する。

1. `U->>VUE: submit login form`
2. `VUE->>SEC: POST /api/login (form) + CSRF`
3. `SEC->>SEC: authenticate`
4. `SEC-->>VUE: 200 OK + Set-Cookie`
5. `VUE->>U: redirect to /timeline`

Page Controller は `GET /login` の HTML 返却のみを担当する表現に分ける。

`02_architecture.md` は上記表現で更新済み。

### 3. Low: API Controller 例に投稿詳細取得 API がない

#### 設計

API Controller 例には投稿一覧、検索、地図検索、返信一覧、マイページ系、作成系 API が列挙されているが、投稿詳細取得 API がない。

- `02_architecture.md:76-89`

#### 実装

投稿詳細取得 API は `PostRestController` に実装されている。

- `GET /api/posts/{postId}`
- `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/PostRestController.java:176-194`

#### 差分

Controller 例の API 一覧から、実装済みの投稿詳細取得 API が漏れている。

#### 影響

- 投稿詳細モーダルや詳細取得の API 経路がアーキテクチャ資料から読み取れない
- `05_screen_spec.md` やフロント実装の投稿詳細導線と対応づけにくい

#### 修正方針

API Controller 例に `GET /api/posts/{postId}` を追加する。

`02_architecture.md` に追加済み。

### 4. Low: 画像配信の local / S3 差分が図では単一の Image Storage に丸められている

#### 設計

アーキテクチャ図では画像保存先が `Image Storage` として抽象化されている。

- `02_architecture.md:32-33`
- `02_architecture.md:151`

#### 実装

画像保存・配信は環境ごとに異なる。

- local は `app.storage.type=LOCAL` でローカルファイルシステムを利用する
  - `../footprint/src/main/resources/application-local.yml:23-31`
- local の `/images/**` は Spring MVC の ResourceHandler でローカルディレクトリへマッピングする
  - `../footprint/src/main/java/jp/i432kg/footprint/config/WebMvcConfig.java:51-68`
- stg/prod は既定で S3 を利用し、API レスポンスには presigned URL を返す
  - `../footprint/src/main/resources/application-stg.yml:28-43`
  - `../footprint/src/main/resources/application-prod.yml:24-38`
  - `../footprint/src/main/java/jp/i432kg/footprint/presentation/helper/S3ImageUrlResolver.java:34-58`

#### 差分

図では保存先・配信経路の環境差分までは表現していない。

#### 判断

基本設計のアーキテクチャ図としては `Image Storage` に抽象化して問題ない。local/S3/presigned URL の詳細は `01_overview.md`, `08_deployment.md`, 認証・認可設計、実装設定で扱えば十分である。

#### 修正方針

対応不要。必要であれば図の補足に `local は /images/**、stg/prod は S3 presigned URL` と一文追記する程度でよい。

### 5. Low: ログ出力箇所が Application Services 起点に見えるが、実装は複数層にまたがる

#### 設計

アーキテクチャ図では `SVC --> LOG` として、Application Services からログへ向かう表現になっている。

- `02_architecture.md:61-64`
- `02_architecture.md:153`

#### 実装

ログは Application Services だけでなく、filter、interceptor、security handler、storage 実装など複数箇所から出力される。

- trace/access filter は SecurityContextHolderFilter 前後に登録される
  - `../footprint/src/main/java/jp/i432kg/footprint/config/SecurityConfig.java:163-164`
- controller の `@LogOperation` は interceptor で request 文脈へ設定される
  - `../footprint/src/main/java/jp/i432kg/footprint/config/WebMvcConfig.java:41-44`
- 認証成功/失敗 handler も auth ログを出力する
  - `../footprint/src/main/java/jp/i432kg/footprint/infrastructure/security/ApiAuthenticationSuccessHandler.java:39-43`
  - `../footprint/src/main/java/jp/i432kg/footprint/infrastructure/security/ApiAuthenticationFailureHandler.java:33-39`

#### 差分

図だけを見るとログ出力が Application Services 層に閉じているように見えるが、実装では横断的関心事として複数層に配置されている。

#### 判断

基本設計の図としてはログ基盤への依存を簡略化していると見なせるため、修正必須ではない。詳細なログ出力点は `06_log_design.md` と実装で管理する粒度である。

#### 修正方針

対応不要。より正確にするなら `LOG` への矢印を Application Services だけでなく、Security / Controller / Filter からも出す、または補足に「ログは filter / interceptor / service / storage 等から出力される」と追記する。

## 実装と一致している主な記載

- Page Controller が `/login`, `/`, `/timeline`, `/map`, `/search`, `/mypage` を返す
  - `../footprint/src/main/java/jp/i432kg/footprint/presentation/web/RootController.java:38-91`
- Vue Router を使わず、Vite multi-entry の Vue UI を各画面に mount する
  - `../footprint-front/vite.config.js:37-46`
- local は Vite dev server、stg/prod は build 済み frontend asset を Spring Boot static 配下に取り込む
  - `../footprint/src/main/resources/application-local.yml:42-58`
  - `../footprint/Dockerfile:1-25`
- API は Spring MVC の REST Controller として実装されている
  - `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/PostRestController.java`
  - `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/ReplyRestController.java`
  - `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/UserRestController.java`
- 認証必須範囲は設計どおり、ログイン画面・サインアップ導線・静的アセット・ヘルスチェック等を除き原則認証必須である
  - `../footprint/src/main/java/jp/i432kg/footprint/config/SecurityConfig.java:98-127`
- フロントエンドは別リポジトリとして checkout され、Docker build 時に backend へ取り込まれる
  - `../footprint/.github/workflows/deploy-stg.yml:42-48`
  - `../footprint/Dockerfile:1-25`
