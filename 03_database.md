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

| カラム名          | 型            | NULL | 説明               |
|---------------|--------------|------|------------------|
| id            | BIGINT       | NO   | 主キー              |
| username      | VARCHAR(50)  | NO   | 表示用ユーザー名         |
| email         | VARCHAR(255) | NO   | ログインID           |
| password_hash | VARCHAR(255) | NO   | ハッシュ化パスワード       |
| status        | VARCHAR(32)  | NO   | アカウント状態（ACTIVE等） |
| last_login_at | DATETIME     | YES  | 最終ログイン日時         |
| created_at    | DATETIME     | NO   | 作成日時             |
| updated_at    | DATETIME     | NO   | 更新日時             |

制約：
- email は UNIQUE
- username は UNIQUE

---

### 3.2 posts

| カラム名         | 型            | NULL | 説明         |
|--------------|--------------|------|------------|
| id           | BIGINT       | NO   | 主キー        |
| user_id      | BIGINT       | NO   | 投稿者        |
| caption      | TEXT         | YES  | 投稿本文       |
| has_location | BOOLEAN      | NO   | 位置情報有無     |
| lat          | DECIMAL(9,6) | YES  | 緯度         |
| lng          | DECIMAL(9,6) | YES  | 経度         |
| taken_at     | DATETIME     | YES  | 撮影日時（EXIF） |
| created_at   | DATETIME     | NO   | 作成日時       |
| updated_at   | DATETIME     | NO   | 更新日時       |

制約：
- user_id → users.id（外部キー）

---

### 3.3 post_images

将来的な複数画像対応を見据え、投稿と画像は分離する。

| カラム名           | 型             | NULL | 説明              |
|----------------|---------------|------|-----------------|
| id             | BIGINT        | NO   | 主キー             |
| post_id        | BIGINT        | NO   | 紐づく投稿           |
| sort_order     | INT           | NO   | 表示順             |
| storage_type   | VARCHAR(16)   | NO   | LOCAL / S3 等    |
| path           | VARCHAR(1024) | NO   | 保存パスまたはオブジェクトキー |
| content_type   | VARCHAR(128)  | NO   | MIMEタイプ         |
| size_bytes     | BIGINT        | NO   | ファイルサイズ         |
| width          | INT           | YES  | 横幅              |
| height         | INT           | YES  | 高さ              |
| exif_available | BOOLEAN       | NO   | EXIF取得可否        |
| created_at     | DATETIME      | NO   | 作成日時            |

制約：
- post_id → posts.id（外部キー）
- (post_id, sort_order) は UNIQUE

MVPでは1投稿1画像とし、`sort_order = 0` で運用する。

---

### 3.4 replies

返信は無制限階層に対応するため、自己参照構造（Adjacency List）を採用する。

| カラム名            | 型        | NULL | 説明   |
|-----------------|----------|------|------|
| id              | BIGINT   | NO   | 主キー  |
| post_id         | BIGINT   | NO   | 対象投稿 |
| user_id         | BIGINT   | NO   | 投稿者  |
| parent_reply_id | BIGINT   | YES  | 親返信  |
| body            | TEXT     | NO   | 返信本文 |
| created_at      | DATETIME | NO   | 作成日時 |
| updated_at      | DATETIME | NO   | 更新日時 |

制約：
- post_id → posts.id
- user_id → users.id
- parent_reply_id → replies.id（自己参照）

---

## 4. インデックス設計

### 4.1 タイムライン（無限スクロール・シーク法）

タイムラインは新着順とし、シーク法を採用する。

例：
```sql
SELECT *
FROM posts
WHERE created_at < :cursor_created_at
ORDER BY created_at DESC
LIMIT 20;
```

インデックス：

- posts(created_at DESC, id DESC)

※ created_at が同一の場合の安定ソートのため id を併用する。

---

### 4.2 マイページ

- (posts.user_id, posts.created_at DESC)

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

投稿詳細表示時は post_id で取得し、
アプリケーション側でツリー構築する。

---

## 5. 想定クエリ

### 5.1 タイムライン取得（無限スクロール）

- created_at をカーソルとして利用
- 降順でページング

### 5.2 投稿詳細取得

- post 本体取得
- post_images 取得
- replies を post_id で取得

### 5.3 地図範囲検索（bbox）

- has_location = 1
- lat/lng を範囲条件で検索

### 5.4 マイページ

- user_id = ログインユーザーID
- 投稿一覧／返信一覧を取得

---

## 6. 今後の拡張想定

- いいね機能（likes テーブル追加）
- 論理削除（deleted_at 追加）
- 通知機能
- 高速スレッド探索のための Closure Table 導入
- Spatial Index導入