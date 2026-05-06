# 07_authz_authn.md 設計・実装差分レビュー

## レビュー対象

| 区分 | 対象 |
| --- | --- |
| 設計資料 | `07_authz_authn.md` |
| バックエンド実装 | `../footprint/src/main/java/jp/i432kg/footprint/config/SecurityConfig.java`, `../footprint/src/main/java/jp/i432kg/footprint/infrastructure/security/**` |
| バックエンド設定 | `../footprint/src/main/resources/application-*.yml` |
| フロントエンド実装 | `../footprint-front/src/services/apiClient.js`, `../footprint-front/src/services/userService.js` |

## 指摘一覧

| No. | 重要度 | 対象 | 指摘内容 | 実装との差分 | 推奨対応 | ステータス |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Medium | CSRF対策 | CSRFトークンの実装方式が未確定表現のままになっている。 | 実装は `CookieCsrfTokenRepository.withHttpOnlyFalse()` を使い、`XSRF-TOKEN` Cookie と `X-XSRF-TOKEN` header で送信している。 | 設計を実装に合わせ、Cookie名・header名・Cookie属性を明記する。 | 対応不要 |
| 2 | Medium | Cookie属性 | Cookie属性が推奨値に留まり、環境別の実装値が記載されていない。 | `JSESSIONID` は local: `Secure=false`, stg/prod: `Secure=true`, 共通で `HttpOnly=true`, `SameSite=Lax`。CSRF Cookie は `HttpOnly=false`, `SameSite=Lax`, local/dev以外で `Secure=true`。 | 環境別の Cookie 属性を設計へ追加する。 | 対応不要 |
| 3 | Medium | エラー応答方針 | `/api` のエラー応答は `ProblemDetail` 基本形式として統一すると記載されている。 | Controller 例外は `ProblemDetail` だが、Spring Security filter chain の 401/403 は `sendError` で返しており ProblemDetail ではない。 | 現状を正とするなら Security handler の 401/403 は例外として補足する。将来統一するなら改善タスクとして分ける。 | 対応不要 |
| 4 | Low | ADR参照 | ADR参照先が同一リポジトリ前提のパスになっている。 | ADR は基本設計資料リポジトリではなく、バックエンドリポジトリ配下にある。 | `../footprint/docs/adr/adr_024_problem_detail_error_response_policy.md` に更新し、別リポジトリの ADR であることを明記する。 | 対応済み |
| 5 | Low | 公開エンドポイント | 実装では `/signup` も permitAll になっているが、設計の公開画面に記載がない。 | `SecurityConfig` は `/signup` を許可している。一方、画面仕様では新規登録はログイン画面内モーダルであり、独立画面を持たない。 | `/signup` が不要なら実装から許可設定を削除する。互換目的で残すなら設計に「現時点では画面なし」と補足する。 | 対応不要 |
| 6 | Low | OpenAPI公開範囲 | 設計では OpenAPI を「開発環境のみ」の公開としている。 | 実装は `SecurityConfig` 上は local/dev のみ permitAll。stg は springdoc 自体は有効だが、Security 上は認証必須になる。 | 設計に「local/dev は未認証で参照可、stg は有効だが認証必須、prod は無効」を補足する。 | 対応済み |

## 詳細

### 1. CSRFトークン実装方式の未反映

設計では、CSRF トークンの実装方式を「CookieにCSRFトークン、metaタグに埋め込み等はUI実装に合わせて決定」としている。

実装では方式が確定している。

- サーバ: `CookieCsrfTokenRepository.withHttpOnlyFalse()`
- Cookie名: `XSRF-TOKEN`
- Header名: `X-XSRF-TOKEN`
- フロント: 安全メソッド以外で `XSRF-TOKEN` Cookie を読み取り、`X-XSRF-TOKEN` header に設定する

参照:

- 設計: `07_authz_authn.md` 38-43行目
- 実装: `../footprint/src/main/java/jp/i432kg/footprint/config/SecurityConfig.java` 74-84行目
- 実装: `../footprint/src/main/java/jp/i432kg/footprint/config/SecurityConfig.java` 169-173行目
- 実装: `../footprint-front/src/services/apiClient.js` 3-19行目
- 実装: `../footprint-front/src/services/apiClient.js` 207-212行目

修正方針:

- CSRF方式を `XSRF-TOKEN` Cookie + `X-XSRF-TOKEN` header として明記する。
- `XSRF-TOKEN` はフロント JS が読む必要があるため `HttpOnly=false` であることを補足する。

クローズ理由:

- 基本設計では「CSRF対策を有効化し、更新系リクエストでフロントがトークンを送付する」方針まで記載できていれば十分と判断する。
- Cookie名、Header名、`CookieCsrfTokenRepository` の採用有無は詳細設計・実装レベルで管理する。

### 2. Cookie属性の環境別実装値不足

設計では Cookie 属性を推奨値として記載しているが、具体値はデプロイ方式確定後に調整するとしている。

実装では `JSESSIONID` と `XSRF-TOKEN` の属性が環境別に決まっている。

`JSESSIONID`:

- local: `HttpOnly=true`, `Secure=false`, `SameSite=Lax`
- stg/prod: `HttpOnly=true`, `Secure=true`, `SameSite=Lax`

`XSRF-TOKEN`:

- `HttpOnly=false`
- `SameSite=Lax`
- local/dev: `Secure=false`
- stg/prod: `Secure=true`

参照:

