# 03_database.md 実装差分レビュー

## レビュー概要

`03_database.md` とバックエンドリポジトリの DB migration / MyBatis mapper / 関連ドメインサービスを照合した。
主な差分は、マイページ返信一覧向け index、返信一覧の並び順、ADR 参照先の表現である。
`username` の重複許容と FK の削除時動作は方針を確定し、設計書と実装を修正済みである。

## 指摘一覧

| No | 重要度 | 指摘内容 | 対象 | 実装との差分 | 修正方針 | ステータス |
|---:|---|---|---|---|---|---|
| 1 | High | `users.username` が UNIQUE と記載されているが、実装では一意制約がない | `users` 制約 | migration は `public_id` と `email` のみ UNIQUE。アプリ側も email 重複のみを検査している | `username` は重複許容を正とし、設計書から UNIQUE 制約を削除する | 対応済み |
| 2 | Medium | FK の `ON DELETE CASCADE` が設計書に記載されていない | `post_images`, `replies` 制約 | 実装は `post_images.post_id`、`replies.post_id`、`replies.parent_id` に `ON DELETE CASCADE` を設定していた | `posts -> post_images` のみ cascade とし、返信は application 層で削除ユースケースを明示制御する | 対応済み |
| 3 | Medium | マイページ返信一覧向け index が設計書にあるが、migration に存在しない | インデックス設計 4.2 | 設計は `(replies.user_id, replies.created_at DESC)` を記載しているが、実装は `replies.user_id` 起点の index を作成していなかった | migration に `idx_replies_user_timeline` を追加し、`id DESC` まで含める | 対応済み |
| 4 | Medium | 返信一覧取得の設計は `created_at` 順 index を前提に見えるが、実装 SQL は top-level / nested replies に `ORDER BY` がない | 返信取得 4.4 / 想定クエリ 5.2 | `findTopLevelRepliesByPostId` と `findNestedRepliesByParentId` は `ORDER BY` なしで取得していた | 新しい順を正とし、`ORDER BY created_at DESC, id DESC` と対応 index を追加する | 対応済み |
| 5 | Low | マイページ投稿一覧の index が `ORDER BY created_at DESC, id DESC` と完全一致していない | インデックス設計 4.2 | 実装 SQL は `ORDER BY p.created_at DESC, p.id DESC` だが、migration の index は `(user_id, created_at DESC)` までだった | `(user_id, created_at DESC, id DESC)` に修正する | 対応済み |
| 6 | Low | `docs/adr/...` 参照が別リポジトリ内 ADR であることを明示していない | ページング設計メモ | ADR はバックエンドリポジトリ `../footprint/docs/adr/` 配下に存在する | 相対パスで `../footprint/docs/adr/...` と記載し、別リポジトリの ADR であることを補足する | 対応済み |
| 7 | Low | 章番号が `7` の後に `6` へ戻っている | ページング設計メモ / 今後の拡張想定 | 実装差分ではないが、資料として読み順が崩れている | 章番号を `6. ページング設計メモ`、`7. 今後の拡張想定` に修正する | 対応済み |

## 詳細

### 1. `users.username` が UNIQUE と記載されているが、実装では一意制約がない

#### 設計書

`03_database.md` は `users` の制約として以下を記載している。

- `public_id` は UNIQUE
- `email` は UNIQUE
- `username` は UNIQUE

参照: `03_database.md:86-89`

#### 実装

migration では `users` に対して `uq_users_public_id` と `uq_users_email` のみ定義している。
`username` の UNIQUE 制約はない。

参照:

- `../footprint/src/main/resources/db/migration/V1__init.sql:26-27`
- `../footprint/src/main/resources/db/migration/V1__init.sql:30-31`

アプリ側でも `UserDomainService` は email 重複のみを検査しており、username 重複検査はない。

参照:

- `../footprint/src/main/java/jp/i432kg/footprint/domain/service/UserDomainService.java:33-38`
- `../footprint/src/main/resources/jp/i432kg/footprint/infrastructure/datasource/mapper/repository/UserMapper.xml:19-23`

#### 影響

