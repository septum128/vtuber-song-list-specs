# SongList バックエンド仕様書

## 1. 概要

VTuber が歌枠配信で歌った楽曲をデータベース化・検索できるシステムのバックエンド仕様書。
YouTube のコメント欄から AI（OpenAI）を用いてセトリを自動抽出し、会員による修正提案・管理者の承認フローも備える。

本書は `src/`・`migration/`・`config/`・`.github/workflows/`・`Dockerfile`・`docker-compose.yaml` の実装を正とする。
テーブルの詳細は `tables.md` を参照。

---

## 2. 技術スタック

| カテゴリ | 技術 | バージョン | 備考 |
|---------|------|----------|------|
| 言語 | Rust | edition 2021 | `rustfmt` `max_width = 100` |
| フレームワーク | Loco | 0.16 | SaaS スターターテンプレート由来 |
| Web サーバー | Axum | 0.8 | |
| ORM | SeaORM | 1.1 | `sqlx-postgres` / `sqlx-sqlite` / `runtime-tokio-rustls` |
| DB マイグレーション | `sea-orm-migration` | - | `migration/` クレート |
| データベース | PostgreSQL | - | 開発では SQLite も可 |
| 非同期ランタイム | Tokio | 1.45 | `rt-multi-thread` |
| ジョブ実行 | Loco `BackgroundWorker`（`workers.mode: BackgroundAsync`） | - | プロセス内非同期。テスト時は `ForegroundBlocking` |
| スケジューラ | Loco 組み込みスケジューラ（`scheduler:` / `cargo loco scheduler`） | - | cron は UTC |
| 認証 | JWT（`loco_rs::auth::jwt`） | - | API キー認証も実装あり |
| バリデーション | `validator` | 0.20 | |
| ビューエンジン | Tera + `fluent-templates` | Tera 1.20 / `fluent-templates` 0.13 | i18n 用。API レスポンスは基本 JSON。メールテンプレートは `tera::Tera::one_off` で直接レンダリングする（`src/mailers/auth.rs`） |
| HTTP クライアント | `reqwest` | 0.12 | `json` フィーチャ |
| 日付 | `chrono` | 0.4 | |
| その他 | `regex` 1.11, `uuid` 1.6, `serde` / `serde_json` 1, `axum-extra` 0.10 | | |
| テスト | `insta`（スナップショット）, `serial_test`, `rstest`, `loco-rs` testing | - | 実 PostgreSQL 必須（モックなし） |

> **Redis について**: `Cargo.toml` に Redis 依存はなく、`config/` にもキュー設定はない。ワーカーは `BackgroundAsync`（プロセス内）で動作する。CI のみ Redis サービスと `REDIS_URL` を用意している。

### 2.1 エントリポイント

- `src/bin/main.rs` — `loco_rs::cli::main::<App, Migrator>()` を起動。
- `src/app.rs` — `Hooks` trait を `App` 構造体に実装。ルート・ワーカー・タスク・イニシャライザ・マイグレーション・シードを配線。
- `src/lib.rs` — `app` / `controllers` / `models` / `workers` / `tasks` / `mailers` / `views` / `initializers` / `data` モジュールを公開。

### 2.2 外部サービス連携

| サービス | 用途 | 認証方式 | 実装 |
|---------|------|---------|------|
| YouTube RSS フィード | チャンネルの最新動画一覧取得 | 不要 | `src/workers/youtube_client.rs` |
| YouTube Data API v3 | 動画情報・コメント・チャンネルアイコン取得 | API Key（`GOOGLE_API_KEY`） | 同上 |
| OpenAI Chat Completions API | コメントからセトリ JSON 抽出（`gpt-4o-mini`） | Bearer Token（`OPENAI_API_KEY`） | `src/workers/openai_client.rs` |
| Spotify Web API | 曲名からアーティスト名を補完 | Client Credentials（`SPOTIFY_CLIENT_ID` / `SPOTIFY_CLIENT_SECRET`） | `src/workers/spotify_client.rs` |
| Slack Incoming Webhook | セトリ自動作成の結果通知 | Webhook URL（`SLACK_WEBHOOK_URL`、任意） | `src/workers/slack_client.rs` |
| Cloudflare Email Sending REST API | 認証・通知メール送信（既定経路） | Bearer Token（`CLOUDFLARE_API_TOKEN`）+ `CLOUDFLARE_ACCOUNT_ID` | `src/mailers/cloudflare_client.rs` / `src/mailers/cloudflare_worker.rs` |
| SMTP | 上記のフォールバック（`CLOUDFLARE_API_TOKEN` 未設定時。開発 / テスト用） | `config` の `mailer.smtp` | `src/mailers/auth.rs` |

---

## 3. データモデル

テーブル・カラム・インデックス・外部キー・Enum の詳細は `tables.md` を参照。

### 3.1 テーブル一覧

