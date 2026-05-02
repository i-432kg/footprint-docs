# DB設計

## 1. 設計方針

本サービスは匿名型SNSであるが、認証管理のため内部的なユーザー識別子を保持する。
表示名（username）は保持するが、プロフィール情報やフォロー関係は持たない。

設計方針は以下の通り。

- MVPでは小規模運用を前提とし、シンプルなRDB設計とする
- 将来機能（複数画像投稿、いいね、削除、通知等）を追加しやすい構造とする
- 返信は無制限階層に対応可能な構造とする
- タイムラインは新着順表示とし、無限スクロール（シーク法）を採用する
- 地図表示は表示範囲（Bounding Box）検索に対応する
- 公開 API / URL では `public_id` を利用し、内部ソートや seek 条件の安定化には `id` を併用する
- `id` は DB 内部の主キーとして扱い、API レスポンスや外部公開には出さない
- `public_id` はフロントエンドへ返却してよい公開識別子として扱い、外部から受け取る参照キーにも利用する
- FK にも `public_id` を使い、フロントエンドから受け取った公開 ID を内部主キーへ変換せずに検索・結合できるようにする

DBは MySQL 8.x を前提とする。

### 1.1 `id` と `public_id` の役割分担

- `id`
  - DB の内部主キー
  - `AUTO_INCREMENT BIGINT` による一意識別子
  - 内部ソート、seek 条件の境界判定、DB 内部の安定した順序付けに利用する
  - 連番で推測しやすいため、API レスポンスや外部公開には使わない

- `public_id`
  - API / URL / フロントエンド連携で利用する公開識別子
  - 推測しにくい ID を使うことで、単純な連番推測を避ける
  - フロントエンドから受け取る参照キーとしても利用する

### 1.2 FK に `public_id` を使う理由

本設計では、`posts.user_id -> users.public_id` のように FK も `public_id` を参照する。

理由は以下の通り。

- API 入出力で扱う識別子が `public_id` に統一される
- フロントエンドから受け取った `public_id` をもとに検索する際、都度 `id` へ引き直さずにインデックスを効かせやすい
- 参照・結合条件が API の公開識別子と一致するため、アプリケーション層での主キー変換処理を減らせる

一方で、`id` は DB 内部の主キーとして残し、`created_at` と組み合わせた安定ソートや seek 条件の境界判定に利用する。

---

## 2. ER概要

主要エンティティは以下の通り。

- users
- posts
- post_images
- replies

### リレーション

- users 1 --- N posts
- posts 1 --- N post_images
- posts 1 --- N replies
- replies 1 --- N replies（自己参照：無制限階層）

---

## 3. テーブル定義

### 3.1 users

| カラム名 | 型 | NULL | 説明 |
|---|---|---|---|
| id | BIGINT | NO | 内部主キー |
| public_id | CHAR(26) | NO | API / 外部公開用 ID |
| username | VARCHAR(50) | NO | 表示用ユーザー名 |
| email | VARCHAR(255) | NO | ログイン ID |
| password_hash | VARCHAR(255) | NO | ハッシュ化パスワード |
| birthdate | DATE | NO | 生年月日 |
| is_active | BOOLEAN | NO | メール疎通確認状態。`true` は確認 OK、`false` は確認 NG |
| disabled | BOOLEAN | NO | 退会状態。`true` は退会済み、`false` は未退会 |
| disabled_at | DATETIME | YES | 退会日時 |
| last_login_at | DATETIME | YES | 最終ログイン日時 |
| created_at | DATETIME | NO | 作成日時 |
| updated_at | DATETIME | NO | 更新日時 |

制約：
- public_id は UNIQUE
- email は UNIQUE

`username` は表示用ユーザー名であり、重複を許容する。

状態系カラムの扱い:

- `is_active`
  - メール認証による疎通確認状態を表す
  - `true`: 疎通確認 OK
  - `false`: 疎通確認 NG
  - 現時点ではメール疎通確認機能が未実装のため未使用カラムとする

- `disabled`
  - ユーザーの退会状態を表す
  - `true`: 退会済み
  - `false`: 未退会
  - 現時点では退会機能が未実装のため未使用カラムとする

- `disabled_at`
  - 退会日時を表す
  - `disabled=true` に遷移した時点で設定する想定
  - 現時点では退会機能が未実装のため未使用カラムとする

