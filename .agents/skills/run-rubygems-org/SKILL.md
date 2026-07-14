---
name: run-rubygems-org
description: Run rubygems.org locally — boot the Rails server, hit it with curl, screenshot pages via bin/playwright, or invoke internal code with bin/rails runner. Use when asked to run, start, boot, smoke-test, screenshot, or poke models/jobs/controllers in the rubygems.org / gemcutter app.
---

# rubygems.orgを実行する

Rails 8アプリ(内部名: `gemcutter`)で、Postgres・OpenSearch・Memcachedが`127.0.0.1:5432/9200/11211`で
到達可能である必要があります。`smoke.sh`はこれらのポートをプローブし、周辺サービスがDocker・ネイティブ
インストール・ホスト上で既に起動済みのいずれであっても動作します。それ以外(Ruby 4, Puma, headless Chrome)は
すべてホスト上で実行されます。

**エージェントが使うべきパスは`./.agents/skills/run-rubygems-org/smoke.sh`です** — このスクリプトが
周辺サービスを起動し、Railsを:3000で(未起動なら)起動し、主要4エンドポイントにcurlし、Chromiumバイナリが
利用可能であれば`tmp/run-skill/`に3枚のPNGスクリーンショットを書き出します。

このドキュメント内のパスはすべてリポジトリルートからの相対パスです。

> 正規の設置場所: `.agents/skills/run-rubygems-org/` — Codex CLI, OpenCode, Gemini CLIが採用している
> ツール横断のskill規約です。Claude Codeは`.claude/skills/run-rubygems-org/`のシンボリックリンク経由で
> これを読み込みます。
>
> **`smoke.sh`が失敗したら、自己判断で対処する前に下記「このskillが失敗したとき」を確認してください** —
> 大抵の場合、[`LEARNINGS.md`](LEARNINGS.md)に解決策が載っています。

## 前提条件(初回のみ)

Ruby 4(`.ruby-version`に基づき`mise`/`asdf`経由で導入)と、`127.0.0.1:5432/9200/11211`で到達可能な
Postgres + OpenSearch + Memcachedが必要です。`smoke.sh`はこれらが*どうやって*動いているか(Docker、
brew services、apt、もしくは既にホスト上で稼働中)は問わず、ポートをプローブするだけです。

まっさらな環境の場合、好きな方を選んでください。

```bash
# パターンA: Docker(3つの周辺サービスをまとめて1コマンドで)
mise install                  # .ruby-versionに従いruby 4.0.5を導入
docker compose up -d db cache search
bin/setup                     # bundle, db:prepare, db:seed, playwrightのインストール

# パターンB: ネイティブ(macOS / Homebrew)
mise install
brew install postgresql@14 memcached
brew services start postgresql@14
brew services start memcached
# OpenSearchはHomebrewにないため、"ネイティブ"構成でもDocker経由で実行する:
docker run -d -p 9200:9200 \
  -e discovery.type=single-node -e DISABLE_SECURITY_PLUGIN=true \
  -e 'OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m' \
  opensearchproject/opensearch:2.13.0
bin/setup
```

Linux/apt向けの手順は`CONTRIBUTING.md`の「Development Setup」を参照してください。`bin/setup`は
`bin/playwright install --with-deps chromium`を実行します。`smoke.sh`は`bin/playwright screenshot`を
呼び出しており、同じバンドル済みChromiumを使うため、別途ブラウザを用意する必要はありません。

まっさらな環境で陥りがちな2つの罠:

- **`mise install`だけではPATHにRubyが通りません(非対話シェルの場合)。** 先にactivateする
  (`eval "$(mise activate bash --shims)"`)か、コマンドの前に`mise exec --`を付けてください。
  そうしないと`bin/setup`がシステムのRubyに対して実行され、bundlerがバージョン不一致で失敗します。