| テーブル名 | 説明 |
|----------|------|
| users | ユーザーアカウント（JWT 認証） |
| channels | YouTube チャンネル情報 |
| videos | YouTube 動画・ライブ配信情報 |
| comments | YouTube 動画のコメント（セトリ情報源） |
| song_items | セトリの各曲エントリ（コンテナ） |
| song_diffs | セトリ曲情報の変更履歴・修正提案 |
| favorites | ユーザーのお気に入り曲 |

### 3.2 モデル層の構成

- `src/models/_entities/` — SeaORM の自動生成エンティティ（手動編集禁止）。
- `src/models/<name>.rs` — エンティティ上のビジネスロジック層。Enum 定数（`KIND_*` / `STATUS_*`）、ファインダ（`find_paginated` 等）、`ActiveModel` の生成・更新メソッドを実装。
- `src/models/mod.rs` に各モデルを登録。

### 3.3 SongItem と SongDiff の関係

`song_items` は曲エントリのコンテナで、実データ（`time` / `title` / `author`）は `song_diffs` に持つ。
`song_items.latest_diff_id` は最新の確定 diff への非正規化参照（DB 上の FK 制約なし）。この値が NULL の曲は未確定として扱い、公開 API では返さない。

diff の承認（`song_diffs::Model::approve`）または管理画面・AI からの確定 diff 作成時に `latest_diff_id` が更新される。

---

## 4. API エンドポイント

ルートは `src/app.rs` の `App::routes` で登録。`AppRoutes::with_default_routes()` により Loco の組み込みルート（`GET /_health`, `GET /_ping`）も有効。
インデックスルートは各コントローラで `add("/", ...)` として定義されており、パスは末尾スラッシュ付き（例 `/api/channels/`）。

### 4.1 認証（アプリ独自・`src/controllers/sessions.rs`）

フロントエンドはこちらを使用する。ログイン識別子は `name`（`users::Model::find_by_name`）で、`email` は登録時に本人が入力した実アドレスをそのまま保持する。

| Method | Path | 認証 | 説明 |
|--------|------|------|------|
| POST | `/api/user` | 不要 | ユーザー登録（`name`, `email`, `password`, `password_confirmation`）。成功時 **201**・`{ message, user: { id, name, kind }, token }` |
| GET | `/api/user` | JWT | ログイン中ユーザー取得（`{ id, name, kind }`） |
| POST | `/api/session` | 不要 | ログイン（`name`, `password`）。`{ message, user, token }` |
| DELETE | `/api/session` | JWT | ログアウト（`{ message: "ログアウトしました", user: null }`。トークン破棄はクライアント側） |

- 失敗時は `400 Bad Request`・`{ "message": "..." }`。登録時のメッセージは発生原因で切り替える。
  - `ModelError::EntityAlreadyExists`（email 重複）→「このメールアドレスは既に登録されています」
  - `ModelError::Message("name already exists")`（name 重複）→「このユーザー名は既に使われています」
  - その他（email 形式など `validator` のエラー）→「入力内容をご確認ください（メールアドレスの形式などを確認してください）」
  - ログイン失敗 →「ユーザー名またはパスワードが違います」
- 登録成功時は `set_email_verification_sent` で `email_verification_token` / `email_verification_sent_at` を記録し、ウェルカムメール（メールアドレス確認リンク付き）を送信する。`ADMIN_NOTIFICATION_EMAIL` が設定されていれば管理者宛の新規登録通知も送信する（この通知の失敗は `tracing::error!` でログ出力するのみで、登録レスポンスは成功扱い）。
- メール認証はログインの要件ではない（`password` の一致のみ検証）。

### 4.2 認証（Loco SaaS スターター既定・`src/controllers/auth.rs`）

プレフィックス `/api/auth`。JSON ボディで `email` を用いる。主にテストで使用。レスポンスは `LoginResponse { token, pid, name, is_verified }` 等。

| Method | Path | 説明 |
|--------|------|------|
| POST | `/api/auth/register` | 登録 + welcome メール送信 + 管理者通知メール送信（`ADMIN_NOTIFICATION_EMAIL` 設定時のみ。失敗はログ出力のみ） |
| POST | `/api/auth/verify/{token}` | メール認証（GET でも可） |
| POST | `/api/auth/login` | ログイン（`email`, `password`） |
| POST | `/api/auth/forgot` | パスワードリセットトークン発行・メール送信 |
| POST | `/api/auth/reset` | パスワードリセット実行 |
| GET | `/api/auth/current` | 現在ユーザー（`{ pid, name, email }`） |
| POST | `/api/auth/magic-link` | マジックリンク発行（`@example.com` / `@gmail.com` ドメインのみ許可） |
| GET | `/api/auth/magic-link/{token}` | マジックリンク検証（5分有効・32文字トークン） |
| POST | `/api/auth/resend-verification-mail` | 認証メール再送 |

### 4.3 チャンネル（公開・`src/controllers/channels.rs`）

| Method | Path | 認証 | 説明 |
|--------|------|------|------|
| GET | `/api/channels/` | 不要 | `kind = published(100)` のみ、`id` 昇順 |
| GET | `/api/channels/{id}` | 不要 | ID 指定取得（非公開チャンネルも返る）。未存在は 404 |

