# 修正履歴

## fix/issue-13: セトリ削除時の JSON パースエラー

**症状:** セトリ管理画面でセトリを削除すると「Failed to execute 'json' on 'Response': Unexpected end of JSON input」が表示される。

**原因:** バックエンドの削除エンドポイントが `format::empty()` で HTTP 200 + 空ボディを返していたが、フロントエンドの `apiFetch` は HTTP 204 のみ空レスポンスとして扱い、それ以外は `res.json()` を呼び出すため空ボディでパースエラーになっていた。

**修正内容:**

- `src/controllers/admin/song_items.rs`: 削除成功時のレスポンスを `format::empty()` (HTTP 200) から `StatusCode::NO_CONTENT` (HTTP 204) に変更
- `src/controllers/admin/channels.rs`: 同上
- `frontend/src/utils/api.ts`: `res.json()` を直接呼ぶのではなく `res.text()` でボディを取得し、空なら `undefined` を返す方式に変更
- `frontend/src/utils/__tests__/api.test.ts`: モックを `json()` から `text()` を返す形式に更新
- `.gitignore`: `frontend/tsconfig.tsbuildinfo`（TypeScript インクリメンタルビルドキャッシュ）を追加し、トラッキングからも除外

---

## fix/issue-16: モバイルのハンバーガーメニューが動作しない

**症状:** モバイル表示でハンバーガーメニューボタンを押してもメニューが開かない。

**原因:** Navbar の `data-bs-toggle="collapse"` は Bootstrap JS を必要とするが、`_app.tsx` に Bootstrap JS の読み込みがなく、CSS のみが読み込まれていた。

**修正内容:**

- `frontend/src/pages/_app.tsx`: `useEffect` で `import("bootstrap")` を追加し、クライアントサイドでのみ Bootstrap JS を動的インポートするよう修正
- `frontend/src/types/bootstrap.d.ts`: Bootstrap 5 には型定義が同梱されていないため、モジュール宣言ファイルを新規作成

---

## fix/pnpm-ignored-builds: pnpm v10 でネイティブモジュールのビルドスクリプトが実行されない

**症状:** pnpm v10 環境で `pnpm install` を実行すると、`@parcel/watcher` / `sharp` / `unrs-resolver` のネイティブモジュールのビルドスクリプトが実行されずインストールが不完全になる。

**原因:** pnpm v10 はセキュリティ強化のためデフォルトで依存パッケージのビルドスクリプト実行をブロックするようになり、`package.json` 側で明示的な許可リストが必要だった。

**修正内容:**

- `frontend/package.json`: `pnpm.onlyBuiltDependencies` に `["@parcel/watcher", "sharp", "unrs-resolver"]` を追加し、該当パッケージのビルドスクリプト実行を明示的に許可。

---

## fix/issue-69: 動画管理の配信日が UTC 基準でずれる

**症状:** 動画編集フォームの「配信日」が実際の配信日時（JST）より 9 時間前の値で表示され、そのまま保存すると配信日がさらにずれていく。

**原因:** `published_at`（DB は UTC の `timestamptz`）を `slice(0, 16)` でそのまま `datetime-local` 入力に流し込み、送信時も `new Date(value).toISOString()` でローカルタイムゾーン依存の変換をしていた。

**修正内容:**

- `frontend/src/containers/Admin/VideoForm.tsx`: `toJstDatetimeLocal()` / `fromJstDatetimeLocalToUtc()` を追加。DB の UTC 値に +9h して JST として表示し、入力された JST 値を `${value}:00Z` として解釈してから −9h し UTC の ISO 文字列で API へ送信するよう変更。

---

## fix/86-pnpm-lockfile-outdated: pnpm-lock.yaml の specifier が package.json とズレて Vercel ビルドが失敗する

**症状:** Vercel でのフロントエンドビルドが `ERR_PNPM_OUTDATED_LOCKFILE` で失敗する。

**原因:** `react` / `sass` / `styled-components` / `swr` / `@testing-library/*` / `@types/*` / `@typescript-eslint/*` / `eslint` / `eslint-config-next` / `typescript` など 17 個の依存関係で、`pnpm-lock.yaml` の specifier が `package.json` に記載のバージョン範囲とズレており、`--frozen-lockfile` 相当のインストールが拒否されていた。

**修正内容:**

- `frontend/pnpm-lock.yaml`: `pnpm install` で再生成し `package.json` と同期（解決される依存バージョン自体への影響はなし）。

---

## hotfix/89-cf-deploy-api-base-url-secret: Cloudflare Workers デプロイ時に本番 API が全滅する

**症状:** `main` マージによる Cloudflare Workers への自動デプロイ後、本番サイトの `/api/*` へのリクエストが全て 500 エラーになる。

