# 05_screen_spec.md 設計・実装差分レビュー

## レビュー対象

| 区分 | 対象 |
| --- | --- |
| 設計資料 | `05_screen_spec.md` |
| バックエンド実装 | `../footprint/src/main/java/jp/i432kg/footprint/presentation/web/RootController.java` |
| フロントエンド実装 | `../footprint-front/src/entries/*`, `../footprint-front/src/components/*` |

## 指摘一覧

| No. | 重要度 | 対象 | 指摘内容 | 実装との差分 | 推奨対応 | ステータス |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Low | 画面一覧 | ルートパス `/` が画面一覧に記載されていない。 | `RootController` は `GET /` をタイムライン画面として返す。 | 画面一覧または補足に「`GET /` はインデックス画面としてタイムラインを表示する」を追加する。 | 対応済み |
| 2 | Medium | SCR-02 タイムライン | 投稿一覧の表示項目に「投稿本文」「投稿者username」が含まれている。 | 投稿カード一覧は画像、位置情報、投稿日を表示しており、投稿本文と投稿者usernameは表示していない。APIレスポンスにも投稿者usernameは含まれていない。 | 投稿者usernameは表示項目から外し、投稿本文は詳細モーダルで表示する旨を補足する。 | 対応済み |
| 3 | Medium | SCR-03 地図表示 | 地図表示範囲変更時に自動で bbox 検索すると読める。 | 実装では移動・ズーム終了時に `hasMoved` を立て、「このエリアで再検索」ボタン押下時だけ API を実行する。 | 実装を正として、範囲変更時は再検索ボタンを表示し、ボタン押下で bbox 検索する仕様へ更新する。 | 対応済み |
| 4 | Medium | SCR-03 地図表示 | 地図画面の API 一覧に子返信取得 API がない。 | 地図画面の投稿詳細モーダルはタイムラインと同一で、子返信展開時に `GET /api/replies/{parentReplyId}` を利用する前提。 | SCR-03 の API 一覧に子返信一覧取得を追加する。 | 対応済み |
| 5 | Medium | SCR-04 マイページ | 自分の投稿一覧・返信一覧が「無限スクロール」と記載されている。 | 実装はタブ切り替えと「もっと読み込む」ボタンによる追加取得。IntersectionObserver による自動読み込みではない。 | 実装を正として、ボタン押下による追加読み込みに修正する。 | 対応済み |
| 6 | Medium | SCR-05 検索 | 検索画面の URL パラメータが記載されていない。 | ヘッダー検索は `/search?q=...` へ遷移し、検索画面は `q` を読み取って API の `keyword` に渡す。 | URL を `GET /search?q=` とし、画面 URL は `q`、API パラメータは `keyword` であることを明記する。 | 対応済み |
| 7 | Low | ADR参照 | seek pagination の ADR 参照先が同一リポジトリ前提のパスになっている。 | ADR は基本設計資料リポジトリではなく、バックエンドリポジトリ配下にある。 | `../footprint/docs/adr/adr_023_seek_pagination_boundary.md` へ更新し、別リポジトリの ADR であることを明記する。 | 対応済み |
| 8 | Low | SCR-03 地図表示 | 初期表示時の現在地取得とフォールバック地点が記載されていない。 | 実装では Geolocation API で現在地取得を試み、失敗時は皇居付近を初期表示して投稿を取得する。 | 基本設計の粒度では補足扱いとする。 | 補足扱い |

## 詳細

### 1. ルートパス `/` の画面定義不足

`05_screen_spec.md` の画面一覧は `/login`, `/timeline`, `/map`, `/mypage`, `/search` のみを定義している。

一方、実装では `RootController` が `GET /` を受け、`timeline` テンプレートを返す。

- 設計: `05_screen_spec.md` 5-11行目
- 実装: `../footprint/src/main/java/jp/i432kg/footprint/presentation/web/RootController.java` 38-40行目

修正方針:

- 画面一覧にインデックス画面を追加する。
- または補足として「`GET /` はタイムライン画面を返す」を明記する。

対応結果:

- `05_screen_spec.md` の画面一覧補足に「`GET /` はインデックス画面としてタイムライン画面を表示する」を追加した。

### 2. タイムライン投稿一覧の表示項目差分

設計では投稿一覧に「投稿本文」「投稿者username」を表示するとしている。

実装の `PostCard` は以下を表示している。

- 投稿画像
- 位置情報の有無に応じた緯度経度または「位置情報不明」
- 投稿日時

投稿本文は一覧カードでは表示されず、投稿詳細モーダル側で表示する構成になっている。また、投稿者usernameは現在の投稿 API レスポンスに含まれないため、画面側でも表示していない。

- 設計: `05_screen_spec.md` 81-87行目
- 実装: `../footprint-front/src/components/post/PostCard.vue` 20-35行目
- 実装: `../footprint-front/src/entries/timeline/App.vue` 136-145行目

修正方針:

- 投稿者usernameは投稿一覧の表示項目から外す。
- 投稿本文は投稿一覧カードではなく、投稿詳細モーダルで表示する旨を補足する。

対応結果:

- `05_screen_spec.md` の投稿一覧表示項目から投稿者usernameを削除した。
- 投稿本文は一覧カードでは表示せず、投稿詳細モーダルで表示する旨を補足した。

### 3. 地図表示範囲変更時の検索トリガー差分