レスポンス: `{ id, channel_id, name, custom_name, twitter_id, icon_url, kind, status }`

### 4.4 動画（公開・`src/controllers/videos.rs`）

| Method | Path | 認証 | 説明 |
|--------|------|------|------|
| GET | `/api/videos/` | 不要 | `published = true` のみ、`published_at` 降順 |
| GET | `/api/videos/{id}` | 不要 | ID 指定取得（未存在は 404） |

クエリパラメータ（`GET /api/videos/`）:

| パラメータ | 型 | 説明 |
|----------|-----|------|
| `channel_id` | integer | チャンネルで絞込 |
| `query` | string | タイトル部分一致 |
| `since` / `until` | string(`YYYY-MM-DD`) | `published_at` の範囲（`since` は 00:00:00、`until` は 23:59:59 を UTC 換算） |
| `only_song_lives` | integer | 1 以上で「`active` な `song_items` が 1 件以上ある動画」に限定（EXISTS サブクエリ） |
| `page` | integer | 1 始まり（既定 1） |
| `count` | integer | 1 ページ件数（既定 20） |

レスポンス: `{ id, channel_id, video_id, title, kind, status, published, published_at }`

### 4.5 セトリ（公開・`src/controllers/song_items.rs`）

生 SQL（`song_items` ⋈ `videos` ⋈ `channels` ⋈ `song_diffs`）で取得。`latest_diff_id IS NOT NULL AND videos.published = true` が常に付与される。

| Method | Path | 認証 | 説明 |
|--------|------|------|------|
| GET | `/api/song_items/` | 不要 | ソート `videos.published_at DESC, song_diffs.time ASC NULLS LAST` |
| GET | `/api/song_items/{id}` | 不要 | ID 指定取得 |

クエリパラメータ:

| パラメータ | 型 | 説明 |
|----------|-----|------|
| `channel_id` | integer | チャンネルで絞込 |
| `video_id` | integer | 動画で絞込 |
| `query` | string | `song_diffs.title` または `song_diffs.author` の ILIKE 部分一致 |
| `since` / `until` | string(`YYYY-MM-DD`) | `videos.published_at` の範囲 |
| `video_title` | string | 動画タイトルの ILIKE 部分一致 |
| `page` / `count` | integer | ページング（既定 20） |

レスポンス: `{ id, video_id, latest_diff_id, diff: { title, author, time }, video: { id, video_id, title, channel_id, channel_custom_name, kind, published_at } }`

### 4.6 会員機能（JWT 必須・`src/controllers/member/`）

`auth::JWT` エクストラクタで検証し、`users::Model::find_by_pid` で存在確認。存在しなければエラー。

**セトリ修正（`song_diffs.rs`）**

| Method | Path | 説明 |
|--------|------|------|
| GET | `/api/member/song_items/{song_item_id}/song_diffs/` | 修正履歴一覧（`created_at` 昇順） |
| POST | `/api/member/song_items/{song_item_id}/song_diffs/` | 修正提案（`time?`, `title?`, `author?`）。**201** |

- `made_by_id` はログインユーザー ID をサーバー側で付与。`kind = manual`。
- 管理者（`kind = 10`）の投稿は即 `status = approved` となり `song_item.latest_diff_id` を更新。
- 一般ユーザーの投稿は `status = pending`（管理者承認待ち）。

**お気に入り（`favorites.rs`）**

| Method | Path | 説明 |
|--------|------|------|
| GET | `/api/member/favorites/ids` | お気に入り曲 ID 一覧（`{ song_item_ids: [] }`） |
| GET | `/api/member/favorites/` | お気に入り曲の詳細一覧（`/api/song_items` と同形式・`favorites.created_at` 降順） |
| POST | `/api/member/favorites/` | 追加（`{ song_item_id }`）。既存なら 200、新規なら 201 |
| DELETE | `/api/member/favorites/{song_item_id}` | 削除 |

### 4.7 管理者機能（JWT + `kind = admin(10)` 必須・`src/controllers/admin/`）

`require_admin`（`src/controllers/admin/mod.rs`）で `user.kind != 10` なら `401 Unauthorized`（「管理者権限が必要です」）。

**チャンネル管理（`channels.rs`）**

| Method | Path | 説明 |
|--------|------|------|
| GET | `/api/admin/channels/` | 全チャンネル（非公開含む）、`id` 昇順 |
| POST | `/api/admin/channels/` | 作成（`channel_id`, `name?`, `custom_name`, `twitter_id?`, `kind?`）。YouTube API でアイコン取得を試行し `icon_url` を保存、`SongItemsCreatorWorker` を投入。**201** |
| PATCH | `/api/admin/channels/{id}` | 更新（`name?`, `custom_name?`, `twitter_id?`, `kind?`, `fetch_icon?`）。`name` / `twitter_id` は JSON `null` で明示的にクリア可 |
| DELETE | `/api/admin/channels/{id}` | 削除。成功時 **204** |

**動画管理（`videos.rs`）**

