---
alwaysApply: true
---

# プロジェクト概要

README.md を読んで理解してください

企業のテックブログの更新をまとめてRSS配信するサイト。フィード一覧（`src/resources/feed-info-list.ts`）を元に、GitHub Actions が定期的に各社ブログをクロールし、まとめフィード（RSS/Atom/JSON）と静的サイト（Eleventy）を生成して GitHub Pages に公開する。

# コマンド

```bash
npm install                # 依存関係インストール（Node.js >= 24）

npm run feed-generate        # フィードを取得してまとめフィード（src/site/feeds, src/site/blog-feeds）を生成
npm run site-serve           # localhost:8080 でサイトを確認（feed-generate 後に実行）
npm run site-build           # Eleventy でサイトをビルド（public/ に出力）
npm run build                 # feed-generate + site-build

npm run lint                  # lint-biome + lint-ts + lint-secretlint をまとめて実行
npm run lint-biome            # biome check --write（フォーマット + lint、自動修正あり）
npm run lint-ts               # tsc --noEmit
npm run lint-secretlint       # secretlint '**/*'

npm run test                  # vitest run（tests/external 含む全テスト）
npm run test-internal         # tests/external を除いた通常テスト
npm run test-external         # 実際に外部フィードへアクセスするテスト（vitest.external.config.ts）
npm run test-coverage         # カバレッジ計測（tests/external を除く）

npx vitest run tests/feed-generator.test.ts   # 単一テストファイルのみ実行
npx vitest run -t "テスト名"                    # 名前でテストを絞って実行

npm run cache-prune           # 古いフィード/OGPキャッシュの削除
```

# アーキテクチャ

## フィード生成パイプライン（`src/cli/generate-feed-command.ts`）

`npm run feed-generate` が実行するメインフロー。`src/feed/` 配下のクラスを順番に呼び出す:

1. **`FeedCrawler`**（`feed-crawler.ts`） — `FEED_INFO_LIST` の全ブログの RSS/Atom を並列取得（`@supercharge/promise-pool`）。各記事の OGP 情報取得、はてなブックマーク数取得も行う。取得結果は `@11ty/eleventy-fetch` でキャッシュされる（`eleventy-cache-option.ts`）。
2. **`FeedGenerator`**（`feed-generator.ts`） — 取得した全記事を1つのまとめフィード（`feed` パッケージの `Feed` インスタンス）に変換し、Atom/RSS/JSON文字列を生成。記事タイトルは `記事タイトル | ブログ名` の形式にする。
3. **`FeedStorer`**（`feed-storer.ts`） — 生成したフィードを `src/site/feeds/`（配信用XML/JSON）、`src/site/blog-feeds/`（Eleventyのサイト生成用データ、`BlogFeed` 型）に書き出す。
4. **`FeedValidator`**（`feed-validator.ts`） — 生成したフィードの XML/構造をバリデーション。失敗時は例外を投げてワークフローを失敗させる。

`src/cli/prune-cache-command.ts`（`prune-cache.ts`）はキャッシュ削除、`src/cli/prepare-site-command.ts` はサイト生成前の画像プリフェッチ（必須ではない高速化用）、`src/cli/register-index-command.ts` は Google Indexing API への URL 登録を行う。

## サイト生成（Eleventy）

- `eleventy.config.ts` が設定エントリ。`src/site` が入力、`public` が出力。
- `src/site/_data/*.js` は Eleventy のグローバルデータで、実体は `src/site/_data/lib/*.ts`（テスト対象）を呼び出す薄いラッパー。`blog-feeds.ts` が `blog-feeds.json` を読み込み、`feed-items-chunks.ts` / `feed-items-hot.ts` がページング・人気記事抽出のロジックを持つ。
- テンプレートは `.11ty.ts`（TypeScriptで書かれ、JSテンプレートとして扱われる。`eleventyConfig.addExtension('11ty.ts', ...)`）。ロジックを持つコンポーネント（`src/site/_includes/components/*.ts`）はカバレッジ対象、それ以外のテンプレート/スタイル/スクリプトは対象外（`vitest.config.ts` の `coverage.exclude` 参照）。

## 設定値

`src/common/constants.ts` にサイトURL・フィード設定・並列数・キャッシュ有効期間などを集約。フォークして独自サイトを運用する場合はここと `feed-info-list.ts` を書き換える。

## テスト

- `tests/*.test.ts` — 通常のユニットテスト（外部通信なし、モック・fixture使用。`tests/helpers/site-data-fixtures.ts` 参照）。
- `tests/external/*.test.ts` — 実際に外部URLへリクエストする統合テスト。`vitest.external.config.ts` で別設定・別コマンド（`test-external`）から実行され、通常の `test`/`lint` では走らない。

# ルール

## フィード追加のルール

ユーザーにフィードを追加してと言われたら、以下の手順でフィードを追加してください

1. フィード追加の最低限の情報をユーザーに聞く
  - 企業名
  - RSSフィードURL
2. ブランチ作成
   `git checkout -b chore/new-feed-<企業名の英語>`
3. フィードを追加
   `src/resources/feed-info-list.ts` の `FEED_INFO_LIST` を更新
4. コミット
   `git commit -am 'chore(feed): <企業名など> 追加`
5. プッシュ
   `git push origin chore/new-feed-<企業名の英語>`
6. プルリクを作成
   pull_request_template.md を参考にプルリクを作成してください
   ghコマンドがあればそれを使用し、なければユーザーに作成方法を指示してください

すべての手順は都度ユーザーに同意を得て進めてください