- **Linuxのネイティブ Postgresは`trust`認証が必要です。** `config/database.yml`は同じ`postgres`
  ロールに対してdev(`devpassword`)とtest(`testpassword`)で*異なる*パスワードを期待しており、
  これはサーバーがパスワードを検証しない場合のみ成立します。Dockerは`POSTGRES_HOST_AUTH_METHOD=trust`
  で、Homebrewはデフォルトでローカル接続を信頼するためこれを満たしますが、aptのPostgresのデフォルトは
  `scram-sha-256`です。`pg_hba.conf`のlocal/`127.0.0.1`エントリを`trust`に設定してPostgresを再起動する
  か、`bin/rails db:prepare`が`password authentication failed for user "postgres"`で失敗します。

## 実行(エージェント向けパス — 推奨)

```bash
./.agents/skills/run-rubygems-org/smoke.sh
```

このコマンド1つで以下が行われます。

1. Postgres(`:5432`)、Memcached(`:11211`)、OpenSearch(`:9200`)をプローブする。既にサービスが
   listenしていれば(ネイティブのbrew/apt install、または他の起動中コンテナ)そのままにする。何かが
   落ちていて`docker compose`と`docker-compose.yml`の両方が使えれば`docker compose up -d <service>`に
   フォールバックする。どちらも使えなければ、そのサービス用の正確なインストールコマンドを表示して失敗する。
2. :3000で何も動いていなければ、`nohup`配下で`bundle exec rails server -p 3000 -b 127.0.0.1`を起動する
   (pid→`tmp/run-skill/rails.pid`, log→`tmp/run-skill/rails.log`)。既に動いていればそれを再利用する。
3. `GET /`が200を返すまでポーリングする(タイムアウト60秒 — コールドスタートは約3秒だが、コード変更後の
   初回起動はTailwindコンパイルのため約10〜15秒かかる)。
4. 主要4エンドポイントにcurlし、200以外またはレスポンス内に想定文字列が見つからない場合は明確に失敗する:
   - `/`(ホームページHTML)
   - `/api/v1/gems/rubygem0.json`(gem JSON API)
   - `/versions`(compact_index — `bundle`や`gem`が取得するもの)
   - `/gems/rubygem0`(gem詳細ページ)
5. headless Chromeで幅1280pxの3ページをスクリーンショット → `tmp/run-skill/{home,gem,signin}.png`

`rubygem0`は`db:seed`が作成するため存在します。DBをリセットした場合は`bin/rails db:seed`を再実行してください。

`smoke.sh`が起動したサーバーを止める場合:

```bash
kill "$(cat tmp/run-skill/rails.pid)"
```

## 実行(人間向けパス)

```bash
bin/rails s                  # :3000でフォアグラウンド起動
```

何も開きません — ただlistenするだけです。headlessでは実用性がないため、エージェントがアプリを操作する
場合は、スクリーンショットとHTTPアサーションが得られる`smoke.sh`を常に使ってください。

## 直接呼び出し(内部コードを触る)

このリポジトリのほとんどのPRは、HTML表示面ではなくモデル・ジョブ・ポリシー・コントローラーに触れます。
そうしたケースでは`smoke.sh`を飛ばして、コードを直接呼び出してください。以下はすべて開発用DBに対して
実行され、サーバーを再起動しなくてもローカルの変更を反映します。

```bash
# ワンライナー: import + call + observe(AGENTS.mdの例に対応)
bin/rails runner 'p Rubygem.find_by(name: "rubygem0")&.versions&.count'

# REPL — 探索的な作業や複数ステップの作業向け
bin/rails c

# 単一テストファイルを実行(モデル/ジョブの変更に対する最速のフィードバックループ)
bin/rails test test/models/rubygem_test.rb
bin/rails test test/models/rubygem_test.rb:42      # 単一行
bin/rails test -n /pattern/                         # 名前パターン

# 開発用DBに対してジョブをキューイング/インライン実行する
bin/rails runner 'YourJob.new.perform(Rubygem.last.id)'
```

ジョブ/ポリシーがgemインデックスやOpenSearchの最新状態に依存する場合は、先にインデクサーを実行してください。

```bash
bundle exec rake gemcutter:index:update             # compact_index / specs.*.gz
bundle exec rake searchkick:reindex CLASS=Rubygem   # OpenSearch
```