| Method | Path | 説明 |
|--------|------|------|
| GET | `/api/admin/videos/` | `channel_id?`, `only_song_lives?`(bool・タイトルのキーワード一致), `page?`, `count?`。`published_at` 降順 |
| POST | `/api/admin/videos/` | 作成（`video_id`（URL でも可）, `channel_id`）。重複チェック → YouTube API 取得 → 保存 → `SongItemsCreatorWorker` 投入 |
| POST | `/api/admin/videos/bulk` | TSV 一括登録（`{ tsv }`。1行目ヘッダをスキップ、`チャンネル custom_name <TAB> URL`）。`{ succeeded, skipped, failed }` を返す |
| POST | `/api/admin/videos/bulk_fetch_setlist` | `{ video_ids: [], force? }` → `SetlistFetchWorker` 投入 |
| POST | `/api/admin/videos/bulk_publish` | `{ video_ids: [], published }` → 対象動画の `published` のみ一括更新（`published_at` は変更しない）。`{ updated }` を返す |
| PATCH | `/api/admin/videos/{id}` | 更新（`title?`, `published?`, `kind?`, `status?`, `published_at?`（RFC3339）） |
| POST | `/api/admin/videos/{id}/fetch_setlist` | `{ force? }` → `SetlistFetchWorker` 投入 |

**セトリ管理（`song_items.rs`）**

| Method | Path | 説明 |
|--------|------|------|
| GET | `/api/admin/song_items/?video_id=&page=&count=` | 動画単位の一覧（`song_diffs.time` 昇順、既定 count 50） |
| POST | `/api/admin/song_items/` | 作成（`video_id`, `time?`, `title?`, `author?`）。`song_item` + 承認済み manual diff を生成。**201** |
| POST | `/api/admin/song_items/bulk` | `{ video_id, items: [{ time?, title?, author? }] }` → `{ created }` |
| DELETE | `/api/admin/song_items/{id}` | 削除（`latest_diff_id` を NULL 化 → `song_diffs` 削除 → `song_item` 削除）。成功時 **204** |

**修正承認（`song_diffs.rs`）**

| Method | Path | 説明 |
|--------|------|------|
| GET | `/api/admin/song_diffs/?status=&page=&count=` | `created_at` 降順。`status` で絞込可 |
| PATCH | `/api/admin/song_diffs/{id}/approve` | `status = approved` + `song_item.latest_diff_id` 更新 |
| PATCH | `/api/admin/song_diffs/{id}/reject` | `status = rejected` |

---

## 5. 認証・認可

### 5.1 認証方式

- **JWT**（`loco_rs::auth::jwt`）。トークンの subject は `users.pid`（UUID）。有効期限は既定 604800 秒（7日）。
- 署名鍵は `auth.jwt.secret`（本番は環境変数 `JWT_SECRET`、開発 / テストは `config` 直書き）。
- パスワードは Argon2（`loco_rs::hash`）でハッシュ化。
- `users.pid` / `users.api_key` はエンティティの `before_save` で挿入時に採番（`api_key` は `lo-<uuid>`）。
- ログイン識別子は `users.name`（unique index `idx-users-name-unique`）。`users.email` も unique で、メール送信先として使用する。`name` の重複はモデル層（`create_with_password` のトランザクション内）でも明示的にチェックする。
- **API キー認証**も `Authenticable` 実装として存在するが、フロントエンドでは未使用。
- マジックリンク認証（`MAGIC_LINK_LENGTH = 32` / `MAGIC_LINK_EXPIRATION_MIN = 5`）。

### 5.2 アクセス制御

| 対象 | 条件 | 未認証・権限不足時 |
|------|------|-------------------|
| 公開 API（`/api/channels` `/api/videos` `/api/song_items` `/api/user`(POST) `/api/session`(POST) `/api/auth/*`） | 誰でも可 | - |
| `/api/member/*`・`GET /api/user`・`DELETE /api/session` | 有効な JWT + 実在ユーザー | 401 |
| `/api/admin/*` | 有効な JWT + `users.kind == 10` | 401 |

### 5.3 SongDiff 承認フロー

```
[一般ユーザー] POST /api/member/song_items/:id/song_diffs
    → kind: manual, status: pending で保存
    → 管理者が /api/admin/song_diffs で承認 or 却下
        approved → song_item.latest_diff_id を更新
        rejected → song_item は変更なし

[管理者] POST /api/member/song_items/:id/song_diffs
    → kind: manual, status: approved で即時保存 + latest_diff_id 更新

[管理画面から直接作成] POST /api/admin/song_items
    → song_item + kind: manual, status: approved の diff を生成

[AI 自動抽出] SongItemsCreatorWorker / SetlistFetchWorker
    → kind: auto, status: approved, comment_id 付きで生成
```

---

## 6. セトリ自動作成処理

### 6.1 実行トリガー

