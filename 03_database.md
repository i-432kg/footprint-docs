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

DBは MySQL 8.x を前提とする。

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
| public_id | CHAR(26) | NO | 外部公開用 ID |
| username | VARCHAR(50) | NO | 表示用ユーザー名 |
| email | VARCHAR(255) | NO | ログイン ID |
| password_hash | VARCHAR(255) | NO | ハッシュ化パスワード |
| birthdate | DATE | NO | 生年月日 |
| is_active | BOOLEAN | NO | アカウント認証状態 |
| disabled | BOOLEAN | NO | 無効化状態 |
| disabled_at | DATETIME | YES | 無効化日時 |
| last_login_at | DATETIME | YES | 最終ログイン日時 |
| created_at | DATETIME | NO | 作成日時 |
| updated_at | DATETIME | NO | 更新日時 |

制約：
- public_id は UNIQUE
- email は UNIQUE
- username は UNIQUE

---

### 3.2 posts

| カラム名 | 型 | NULL | 説明 |
|---|---|---|---|
| id | BIGINT | NO | 内部主キー |
| public_id | CHAR(26) | NO | 外部公開用 ID |
| user_id | CHAR(26) | NO | 投稿者の `users.public_id` |
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
| post_id | CHAR(26) | NO | 紐づく投稿の `posts.public_id` |
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
- post_id → posts.public_id（外部キー）
- (post_id, sort_order) は UNIQUE

MVPでは1投稿1画像とし、`sort_order = 0` で運用する。

---

### 3.4 replies

返信は無制限階層に対応するため、自己参照構造（Adjacency List）を採用する。

| カラム名 | 型 | NULL | 説明 |
|---|---|---|---|
| id | BIGINT | NO | 内部主キー |
| public_id | CHAR(26) | NO | 外部公開用 ID |
| post_id | CHAR(26) | NO | 対象投稿の `posts.public_id` |
| user_id | CHAR(26) | NO | 投稿者の `users.public_id` |
| parent_id | CHAR(26) | YES | 親返信の `replies.public_id` |
| message | TEXT | NO | 返信本文 |
| child_count | INT | NO | 子返信数 |
| created_at | DATETIME | NO | 作成日時 |
| updated_at | DATETIME | NO | 更新日時 |

制約：
- public_id は UNIQUE
- post_id → posts.public_id
- user_id → users.public_id
- parent_id → replies.public_id（自己参照）

---

## 4. インデックス設計

### 4.1 タイムライン（無限スクロール・シーク法）

タイムラインは新着順とし、シーク法を採用する。

API では `lastId` / `size` を使う。
`lastId` は公開 ID だが、サーバー側ではその行の `created_at` と内部 `id` を参照して次ページ条件を構築する。

概念上の条件は以下とする。

```sql
SELECT *
FROM posts
WHERE created_at < :lastCreatedAt
   OR (created_at = :lastCreatedAt AND id < :lastInternalId)
ORDER BY created_at DESC, id DESC
LIMIT :size;
```

インデックス：

- posts(created_at DESC, id DESC)

`ORDER BY` に `id` を含める場合、seek 条件も同じキー集合でそろえる。
`created_at` だけで境界を切ると、同一 timestamp の行でページ境界の取りこぼしが起こりうる。

---

### 4.2 マイページ

- (posts.user_id, posts.created_at DESC)
- (replies.user_id, replies.created_at DESC)

マイページの無限スクロールも `lastId` / `size` を使う。
境界条件はタイムラインと同様に、ソートキーと一致する複合条件で扱う。

---

### 4.3 地図表示（Bounding Box検索）

地図表示では現在表示範囲内の投稿のみ取得する。

例：
```sql
SELECT *
FROM posts
WHERE has_location = 1
AND lat BETWEEN :minLat AND :maxLat
AND lng BETWEEN :minLng AND :maxLng;
```

インデックス：

- (has_location, lat, lng)

将来的に高負荷となった場合は、
Spatial Index（POINT型）への移行を検討する。

---

### 4.4 返信取得

- (replies.post_id, created_at)
- (replies.parent_reply_id, created_at)
- ORDER BY created_at DESC, id DESC を採る場合は seek 条件も複合化する

投稿詳細表示時は post_id で取得し、
アプリケーション側でツリー構築する。

---

## 5. 想定クエリ

### 5.1 タイムライン取得（無限スクロール）

- API パラメータは `lastId` / `size`
- 最後に取得した投稿 ID を次回 `lastId` に使う
- 降順ソートと一致する複合 seek 条件でページングする

### 5.2 投稿詳細取得

- post 本体取得
- post_images 取得
- replies を post_id で取得

### 5.3 地図範囲検索（bbox）

- has_location = 1
- lat/lng を範囲条件で検索

### 5.4 マイページ

- user_id = ログインユーザーID
- 投稿一覧／返信一覧を `lastId` / `size` で取得

---

## 7. ページング設計メモ

- クライアントは最後に受け取った要素の公開 ID を `lastId` として送る
- レスポンスは配列を返し、`page.nextCursor` は返さない
- 次ページの有無は、返却件数が `size` 未満になること、または次回取得が空になることで判断する
- 詳細な方針は `docs/adr/adr_023_seek_pagination_boundary.md` に従う

---

## 6. 今後の拡張想定

- いいね機能（likes テーブル追加）
- 論理削除（deleted_at 追加）
- 通知機能
- 高速スレッド探索のための Closure Table 導入
- Spatial Index導入