## このskillが失敗したとき(学習して教えるループ)

このskillは[`LEARNINGS.md`](LEARNINGS.md)という進行中のログを保持しており、各失敗を一度限りのコストに
留め、繰り返さないようにしています。**`smoke.sh`がTroubleshootingテーブルにまだない原因で失敗したら:**

1. **まず[`LEARNINGS.md`](LEARNINGS.md)を読む** — エラー文字列をCtrl-Fしてください。既に誰かが
   解決している可能性があります。
2. **`tmp/run-skill/diagnostic.txt`を読む** — `smoke.sh`は失敗するたびに、環境の構造化スナップショット
   (ポート、docker状態、railsログの末尾、ruby/bundlerのバージョン)を書き出します。
3. **修正する。** 効果があった正確なコマンドを記録してください。
4. **[`LEARNINGS.md`](LEARNINGS.md)にエントリを追加する。** ファイル冒頭のテンプレートを使ってください。
   自分にとって「当たり前」の修正でも省略しないこと — 深夜に新しいコンテナで作業する次のエージェントに
   とっては当たり前ではありません。
5. **安定した修正は"卒業"させる。** その失敗モードが再発しやすい(単発の環境特有の問題ではない)場合:
   - 下記Troubleshootingテーブルに行を追加する(症状→修正)、および/または
   - `smoke.sh`を自動検出・自動回復するよう更新する(プローブを追加する、dockerフォールバックを拡張するなど)
   - その上でLEARNINGS.mdのエントリを**Status: graduated**にし、該当のcommit/PRへのリンクを付ける

これはskillが自己学習する仕組みです。`LEARNINGS.md`は`SKILL.md`/`smoke.sh`へのあらゆる変更の背後にある
*理由*であり、卒業後も残しておいてください。

## テスト

```bash
bin/rails test                                  # unit/integration
bin/rails test test/models/rubygem_test.rb:42   # 単一テスト
DB_HOST=db bin/rails test                       # Postgresがlocalhostにない場合のみ(例: dev container) — Gotchas参照
bin/rails test:system                           # Playwright + Chrome
bin/ci                                          # CIスイート全体(lint + brakeman + tests)
```

## つまずきやすい点(Gotchas)

- **`.env.local`はtest環境で除外されます。** dotenvは開発環境では`.env.local`を読み込みますがtestでは
  読み込まないため、そこで設定した`DB_HOST`はテストからは見えません。`config/database.yml`はdev・test
  双方で`DB_HOST`のデフォルトを`localhost`(TCP)としているため、標準的なローカル構成では`DB_HOST`は
  一切不要ですが、Postgresが別ホスト(例: dev containerの`db`)にある場合はインラインで渡してください:
  `DB_HOST=db bin/rails test`。
- **`/up`エンドポイントは存在しません。** Rails 8のデフォルトヘルスチェックは配線されていないため、
  readiness probeには`GET /`(ホームページ)を使ってください。`/up`は404を返します。
- **初回起動時は"Listening on …"の前に約50行のDatadog APM/CI-Visibilityのノイズが出力されます。**
  これは正常です — 開発環境では`dd-trace-rb`が自動でロードされます。失敗と勘違いしないでください。
- **OpenSearchが`status: yellow`を報告することがあります。** シングルノードクラスタでは想定内です。
  smokeスクリプトは`_cluster/health`が応答することだけを要求しており、greenである必要はありません。
- **シードされるgemは`rubygem0`/`rubygem1`/`rubygem2`であり、**実在のgemではありません。
  `/api/v1/gems/rails.json`は"This rubygem could not be found."を返します — smokeチェックでは
  常に`rubygem0`を使ってください。
- **Tailwindは別プロセスで動きます**(`bin/rails tailwindcss:watch`が開発環境で自動的に起動します)。
  起動後、非同期にCSSを書き出すため、スクリーンショットでホームページのスタイルが崩れている場合は
  さらに2〜3秒待ってください。
- **`bin/setup`は開発用DBのスキーマを破壊的に変更します。** `db:prepare`を実行し、作成・マイグレーション
  を行います。実際の匿名化データを読み込むには`script/load-pg-dump`を使ってください(AGENTS.md参照)。