| 経路 | 実装 |
|------|------|
| 定期実行 | Loco スケジューラ（`config/production.yaml` の `scheduler:`）が毎日 **00:00 UTC = 09:00 JST**（cron `0 0 0 * * *`）にタスク `create_recent_videos_song_items` を実行。本番は `start --all` で起動 |
| 手動タスク | `cargo loco task create_recent_videos_song_items`（`src/tasks/create_recent_videos_song_items.rs`） |
| チャンネル / 動画の追加時 | `POST /api/admin/channels` `POST /api/admin/videos` `POST /api/admin/videos/bulk` が完了後に `SongItemsCreatorWorker` を投入 |
| 既存動画の個別再取得 | `POST /api/admin/videos/{id}/fetch_setlist` / `POST /api/admin/videos/bulk_fetch_setlist` → `SetlistFetchWorker` |

いずれもエントリは `SongItemsCreatorWorker`（`src/workers/song_items_creator.rs`）または `SetlistFetchWorker`（`src/workers/setlist_fetch_worker.rs`）。

### 6.2 SongItemsCreatorWorker の処理フロー

```
1. fetch_and_insert_videos（全チャンネル）
   ├ YouTube RSS フィード（feeds/videos.xml）から動画エントリを取得
   ├ DB 未登録のものだけ YouTube Data API v3（snippet,contentDetails,liveStreamingDetails）で詳細取得
   ├ liveBroadcastContent == "upcoming"（配信開始前）はスキップ
   ├ duration <= 60 秒（ショート）はスキップ
   ├ published_at: ライブは actualStartTime、それ以外は RSS の <published>
   └ videos を status=fetched(10), published=false, kind=0 で挿入
      （response_json に published_at の採用元と生値を記録）

2. process_song_items（status != completed かつ != comments_disabled の動画）
   ├ タイトルが歌枠キーワードを含むか判定（is_song_live、大文字小文字無視）
   │   歌枠 / うたわく / 歌ってみた / 弾き語り / 歌配信 / カラオケ / karaoke / singing / song
   ├ YouTube Data API v3 で全コメント取得（commentThreads, maxResults=100, ページネーション）
   │   403（コメント無効）なら video.status=comments_disabled(11) にして終了
   ├ コメントを upsert（comment_id で一意）
   ├ セトリ判定（Comment::is_setlist）: 正規表現 ([0-9]{2}:)?[0-9]?[0-9]:[0-9]{2} が 3 回以上マッチ
   ├ OpenAI Chat API でセトリ判定コメントごとにセトリ JSON を抽出し、動画単位で一旦 `PendingSongEntry`（comment_id, time, title, author）として集約
   │   抽出できたコメントは comment.status=completed(20) にする（重複排除で結果的に 0 件になっても再処理はしない）
   ├ dedup_song_entries: title/author が空のエントリを除外 → time/title/author 順にソート → title+author を正規化（前後空白除去・全角スペース統一・連続空白圧縮・小文字化）した値で重複除去
   └ 重複排除後の各エントリを song_item + song_diff（kind=auto, status=approved, comment_id）で作成
      → 1 件でも作成できたら video.status=song_items_created(20)

3. backfill_author_from_history（status=song_items_created の動画）
   ├ latest_diff の author が空の diff について
   ├ 同一 title・status=approved・author 非空の最新 diff を検索して author をコピー
   └ video.status=fetched_history(25)

4. backfill_author_from_spotify（status=fetched_history の動画）
   ├ author が空の diff について Spotify search（2 パターン: "title", "title track"）
   ├ ヒット → author を設定、diff.status=35 / ヒットなし → diff.status=30
   └ video.status=completed(40)

5. Slack 通知（SLACK_WEBHOOK_URL があれば成功・失敗いずれも通知）
```

### 6.3 SetlistFetchWorker（既存動画向け）

- 引数: `{ video_ids: [i32], force: bool }`。
- `force = true`: 対象動画の `song_items` / `song_diffs` を全削除し `status` を `fetched(10)` にリセットしてから再処理。
- `force = false`: `status >= song_items_created(20)` の動画はスキップ。
- 各動画に対し `process_song_items_for_video` → `backfill_history_for_video` → `backfill_spotify_for_video` を実行。

### 6.4 OpenAI 連携（`src/workers/openai_client.rs`）

- エンドポイント: `POST https://api.openai.com/v1/chat/completions`、モデル `gpt-4o-mini`。
- system プロンプトで `[{"time":"HH:MM:SS","title":"曲名","author":"アーティスト名"}]` 形式の JSON のみを返させる。`author` 不明時は空文字列。
- 応答から markdown コードフェンスを除去し、`time` を `HH:MM:SS` に正規化（`MM:SS` → `00:MM:SS`）。

### 6.5 Spotify 補完（`src/workers/spotify_client.rs`）

1. `POST https://accounts.spotify.com/api/token`（Basic 認証・`grant_type=client_credentials`）で Bearer Token 取得。
2. `GET https://api.spotify.com/v1/search?q=...&type=track&limit=1` を「曲名」「曲名 + " track"」の 2 パターンで検索。
3. 最初にヒットしたトラックの先頭アーティスト名を返す。

### 6.6 YouTube データ取得方式（`src/workers/youtube_client.rs`）