---

### 3.2 posts

| カラム名 | 型 | NULL | 説明 |
|---|---|---|---|
| id | BIGINT | NO | 内部主キー |
| public_id | CHAR(26) | NO | API / 外部公開用 ID |
| user_id | CHAR(26) | NO | 投稿者の `users.public_id` を参照する公開 FK |
| caption | TEXT | YES | 投稿本文 |
| has_location | BOOLEAN | NO | 位置情報有無 |
| latitude | DECIMAL(9,6) | YES | 緯度 |
| longitude | DECIMAL(9,6) | YES | 経度 |
| taken_at | DATETIME | YES | 撮影日時（EXIF） |
| created_at | DATETIME | NO | 作成日時 |
| updated_at | DATETIME | NO | 更新日時 |

制約：
- public_id は UNIQUE
- user_id → users.public_id（外部キー）

---

### 3.3 post_images

将来的な複数画像対応を見据え、投稿と画像は分離する。

| カラム名 | 型 | NULL | 説明 |
|---|---|---|---|
| id | BIGINT | NO | 内部主キー |
| post_id | CHAR(26) | NO | 紐づく投稿の `posts.public_id` を参照する公開 FK |
| sort_order | INT | NO | 表示順 |
| storage_type | VARCHAR(16) | NO | `LOCAL` / `S3` |
| object_key | VARCHAR(1024) | NO | 保存先オブジェクトキー |
| file_extension | VARCHAR(128) | NO | ファイル拡張子 |
| size_bytes | BIGINT | NO | ファイルサイズ |
| width | INT | YES | 横幅 |
| height | INT | YES | 高さ |
| exif_available | BOOLEAN | NO | EXIF取得可否 |
| created_at | DATETIME | NO | 作成日時 |

制約：
- post_id → posts.public_id（外部キー、ON DELETE CASCADE）
- (post_id, sort_order) は UNIQUE

MVPでは1投稿1画像とし、`sort_order = 0` で運用する。
投稿画像レコードは投稿に完全従属するため、投稿削除時は DB の cascade により削除する。

---

### 3.4 replies

返信は無制限階層に対応するため、自己参照構造（Adjacency List）を採用する。

| カラム名 | 型 | NULL | 説明 |
|---|---|---|---|
| id | BIGINT | NO | 内部主キー |
| public_id | CHAR(26) | NO | API / 外部公開用 ID |
| post_id | CHAR(26) | NO | 対象投稿の `posts.public_id` を参照する公開 FK |
| user_id | CHAR(26) | NO | 投稿者の `users.public_id` を参照する公開 FK |
| parent_id | CHAR(26) | YES | 親返信の `replies.public_id` を参照する公開 FK |
| message | TEXT | NO | 返信本文 |
| child_count | INT | NO | 子返信数。都度 `COUNT(*)` を避けるため親返信に保持する派生値 |
| created_at | DATETIME | NO | 作成日時 |
| updated_at | DATETIME | NO | 更新日時 |

制約：
- public_id は UNIQUE
- post_id → posts.public_id
- user_id → users.public_id
- parent_id → replies.public_id（自己参照）

返信はユーザー生成コンテンツとして扱い、投稿や親返信の削除時に DB の cascade では削除しない。
削除ユースケースを実装する場合は、application 層で返信ツリーや `child_count` との整合性を明示的に制御する。

`child_count` の扱い:

- `child_count` は親返信が保持する子返信数の派生値とする
- 投稿詳細や返信ツリー表示で都度 `COUNT(*)` を実行すると負荷が高くなりやすいため、親返信レコードへキャッシュする
- 更新責務は application 層が持つ
  - 返信作成時に、親返信が存在する場合は application service が保存処理と同一トランザクション内で `child_count` を加算する
  - 現実装も `ReplyCommandService` から `ReplyRepository.increaseReplyCount(...)` を呼ぶ構成としている
- 将来、返信削除を実装する場合も application 層が減算責務を持つ前提とする

注意点:

- `child_count` は派生値であり、元データと不整合を起こしうる
- 返信作成だけでなく、将来の返信削除・移動・再構築でも同じ整合性ルールを守る必要がある
- application 層を経由しない DB 直更新や別経路のバッチ処理を導入する場合は、`child_count` の整合性維持策を別途定義する必要がある
- 高並行で親返信へ子返信が追加される場合は、ロストアップデートを避ける更新方式で実装する必要がある

