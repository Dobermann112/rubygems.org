# RubyGems.org

Rubyコミュニティのgemホスティングサービス — Ruby on Railsアプリ(内部名: `gemcutter`)。

## スタック

- Ruby 4.0.x (`.ruby-version`), Rails 8.1, Bundler/RubyGems 4.0.x
- PostgreSQL (>= 14), OpenSearch 2.13.0, Memcached — いずれもアプリ・テストの実行に必須
- システムテスト用にChrome + Playwright

## セットアップ

周辺サービスはデフォルトでDocker経由で起動します。Dockerが未インストールの場合は
[CONTRIBUTING.md](CONTRIBUTING.md#setting-up-the-environment)のネイティブインストール手順
(macOSはbrew、Debian/Ubuntuはapt)を参照してください。

```bash
docker compose up        # postgres, opensearch, memcachedを起動(アプリ本体は起動しない)
bin/setup                # 依存関係, db:prepare, db:seed, playwright, アセットビルド
```

## 開発用データ

`db:seed`以上の実データをシードする場合(開発用DBに対して):

```bash
# 匿名化された週次本番DBダンプ(-cでS3から最新版をダウンロード; DBをDROPして再作成)
script/load-pg-dump -c -d rubygems_development ~/Downloads/public_postgresql.tar

# pushパイプライン経由で.gemファイルをインポート("gem-author"ユーザーはdb:seedで作成済み)
bundle exec rake "gemcutter:import:process[vendor/cache,gem-author]"
bundle exec rake gemcutter:index:update   # インポート後にgem specsインデックス(compact index / specs.*.gz)を再構築
bundle exec rake searchkick:reindex CLASS=Rubygem   # OpenSearchの検索インデックスを再構築(gemインデックスとは別)
```

ダンプ: <https://rubygems.org/pages/data>。既にダウンロード済みのファイルを使う場合は`-c`を外してください。

## コマンド

```bash
bin/rails s            # アプリを:3000で起動
bin/rails test:all     # 全テスト(unit + system)
bin/rails test         # system以外のテスト
bin/rails test test/models/rubygem_test.rb:42   # 単一ファイル・単一行
bin/rails test -n /pattern/                     # 名前パターンに一致するテスト
bin/rails test:system  # systemテスト(Playwright/Chrome)
bin/ci             # ローカルでCIスイート全体を実行(config/ci.rb)
```

**`DB_HOST`について:**

- 標準的なインストール: ローカルのPostgresソケットがデフォルトで使われ、設定不要
- ネットワーク越しのホスト(例: dev container): 設定が必須。テスト環境ではdotenvが`.env.local`を
  除外するため、インラインで指定する:

```bash
DB_HOST=db rails test   # Postgresがローカルにない場合のみ
```

1Passwordにアクセスできるチームメンバーは、任意のコマンドの前に`script/dev`を付けることで
開発用シークレットを読み込めます — [CONTRIBUTING.md](CONTRIBUTING.md#developing-with-dev-secrets)を参照。

## Lint & セキュリティ(CIが落ちる原因になるため)

以下はすべて`bin/ci`の一部として実行されます。個別に試したい場合はこちらを使ってください。

```bash
bin/rubocop              # Rubyのスタイルチェック
bin/herb analyze         # ERBのlint
bin/prettier             # JSのスタイルチェック
rake format              # Ruby + JSを自動修正
bin/brakeman --quiet --no-pager --exit-on-warn --exit-on-error
bin/importmap audit      # JS依存関係の脆弱性監査
```

## アーキテクチャ (`app/`)

- **2つのインターフェース**: Webサイト(`controllers/`配下のHTMLコントローラー)とgemクライアント用
  API(`controllers/api/v1`、`controllers/api/v2`、および`bundle`/`gem`が取得する
  compact_indexエンドポイント`/versions`, `/info/:gem_name`, `/names`)
- `models/`, `controllers/`, `views/` — 標準的なRails構成
- `components/` — ViewComponentによるUIコンポーネント
- `policies/` — Punditによる認可
- `jobs/` — バックグラウンドジョブ(GoodJob + Shoryuken/SQS)
- `avo/` — Avo管理画面
- `tasks/` — maintenance_tasks(データマイグレーション)
- フロントエンド: Propshaft + importmap(JSバンドラーなし)、`app/javascript/controllers/`配下の
  Stimulusコントローラー、Tailwind CSS
- Flipperによるフィーチャーフラグ(`flipper-active_record`; UIは`/features`)
- 認証はClearance。gemファイルはS3(本番)または`server/`(開発)に保存
- `lib/compact_index*`, `lib/rstuf*`, `lib/gemcutter` — gemインデックス、TUF署名、コアドメイン

## テスト

- Minitestと`shoulda-context`(`class FooTest < ActiveSupport::TestCase`, `should`ブロック) + `shoulda-matchers`
- テストデータには`factory_bot`を使用(`create(:rubygem)`) — fixturesではない。ファクトリは`test/factories/`
- モックには`mocha`。systemテストはCapybara + Playwright
- `test/`は`app/`のディレクトリ構成をミラーする。**テストなしのコントリビューションは受け付けられません**

## ガードレール

- gemは**yank(取り下げ)されるのみで、物理削除はされない**: yankすると`Deletion`が作られ、
  バージョンの`indexed`が`false`になる
- 特に注意して変更すべきセキュリティ関連のサブシステム(変更には十分なテストが必要):
  - **認証・セッション** — Clearance (`app/models/user.rb`)
  - **多要素認証(MFA)** — TOTP + WebAuthn (`app/models/concerns/user_*_methods.rb`, `webauthn_*.rb`)
  - **APIキーとスコープ** — `api_key.rb`, `api_key_rubygem_scope.rb`(push/yank等の権限)
  - **Trusted publishing (OIDC)** — GitHub Actions等によるキーレスpublish(`app/models/oidc/**`); アクセスポリシーとトークン交換
  - **gem pushパイプライン** — `app/models/pusher.rb`(アップロードされたgemの検証と取り込み)
  - **所有権(Ownership)** — `app/models/ownership.rb`(誰がpush/yankできるか)
  - **証明・来歴(Attestations/provenance)** — `app/models/attestation.rb`, Sigstore証明書チェーン
  - **TUF署名** — `lib/rstuf*`
- シークレットや本番データは絶対にコミットしない。[開発用シークレットのワークフロー](CONTRIBUTING.md#developing-with-dev-secrets)を使うこと

## 規約

- masterは常にfast-forward可能な状態を保つこと。変更の際は必ずここからブランチを切る
- pushする前にローカルで`bin/ci`を実行すること — CIと同じ内容(lint, security, tests)を再現できる
- ユーザー向け文字列: `config/locales/en.yml`にキーを追加し、`bin/fill-locales`を実行して
  他のロケールに反映すること