- **スクリーンショットは(見つけてきたChromiumではなく)`bin/playwright`経由で撮影されます。**
  `bin/playwright`は`playwright-ruby-client` gemのバージョンに固定されたNode CLIで、
  `playwright screenshot`は自身がバンドルしたブラウザを解決するため、Chromiumがディスクのどこにあるかは
  気にする必要がありません。キャッシュを消してしまった場合は`bin/playwright install --with-deps chromium`
  で再取得できます。

## Troubleshooting

| 症状 | 対処 |
|---|---|
| `could not connect to server: Connection refused`(psql/Rails) | Postgresが起動していません。Docker: `docker compose up -d db`。ネイティブ(macOS): `brew services start postgresql@14`。 |
| `localhost:9200`に対する`Faraday::ConnectionFailed` | OpenSearchが停止しているか起動中です。`smoke.sh`はdockerが利用可能であれば自動起動を試みます — そうでなければ前提条件のOpenSearchの`docker run`行を参照してください。`/_cluster/health`が応答するまで約5秒待ってください。 |
| `smoke.sh`の失敗: `<svc> is not responding ... docker compose isn't available here` | 該当ポートに何も見つからず、PATH上にdockerも無い状態です。エラーメッセージにそのサービス用の正確なネイティブインストールコマンドが表示されるので、それを実行してから`smoke.sh`を再実行してください。 |
| `Address already in use - bind(2) for "127.0.0.1" port 3000` | 以前のRailsがまだ生きています: `pkill -f 'puma.*3000'`(または`smoke.sh`が起動した場合は`kill $(cat tmp/run-skill/rails.pid)`)。 |
| Smokeが`homepage never returned 200 (last 500)`と表示 | `tail -100 tmp/run-skill/rails.log`を確認してください — 大抵はマイグレーション漏れ(`bin/rails db:migrate`)かDBシード漏れ(`bin/rails db:seed`)です。 |
| Smokeが`gem JSON API failed (expected 200, got 404)`と表示 | DBがまっさらでシードされていません: `bin/rails db:seed`で`rubygem0`が再作成されます。 |
| smokeの出力に`screenshot <name> failed` | 固定されたplaywrightバージョン用のChromiumが未インストールです: `bin/playwright install --with-deps chromium`。 |
| smokeの出力に`bin/playwright unavailable` | PATH上に`node`が無いか`bin/playwright`が見つかりません — Node(または`mise`/`asdf`)をインストールしてから再実行してください。 |
| スクリーンショットが白紙/空白 | サーバーは200を返しているが、クライアント側でページがエラーになっています。`tmp/run-skill/rails.log`を開き、直前のリクエストを確認してください — Tailwindがまだ準備できていないケースが最も多い原因です。`smoke.sh`を再実行してください。 |
| `bin/setup` / `db:prepare`中の`FATAL: password authentication failed for user "postgres"` | パスワード認証を使うネイティブPostgres(aptのデフォルト)です。devとtestは同じ`postgres`ロールに異なるパスワードを期待するため、サーバー側でパスワードを検証しない設定にする必要があります: `pg_hba.conf`のlocal/`127.0.0.1`エントリを`trust`に設定し、Postgresを再起動してから再実行してください。前提条件を参照。 |
| `mise install`直後の`Your Ruby version is 3.x, but your Gemfile specified 4.0.x` | このシェル(非対話)でmiseがactivateされていません: `eval "$(mise activate bash --shims)"`、またはコマンドの前に`mise exec --`を付けてから再実行してください。 |
| `docker pull`が`You have reached your unauthenticated pull rate limit`で失敗 | Docker Hubのレート制限です。まっさらなコンテナ/CIのegress IPからよく発生します。OpenSearchにはミラーがあります: 前提条件の`docker run`行のイメージを`public.ecr.aws/opensearchproject/opensearch:2.13.0`に差し替えてください。Postgresとmemcachedはapt/brewから入れることもできます(Postgres認証の注意点は前提条件を参照)。 |