| 処理 | 取得方法 |
|------|---------|
| チャンネルの最新動画一覧 | RSS フィード `https://www.youtube.com/feeds/videos.xml?channel_id=...`（正規表現でパース） |
| 動画情報 | Data API v3 `videos`（`part=snippet,contentDetails,liveStreamingDetails`） |
| コメント一覧 | Data API v3 `commentThreads`（`part=snippet`, `maxResults=100`, ページネーション） |
| チャンネルアイコン | Data API v3 `channels`（`part=snippet`、`thumbnails.default.url`） |

---

## 7. ワーカー・タスク・スケジューラ・イニシャライザ・メーラー

### 7.1 ワーカー（`src/app.rs::connect_workers` で登録）

| 名前 | 用途 |
|------|------|
| `SongItemsCreatorWorker` | セトリ自動作成パイプライン一式 |
| `SetlistFetchWorker` | 指定動画のセトリ再取得 |
| `CloudflareMailerWorker` | `mailer::Email` を Cloudflare Email Sending REST API 経由で送信（`src/mailers/cloudflare_worker.rs`） |

### 7.2 タスク（`src/app.rs::register_tasks` で登録）

| タスク名 | 用途 | 実装 |
|---------|------|------|
| `create_recent_videos_song_items` | `SongItemsCreatorWorker` を同期実行 | `src/tasks/create_recent_videos_song_items.rs` |
| `fetch_channel_icons` | `icon_url` が NULL のチャンネルについて YouTube API からアイコン URL を取得・保存 | `src/tasks/fetch_channel_icons.rs` |

実行例: `cargo loco task fetch_channel_icons`

### 7.3 スケジューラ

- `config/production.yaml` の `scheduler.jobs.create_recent_videos_song_items`（`schedule: "0 0 0 * * *"`、UTC = 09:00 JST）。
- スタンドアロン用 `config/scheduler.yaml`（`cargo loco scheduler --config config/scheduler.yaml`）。

### 7.4 イニシャライザ

- `ViewEngineInitializer`（`src/initializers/view_engine.rs`）— Tera ビューエンジンを構築。`assets/i18n` が存在すれば `fluent-templates` の `t` 関数を登録（`en-US` 既定、`assets/i18n/_shared.ftl` を共有リソースに）。

### 7.5 メーラー（`src/mailers/`）

| ファイル | 役割 |
|---------|------|
| `auth.rs` | `AuthMailer`。テンプレートのレンダリングと送信経路の振り分け |
| `cloudflare_client.rs` | `CloudflareEmailClient`。Cloudflare Email Sending REST API（`POST https://api.cloudflare.com/client/v4/accounts/{account_id}/email/sending/send`）へ `reqwest` で送信 |
| `cloudflare_worker.rs` | `CloudflareMailerWorker`。`BackgroundWorker<mailer::Email>` 実装。実行時に `CLOUDFLARE_ACCOUNT_ID` / `CLOUDFLARE_API_TOKEN` を読んでクライアントを生成 |

**メール種別とテンプレート**（`src/mailers/auth/<kind>/{subject,html,text}.t`）:

| メソッド | テンプレート | 宛先 | 内容 |
|---------|------------|------|------|
| `send_welcome` | `welcome` | 登録ユーザー | 日本語・英語併記のメールアドレス確認メール。件名「【Vtuber-Song.com】ご登録ありがとうございます / Welcome to Vtuber-Song.com」。リンクは `{domain}/api/auth/verify/{verifyToken}` |
| `forgot_password` | `forgot` | 対象ユーザー | パスワードリセット（Loco スターター既定の英文テンプレート） |
| `send_magic_link` | `magic_link` | 対象ユーザー | マジックリンク（同上）。`magic_link_token` が無い場合はエラー |
| `notify_admin_of_registration` | `admin_notification` | `ADMIN_NOTIFICATION_EMAIL` | 新規登録の通知（名前・メールアドレス）。env 未設定・空文字なら何もせず `Ok(())` |

**テンプレートのレンダリング**: Loco の `mail_template`（`include_dir!`）ではなく `include_str!` で各テンプレートを埋め込み、`tera::Tera::one_off` でレンダリングする。HTML のみ autoescape を有効にし、subject / text は無効（`render` / `render_email` ヘルパ）。

**送信経路の振り分け**（`AuthMailer::dispatch`）:

1. `AuthMailer::opts()` の `from` は `CLOUDFLARE_EMAIL_FROM`（未設定時は `mailer::DEFAULT_FROM_SENDER`）。
2. `CLOUDFLARE_API_TOKEN` が設定されていれば `CloudflareMailerWorker::perform_later` にキューイング（staging / production）。
3. 未設定なら Loco 組み込みの SMTP メーラー（`Self::mail`）にフォールバック。`development` / `test` 以外の環境では「送信されない可能性がある」旨を `tracing::error!` で警告する。

**メール内リンクの URL**: `ctx.config.server.full_url()`（`host:port`）ではなく環境変数 `APP_HOST` を直接参照する（`public_app_url()`、未設定時 `http://localhost:5150`）。Railway のエッジでは公開ドメインが 443 で TLS を終端し内部の別ポートへルーティングするため、`:{port}` を付けると壊れた URL になる（`fix_spec.md` の `fix/issue-163` 参照）。

