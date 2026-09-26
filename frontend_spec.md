# Frontend 解析レポート

`frontend/` の実装（`package.json`・`src/`・`next.config.ts`・`wrangler.jsonc`）を正とする。

## 1. プロジェクト構成

```
frontend/
├── src/
│   ├── components/        # 再利用可能な UI コンポーネント
│   │   ├── Common/        # Alert / Footer / Head / Loading / Modal
│   │   ├── Navbar/        # ナビゲーション（Navbar.tsx + .module.scss）
│   │   ├── SongLists/     # 曲・動画・チャンネル・修正関連の表示コンポーネント
│   │   └── __tests__/     # Pagination / SongDiffList のテスト
│   ├── containers/        # ページ固有のロジックを持つコンテナ
│   │   ├── Admin/         # 管理画面（チャンネル / 動画 / セトリ / 修正承認 / 各種フォーム）
│   │   ├── Channels/      # チャンネル一覧・詳細
│   │   ├── Favorites/     # お気に入り一覧
│   │   ├── Session/       # ログインフォーム
│   │   └── User/          # ユーザー登録フォーム
│   ├── context/           # グローバル状態（Context API）
│   │   ├── AlertsProvider.tsx
│   │   └── ThemeProvider.tsx
│   ├── hooks/             # SWR ベースのデータ取得・操作フック
│   ├── layouts/           # DefaultLayout / AdminLayout
│   ├── pages/             # Next.js Pages Router
│   ├── resources/         # 型定義（types.ts）・Enum（enums.ts）
│   ├── styles/            # globals.scss / variables.scss
│   ├── types/             # bootstrap.d.ts（型定義補完）
│   └── utils/             # api.ts（fetch ラッパ）/ storage.ts / videoExport.ts
├── next.config.ts
├── wrangler.jsonc         # Cloudflare Workers 設定
├── open-next.config.ts    # OpenNext アダプタ設定
├── jest.config.ts / jest.setup.ts
└── eslint.config.mjs
```

---

## 2. 技術スタック

| カテゴリ | ライブラリ | `package.json` の指定 |
|---|---|---|
| フレームワーク | Next.js（Pages Router） | `^15.5.21` |
| UI | React / React DOM | `^19.0.0` |
| 言語 | TypeScript | `^5.7.0` |
| データフェッチ | SWR | `^2.3.0` |
| フォーム | React Hook Form | `^7.54.0` |
| CSS フレームワーク | Bootstrap（SCSS 直接インポート） | `^5.3.8` |
| CSS-in-JS | styled-components | `^6.1.19` |
| SCSS | sass | `^1.85.0` |
| 日付 | date-fns | `^4.1.0` |
| テスト | Jest / Testing Library | `jest ^29.7.0` / `@testing-library/react ^16.3.0` |
| デプロイアダプタ | `@opennextjs/cloudflare` | `^1.20.2` |
| Cloudflare CLI | wrangler | `^4.127.0` |
| パッケージマネージャ | pnpm | `pnpm@10.32.1`（`packageManager`） |

---

## 3. ルーティング構成（`src/pages/`）

| ルート | ファイル | 説明 |
|---|---|---|
| `/` | `index.tsx` | ホーム（説明テキスト。`NEXT_PUBLIC_APP_ENV === "staging"` で見出しを追加表示） |
| `/channels` | `channels/index.tsx` | チャンネル一覧 |
| `/channels/[id]` | `channels/[id].tsx` | チャンネル詳細（「曲一覧」/「動画一覧」タブ、検索・ページングを URL クエリで保持） |
| `/favorites` | `favorites.tsx` | お気に入り一覧（未ログインは `/session/new` へリダイレクト） |
| `/session/new` | `session/new.tsx` | ログイン（ログイン済みは `/` へ） |
| `/user/new` | `user/new.tsx` | ユーザー登録（ログイン済みは `/` へ） |
| `/admin` | `admin/index.tsx` | `/admin/song_diffs` へリダイレクト |
| `/admin/song_diffs` | `admin/song_diffs.tsx` | 修正承認（承認待ち / 承認済み / 却下 / すべて で絞込） |
| `/admin/channels` | `admin/channels/index.tsx` | チャンネル管理（一覧・編集・削除） |
| `/admin/channels/new` | `admin/channels/new.tsx` | チャンネル新規作成 |
| `/admin/videos` | `admin/videos/index.tsx` | 動画管理（一覧・編集・追加・TSV 一括登録・セトリ取得・エクスポート） |
| `/admin/videos/[id]/song_items` | `admin/videos/[id]/song_items.tsx` | セトリ管理（一覧・作成・一括インポート・削除） |
| `/404`, `/500` | `404.tsx`, `500.tsx` | エラーページ |