設計では、地図表示範囲変更時に bbox を算出し、bbox を用いて投稿取得・マーカー更新を行うと記載している。

実装では、表示範囲変更時に API は呼ばず、`hasMoved` を `true` にして「このエリアで再検索」ボタンを表示する。API 呼び出しはボタン押下時に実行される。

- 設計: `05_screen_spec.md` 128-130行目
- 実装: `../footprint-front/src/components/post/PostMap.vue` 162-191行目
- 実装: `../footprint-front/src/components/post/PostMap.vue` 318-330行目

修正方針:

- 実装を正として、以下の流れに修正する。
  - 地図移動・ズーム終了時に bbox を算出可能な状態にする。
  - 表示範囲が変わった場合は「このエリアで再検索」ボタンを表示する。
  - ボタン押下時に bbox を用いて投稿を取得し、マーカーを更新する。

対応結果:

- `05_screen_spec.md` の SCR-03 処理を、範囲変更時は再検索ボタンを表示し、ボタン押下時に bbox 検索する仕様へ更新した。

### 4. 地図画面 API 一覧の子返信取得不足

地図画面の投稿詳細モーダルはタイムライン画面と同一仕様と記載されているが、API 一覧には子返信取得 API が含まれていない。

タイムライン画面では `GET /api/replies/{parentReplyId}` が明記されているため、地図画面側にも同じ API を記載するのが自然。

- 設計: `05_screen_spec.md` 136-143行目
- 設計: `05_screen_spec.md` 99-105行目

修正方針:

- SCR-03 の API 一覧に「子返信一覧取得：`GET /api/replies/{parentReplyId}`」を追加する。

対応結果:

- `05_screen_spec.md` の SCR-03 API 一覧に、子返信一覧取得 `GET /api/replies/{parentReplyId}` を追加した。

### 5. マイページの追加読み込み方式差分

設計では自分の投稿一覧・返信一覧を無限スクロールとしている。

実装では、投稿と返信履歴をタブで切り替え、それぞれ「もっと読み込む」ボタンで追加取得している。

- 設計: `05_screen_spec.md` 158-162行目
- 実装: `../footprint-front/src/entries/mypage/App.vue` 49-92行目
- 実装: `../footprint-front/src/entries/mypage/App.vue` 150-155行目
- 実装: `../footprint-front/src/entries/mypage/App.vue` 194-205行目
- 実装: `../footprint-front/src/entries/mypage/App.vue` 243-255行目

修正方針:

- 実装を正として、マイページは「投稿」「返信履歴」タブを持つことを明記する。
- 一覧の追加取得方式は「もっと読み込む」ボタンに修正する。

対応結果:

- `05_screen_spec.md` の SCR-04 表示内容に「自分の投稿」「返信履歴」のタブ切り替えを追加した。
- 自分の投稿一覧・返信一覧の追加取得方式を「もっと読み込む」ボタンへ修正した。

### 6. 検索画面 URL パラメータの記載不足

設計では検索画面 URL を `GET /search` としているが、実装ではヘッダー検索から `/search?q=...` へ遷移する。

検索画面は URL の `q` を読み取り、API 呼び出し時に `keyword` として渡す。

- 設計: `05_screen_spec.md` 181-188行目
- 実装: `../footprint-front/src/components/layout/TheHeader.vue` 51-63行目
- 実装: `../footprint-front/src/entries/search/App.vue` 24-55行目

修正方針:

- SCR-05 の URL を `GET /search?q=` に更新する。
- 「画面 URL の検索語は `q`、API パラメータは `keyword`」と補足する。

対応結果:

- `05_screen_spec.md` の SCR-05 URL を `GET /search?q=` に更新した。
- 画面 URL の検索語は `q`、API 呼び出し時は `keyword` パラメータとして渡す旨を補足した。

### 7. ADR 参照先のパス差分

設計では `docs/adr/adr_023_seek_pagination_boundary.md` を参照しているが、このパスは基本設計資料リポジトリ内の相対パスとして解釈される。

実際の ADR はバックエンドリポジトリ配下にあるため、基本設計資料からは `../footprint/docs/adr/adr_023_seek_pagination_boundary.md` と参照する必要がある。

- 設計: `05_screen_spec.md` 112行目
- 設計: `05_screen_spec.md` 172行目
- 実ファイル: `../footprint/docs/adr/adr_023_seek_pagination_boundary.md`

修正方針:

- 参照先を `../footprint/docs/adr/adr_023_seek_pagination_boundary.md` に更新する。
- ADR は別リポジトリの資料であることを補足する。

対応結果:

- `05_screen_spec.md` の ADR 参照先を `../footprint/docs/adr/adr_023_seek_pagination_boundary.md` に更新した。
- ADR は別リポジトリ（バックエンド）の資料であることを明記した。

### 8. 地図初期表示仕様の補足不足

実装では、地図初期化時に Geolocation API で現在地取得を試みる。取得できた場合は現在地周辺を表示し、失敗または非対応の場合は皇居付近を初期表示して投稿を取得する。

設計には初期表示地点や現在地取得に関する記載がない。

- 設計: `05_screen_spec.md` 116-144行目
- 実装: `../footprint-front/src/components/post/PostMap.vue` 247-299行目

修正方針:

- 基本設計の粒度では、詳細な Geolocation の成否分岐までは補足扱いとする。

対応結果:

- `05_screen_spec.md` への追記は行わず、レビュー上で補足扱いとして整理した。