設計書を読むと表示名が一意である前提に見えるが、実装では同じ username のユーザー登録を許容する。
表示名を匿名 SNS 上の非一意な表示ラベルとして扱うなら実装が自然だが、設計書の制約と矛盾している。

#### 修正方針

`username` は重複許容を正とし、`03_database.md` から `username は UNIQUE` を削除した。
表示名は匿名 SNS 上の表示ラベルとして扱い、ユーザーの一意性は `public_id` と `email` で担保する。

ステータス: 対応済み

### 2. FK の `ON DELETE CASCADE` 方針が設計書と実装で整理されていない

#### 設計書

`post_images` と `replies` は FK 参照先のみ記載していた。

参照:

- `03_database.md:151-153`
- `03_database.md:175-179`

#### 実装

migration では以下の FK に `ON DELETE CASCADE` が設定されていた。

- `post_images.post_id -> posts.public_id`
- `replies.post_id -> posts.public_id`
- `replies.parent_id -> replies.public_id`

参照:

- `../footprint/src/main/resources/db/migration/V1__init.sql:75-77`
- `../footprint/src/main/resources/db/migration/V1__init.sql:93-99`

#### 影響

投稿削除や親返信削除を将来実装する際、DB が返信を cascade delete すると、返信ツリー、`child_count`、将来の論理削除・監査・通知との整合性が application 層から見えにくくなる。
一方、`post_images` は投稿に完全従属する DB 行であり、投稿削除時に cascade しても業務判断が薄い。

#### 修正方針

`posts -> post_images` のみ DB cascade とする。
`posts -> replies`、`replies -> replies` は cascade せず、削除ユースケースを実装する場合は application 層で返信ツリーや `child_count` との整合性を明示的に制御する。

対応内容:

- `03_database.md` に `post_images.post_id` の `ON DELETE CASCADE` と、返信は cascade しない方針を追記した
- `../footprint/src/main/resources/db/migration/V1__init.sql` から `replies.post_id` と `replies.parent_id` の `ON DELETE CASCADE` を削除した

ステータス: 対応済み

### 3. マイページ返信一覧向け index が設計書にあるが、migration に存在しない

#### 設計書

マイページ向け index として以下を記載している。

- `(posts.user_id, posts.created_at DESC, posts.id DESC)`
- `(replies.user_id, replies.created_at DESC, replies.id DESC)`

参照: `03_database.md:238-245`

#### 実装

posts 側には `idx_posts_user_timeline` がある。
replies 側には `user_id` 起点の index がなかったため、`idx_replies_user_timeline` を追加した。

参照:

- `../footprint/src/main/resources/db/migration/V1__init.sql:55-56`
- `../footprint/src/main/resources/db/migration/V1__init.sql:102-107`

一方、マイページ返信一覧 SQL は `r.user_id = #{userId}` で絞り込み、`created_at DESC, id DESC` で並べる。

参照: `../footprint/src/main/resources/jp/i432kg/footprint/infrastructure/datasource/mapper/query/ReplyQueryMapper.xml:42-77`

#### 影響

返信数が増えると、マイページ返信一覧で `user_id` 絞り込みとソートの効率が設計どおりにならない可能性がある。

#### 修正方針

実装へ `idx_replies_user_timeline` を追加し、マイページ返信一覧 SQL の `ORDER BY r.created_at DESC, r.id DESC` に合わせて `id DESC` まで含めた。

対応内容:

- `../footprint/src/main/resources/db/migration/V1__init.sql` に `CREATE INDEX idx_replies_user_timeline ON replies (user_id, created_at DESC, id DESC);` を追加した

ステータス: 対応済み

### 4. 返信一覧取得の設計は `created_at` 順 index を前提に見えるが、実装 SQL は top-level / nested replies に `ORDER BY` がない

#### 設計書

返信取得向け index として以下を記載していた。

- `(replies.post_id, created_at)`
- `(replies.parent_id, created_at)`

参照: `03_database.md:271-279`

#### 実装

top-level replies と nested replies の取得 SQL は `post_id` / `parent_id` で絞り込むが、`ORDER BY` がなかった。

参照:

- `../footprint/src/main/resources/jp/i432kg/footprint/infrastructure/datasource/mapper/query/ReplyQueryMapper.xml:17-28`
- `../footprint/src/main/resources/jp/i432kg/footprint/infrastructure/datasource/mapper/query/ReplyQueryMapper.xml:30-40`

#### 影響

DB は `ORDER BY` なしの結果順を保証しない。
返信一覧は新しい順を正とし、同一 `created_at` の順序を安定させるため `id DESC` を併用する。

#### 修正方針

返信一覧は `ORDER BY created_at DESC, id DESC` で取得する。

対応内容:

- `03_database.md` の返信取得 index と表示順を `created_at DESC, id DESC` に更新した
- `../footprint/src/main/resources/jp/i432kg/footprint/infrastructure/datasource/mapper/query/ReplyQueryMapper.xml` に `ORDER BY created_at DESC, id DESC` を追加した
- `../footprint/src/main/resources/db/migration/V1__init.sql` の返信取得 index を `(post_id, created_at DESC, id DESC)`、`(parent_id, created_at DESC, id DESC)` に更新した

ステータス: 対応済み

### 5. マイページ投稿一覧の index が `ORDER BY created_at DESC, id DESC` と完全一致していない

#### 設計書

マイページのページングは、タイムラインと同様に `created_at` / `id` の複合条件で扱うと記載している。
一方、index は `(posts.user_id, posts.created_at DESC)` までの記載だった。

参照: `03_database.md:238-245`

#### 実装

マイページ投稿一覧 SQL は `ORDER BY p.created_at DESC, p.id DESC` を使う。
migration の index は `(user_id, created_at DESC)` であり、`id DESC` は含まれていなかった。

参照:

- `../footprint/src/main/resources/jp/i432kg/footprint/infrastructure/datasource/mapper/query/PostQueryMapper.xml:112-148`
- `../footprint/src/main/resources/db/migration/V1__init.sql:55-56`

#### 影響

`created_at` が同一の投稿が多い場合、`id` を含む安定ソートの意図に対して index が完全にはそろっていなかった。
`ORDER BY` と index の対応を明確にする必要がある。

#### 修正方針

index を `(user_id, created_at DESC, id DESC)` に修正した。

対応内容:

- `03_database.md` のマイページ投稿一覧 index を `(posts.user_id, posts.created_at DESC, posts.id DESC)` に更新した
- `../footprint/src/main/resources/db/migration/V1__init.sql` の `idx_posts_user_timeline` を `(user_id, created_at DESC, id DESC)` に更新した

ステータス: 対応済み

### 6. `docs/adr/...` 参照が別リポジトリ内 ADR であることを明示していない

#### 設計書

ページング設計メモで以下を参照している。

- `docs/adr/adr_023_seek_pagination_boundary.md`
- `docs/adr/adr_025_seek_pagination_query_split.md`

参照: `03_database.md:310-316`

#### 実装・配置

該当 ADR はバックエンドリポジトリ配下に存在する。

参照:

- `../footprint/docs/adr/adr_023_seek_pagination_boundary.md`
- `../footprint/docs/adr/adr_025_seek_pagination_query_split.md`

#### 影響

`footprint-docs` リポジトリ内の相対パスとして読むと参照先が存在しない。

#### 修正方針

別リポジトリの ADR であることを明記し、相対パスを `../footprint/docs/adr/...` に更新した。

対応内容:

- `03_database.md` の ADR 参照を `../footprint/docs/adr/adr_023_seek_pagination_boundary.md` に更新した
- `03_database.md` の ADR 参照を `../footprint/docs/adr/adr_025_seek_pagination_query_split.md` に更新した
- いずれも別リポジトリの ADR であることを明記した

ステータス: 対応済み

### 7. 章番号が `7` の後に `6` へ戻っている

#### 設計書

`## 7. ページング設計メモ` の後に `## 6. 今後の拡張想定` が続いている。

参照:

- `03_database.md:310`
- `03_database.md:320`

#### 影響

実装差分ではないが、基本設計資料として章立てが読みづらい。

#### 修正方針

章番号を以下へ修正する。

- `## 6. ページング設計メモ`
- `## 7. 今後の拡張想定`

ステータス: 対応済み