`/admin/*` は `AdminLayout` が `user.kind !== 10`（`UserKind.ADMIN`）のとき `/` へリダイレクトする。

> 旧仕様にあった `/about` `/inquiry` `/maintenance` は存在しない。問い合わせ導線は `/`（ホーム）のテキストリンク（Twitter / メール / GitHub Issue）。

---

## 4. 状態管理

### グローバル状態（Context API）

**ThemeProvider**（`src/context/ThemeProvider.tsx`）
- ライト / ダークの 2 値。既定は `light`。
- `localStorage`（キー `theme`）に永続化し、`document.documentElement` の `data-bs-theme` 属性を切り替える。
- `useTheme()` を提供。

**AlertsProvider**（`src/context/AlertsProvider.tsx`）
- 通知の種別は `danger` / `notice` / `success`。
- 追加から 3 秒（`AUTO_DISMISS_MS = 3000`）で自動消去。
- Next.js Router の `routeChangeStart` で全消去。
- `useAlerts()`（`alerts` / `addAlert` / `removeAlert` / `clearAlerts`）を提供。

### サーバーデータ（SWR）

- 全データ取得は SWR。フックは `src/hooks/` に集約。
- 認証必須のフックは SWR キーを `[url, token]` のタプルにし、トークンがなければ `null` キーで停止。
- 変更操作は `useSWRConfig().mutate` で関連キャッシュを再検証（キーの一部一致で一括 mutate するものもある）。
- `revalidateOnFocus: false` を多くのフックで指定。

### URL クエリ

- チャンネル詳細のタブ・ページ・検索条件（`tab` / `page` / `vpage` / `q` / `since` / `until` / `video_title`）を `router.push(..., { shallow: true })` で URL に保持。

### 認証トークン

- `localStorage`（キー `auth_token`）に保存（`src/utils/storage.ts`）。SSR 環境では `null` を返す。

---

## 5. API 通信

### 5.1 fetch ラッパ（`src/utils/api.ts`）

- `apiFetch<T>(path, { auth?, ...RequestInit })`。
- `auth: true` のとき `Authorization: Bearer <token>` を付与（トークンは `localStorage`）。
- `Content-Type: application/json` を既定付与。
- 非 2xx はボディ（`{ message } | { description }`）からメッセージを取り出して `Error` を throw。
- レスポンスボディは `res.text()` で受け、空文字なら `undefined` を返す（204 / 空ボディ対策。`fix_spec.md` の `fix/issue-13` 参照）。
- `buildQuery(params)` で `undefined` / 空文字を除外したクエリ文字列を生成。

### 5.2 使用エンドポイント

| エンドポイント | メソッド | 使用フック |
|---|---|---|
| `/api/channels` `/api/channels/{id}` | GET | `useChannels` / `useChannel` |
| `/api/videos` `/api/videos/{id}` | GET | `useVideos` / `useVideo` |
| `/api/song_items` `/api/song_items/{id}` | GET | `useSongItems` / `useSongItem` |
| `/api/user` | GET / POST | `useAuth`（現在ユーザー・登録） |
| `/api/session` | POST / DELETE | `useAuth`（ログイン・ログアウト） |
| `/api/member/song_items/{id}/song_diffs` | GET / POST | `useSongDiffs` / `useCreateSongDiff` |
| `/api/member/favorites` `/api/member/favorites/ids` `/api/member/favorites/{id}` | GET / POST / DELETE | `useFavorites`（`useFavoriteIds` / `useFavoriteList` / `useToggleFavorite`） |
| `/api/admin/channels` `/api/admin/channels/{id}` | GET / POST / PATCH / DELETE | `useAdminChannels` / `useAdminChannelActions` |
| `/api/admin/videos` `/api/admin/videos/{id}` `/api/admin/videos/bulk` `/api/admin/videos/{id}/fetch_setlist` `/api/admin/videos/bulk_fetch_setlist` `/api/admin/videos/bulk_publish` | GET / POST / PATCH | `useAdminVideos` / `useAdminVideoActions` |
| `/api/admin/song_items` `/api/admin/song_items/bulk` `/api/admin/song_items/{id}` | GET / POST / DELETE | `useAdminSongItems` / `useAdminSongItemActions` |
| `/api/admin/song_diffs` `/api/admin/song_diffs/{id}/approve` `/api/admin/song_diffs/{id}/reject` | GET / PATCH | `useAdminSongDiffs` / `useAdminSongDiffActions` |

