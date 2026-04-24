# Screen Specification

## 1. 画面一覧

| 画面ID | 画面名 | URL | ログイン必須 |
|--------|--------|-----|--------------|
| SCR-01 | ログイン（登録含む） | /login | × |
| SCR-02 | タイムライン | /timeline | ○ |
| SCR-03 | 地図表示 | /map | ○ |
| SCR-04 | マイページ | /mypage | ○ |
| SCR-05 | 検索 | /search | ○ |

補足：
- 新規登録は独立ページを持たず、ログイン画面内のモーダルで切り替えて表示する。
- 投稿詳細は独立ページを持たず、タイムライン画面および地図表示画面からモーダルで表示する。

---

## SCR-01 ログイン画面（登録含む）

### 概要
ログインフォームと新規登録フォームをモーダルで切り替えて表示する。
認証はSpring Security標準フォームログインを使用する。

### URL
`GET /login`

### UI構成
- ログインフォーム（デフォルト表示）
- 新規登録フォーム（モーダル切り替えで表示）

### 入力項目（ログイン）
| 項目 | 型 | 必須 |
|------|----|------|
| loginId | text | ○ |
| password | password | ○ |

### 入力項目（新規登録）
| 項目 | 型 | 必須 |
|------|----|------|
| userName | text | ○ |
| email | text | ○ |
| password | password | ○ |
| birthDate | date | ○ |

### 処理（ログイン）
- `POST /api/login`（フォーム送信）
- 認証成功時、タイムラインへリダイレクト
- 認証失敗時、エラーメッセージ表示

### 処理（新規登録）
- `POST /api/users` を実行
- 登録成功後は自動ログインする

---

## SCR-02 タイムライン画面

### 概要
投稿を新着順で表示する。無限スクロール（シーク法）を採用する。

### URL
`GET /timeline`

### 認可
ログイン必須

### 構成
- Thymeleafでページ描画
- Vueをマウント
- Axiosで `/api/posts` を取得

### UI構成
- 投稿作成ボタン
- 投稿一覧（カード）
- 投稿詳細モーダル
- 投稿作成モーダル

### 表示項目（投稿一覧）
- 投稿画像（サムネイル）
- 投稿本文
- 投稿者username
- 投稿日時
- 位置情報アイコン（has_location=trueの場合）

### 投稿作成（モーダル）
- 投稿作成ボタン押下でモーダル表示（ログイン必須）
- 入力：画像（必須）＋本文（任意）
- 投稿成功時：タイムライン先頭に反映（または再取得）

### 投稿詳細表示（モーダル）
- 投稿カード/画像クリックでモーダル表示
- モーダル内で投稿本文・画像・返信一覧を表示
- 返信投稿はログイン必須

### API
- タイムライン取得：`GET /api/posts?lastId=&size=`
- 投稿作成：`POST /api/posts`（multipart）
- 投稿詳細取得：`GET /api/posts/{postId}`
- 返信一覧取得：`GET /api/posts/{postId}/replies`
- 返信投稿：`POST /api/replies/{postId}/reply`

### ページネーション
- クエリパラメータは `lastId` / `size`
- クライアントは最後に受け取った投稿 ID を次回 `lastId` に使う
- レスポンスは配列であり、`page.nextCursor` は返さない
- スクロール末尾到達で次ページ取得
- 詳細な seek 条件は `docs/adr/adr_023_seek_pagination_boundary.md` に従う

---

## SCR-03 地図表示画面

### 概要
Leafletを使用して地図表示する。
現在の表示範囲（bbox）に含まれる投稿を取得し、マーカー表示する。

### URL
`GET /map`

### 認可
ログイン必須

### 処理
- 地図表示範囲変更時に bbox を算出
- bbox を用いて投稿を取得し、マーカーを更新する

### 表示内容
- マーカー（位置情報を持つ投稿のみ）
- マーカークリックで投稿詳細モーダルを表示する

### 投稿詳細表示（モーダル）
- タイムライン画面と同一仕様のモーダルを表示する（投稿詳細＋返信）

### API
- 地図投稿取得：`GET /api/posts/search/map?minLat=&maxLat=&minLng=&maxLng=`
- 投稿詳細取得：`GET /api/posts/{postId}`
- 返信一覧取得：`GET /api/posts/{postId}/replies`
- 返信投稿：`POST /api/replies/{postId}/reply`

---

## SCR-04 マイページ

### 概要
ログイン中ユーザーの情報および履歴を表示する。

### URL
`GET /mypage`

### 認可
ログイン必須

### 表示内容
- username表示（ログイン中アカウントの確認）
- 自分の投稿一覧（無限スクロール）
- 自分の返信一覧（無限スクロール）

### API
- 自分のユーザー情報：`GET /api/users/me`
- 自分の投稿一覧：`GET /api/users/me/posts?lastId=&size=`
- 自分の返信一覧：`GET /api/users/me/replies?lastId=&size=`

### ページネーション
- クエリパラメータは `lastId` / `size`
- クライアントは最後に受け取った投稿 ID / 返信 ID を次回 `lastId` に使う
- レスポンスは配列であり、`page.nextCursor` は返さない
- 詳細な seek 条件は `docs/adr/adr_023_seek_pagination_boundary.md` に従う

---

## SCR-05 検索画面

### 概要
キーワードに基づいて投稿を検索する。結果一覧は無限スクロールで追加取得する。

### URL
`GET /search`

### 認可
ログイン必須

### API
- 検索結果取得：`GET /api/posts/search?keyword=&lastId=&size=`

### ページネーション
- クエリパラメータは `lastId` / `size`
- クライアントは最後に受け取った投稿 ID を次回 `lastId` に使う
- レスポンスは配列であり、`page.nextCursor` は返さない
