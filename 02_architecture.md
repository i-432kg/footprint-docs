# アーキテクチャ構成

```mermaid
    flowchart LR

%% =====================
%% User
%% =====================
    U["User<br/>Browser (Mobile / PC)"]

%% =====================
%% Frontend
%% =====================
    PAGE["Server Rendered Page<br/>(Thymeleaf)"]
    VUE["Vue UI<br/>(Mounted in Page)<br/>No Vue Router"]
    MAP["Map UI<br/>(Leaflet)"]
    TILES[Map Tiles Provider]

%% =====================
%% Backend - Controllers separated
%% =====================
    CTRL_PAGE["Page Controller<br/>(Returns HTML)"]
    VIEW[Thymeleaf View Rendering]

    CTRL_API["API Controller<br/>(Returns JSON)"]

    SEC["Spring Security<br/>(Session Cookie)"]
    SVC[Application Services]
    EXIF[EXIF Extractor]
    LOG[Logging]

    DB[(Database<br/>MySQL)]
    IMG[(Image Storage)]

%% =====================
%% Page Navigation (MPA)
%% =====================
    U -->|HTTPS GET/POST| CTRL_PAGE
    CTRL_PAGE --> SEC
    CTRL_PAGE --> VIEW
    VIEW --> PAGE
    PAGE --> U

%% =====================
%% Vue mounted inside page
%% =====================
    PAGE --> VUE
    VUE --> MAP
    MAP --> TILES

%% =====================
%% API calls from Vue
%% =====================
    VUE -->|fetch JSON API| CTRL_API
    CTRL_API --> SEC
    CTRL_API --> SVC

%% =====================
%% Backend internal flow
%% =====================
    SVC --> DB
    SVC --> IMG
    SVC --> EXIF
    SVC --> LOG
```
### Controller例

- Page Controller
    - GET /login
    - GET /
    - GET /timeline
    - GET /map
    - GET /search
    - GET /mypage

- API Controller
    - GET /api/posts
    - GET /api/posts/search
    - GET /api/posts/search/map
    - GET /api/posts/{postId}/replies
    - GET /api/replies/{parentReplyId}
    - GET /api/users/me
    - GET /api/users/me/posts
    - GET /api/users/me/replies
    - POST /api/posts
    - POST /api/replies/{postId}/reply
    - POST /api/users
    - POST /api/login
    - POST /api/logout


```mermaid
sequenceDiagram
    autonumber
    actor U as User (Browser)
    participant PC as Page Controller (HTML)
    participant TV as Thymeleaf (View Rendering)
    participant VUE as Vue UI (mounted, no Vue Router)
    participant AC as API Controller (JSON)
    participant SEC as Spring Security (Session Cookie)
    participant SVC as Application Services
    participant EX as EXIF Extractor
    participant DB as DB (MySQL)
    participant IMG as Image Storage

    rect rgba(200, 200, 200, 0.15)
        note over U,TV: Page navigation (MPA)
        U->>PC: GET /timeline
        PC->>SEC: authorize
        PC->>TV: render HTML
        TV-->>U: 200 HTML
        note over U,VUE: Vue mounts inside the page
        U->>VUE: mount Vue (client-side)
    end

    rect rgba(200, 200, 200, 0.15)
        note over U,SEC: Login (session cookie)
        U->>PC: POST /api/login (form)
        PC->>SEC: authenticate
        SEC-->>PC: session created
        PC-->>U: 200/302 + Set-Cookie
    end

    rect rgba(200, 200, 200, 0.15)
        note over VUE,IMG: Create post (Vue -> API)
        VUE->>AC: POST /api/posts (multipart) + Cookie
        AC->>SEC: authorize
        SEC-->>AC: OK
        AC->>SVC: create post
        SVC->>EX: extract GPS (optional)
        EX-->>SVC: location (optional)
        SVC->>IMG: save image
        SVC->>DB: save post + location
        DB-->>SVC: OK
        SVC-->>AC: OK
        AC-->>VUE: 201 Created
    end

    rect rgba(200, 200, 200, 0.15)
        note over VUE,DB: Map view (Vue -> API)
        VUE->>AC: GET /api/posts/search/map?minLat=...&maxLat=...&minLng=...&maxLng=...
        AC->>DB: query posts
        DB-->>AC: posts
        AC-->>VUE: 200 OK
    end
```

### 図の補足
- 認証：Spring Security によるセッションCookie
- 公開範囲：ログイン画面、サインアップ導線、静的アセット、ヘルスチェック以外は原則認証必須
- 画像：アップロード後にEXIF（GPS）を抽出し、投稿に位置情報を紐付ける
- 地図：フロントでLeafletを用いて表示し、投稿データはAPIから取得する
- ログ：アプリ/認証/アクセスを分類して出力する（出力先はデプロイ方式決定後に確定）
- frontend は別リポジトリで管理し、Docker build 時に build 成果物を backend へ取り込む
- 投稿/返信/ユーザー API は `public_id` / ULID を公開識別子として利用する
- 返信取得はトップレベル返信とネスト返信で API 経路を分ける
