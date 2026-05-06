# `01_overview.md` 設計実装差分レビュー

## 概要

- 対象設計書: `01_overview.md`
- 対象実装:
  - バックエンド: `../footprint`
  - フロントエンド: `../footprint-front`
- 確認日: 2026-05-02
- 目的: `01_overview.md` の現行記載と実装を突き合わせ、概要書更新時に修正すべき差分を洗い出す

## 結論

`01_overview.md` は大枠のサービス像とは整合しているが、技術スタック、環境構成、DB、主要エンティティ、画面構成、認証 Cookie、画像アップロード制限、参照先リンクに実装との差分がある。

特に Java バージョン、local DB、永続化方式、Vue Router の記載、`Like` エンティティ、H2 Console、`Dev` 環境表記は実装と明確に異なるため、概要書更新時に優先して修正する。

## 指摘一覧

| No | 重要度 | 指摘内容 | 対象箇所 | 修正方針 | 対応状況 |
| --- | --- | --- | --- | --- | --- |
| 1 | High | Backend の Java バージョンが実装と異なる | 技術スタック Backend | Java 21 に更新する | 対応済み |
| 2 | High | Spring Data JPA が記載されているが実装にはない | 技術スタック Backend | 永続化は MyBatis、migration は Flyway として記載する | 対応済み |
| 3 | High | Local DB が H2 と記載されているが実装は MySQL | 技術スタック Backend / Local 環境 | H2 Console を削除し、Local DB を Docker Compose の MySQL 8.4 に更新する | 対応済み |
| 4 | High | 環境名が `Dev` と記載されているが実装は `stg` | 環境 | `Local / Stg / Prod` に更新する | 対応済み |
| 5 | Medium | Frontend の Lint に Oxlint が記載されているが実装にはない | 技術スタック Frontend | Lint を ESLint に更新する | 対応済み |
| 6 | Medium | Vue Router 利用の記載があるが実装にはない | 構成方針 | Spring MVC / Thymeleaf + Vue 3 multi-entry + Pinia として記載する | 対応済み |
| 7 | Medium | 主要エンティティに `Like` が含まれているが実装にはない | データの特徴 | `Like` を削除し、位置情報は Post の緯度・経度として保持する旨を記載する | 対応済み |
| 8 | Medium | 認証 Cookie の記載が cookie 種別と環境差分を表現していない | セキュリティ | 基本設計の粒度では現行記載で十分なため、詳細設計・実装設定で管理する | 対応不要 |
| 9 | Medium | 画像アップロード許可形式がフロントとサーバーで一致していない | アップロード制限 | 基本設計では許可方針が示されていれば十分なため、詳細設計・実装で管理する | 対応不要 |
| 10 | Low | サインアップ後に自動ログインする実装が概要にない | 提供機能 | 基本設計ではユーザー登録とログインの提供機能が示されていれば十分なため、画面/API 詳細で管理する | 対応不要 |
| 11 | Low | 「匿名型」の説明が実装上のアカウント情報保持とややずれている | 目的 / 想定ユーザー | 投稿表示上の匿名性として説明を補正する | 対応済み |
| 12 | Low | 投稿検索の具体仕様が概要にない | 提供機能 / ユースケース | 基本設計では投稿検索の提供機能が示されていれば十分なため、API/画面仕様で管理する | 対応不要 |
| 13 | Low | 地図検索 API は bbox 指定である点が概要にない | ユースケース | 基本設計では地図表示・探索の方針が示されていれば十分なため、API/画面仕様で管理する | 対応不要 |
| 14 | Low | 返信 API と返信ツリーの実装詳細が概要に反映されていない | ユースケース | 基本設計ではスレッド形式の返信方針が示されていれば十分なため、API/画面仕様で管理する | 対応不要 |
| 15 | Low | 仕様参照先のファイル名が実ファイルと異なる | 仕様・設計の参照先 | 実ファイル名へ修正し、ADR はバックエンドリポジトリの相対パスとして明記する | 対応済み |

## 指摘事項

### 1. High: Backend の Java バージョンが実装と異なる

#### レビュー時点の記載

- Backend の言語: Java 17
  - `01_overview.md:267`

#### 実装

- Gradle toolchain は Java 21
  - `../footprint/build.gradle.kts:11-14`

#### 差分

概要書は Java 17 としているが、実装は Java 21 でビルドする前提になっている。

#### 修正方針

Backend の言語を `Java 21` に更新する。

### 2. High: 永続化方式に Spring Data JPA が含まれているが実装にはない

#### レビュー時点の記載

- 永続化: Spring Data JPA / MyBatis 併用
  - `01_overview.md:273`

#### 実装