---

## 8. 環境変数

コード / `config` で実際に参照しているもの。

| 変数名 | 参照箇所 | 説明 |
|-------|---------|------|
| `DATABASE_URL` | `config/*.yaml` | DB 接続 URI |
| `JWT_SECRET` | `config/production.yaml` | JWT 署名鍵（本番） |
| `GOOGLE_API_KEY` | ワーカー / `admin` コントローラ / `fetch_channel_icons` | YouTube Data API v3 キー |
| `OPENAI_API_KEY` | `SongItemsCreatorWorker` / `SetlistFetchWorker` | OpenAI API キー |
| `SPOTIFY_CLIENT_ID` / `SPOTIFY_CLIENT_SECRET` | 同上 | Spotify Client Credentials |
| `SLACK_WEBHOOK_URL` | `SongItemsCreatorWorker` | Slack 通知先（任意。未設定なら通知スキップ） |
| `PORT` | `config/production.yaml` | 待受ポート（既定 5150） |
| `APP_HOST` | `config/production.yaml` / `src/mailers/auth.rs::public_app_url` | メール内リンクの組み立てに使う公開 URL。**ポートを含めない**（例 `https://vtuber-song-list-staging.up.railway.app`）。既定 `http://localhost:5150` |
| `ADMIN_NOTIFICATION_EMAIL` | `AuthMailer::notify_admin_of_registration` | 新規会員登録の通知先（任意。未設定・空文字なら通知スキップ） |
| `CLOUDFLARE_API_TOKEN` | `AuthMailer::dispatch` / `CloudflareMailerWorker` | Cloudflare Email Sending の API トークン。**設定されているかどうかが送信経路の判定条件**（未設定なら SMTP フォールバック） |
| `CLOUDFLARE_ACCOUNT_ID` | `CloudflareMailerWorker` | Cloudflare アカウント ID（送信 API の URL に使用） |
| `CLOUDFLARE_EMAIL_FROM` | `AuthMailer::opts` | 送信元アドレス。Cloudflare Email Sending でオンボーディング済みのドメインであること |
| `FRONTEND_URL` | `config/production.yaml` | CORS の許可オリジン（既定 `http://localhost:3001`） |
| `DB_CONNECT_TIMEOUT` / `DB_IDLE_TIMEOUT` / `DB_MIN_CONNECTIONS` / `DB_MAX_CONNECTIONS` | `config/development.yaml` / `config/test.yaml` | コネクションプール設定 |
| `BUILD_SHA` / `GITHUB_SHA` | `src/app.rs::app_version`（コンパイル時 `option_env!`） | バージョン表示用 |
| `REDIS_URL` | CI のみ | CI のテストジョブで注入（`config` からの参照はなし） |

---

## 9. インフラ・デプロイ

### 9.1 Dockerfile（バックエンド）

マルチステージビルド。

| ステージ | 内容 |
|---------|------|
| builder | `rust:slim`。`pkg-config` / `libssl-dev` を導入し `cargo build --release` |
| runtime | `ubuntu:24.04`。`ca-certificates` / `libssl3` のみ。`vtuber_song_list-cli` バイナリ・`config/`・`assets/` をコピー |

- 公開ポート: 5150
- 起動コマンド: `./vtuber_song_list-cli start --environment production --all`（`--all` により Web サーバ + ワーカー + スケジューラを同一プロセスで起動）

### 9.2 docker-compose.yaml

ローカル開発用の PostgreSQL のみを定義。

| サービス | イメージ | 用途 |
|---------|---------|------|
| postgres | `postgres:17` | 開発 DB（`vtuber-song-list_development` / user・pass ともに `loco`）。ポート 5432、`postgres_data` ボリューム |

> アプリ / Redis / nginx / Next.js のサービス定義はない。

### 9.3 config（環境別）

| ファイル | 主な差分 |
|---------|---------|
| `config/development.yaml` | ポート 5150、`binding: localhost`、SMTP `localhost:1025`、`workers.mode: BackgroundAsync`、`auto_migrate: true`、JWT 秘密は直書き |
| `config/test.yaml` | `workers.mode: ForegroundBlocking`、`mailer.stub: true`、DB `vtuber-song-list_test`、`dangerously_truncate` / `dangerously_recreate: true` |
| `config/production.yaml` | `logger.format: json` / `level: info`、`PORT` / `APP_HOST` / `FRONTEND_URL` を env 化、CORS + static ミドルウェア、`scheduler:` を定義、DB は `DATABASE_URL`、JWT は `JWT_SECRET` |
| `config/scheduler.yaml` | スタンドアロンのスケジューラ実行用 |

### 9.4 フロントエンドのデプロイ

Cloudflare Workers（OpenNext アダプタ `@opennextjs/cloudflare`）。詳細は `frontend_spec.md`。

---

## 10. テスト

### 10.1 構成

