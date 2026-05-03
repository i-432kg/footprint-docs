# 06_log_design.md 設計・実装差分レビュー

## レビュー対象

| 区分 | 対象 |
| --- | --- |
| 設計資料 | `06_log_design.md` |
| バックエンド実装 | `../footprint/src/main/java/jp/i432kg/footprint/logging/**`, `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/**`, `../footprint/src/main/java/jp/i432kg/footprint/infrastructure/security/**` |
| バックエンド設定 | `../footprint/src/main/resources/application*.yml` |
| フロントエンド実装 | `../footprint-front/src/utils/logger.js`, `../footprint-front/src/constants/logEvents.js`, `../footprint-front/src/services/**` |

## 指摘一覧

| No. | 重要度 | 対象 | 指摘内容 | 実装との差分 | 推奨対応 | ステータス |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Medium | 構造化ログ | JSONログを基本とする方針はあるが、環境別の出力形式が記載されていない。 | stg/prod は console に logstash JSON、local はデフォルト text、`local-logfile` profile を重ねた場合だけ file に JSON 出力する。 | 設計に環境別ログ出力方式を追加する。 | 対応済み |
| 2 | Low | ログカテゴリ | 設計のカテゴリ名・JSON例は `access` だが、実装のサーバ logger 名は `footprint.access` など。 | `LoggingCategories` は `footprint.access`, `footprint.auth`, `footprint.app`, `footprint.audit` を定義している。 | 設計で「論理カテゴリ」と「実 logger 名」を分けて記載するか、JSON例を実装に合わせる。 | 対応済み |
| 3 | Medium | イベント定義 | 実装済みイベントがイベント定義に不足している。 | `HTTP_ACCESS`, `POST_SEARCH_FETCH`, `USER_CREATE_SUCCESS`, `REQUEST_VALIDATION_FAIL`, `LAST_LOGIN_UPDATE_FAILED`, storage/seed 系イベントなどが実装に存在する。 | 7章に実装済みイベントを追記する。storage/seed 系は「インフラ/seed補助イベント」として別枠にするのがよい。 | 対応不要 |
| 4 | Medium | SCR-05 検索 | 検索画面のログ要件が画面・ユースケース別ログ要件にない。 | バックエンドは `GET /api/posts/search` を `POST_SEARCH_FETCH` として access ログへ出力する。フロントも検索実行・検索APIログを持つ。 | 8章に SCR-05 検索を追加し、`POST_SEARCH_FETCH` とフロント `POST_SEARCH_EXECUTE` の扱いを記載する。 | 対応不要 |
| 5 | Medium | SCR-01 登録 | 登録重複の記載が現在仕様と異なる。 | 実装は email 重複のみを事前チェックする。username 重複は許容仕様で、username の UNIQUE 制約も前提ではない。 | `username/email重複（UNIQUE制約）` を email 重複に修正し、username 重複はログ要件に含めない。 | 対応済み |
| 6 | Low | SCR-03 地図表示 | フロントの bbox 変更ログにサンプリング/デバウンス前提とあるが、実装はその前提と一致していない。 | 実装は `moveend` ごとに `POST_MAP_MOVE` を出力し、サンプリング/デバウンスはしていない。API 実行は「このエリアで再検索」ボタン押下時。 | 地図ログ要件を現実装に合わせて「moveend の UI ログ」と「再検索ボタン押下による API ログ」に分ける。 | 対応済み |
| 7 | Medium | 地図APIフロントログ | 地図検索 API のフロントログイベントがサーバイベントと揃っていない。 | `postService.searchMap` は `POST_SEARCH_FETCH` を使っており、サーバ側の地図検索イベント `POST_MAP_BBOX_FETCH` と不一致。 | 実装を正すならフロントの地図検索APIログを `POST_MAP_BBOX_FETCH` に変更する。設計を正す場合は差異を明記するが、イベント名は揃える方が望ましい。 | 対応済み |
| 8 | Medium | 共通キー | 独自例外・想定外例外の app ログに `event` が付かないケースがある。 | `GlobalExceptionHandler` の想定外例外ログは `errorCode` のみ、独自例外ログも `errorCode` と details のみで `event` を出していない。 | 共通キーとして `event` を必須にするなら実装へ event を追加する。必須ではないなら設計の「最低限」から例外ログの扱いを分ける。 | 対応保留 |

## 詳細

### 1. 環境別ログ出力方式の不足