- 依存関係は MyBatis Starter、Flyway、MySQL connector
- JPA starter や Spring Data JPA 依存は存在しない
  - `../footprint/build.gradle.kts:55-68`

#### 差分

概要書は JPA と MyBatis の併用としているが、実装は MyBatis を中心にした永続化である。

#### 修正方針

永続化は `MyBatis`、DB migration は `Flyway` として記載する。

### 3. High: Local DB が H2 と記載されているが実装は MySQL

#### レビュー時点の記載

- 開発/検証: H2 Console
  - `01_overview.md:275`
- Local DB: H2 DB
  - `01_overview.md:296`

#### 実装

- `compose.yaml` で MySQL 8.4 を起動する
  - `../footprint/compose.yaml:2-11`
- local profile は `.env` の `SPRING_DATASOURCE_URL` を参照する
  - `../footprint/src/main/resources/application-local.yml:4-7`
- `.env.example` の local URL は `jdbc:mysql://localhost:3306/footprint_local`
  - `../footprint/.env.example:7-10`

#### 差分

概要書は local DB を H2 としているが、実装は local も MySQL を利用する構成である。H2 Console 利用の実装も確認できない。

#### 修正方針

Local DB を `Docker Compose で起動する MySQL 8.4` に更新し、H2 Console の記載を削除する。

### 4. High: 環境名が `Dev` と記載されているが実装は `stg`

#### レビュー時点の記載

- 環境: Local / Dev / Prod
  - `01_overview.md:288`
- `9.2 Dev（検証環境）`
  - `01_overview.md:304`

#### 実装

- 設定ファイルは `application-local.yml`, `application-stg.yml`, `application-prod.yml`
  - `../footprint/src/main/resources/application-local.yml`
  - `../footprint/src/main/resources/application-stg.yml`
  - `../footprint/src/main/resources/application-prod.yml`
- stg は MySQL、S3、structured logging、OpenAPI 有効の構成
  - `../footprint/src/main/resources/application-stg.yml:1-60`

#### 差分

概要書は Dev 環境としているが、実装上の検証環境 profile は `stg` である。

#### 修正方針

環境表記を `Local / Stg / Prod` に更新する。

### 5. Medium: Frontend の Lint に Oxlint が記載されているが実装にはない

#### レビュー時点の記載

- Lint: ESLint / Oxlint
  - `01_overview.md:264`

#### 実装

- npm scripts は `eslint . --cache` と `eslint . --fix --cache`
- `oxlint` 依存や script は存在しない
  - `../footprint-front/package.json:14-15`
  - `../footprint-front/package.json:25-36`

#### 差分

概要書には Oxlint が含まれているが、実装は ESLint のみである。

#### 修正方針

Lint を `ESLint` に更新する。

### 6. Medium: Vue Router 利用の記載があるが実装にはない

#### レビュー時点の記載

- SPA 領域では Vue Router / Pinia を用いて状態と画面遷移を管理する
  - `01_overview.md:286`

#### 実装

- `vue-router` 依存は存在しない
  - `../footprint-front/package.json:17-24`
- Vite の multi entry として login / map / mypage / search / timeline を定義している
  - `../footprint-front/vite.config.js:37-46`
- 画面遷移は Spring MVC のテンプレート返却と通常リンクで行う
  - `../footprint/src/main/java/jp/i432kg/footprint/presentation/web/RootController.java:38-91`

#### 差分

概要書は Vue Router を使う前提だが、実装は各画面ごとの Vue entry を Thymeleaf テンプレートへ載せる MPA + Vue 構成である。

#### 修正方針

構成方針を `Spring MVC/Thymeleaf による画面単位の MPA と、各画面に mounted する Vue 3 multi-entry 構成` に更新する。状態管理は Pinia のみ記載する。

### 7. Medium: 主要エンティティに `Like` が含まれているが実装にはない

#### レビュー時点の記載

- 主要エンティティ: User / Post / Reply / Like / Image / Location など
  - `01_overview.md:122`

#### 実装

- DB テーブルは `users`, `posts`, `post_images`, `replies`
  - `../footprint/src/main/resources/db/migration/V1__init.sql:10-107`
- Out of scope にも「いいね機能」が含まれている
  - `01_overview.md:50`

#### 差分

概要書のデータ特徴に `Like` が含まれているが、初期リリース対象外かつ DB 実装にも存在しない。

#### 修正方針

主要エンティティから `Like` を削除し、`User / Post / Reply / Image / Location` を中心に整理する。DB 実装上は `Location` は独立テーブルではなく `posts` の緯度経度として保持される点も補足する。

### 8. Medium: 認証 Cookie の記載が実装の cookie 種別と環境差分を表現していない

#### レビュー時点の記載

- Cookie 属性: Secure / HttpOnly / SameSite を設定する
  - `01_overview.md:148`