- 設計: `07_authz_authn.md` 32-37行目
- 実装: `../footprint/src/main/resources/application-local.yml` 15-21行目
- 実装: `../footprint/src/main/resources/application-stg.yml` 20-26行目
- 実装: `../footprint/src/main/resources/application-prod.yml` 16-22行目
- 実装: `../footprint/src/main/java/jp/i432kg/footprint/config/SecurityConfig.java` 68-76行目
- 実装: `../footprint/src/main/java/jp/i432kg/footprint/config/SecurityConfig.java` 169-173行目

修正方針:

- 設計に Cookie ごとの環境別属性表を追加する。

クローズ理由:

- Cookie 属性の詳細は、バックエンドリポジトリの `../footprint/docs/ops/cookie_management.md` で管理する。
- 基本設計資料では推奨属性とセッション Cookie 運用方針までを扱い、環境別の詳細管理は別資料へ委ねる。

### 3. Security handler の 401/403 が ProblemDetail ではない

設計では `/api` のエラー応答を `ProblemDetail` 基本形式として統一するとしている。

実装では Controller / `GlobalExceptionHandler` が扱う例外は `ProblemDetail` 形式に変換される。一方、Spring Security filter chain が処理する認証・認可エラーは `sendError` を使うため、現時点では `ProblemDetail` 形式ではない。

対象:

- ログイン失敗: `401`
- 未認証 API アクセス: `401`
- 認可拒否 / CSRF 拒否: `403`

参照:

- 設計: `07_authz_authn.md` 108-132行目
- 実装: `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/GlobalExceptionHandler.java`
- 実装: `../footprint/src/main/java/jp/i432kg/footprint/infrastructure/security/ApiAuthenticationFailureHandler.java` 33-39行目
- 実装: `../footprint/src/main/java/jp/i432kg/footprint/infrastructure/security/ApiAuthenticationEntryPoint.java` 32-37行目
- 実装: `../footprint/src/main/java/jp/i432kg/footprint/infrastructure/security/ApiAccessDeniedHandler.java` 34-43行目

修正方針:

- 現時点の実装を正とする場合は、「Controller 例外は `ProblemDetail`、Spring Security filter chain の 401/403 は `sendError`」と補足する。
- 将来的に API エラー形式を完全統一する場合は、Security handler の応答生成を `ProblemDetail` へ寄せる改善タスクとして扱う。

クローズ理由:

- 基本設計では 401 / 403 のステータス方針まで記載できていれば十分と判断する。
- Security filter chain の 401 / 403 応答本文形式は API 詳細設計、または将来のエラー形式統一タスクで管理する。

### 4. ADR参照先のパス差分

設計では ADR 参照が `docs/adr/adr_024_problem_detail_error_response_policy.md` になっている。

実際の ADR はバックエンドリポジトリ配下にあるため、基本設計資料からは `../footprint/docs/adr/adr_024_problem_detail_error_response_policy.md` と参照する必要がある。

参照:

- 設計: `07_authz_authn.md` 128-132行目
- 実ファイル: `../footprint/docs/adr/adr_024_problem_detail_error_response_policy.md`

修正方針:

- 参照先を `../footprint/docs/adr/adr_024_problem_detail_error_response_policy.md` に更新する。
- ADR は別リポジトリの資料であることを補足する。

対応結果:

- `07_authz_authn.md` の ADR 参照先を `../footprint/docs/adr/adr_024_problem_detail_error_response_policy.md` に更新した。
- ADR は別リポジトリ（バックエンド）の資料であることを明記した。

### 5. `/signup` の permitAll 設定差分

設計では公開画面として `GET /login` のみを定義している。

実装では `/signup` も permitAll に含まれている。ただし、画面仕様上は新規登録はログイン画面内モーダルで行うため、独立した `/signup` 画面はない。

参照:

- 設計: `07_authz_authn.md` 53-69行目
- 実装: `../footprint/src/main/java/jp/i432kg/footprint/config/SecurityConfig.java` 98-108行目

修正方針:

- `/signup` が不要なら `SecurityConfig` から permitAll 設定を削除する。
- 互換または将来用に残す場合は、設計に「`/signup` は現時点で画面を持たないが許可設定がある」旨を補足する。

クローズ理由:

- `/signup` は現時点で独立画面・APIとして提供しておらず、基本設計上の画面/APIポリシーには影響しない。
- permitAll 設定の整理は実装の不要設定 cleanup として扱い、基本設計資料では対応不要とする。

### 6. OpenAPI公開範囲の補足不足

設計では `/swagger-ui/**`, `/v3/api-docs/**`, `/v3/api-docs.yaml` を開発環境のみ公開としている。

実装では `SecurityConfig` が local/dev profile の場合だけ未認証アクセスを許可する。一方、stg profile では springdoc 自体は有効だが、Security 上は未認証公開されない。

参照:

- 設計: `07_authz_authn.md` 62-65行目
- 実装: `../footprint/src/main/java/jp/i432kg/footprint/config/SecurityConfig.java` 110-117行目
- 実装: `../footprint/src/main/resources/application-stg.yml` 14-18行目
- 実装: `../footprint/src/main/resources/application-prod.yml` 10-14行目

修正方針:

- 設計に「local/dev は未認証で OpenAPI / Swagger UI を参照可、stg は有効だが認証必須、prod は無効」と補足する。

対応結果:

- `07_authz_authn.md` の OpenAPI / Swagger UI 公開範囲に、local/dev は未認証で参照可、stg は有効だが認証必須、prod は無効である旨を追記した。