### 5.3 通信方式

- 認証は **JWT を `localStorage` に保持し `Authorization` ヘッダで送信**（Cookie / CSRF トークンは使用しない）。
- `next.config.ts` の `rewrites()` で `/api/:path*` を `NEXT_PUBLIC_API_BASE_URL`（未設定時 `http://localhost:5150`）へリライト。

---

## 6. データモデル（`src/resources/types.ts`）

```typescript
ChannelType  = { id, channel_id, name, custom_name, twitter_id, icon_url, kind, status }
VideoType    = { id, channel_id, video_id, title, kind, status, published, published_at }
SongItemType = {
  id, video_id, latest_diff_id,
  diff:  { title, author, time },
  video: { id, video_id, title, channel_id, channel_custom_name, kind, published_at }
}
SongDiffType = { id, song_item_id, made_by_id, time, title, author, status, kind, created_at }
UserType     = { id, name, kind }
AuthResponse = { message, user: UserType, token }
```

`src/resources/enums.ts` に `UserKind` / `ChannelKind` / `VideoKind` / `VideoStatus` / `SongDiffStatus` / `SongDiffKind` を定義（値はバックエンドの Enum と対応。詳細は `tables.md`）。

---

## 7. 主要機能

### ユーザー向け

- チャンネル一覧・詳細の閲覧。
- チャンネル詳細で「曲一覧」（曲名・アーティスト・動画タイトル・日付でフィルタ、YouTube のタイムスタンプ付きリンク）と「動画一覧」（歌枠のみ）をタブ切替。
- ユーザー登録（`name` / パスワード）・ログイン。
- ログインユーザーは曲のお気に入り登録（ハートボタン）と、お気に入り一覧ページの閲覧。
- ログインユーザーは各曲の「修正を提案」モーダルから修正履歴の閲覧と `song_diff` の投稿。
- ライト / ダークテーマ切替（Navbar のボタン）。

### 管理者向け（`kind = 10`）

- **修正承認**: `song_diffs` の承認 / 却下、ステータス絞込。
- **チャンネル管理**: 作成（YouTube チャンネル ID）・編集（表示名 / Twitter / 公開区分 / アイコン再取得）・削除。
- **動画管理**:
  - 個別追加（YouTube URL または動画 ID）、TSV 一括登録。
  - チャンネル絞込・「歌枠のみ」フィルタ・表示件数（30 / 50 / 100 件、既定 30）・ページング。「歌枠のみ」と表示件数は `localStorage`（`admin_video_list_only_song_lives` / `admin_video_list_per_page`）に永続化し、他ページから戻っても状態を維持（`fix_spec.md` の `fix/issue-134` 参照）。
  - 動画ごとの「セトリ取得」/「再取得（force）」、複数選択しての一括セトリ取得。
  - 複数選択しての**一括公開 / 一括非公開**（`published` のみ更新、`published_at` は変更しない）。
  - **エクスポート機能**（`src/utils/videoExport.ts`）: 選択した動画を「タイトル`<TAB>`URL」の行に整形し、テキストファイルのダウンロード（`videos_YYYYMMDD_HHMMSS.txt`）またはクリップボードへのコピー。ページを跨いだ選択を `Map<id, VideoType>` で保持。
  - 動画編集時、配信日は JST 基準で表示・入力し UTC に変換して送信（`fix_spec.md` の `fix/issue-69` 参照）。
- **セトリ管理**: 動画単位の `song_item` 一覧・個別作成・一括インポート・削除。

---

## 8. ビルド・デプロイ設定

### scripts（`package.json`）

