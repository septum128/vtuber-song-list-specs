# vtuber-song-list 設計書

[vtuber-song-list](https://github.com/septum128/vtuber-song-list) 本体の設計書・解析ドキュメントを管理するリポジトリです。

VTuber が歌枠配信で歌った楽曲をデータベース化・検索できるシステムで、YouTube のコメント欄から AI（OpenAI）を用いてセトリを自動抽出し、会員による修正提案・管理者の承認フローを備えます。

各ドキュメントは実装コードを正として書かれた解析結果であり、コードの変更に追従して更新する前提のものです。

## ドキュメント一覧

| ファイル | 内容 |
|---------|------|
| [backend_spec.md](./backend_spec.md) | バックエンド仕様書。技術スタック（Rust / Loco / Axum / SeaORM）、API エンドポイント、認証・認可、セトリ自動作成処理（YouTube RSS・Data API・OpenAI・Spotify 連携）、ワーカー・タスク・スケジューラ、インフラ・デプロイ、テスト、CI/CD |
| [frontend_spec.md](./frontend_spec.md) | フロントエンド解析レポート。プロジェクト構成（Next.js Pages Router）、技術スタック、ルーティング、状態管理、API 通信、データモデル、主要機能、ビルド・デプロイ（Cloudflare Workers / OpenNext）、テスト |
| [tables.md](./tables.md) | テーブル定義。`users` / `channels` / `videos` / `comments` / `song_items` / `song_diffs` / `favorites` の各カラム・インデックス・外部キー・Enum の詳細 |
| [fix_spec.md](./fix_spec.md) | 修正履歴。過去の不具合の症状・原因・修正内容を issue 単位で記録 |

## 読む順序の目安

1. `backend_spec.md` の「1. 概要」「2. 技術スタック」でシステム全体像を把握
2. `tables.md` でデータモデルを理解
3. `backend_spec.md` の API・認証・自動作成処理の詳細、または `frontend_spec.md` の画面・状態管理を必要に応じて参照
4. 過去の不具合調査時は `fix_spec.md` を確認

## 運用ルール

- 各仕様書は実装（`src/`・`migration/`・`config/`・`frontend/` 等）を正として記述する。実装と乖離した記述を見つけたら実装に合わせて更新する。
- 新たに調査・修正した内容は、恒久的な仕様なら該当する仕様書に、個別の不具合対応なら `fix_spec.md` に追記する。