設計では「構造化ログ（JSON）を基本」としているが、環境別にどこへ JSON 出力するかが明確ではない。

実装では以下の構成になっている。

- 共通設定で structured logging の付加項目と stacktrace 設定を持つ。
- stg/prod は console に logstash JSON を出力する。
- local はデフォルトでは text ログで、`local-logfile` profile を重ねた場合だけ file に logstash JSON を出力する。

参照:

- 設計: `06_log_design.md` 23-39行目
- 実装: `../footprint/src/main/resources/application.yml` 40-52行目
- 実装: `../footprint/src/main/resources/application-stg.yml` 58-61行目
- 実装: `../footprint/src/main/resources/application-prod.yml` 44-47行目
- 実装: `../footprint/src/main/resources/application-local-logfile.yml` 1-13行目

修正方針:

- 「local: text console、必要時 `local,local-logfile` で JSON file」「stg/prod: JSON console」のように環境別出力を追記する。

対応結果:

- `06_log_design.md` にサーバログの環境別出力方式を追加した。
- フロントログについて、local/stg は browser console、prod は現時点では収集しない方針を追加した。
- prod のフロントログ収集は今後対応する改善策として位置付け、サーバのフロントログ受信APIへ送信して構造化ログとして出力する案を追記した。
- prod フロントログ収集を実装する場合の個人情報抑制、送信失敗時の扱い、高頻度イベント制御、サーバ側制限の原則を追記した。

### 2. ログカテゴリ名と実 logger 名の差分

設計ではカテゴリを `access`, `auth`, `app`, `audit` と記載し、JSON例の `logger` も `access` になっている。

実装ではサーバ側 logger 名として以下を使っている。

- `footprint.access`
- `footprint.auth`
- `footprint.app`
- `footprint.audit`

参照:

- 設計: `06_log_design.md` 70-75行目
- 設計: `06_log_design.md` 204-220行目
- 実装: `../footprint/src/main/java/jp/i432kg/footprint/logging/LoggingCategories.java` 11-29行目

修正方針:

- 設計に「論理カテゴリ: access / auth / app / audit」「実 logger 名: footprint.access / footprint.auth / footprint.app / footprint.audit」を明記する。
- JSONログ例も実装に合わせる。

対応結果:

- `06_log_design.md` に論理カテゴリと実 logger 名の対応表を追加した。
- JSONログ例の `logger` を `access` から `footprint.access` に更新した。

### 3. イベント定義の不足

設計のイベント定義は最低限として記載されているが、実装済みのイベントと差がある。

不足例:

- `HTTP_ACCESS`
- `POST_SEARCH_FETCH`
- `USER_CREATE_SUCCESS`
- `REQUEST_VALIDATION_FAIL`
- `LAST_LOGIN_UPDATE_FAILED`
- local/stg seed 系イベント
- storage / repository / datasource 系の失敗イベント

参照:

- 設計: `06_log_design.md` 119-155行目
- 実装: `../footprint/src/main/java/jp/i432kg/footprint/logging/LoggingEvents.java` 11-239行目
- 実装: `../footprint/src/main/java/jp/i432kg/footprint/logging/LoggingOperations.java` 12-65行目

修正方針:

- 7章に `POST_SEARCH_FETCH` と `USER_CREATE_SUCCESS` など、画面/API要件に直結するイベントを追加する。
- seed/storage/datasource 系は「インフラ補助イベント」として分ける。

クローズ理由:

- 基本設計の粒度を超えるため対応不要とする。
- 実装済みイベントの網羅は、詳細設計またはイベントカタログで管理する。

### 4. SCR-05 検索のログ要件不足

画面・ユースケース別ログ要件には SCR-01 から SCR-04 までしかなく、検索画面の要件がない。

実装では検索 API に以下のログがある。

- サーバ access: `POST_SEARCH_FETCH`
- フロント api: `POST_SEARCH_FETCH`
- フロント ui: `POST_SEARCH_EXECUTE`

参照:

- 設計: `06_log_design.md` 156-190行目
- 実装: `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/PostRestController.java` 103-128行目
- 実装: `../footprint-front/src/constants/logEvents.js` 19-22行目
- 実装: `../footprint-front/src/services/postService.js` 16-22行目

修正方針:

- 8章に `SCR-05 検索` を追加する。
- `access.INFO: /api/posts/search（keyword有無または keywordLength, lastId有無, size, 件数, durationMs）` を定義する。
- フロント補助ログとして `POST_SEARCH_EXECUTE` を記載する。検索語そのものをログに出すかは個人情報・機密情報の方針に合わせて判断する。