- `tests/` 配下。`loco-rs` の testing ヘルパ（`request::<App, _, _>` 等）+ `insta` スナップショット。
- 実 PostgreSQL が必須（`DATABASE_URL`）。モックは使わない。`config/test.yaml` は起動時に DB を再作成・truncate する。
- スナップショットは `tests/**/snapshots/` に配置。
- `serial_test` の `#[serial]` でテストを直列化。

### 10.2 テストファイル

| ファイル | 対象 |
|---------|------|
| `tests/models/users.rs` | `create_with_password` / `find_by_email` / `find_by_pid` / バリデーション / 重複エラー（スナップショット） |
| `tests/requests/auth.rs` | `/api/auth/*`（登録・認証・ログイン・マジックリンク・リセット・再送）（スナップショット）。`ADMIN_NOTIFICATION_EMAIL` 設定時に welcome + 管理者通知の 2 通が送られることも検証 |
| `tests/requests/sessions.rs` | `/api/user`・`/api/session`（`name` 重複 / `email` 重複 / パスワード不一致の 400、登録時の welcome メール送信） |
| `tests/requests/channels.rs` | 公開フィルタ・ID 取得・404 |
| `tests/requests/song_diffs.rs` | 会員の pending 投稿・管理者の即時 approved・未認証拒否・一覧 |
| `tests/requests/admin/song_diffs.rs` | 承認・却下・非管理者拒否・pending 一覧 |
| `tests/requests/prepare_data.rs` / `session_prepare_data.rs` | テストデータ生成ヘルパ |
| `tests/tasks/mod.rs` / `tests/workers/mod.rs` | 空（プレースホルダ） |

ユニットテスト（`#[cfg(test)]` モジュール）:

| 対象 | 内容 |
|------|------|
| `src/mailers/auth.rs` | Tera テンプレートのレンダリング・HTML autoescape |
| `src/mailers/cloudflare_client.rs` | `parse_mailbox`（`Name <addr>` 形式 / 素のアドレス） |

### 10.3 実行

```bash
# 全テスト（PostgreSQL 必須）
DATABASE_URL=postgres://loco:loco@localhost:5432/vtuber-song-list_development \
REDIS_URL=redis://localhost:6379 \
cargo test --all-features --all

# 単体
cargo test <test_name>

# Lint / フォーマット
cargo clippy --all-features -- -D warnings -W clippy::pedantic -W clippy::nursery -W rust-2018-idioms
cargo fmt --all -- --check
```

---

## 11. CI/CD（GitHub Actions）

ワークフローは `.github/workflows/ci.yaml` の 1 本のみ。

**トリガー**: `push`（`master` / `main` / `staging`）・`pull_request`

| ジョブ | 依存 / 実行条件 | 内容 |
|-------|---------------|------|
| `changes` | - | `dorny/paths-filter@v3` で変更パスを判定し `backend` / `frontend` の 2 つの出力を返す |
| `rustfmt` | `changes.outputs.backend == 'true'` | `cargo fmt --all -- --check` |
| `clippy` | 同上 | `cargo clippy --all-features -- -D warnings -W clippy::pedantic -W clippy::nursery -W rust-2018-idioms` |
| `test` | 同上 | `postgres` + `redis` サービスを立て `cargo test --all-features --all`（`DATABASE_URL` / `REDIS_URL` を注入） |
| `frontend-lint` | `changes.outputs.frontend == 'true'` | `frontend/` で `pnpm lint` + `pnpm tsc --noEmit`（Node 22 / pnpm） |
| `frontend-test` | 同上 | `frontend/` で `pnpm test:ci` |
| `deploy` | `changes` / `frontend-lint` / `frontend-test` 成功後、`frontend == 'true'` かつ `push` かつ `main` / `staging` | Cloudflare Workers へデプロイ（`main` → `pnpm cf:deploy`、`staging` → `pnpm cf:deploy:staging`）。GitHub Environments（`... / production` / `... / staging`）と Secrets（`NEXT_PUBLIC_API_BASE_URL` / `NEXT_PUBLIC_APP_ENV` / `CLOUDFLARE_API_TOKEN` / `CLOUDFLARE_ACCOUNT_ID`）を使用 |

**パスフィルタの定義**:

| 出力 | 対象パス |
|------|---------|
| `backend` | `src/**` `migration/**` `tests/**` `config/**` `Cargo.toml` `Cargo.lock` `Dockerfile` `docker-compose.yaml` `.github/workflows/ci.yaml` |
| `frontend` | `frontend/**` `.github/workflows/ci.yaml` |

> バックエンド専用のデプロイジョブは CI には存在しない（`Dockerfile` によるコンテナ運用）。

---

## 12. 開発コマンド

```sh
cargo loco start                       # サーバ起動
cargo loco start --all                 # サーバ + ワーカー + スケジューラ
cargo loco task <name>                 # タスク実行
cargo loco generate migration <name>   # マイグレーション雛形生成
cargo loco db migrate                  # マイグレーション適用
```

新しいモデル / コントローラ / マイグレーションの追加手順は `CLAUDE.md` を参照。