**原因:** `next.config.ts` の `/api/*` リライトは `NEXT_PUBLIC_API_BASE_URL` 未設定時に `http://localhost:5150` へフォールバックする。CI の `deploy` ジョブがビルド時にこの環境変数を渡していなかったため、本番 Worker に `localhost` 宛のリライト設定がビルドインされてしまっていた（発覚時は手動デプロイで応急復旧）。

**修正内容:**

- `.github/workflows/ci.yaml`: `deploy` ジョブに `NEXT_PUBLIC_API_BASE_URL`（GitHub Secrets 経由）と `NEXT_PUBLIC_APP_ENV: production` を環境変数として追加。本番バックエンド URL をリポジトリに平文で残さないため Secrets 経由で注入する。

---

## fix/allow-cloudflare-web-analytics: Cloudflare Web Analytics のビーコンが CSP でブロックされる

**症状:** Cloudflare Web Analytics を自動挿入から JS スニペットの手動インストールに切り替えたところ、ビーコンスクリプトの読み込みと RUM データ送信が Content-Security-Policy でブロックされる。

**原因:** `next.config.ts` の CSP がビーコン配信元（`static.cloudflareinsights.com`）と RUM 送信先（`cloudflareinsights.com`）を許可していなかった。手動インストールでは RUM ビーコンが同一オリジンの `/cdn-cgi/rum` ではなく `https://cloudflareinsights.com/cdn-cgi/rum` へ直接送信されるため、`connect-src` の見直しも必要だった。

**修正内容:**

- `frontend/src/pages/_document.tsx`: `<body>` 末尾に `https://static.cloudflareinsights.com/beacon.min.js`（`data-cf-beacon` 付き）の `<script>` を追加。
- `frontend/next.config.ts`: CSP の `script-src` に `static.cloudflareinsights.com` を追加（PR #94）、続けて `connect-src` に `cloudflareinsights.com` を追加（PR #96）。

---

## fix/issue-98: RSS 取り込み時の published_at の採用元が調査できない

**症状:** RSS 経由で取り込んだ動画の `published_at` が想定と異なる値になっても、`videos.response_json` が空 `{}` 固定のため DB だけでは原因を追えない。

**原因:** ライブ配信では YouTube API の `actualStartTime` を、取得できない場合は RSS の `<published>` を `published_at` に使うが、どちらを採用したかを記録していなかった。

**修正内容:**

- `src/workers/song_items_creator.rs`: `build_published_at()` を追加。`actualStartTime` が無く RSS 日時にフォールバックした場合は `tracing::warn!` でログ出力し、`response_json` に `{ "published_at_source", "actual_start_time", "rss_published" }` を記録するよう変更。

---

## fix/issue-119: セトリ取得処理で複数コメント由来の曲重複インサート

**症状:** 同一動画に複数のセトリ判定コメントがある場合、同じ曲が重複して `song_items` に登録される。

**原因:** `process_song_items_for_video` がコメントごとに抽出したエントリをその場で即座にインサートしており、動画全体を通しての重複チェックをしていなかった。

**修正内容:**

- `src/workers/song_items_creator.rs`: 各コメントから抽出したエントリを `PendingSongEntry`（comment_id, time, title, author）として動画単位で一旦集約するよう変更。抽出済みのコメントはこの時点で `comment.status=completed` にする。
- `dedup_song_entries()` を追加: title/author が空のエントリを除外 → time/title/author 順にソート → `normalize_for_dedup()`（前後空白除去・全角スペースの半角統一・連続空白圧縮・小文字化）した title+author の組で重複除去。
- 重複排除後のエントリのみを `song_item` + `song_diff`（kind=auto, status=approved）として作成。

---

## fix/issue-134: 動画管理一覧の「歌枠のみ」チェックがセトリ画面から戻ると外れる

**症状:** 動画管理一覧で「歌枠のみ」をONにしても、「セトリ」リンクから遷移して戻ると外れている。

**原因:** `VideoList` の `onlySongLives` はコンポーネントローカルな `useState(false)` のみで永続化されておらず、ページ遷移で再マウントされると初期値に戻っていた。

**修正内容:**

- `frontend/src/utils/storage.ts`: `getAdminVideoListOnlySongLives` / `setAdminVideoListOnlySongLives`（`localStorage` キー `admin_video_list_only_song_lives`）を追加。既存の表示件数（`admin_video_list_per_page`）と同じ永続化パターン。
- `frontend/src/containers/Admin/VideoList.tsx`: マウント時に保存値を読み込み、チェック変更時に保存するよう変更。

---

## fix/issue-135: next dev環境でCSPがHMRのevalをブロックし白画面になる

**症状:** `pnpm dev`（`next dev`）でフロントエンドを起動すると、どのページも白画面になる。