クローズ理由:

- 基本設計の粒度を超えるため対応不要とする。
- 個別画面/API単位のログイベント定義は、詳細設計またはイベントカタログで管理する。

### 5. 登録重複ログ要件の差分

設計では `username/email重複（UNIQUE制約）` と記載している。

現在の実装では email の重複だけを事前チェックしており、username の重複は許容する仕様。登録成功時は `USER_CREATE_SUCCESS` を app.INFO として出力している。

参照:

- 設計: `06_log_design.md` 158-164行目
- 実装: `../footprint/src/main/java/jp/i432kg/footprint/application/command/service/UserCommandService.java` 34-68行目

修正方針:

- `username/email重複（UNIQUE制約）` を「email重複」に修正する。
- username 重複は許容仕様のため、ログ要件に含めない。

対応結果:

- `06_log_design.md` の SCR-01 登録ログ要件を `email重複` のみに修正した。
- username 重複および UNIQUE 制約前提の記載を削除した。

### 6. 地図フロントログのサンプリング/デバウンス記載差分

設計では bbox 変更ログを「高頻度のためサンプリング/デバウンス前提」としている。

実装では、地図の `moveend` ごとに `POST_MAP_MOVE` を出力する。API は地図移動時に自動実行せず、「このエリアで再検索」ボタン押下時に実行する。

参照:

- 設計: `06_log_design.md` 181-185行目
- 実装: `../footprint-front/src/components/post/PostMap.vue` 162-185行目
- 実装: `../footprint-front/src/components/post/PostMap.vue` 318-330行目
- 実装: `../footprint-front/src/constants/logEvents.js` 30行目

修正方針:

- 設計を以下のように分ける。
  - フロント ui: `POST_MAP_MOVE` は `moveend` 時に出力する補助ログ。
  - サーバ access / フロント api: `POST_MAP_BBOX_FETCH` は「このエリアで再検索」ボタン押下後の API 実行ログ。
- サンプリング/デバウンスを採用しないなら、その前提記載は削除する。

対応結果:

- `06_log_design.md` の SCR-03 地図表示ログ要件を、`POST_MAP_MOVE` の UI 補助ログと `POST_MAP_BBOX_FETCH` の API 実行ログに分けて記載した。
- サンプリング/デバウンス前提の記載を削除した。

### 7. 地図検索 API のフロントログイベント不一致

サーバ側の地図検索 API は `POST_MAP_BBOX_FETCH` としてログ出力する。

一方、フロントの `postService.searchMap` は `LOG_EVENTS.POST.SEARCH_FETCH` を使っており、出力イベントは `POST_SEARCH_FETCH` になる。

参照:

- 実装: `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/PostRestController.java` 141-165行目
- 実装: `../footprint-front/src/services/postService.js` 25-31行目
- 実装: `../footprint-front/src/constants/logEvents.js` 19-31行目

修正方針:

- フロントの地図検索 API ログイベントを `POST_MAP_BBOX_FETCH` に揃える。
- `LOG_EVENTS.POST` に `MAP_BBOX_FETCH` を追加し、`postService.searchMap` で使用する。

対応結果:

- `../footprint-front/src/constants/logEvents.js` の `LOG_EVENTS.POST` に `MAP_BBOX_FETCH` を追加した。
- `../footprint-front/src/services/postService.js` の地図検索APIログイベントを `POST_MAP_BBOX_FETCH` に変更した。

### 8. 例外ログに event が付かないケース

設計では共通キーに `event` を含めている。

実装では validation 系は `event` を付与しているが、想定外例外と独自例外のログは `errorCode` 中心で、`event` を付与していない。

参照:

- 設計: `06_log_design.md` 23-39行目
- 実装: `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/GlobalExceptionHandler.java` 283-288行目
- 実装: `../footprint/src/main/java/jp/i432kg/footprint/presentation/api/GlobalExceptionHandler.java` 351-372行目

修正方針:

- `event` を共通必須キーとして扱うなら、独自例外・想定外例外ログにも `event` を追加する。
- `errorCode` を主キーにする例外ログを許容するなら、設計上「例外ログは `event` または `errorCode` を持つ」のように分ける。

保留理由:

- 今後対応するタスクとして保留する。
- 現時点では設計・実装変更は行わない。