#### 実装

- session cookie は local で `secure: false`, `http-only: true`, `same-site: lax`
  - `../footprint/src/main/resources/application-local.yml:15-21`
- session cookie は stg/prod で `secure: true`, `http-only: true`, `same-site: lax`
  - `../footprint/src/main/resources/application-stg.yml:20-26`
  - `../footprint/src/main/resources/application-prod.yml:16-22`
- CSRF cookie `XSRF-TOKEN` は SPA/JS から読むため `CookieCsrfTokenRepository.withHttpOnlyFalse()` を利用する
  - `../footprint/src/main/java/jp/i432kg/footprint/config/SecurityConfig.java:75-83`
- CSRF cookie は SameSite=Lax、secure は local/dev 判定で切り替える
  - `../footprint/src/main/java/jp/i432kg/footprint/config/SecurityConfig.java:169-172`

#### 差分

概要書は Cookie 属性を一括で記載しているが、実装は session cookie と CSRF cookie で HttpOnly の扱いが異なり、Secure は local と stg/prod で異なる。

#### 修正方針

基本設計の粒度では、Cookie 属性として Secure / HttpOnly / SameSite を設定する方針が示されていれば十分と判断する。session cookie と CSRF cookie の違い、local と stg/prod の Secure 差分は詳細設計・実装設定で管理するため、`01_overview.md` の修正は不要とする。

### 9. Medium: 画像アップロード許可形式がフロントとサーバーで一致していない

#### レビュー時点の記載

- 形式: JPEG / PNG / GIF / WEBP を許可する
  - `01_overview.md:175`

#### 実装

- サーバー側の許可拡張子は `jpg`, `jpeg`, `png`, `gif`, `webp`
  - `../footprint/src/main/java/jp/i432kg/footprint/domain/value/FileExtension.java:38-43`
- サーバー側は `FileTypeDetector` で JPEG / PNG / GIF / WebP を判定する
  - `../footprint/src/main/java/jp/i432kg/footprint/infrastructure/storage/repository/LocalImageRepositoryImpl.java:150-159`
  - `../footprint/src/main/java/jp/i432kg/footprint/infrastructure/storage/repository/S3ImageRepositoryImpl.java:178-184`
- フロント側のファイル形式チェックは `jpeg|jpg|png|webp` のみで、GIF を許可していない
  - `../footprint-front/src/utils/validationRules.js:37-42`

#### 差分

概要書とサーバー実装は GIF 許可で一致しているが、フロント実装は GIF を拒否する。利用者観点では実装全体として許可形式が一致していない。

#### 修正方針

基本設計の粒度では、画像アップロードで許可する形式の方針が示されていれば十分と判断する。フロントとサーバーの許可形式差分は詳細設計・実装で管理するため、`01_overview.md` の修正は不要とする。

### 10. Low: サインアップ後に自動ログインする実装が概要にない

#### レビュー時点の記載

- ユーザー登録
- ログイン
  - `01_overview.md:35-36`

#### 実装

- `POST /api/users` でユーザー作成後、`request.login(...)` によりログイン状態へ移行する
  - `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/UserRestController.java:172-194`

#### 差分

概要書は登録とログインを独立機能として書いているが、実装では登録完了後に自動ログインする。

#### 修正方針

基本設計の粒度では、ユーザー登録とログインの提供機能が示されていれば十分と判断する。登録後の自動ログインは画面/API 詳細で管理するため、`01_overview.md` の修正は不要とする。

### 11. Low: 「匿名型」の説明が実装上のアカウント情報保持とややずれている

#### レビュー時点の記載

- 画像と位置情報を組み合わせた匿名型投稿サービス
  - `01_overview.md:5-9`
- ニックネームやプロフィールによって識別される SNS に抵抗を感じる人物
  - `01_overview.md:64`

#### 実装

- サインアップではユーザー名、メールアドレス、パスワード、生年月日を保持する
  - `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/request/SignUpRequest.java:24-61`
- 投稿レスポンスには投稿者 username は含まれていない
  - `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/response/PostItemResponse.java:15-46`

#### 差分

実装はアカウント情報として username/email/birthDate を保持する。一方、投稿一覧や投稿詳細のレスポンスには投稿者情報を返していないため、匿名性は「投稿表示上の匿名性」として説明する方が正確である。

#### 修正方針

匿名型の説明を `サービス上はログインユーザーを持つが、投稿表示では投稿者情報を前面に出さない` という表現に更新する。`01_overview.md` は、投稿表示上の匿名性を重視するサービスであることが伝わる表現へ更新済み。

### 12. Low: 投稿検索の具体仕様が概要にない

#### レビュー時点の記載

- 投稿検索
  - `01_overview.md:40`