```json
"dev":              "next dev -p 3001"
"build":            "next build"
"start":            "next start -p 4400"
"lint":             "eslint src --max-warnings 0"
"test":             "jest"
"test:ci":          "jest --ci --coverage"
"cf:preview":       "opennextjs-cloudflare build && opennextjs-cloudflare preview"
"cf:deploy":        "opennextjs-cloudflare build && opennextjs-cloudflare deploy"
"cf:deploy:staging":"opennextjs-cloudflare build -e staging && opennextjs-cloudflare deploy -e staging"
```

### Cloudflare Workers（`wrangler.jsonc` / `open-next.config.ts`）

- `@opennextjs/cloudflare`（OpenNext アダプタ）で Next.js を Cloudflare Workers 向けにビルド。
- `main: ".open-next/worker.js"`、`assets` バインディング `ASSETS`（`.open-next/assets`）。
- `compatibility_flags: ["nodejs_compat", "global_fetch_strictly_public"]`、`preview_urls: true`。
- 環境 `staging`（Worker 名 `vtuber-song-list-frontend-staging`）。本番 Worker 名 `vtuber-song-list-frontend`。
- CI の `deploy` ジョブが `main` → `pnpm cf:deploy`、`staging` → `pnpm cf:deploy:staging` を実行。

### `next.config.ts`

- `compiler.styledComponents: true`。
- `images.unoptimized: true`（Workers ランタイムでは `sharp` が使えないため最適化を無効化）。`remotePatterns` に `img.youtube.com` / `yt3.googleusercontent.com` / `yt3.ggpht.com`。
- `sassOptions.quietDeps: true`（Bootstrap の `@import` 由来の警告抑制）。
- `headers()` で全パスに CSP と `Permissions-Policy` を付与。CSP の `script-src` / `connect-src` は Cloudflare Web Analytics（`static.cloudflareinsights.com` / `cloudflareinsights.com`）を許可（`fix_spec.md` 参照）。`NODE_ENV !== "production"`（`next dev`）では CSP を付与しない（`script-src` に `unsafe-eval` がなく React Refresh の `eval` 実行がブロックされ白画面になるため。`fix_spec.md` の `fix/issue-135` 参照）。本番ビルドの CSP 内容は変更なし。
- `rewrites()` で `/api/:path*` → `${NEXT_PUBLIC_API_BASE_URL}/api/:path*`。

### 環境変数

| 変数名 | 用途 |
|---|---|
| `NEXT_PUBLIC_API_BASE_URL` | API リライト先のバックエンド URL |
| `NEXT_PUBLIC_APP_ENV` | `"staging"` のときホームに Staging 表示 |
| `CLOUDFLARE_API_TOKEN` / `CLOUDFLARE_ACCOUNT_ID` | デプロイ認証（CI Secrets） |

### スタイリング

- `src/pages/_app.tsx` で `@/styles/globals.scss`（Bootstrap 5 SCSS を含む）をグローバルインポート。
- クライアントで `import("bootstrap")` を動的読み込み（ハンバーガーメニュー等の JS 用。`fix_spec.md` の `fix/issue-16` 参照）。
- コンポーネント固有スタイルは SCSS モジュール（例 `Navbar.module.scss`）。
- ダークモードは Bootstrap の `data-bs-theme` 属性。
- フォントは `next/font/google` の Noto Sans JP（weight 400 / 700）。
- `src/pages/_document.tsx` で Cloudflare Web Analytics のビーコンスクリプトを読み込み。

---

## 9. テスト

- Jest（`testEnvironment: jsdom`）+ Testing Library。設定は `jest.config.ts`、セットアップは `jest.setup.ts`（`@testing-library/jest-dom`）。
- `moduleNameMapper` で `@/` エイリアスと SCSS（`identity-obj-proxy`）を解決。TS は `babel-jest` + `next/babel`。
- テストファイル:
  - `src/components/__tests__/Pagination.test.tsx`
  - `src/components/__tests__/SongDiffList.test.tsx`
  - `src/utils/__tests__/api.test.ts`
  - `src/utils/__tests__/storage.test.ts`
  - `src/utils/__tests__/videoExport.test.ts`
- Lint は ESLint 9（`eslint.config.mjs`、`next/core-web-vitals` + `next/typescript`）。CI では `pnpm lint` に加え `pnpm tsc --noEmit` で型チェック。
