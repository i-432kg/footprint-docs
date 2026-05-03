# 08_deployment.md 設計・実装差分レビュー

## レビュー対象

| 区分 | 対象 |
| --- | --- |
| 設計資料 | `08_deployment.md` |
| バックエンド実装 | `../footprint/Dockerfile`, `../footprint/compose.yaml`, `../footprint/.github/workflows/deploy-stg.yml` |
| バックエンド設定 | `../footprint/src/main/resources/application*.yml`, `../footprint/.env.example` |
| フロントエンド実装 | `../footprint-front/package.json`, `../footprint-front/vite.config.js` |

## 指摘一覧

| No. | 重要度 | 対象 | 指摘内容 | 実装との差分 | 推奨対応 | ステータス |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Medium | prod デプロイ | prod は初回リリース時に Railway を利用すると記載されているが、prod 用のデプロイ実装が見当たらない。 | 実装されている GitHub Actions は `deploy-stg.yml` のみ。prod profile はあるが、prod deploy workflow / Railway service 連携はリポジトリ上確認できない。 | prod が未実装なら「prod デプロイは今後対応」と明記する。実装済みなら workflow / 運用手順の参照先を追記する。 | 対応済み |
| 2 | Medium | STG フロントビルド | STG で frontend を build するとあるが、STG 用 build script の指定が設計に明記されていない。 | Dockerfile は `FRONTEND_BUILD_SCRIPT` の既定値を `build` としている。フロントには `build:stg` があるため、Railway 側で `FRONTEND_BUILD_SCRIPT=build:stg` を設定しないと STG mode で build されない。 | STG Railway 変数として `FRONTEND_BUILD_SCRIPT=build:stg` が必要かを明記する。不要なら `build` を使う方針を明記する。 | 対応済み |
| 3 | Medium | CI/CD品質ゲート | STG deploy workflow にテスト/静的解析の実行がなく、Dockerfile も backend test を skip している。 | workflow は変数検証、frontend checkout、Railway CLI install、`railway up` を実行する。Dockerfile は `./gradlew clean bootJar -x test` で jar を作成する。 | STG deploy 前に backend test / frontend lint or build validation を別 job で実行するか、現時点ではデプロイ速度優先で省略している旨を設計に明記する。 | 対応済み |
| 4 | Low | ADR参照 | S3 presigned URL 方針の ADR 参照先が同一リポジトリ前提のパスになっている。 | ADR は基本設計資料リポジトリではなく、バックエンドリポジトリ配下にある。 | `../footprint/docs/adr/adr_021_auth_required_and_temporary_presigned_image_url.md` に更新し、別リポジトリの ADR であることを明記する。 | 対応済み |
| 5 | Low | 設定値の責務 | Railway サービス変数に必要な具体項目が設計上「主な設定値例」に留まっている。 | 実装では DB、Spring profile、S3、STG seed、frontend manifest/build script など複数の環境変数を参照する。workflow は `FRONTEND_REPO`, `FRONTEND_REF`, `RAILWAY_SERVICE`, `FRONTEND_REPO_PAT`, `RAILWAY_TOKEN` のみを検証する。 | 基本設計では例示に留め、詳細はバックエンドリポジトリの `../footprint/docs/ops/env_management.md` で管理する。 | 対応不要 |
| 6 | Medium | STG deploy workflow 改善候補 | 現在の品質ゲートは最低限の backend test / frontend lint / frontend build を満たしているが、deploy 後確認や多重実行制御は未整備。 | `quality-check` job は追加済み。deploy 後 `/actuator/health` 確認、`concurrency`、test report artifact、backend package build、workflow lint は未追加。 | 優先度を分け、短期では `concurrency`、`./gradlew check` 化、backend package build、deploy 後 health check、失敗時 artifact 保存を検討する。依存関係脆弱性チェックや actionlint は PR CI / 定期実行 workflow で担保する。 | 改善候補 |

## 詳細

### 1. prod デプロイ実装の未確認

設計では、prod について「初回リリース時は Railway を利用し、stg と同系統の構成で運用する」としている。