**原因:** `next.config.ts` の `headers()` が全ルートに付与するCSPの `script-src` に `unsafe-eval` が含まれておらず、Next.jsの開発モード（React Refresh / HMRランタイム）が内部で使う `eval`/`new Function` がブロックされ、以降のJS実行が停止していた。本番ビルドはHMRランタイムを含まないため影響しない。

**修正内容:**

- `frontend/next.config.ts`: `headers()` を `process.env.NODE_ENV !== "production"` のとき空配列を返すように変更し、CSP・`Permissions-Policy` を本番ビルドのみに限定。本番ビルドのCSP内容自体は変更なし。

---

## fix/issue-163: ウェルカムメールが英語のLocoスターター既定文面で、確認リンクのURLも壊れている

**症状:** 会員登録時に届くウェルカムメールが「Welcome to Loco!」という英文のスターター既定文面のまま。さらに本文中のメールアドレス確認リンクが `https://example.com:5150/api/auth/verify/...` のようにポート番号付きで組み立てられ、クリックしても到達できない。

**原因:**

- 文面は Loco SaaS スターターのテンプレート（`src/mailers/auth/welcome/{subject,html,text}.t`）を未編集のまま使っていた。
- URL は `ctx.config.server.full_url()` を使っており、これは `{host}:{port}` を連結する。Railway のエッジでは公開ドメインが 443 で TLS を終端し、内部的には別のコンテナポートへルーティングされるため、公開 URL に `:{port}` を付けると壊れる。

**修正内容:**

- `src/mailers/auth/welcome/{subject,html,text}.t`: 日本語・英語併記の Vtuber-Song.com 文面に差し替え。件名は「【Vtuber-Song.com】ご登録ありがとうございます / Welcome to Vtuber-Song.com」。
- `src/mailers/auth.rs`: `public_app_url()` を追加し、環境変数 `APP_HOST`（未設定時 `http://localhost:5150`）を直接参照するよう変更。welcome / forgot / magic link の全テンプレートに渡す `domain` / `host` をこの値に統一。`APP_HOST` にはポート無しの公開 URL を設定する。

---

## fix/issue-166: メールHTMLテンプレート先頭の余分な `;` が本文に混入する

**症状:** 送信されたメールの HTML 本文が `;<html>` で始まり、メールクライアント上で先頭に `;` が表示される。

**原因:** Loco SaaS スターターが生成した `html.t` 3 ファイル（`welcome` / `forgot` / `magic_link`）の 1 行目が `;<html>` になっており、テンプレートの一部としてそのまま出力されていた。

**修正内容:**

- `src/mailers/auth/{welcome,forgot,magic_link}/html.t`: 先頭の `;` を削除（`;<html>` → `<html>`）。

---

## fix/issue-195: loco 1.2 アップグレード後、既存 DB で INT4 の主キーを読めず 500 になる

**症状:** PR #194（`loco-rs` 0.16 → 1.2 / SeaORM 2.0 へのアップグレード）を staging にマージした後、`/api/channels` と `/api/user` が 500 を返す。ログには `error occurred while decoding column "id": mismatched types; Rust type Option<i64> (as SQL type INT8) is not compatible with SQL type INT4` が出力される。

**原因:** マイグレーションのヘルパ `ColType::PkAuto` が作る列の型が、Loco 0.16 では `integer`、Loco 1.x では `bigint` に変わり、エンティティの主キーも `i64` になった。sqlx 0.9 は `INT4` を `i64` にデコードできないが、staging / production の DB は 0.16 時代に作られたため主キーが `integer` のままだった。テストはマイグレーションから新規作成したスキーマ（`bigint`）で動くため検出できなかった。

**修正内容:**

- `migration/src/m20261023_094647_widen_primary_keys_to_bigint.rs` を追加
  - `users` / `channels` / `videos` / `comments` / `song_items` / `song_diffs` / `favorites` の `id` を `ALTER COLUMN ... TYPE bigint`
  - `pg_get_serial_sequence` で取得した連番シーケンスを `ALTER SEQUENCE ... AS bigint`（`serial` 由来の `AS integer` のままだと 2^31-1 で枯渇するため）
  - `song_diffs.id` を参照する `song_items.latest_diff_id` も `bigint` に変更
  - PostgreSQL 以外（SQLite）では何もしない。既に `bigint` の DB では実質 no-op。`down` は何もしない（`integer` に戻すと `i64` のエンティティが壊れるため）
- エンティティ・モデル・コントローラ・ワーカーの主キー / `latest_diff_id` / パスパラメータ / `video_ids` 等を `i64` に統一し、`i32` ⇔ `i64` の変換（`i64::from` / `as i32` キャスト）を削除
- 運用上の注意: staging は `auto_migrate: true` のためデプロイ時に自動適用される。`ALTER COLUMN TYPE` はテーブルを書き直す間ロックを取る