#### 実装

- `GET /api/posts/search` は `keyword` を受け取り、`posts.caption LIKE CONCAT('%', keyword, '%')` で検索する
  - `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/PostRestController.java:103-129`
  - `../footprint/src/main/resources/jp/i432kg/footprint/infrastructure/datasource/mapper/query/PostQueryMapper.xml:151-203`

#### 差分

概要書では検索対象が明記されていない。実装上は投稿コメントの部分一致検索である。

#### 修正方針

基本設計の粒度では、投稿検索の提供機能が示されていれば十分と判断する。検索対象や検索方式は API 仕様・画面仕様で管理するため、`01_overview.md` の修正は不要とする。

### 13. Low: 地図検索 API は bbox 指定である点が概要にない

#### レビュー時点の記載

- 投稿の地図表示（位置情報連動）
  - `01_overview.md:41`
- 地図画面から位置情報に基づいて投稿を探索できる
  - `01_overview.md:102-105`

#### 実装

- `GET /api/posts/search/map` は `minLat`, `maxLat`, `minLng`, `maxLng` を受け取り bbox 内の投稿を返す
  - `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/PostRestController.java:141-167`
- SQL は `has_location = true` と緯度経度の `BETWEEN` で検索する
  - `../footprint/src/main/resources/jp/i432kg/footprint/infrastructure/datasource/mapper/query/PostQueryMapper.xml:205-226`

#### 差分

概要書では「位置情報連動」とだけ書かれているが、実装上は表示範囲の bbox による明示的な地図検索である。

#### 修正方針

基本設計の粒度では、位置情報に基づく地図表示・探索の方針が示されていれば十分と判断する。bbox 指定の API 仕様は API 仕様・画面仕様で管理するため、`01_overview.md` の修正は不要とする。

### 14. Low: 返信 API と返信ツリーの実装詳細が概要に反映されていない

#### レビュー時点の記載

- 投稿への返信
- 返信はスレッド形式で表示される
  - `01_overview.md:38`
  - `01_overview.md:95-98`

#### 実装

- 投稿配下のトップレベル返信は `GET /api/posts/{postId}/replies`
  - `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/PostRestController.java:203-221`
- ネスト返信は `GET /api/replies/{parentReplyId}`
  - `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/ReplyRestController.java:57-75`
- 返信作成は `POST /api/replies/{postId}/reply` で、`parentReplyId` があればネスト返信になる
  - `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/ReplyRestController.java:86-105`
- フロントは `parentReplyId` ごとに子返信を取得・キャッシュする
  - `../footprint-front/src/stores/replyStore.js:5-17`

#### 差分

概要書の「スレッド形式」は実装と方向性は一致しているが、実装はトップレベル返信と子返信を分けて遅延取得する構成である。

#### 修正方針

基本設計の粒度では、返信をスレッド形式で表示する方針が示されていれば十分と判断する。トップレベル返信とネスト返信の取得方式は API 仕様・画面仕様で管理するため、`01_overview.md` の修正は不要とする。

### 15. Low: 仕様参照先のファイル名が実ファイルと異なる

#### レビュー時点の記載

- 認証・認可: `07_security.md`
  - `01_overview.md:334`
- デプロイ: `08_deploy.md`
  - `01_overview.md:335`
- 設計判断ログ（ADR）: `/docs/adr/`
  - `01_overview.md:336`

#### 実態

- 認証・認可の実ファイルは `07_authz_authn.md`
- デプロイの実ファイルは `08_deployment.md`
- `footprint-docs` リポジトリ直下に `/docs/adr/` は存在しない
  - `07_authz_authn.md`
  - `08_deployment.md`

#### 差分

概要書末尾の参照先リンクが実ファイル名と一致していない。

#### 修正方針

参照先を `07_authz_authn.md`, `08_deployment.md` に修正する。ADR の参照先はバックエンドリポジトリの `../footprint/docs/adr/` として明記する。`01_overview.md` は相対パス表現で更新済み。

## 補足

### 実装と一致している主な記載

- MPA と Vue を組み合わせたハイブリッド構成という大枠
- 画像アップロードと EXIF/GPS 抽出
- Leaflet による地図表示
- Spring Security のセッション Cookie 認証
- CSRF 対策
- 画面/API/画像配信を原則認証必須にする方針
- local はローカルファイル、stg/prod は S3 に画像保存する方針
- stg/prod で S3 presigned URL を返す方針
- local/stg の seed / cleanup 運用

### 更新時の優先順位

1. 技術スタックを実装に合わせる
2. Local/Stg/Prod の環境構成を実装に合わせる
3. データの特徴から未実装の `Like` と独立 `Location` 風の表現を整理する
4. 参照先リンクを実ファイル名に修正する