レビュー作成時点では、prod profile は存在するが、リポジトリ上で確認できる GitHub Actions workflow は STG 向けの `deploy-stg.yml` のみであった。

現在は prod デプロイ用 workflow として `deploy-prod.yml` が追加されている。

参照:

- 設計: `08_deployment.md` 44-48行目
- 実装: `../footprint/.github/workflows/deploy-stg.yml` 1-70行目
- 実装: `../footprint/.github/workflows/deploy-prod.yml`
- 実装: `../footprint/src/main/resources/application-prod.yml` 1-47行目

修正方針:

- prod デプロイが未実装なら、`08_deployment.md` の prod は「今後対応」または「初回リリース時に別途整備」と明記する。
- すでに Railway 側の手動運用で存在する場合は、運用手順または参照先を追記する。

対応結果:

- prod デプロイ用 workflow `../footprint/.github/workflows/deploy-prod.yml` が追加されたためクローズする。

### 2. STG フロントビルドモードの明確化不足

設計では、Docker build 内で frontend を build し、成果物を Spring Boot の `static` 配下へ取り込むとしている。

実装では Dockerfile に `FRONTEND_BUILD_SCRIPT` build arg があり、未指定時は `npm run build` が実行される。フロントエンドには `build:stg` が定義されているため、STG 用 mode で build するには Railway 側で `FRONTEND_BUILD_SCRIPT=build:stg` を指定する必要がある。

参照:

- 設計: `08_deployment.md` 33-36行目
- 実装: `../footprint/Dockerfile` 10-13行目
- 実装: `../footprint-front/package.json` 9-13行目

修正方針:

- STG では `FRONTEND_BUILD_SCRIPT=build:stg` を Railway のサービス変数または build arg として設定する、と明記する。
- STG でも通常の `build` を使う方針なら、その理由を明記する。

対応結果:

- `08_deployment.md` の STG Frontend Integration に、Railway のサービス変数で `FRONTEND_BUILD_SCRIPT=build:stg` を指定する旨を追記した。
- `build:stg` はフロント操作ログを browser console に出力するための staging mode build であることを補足した。
- prod はフロント操作ログの console 出力が不要なため、`FRONTEND_BUILD_SCRIPT` は未指定または `build` とする旨を設定値例に追記した。

### 3. CI/CD品質ゲート不足

設計の STG デプロイフローは、checkout、変数検証、frontend checkout、Railway CLI install、`railway up` で構成されている。

実装も同様だったが、デプロイ前に backend test / frontend lint を実行していなかった。また Dockerfile の backend build は `-x test` でテストを skip している。

参照:

- 設計: `08_deployment.md` 50-62行目
- 設計: `08_deployment.md` 183-187行目
- 実装: `../footprint/.github/workflows/deploy-stg.yml`
- 実装: `../footprint/Dockerfile` 27-28行目

修正方針:

- STG deploy 前に backend test / frontend lint or build validation を追加する。
- もしくは、現時点ではデプロイ速度優先で CI 品質ゲートを省略し、別 workflow で担保する方針を設計に明記する。

対応結果:

- `../footprint/.github/workflows/deploy-stg.yml` に `quality-check` job を追加した。
- `quality-check` では MySQL service を起動し、backend の `./gradlew test`、frontend の `npm run lint`、`npm run build:stg` を実行する。
- `deploy` job は `needs: quality-check` により、品質チェック成功時のみ実行する。
- `08_deployment.md` の CI/CD 方針にも STG deploy 前の品質ゲートを追記した。

### 4. ADR参照先のパス差分

設計では S3 presigned URL 方針の ADR 参照が `docs/adr/adr_021_auth_required_and_temporary_presigned_image_url.md` になっている。

実際の ADR はバックエンドリポジトリ配下にあるため、基本設計資料からは `../footprint/docs/adr/adr_021_auth_required_and_temporary_presigned_image_url.md` と参照する必要がある。

参照:

- 設計: `08_deployment.md` 148-154行目
- 実ファイル: `../footprint/docs/adr/adr_021_auth_required_and_temporary_presigned_image_url.md`

修正方針:

- 参照先を `../footprint/docs/adr/adr_021_auth_required_and_temporary_presigned_image_url.md` に更新する。
- ADR は別リポジトリの資料であることを補足する。

対応結果:

- `08_deployment.md` の参照先を `../footprint/docs/adr/adr_021_auth_required_and_temporary_presigned_image_url.md` に更新した。
- ADR はバックエンドリポジトリの資料であることを補足した。

### 5. Railway サービス変数の明確化

設計では設定値の責務として、GitHub Actions Variables / Secrets と Railway のサービス変数を挙げている。ただし必須項目は「主な設定値例」に留まっている。

実装では以下のような環境変数を参照する。

- GitHub Actions: `FRONTEND_REPO`, `FRONTEND_REF`, `RAILWAY_SERVICE`, `FRONTEND_REPO_PAT`, `RAILWAY_TOKEN`
- アプリ runtime: `SPRING_PROFILES_ACTIVE`, `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`
- S3: `APP_STORAGE_S3_*`
- STG seed: `APP_STG_SEED_*`
- frontend build: `FRONTEND_BUILD_SCRIPT`

参照:

- 設計: `08_deployment.md` 63-83行目
- 実装: `../footprint/.github/workflows/deploy-stg.yml` 19-70行目
- 実装: `../footprint/src/main/resources/application-stg.yml` 1-61行目
- 実装: `../footprint/Dockerfile` 10-13行目

修正方針:

- 基本設計では「主な設定値例」のままで十分なら対応不要とする。
- 運用資料として使うなら、必須 Railway 変数一覧を別表または運用資料へ分離して管理する。

対応結果:

- 環境変数の詳細はバックエンドリポジトリの `../footprint/docs/ops/env_management.md` で管理しているため、基本設計資料では対応不要としてクローズする。

### 6. STG deploy workflow の追加改善候補

`../footprint/.github/workflows/deploy-stg.yml` には `quality-check` job を追加済みであり、STG deploy 前に backend test / frontend lint / frontend STG build を確認する最低限の品質ゲートは満たしている。

一方で、CI/CD のベストプラクティスとしては以下の追加チェックを検討できる。

| 優先度 | 追加項目 | 目的 | 推奨方針 |
| --- | --- | --- | --- |
| High | backend test の `check` 化 | 将来 Checkstyle / SpotBugs / JaCoCo verify 等を追加しても品質ゲートに自然に含める | `./gradlew test` から `./gradlew check` への変更を検討する |
| High | deploy 後 health check | Railway 上で起動できたことを確認する | `railway up` 後に STG の `/actuator/health` を確認する |
| Medium | backend package build | テストだけでなく jar 作成まで確認する | `quality-check` に `./gradlew bootJar`、または `./gradlew check bootJar` を追加する |
| Medium | workflow 多重実行制御 | `develop` push と `repository_dispatch` の重複実行による STG deploy 競合を防ぐ | `concurrency: group: deploy-stg` を追加し、`cancel-in-progress: true` を設定する |
| Medium | テストレポート artifact 保存 | CI 失敗時の原因調査をしやすくする | 失敗時に Gradle test report を artifact として保存する |
| Medium | workflow lint | GitHub Actions YAML の構文・式の誤りを早期検知する | `actionlint` を PR CI または定期実行 workflow で実行する |
| Medium | 依存関係脆弱性チェック | ライブラリ脆弱性を検知する | Dependabot / GitHub Dependabot alerts / 定期実行 workflow で担保する |
| Low | GitHub Actions の SHA pinning | 利用 action の supply chain risk を下げる | 厳格運用が必要になった段階で tag 指定から commit SHA pinning へ変更する |

補足:

- deploy workflow には、STG deploy 成否に直結する短時間の品質ゲートを置く。
- 脆弱性チェックや workflow lint は重要だが、毎回の STG deploy に含めるとデプロイ時間と失敗要因が増えるため、PR CI または定期実行 workflow に分離する方が運用しやすい。
- Dockerfile の `./gradlew clean bootJar -x test` は、workflow 側で品質ゲートを担保する前提であれば許容する。