---

## 4. インデックス設計

### 4.1 タイムライン（無限スクロール・シーク法）

タイムラインは新着順とし、シーク法を採用する。

API では `lastId` / `size` を使う。
`lastId` は公開 ID だが、サーバー側ではその行の `created_at` と内部 `id` を参照して次ページ条件を構築する。
初回表示と継続取得では SQL を分ける。

- 初回表示: `ORDER BY created_at DESC, id DESC LIMIT :size`
- 継続取得: `lastId` に対応する行を 1 回だけ解決し、同じソートキー集合で seek 条件を組む

継続取得の概念上の条件は以下とする。

```sql
WITH cursor_post AS (
  SELECT created_at, id
  FROM posts
  WHERE public_id = :lastId
)
SELECT *
FROM posts p
CROSS JOIN cursor_post c
WHERE p.created_at < c.created_at
   OR (p.created_at = c.created_at AND p.id < c.id)
ORDER BY created_at DESC, id DESC
LIMIT :size;
```

インデックス：

- posts(created_at DESC, id DESC)

`ORDER BY` に `id` を含める場合、seek 条件も同じキー集合でそろえる。
`created_at` だけで境界を切ると、同一 timestamp の行でページ境界の取りこぼしが起こりうる。

---

### 4.2 マイページ

- (posts.user_id, posts.created_at DESC, posts.id DESC)
- (replies.user_id, replies.created_at DESC, replies.id DESC)

マイページの無限スクロールも `lastId` / `size` を使う。
境界条件はタイムラインと同様に、ソートキーと一致する複合条件で扱う。
実装では初回表示用 SQL と継続取得用 SQL を分け、継続取得時のみ `lastId` から `created_at` / `id` を解決する。

---

### 4.3 地図表示（Bounding Box検索）

地図表示では現在表示範囲内の投稿のみ取得する。

例：
```sql
SELECT *
FROM posts
WHERE has_location = 1
AND latitude BETWEEN :minLat AND :maxLat
AND longitude BETWEEN :minLng AND :maxLng;
```

インデックス：

- (has_location, latitude, longitude)

将来的に高負荷となった場合は、
Spatial Index（POINT型）への移行を検討する。

---

### 4.4 返信取得

- (replies.post_id, created_at DESC, id DESC)
- (replies.parent_id, created_at DESC, id DESC)
- 返信一覧は `ORDER BY created_at DESC, id DESC` により新しい順で取得する

投稿詳細表示時はトップレベル返信を `post_id` で取得し、
子返信は `parent_id` ごとに別 API で取得する。
現実装は 1 クエリで返信ツリー全体を構築する方式ではない。

---

## 5. 想定クエリ

### 5.1 タイムライン取得（無限スクロール）

- API パラメータは `lastId` / `size`
- 最後に取得した投稿 ID を次回 `lastId` に使う
- 降順ソートと一致する複合 seek 条件でページングする

### 5.2 投稿詳細取得

- post 本体取得
- post_images 取得
- top-level replies を post_id で取得
- nested replies を parent_id で取得

### 5.3 地図範囲検索（bbox）

- has_location = 1
- latitude/longitude を範囲条件で検索

### 5.4 マイページ

- user_id = ログインユーザーID
- 投稿一覧／返信一覧を `lastId` / `size` で取得

---

## 6. ページング設計メモ

- クライアントは最後に受け取った要素の公開 ID を `lastId` として送る
- レスポンスは配列を返し、`page.nextCursor` は返さない
- 次ページの有無は、返却件数が `size` 未満になること、または次回取得が空になることで判断する
- 詳細な方針は、別リポジトリの ADR `../footprint/docs/adr/adr_023_seek_pagination_boundary.md` に従う
- SQL 分割方針は、別リポジトリの ADR `../footprint/docs/adr/adr_025_seek_pagination_query_split.md` に従う

---

## 7. 今後の拡張想定

- いいね機能（likes テーブル追加）
- 論理削除（deleted_at 追加）
- 通知機能
- 高速スレッド探索のための Closure Table 導入
- Spatial Index導入
