# Learnings: running rubygems.org locally

(日本語訳注: このファイルは英語のエラー文字列をCtrl-Fで検索できることが重要なため、以下の冒頭説明のみ翻訳し、
テンプレートおよびEntries本体は原文(英語)のまま維持しています。)

rubygems.orgを起動する際に遭遇した失敗パターンと、有効だった対処法を蓄積している運用ログです。
**このファイルは、skillが学習していくために存在します。** `smoke.sh`が失敗したら:

1. **まずこのファイルを読む** — 誰かが既に解決している可能性があります(エラー文字列をCtrl-F)。
2. **`tmp/run-skill/diagnostic.txt`を読む** — `smoke.sh`は失敗するたびに、環境の構造化スナップショット
   (ポート、docker状態、railsログの末尾、ruby/bundlerのバージョン)を書き出します。
3. **問題を修正する。** 効果があった正確なコマンドをメモしてください。
4. **下にエントリを追加する。** テンプレートを使ってください。具体的に書くこと — 未来の自分/未来のエージェントが
   何の前提知識もない状態でこれを読み、対処できる必要があります。
5. **安定した修正は"卒業"させる。** その失敗が再発しやすい(単発の環境特有の問題ではない)場合、以下のいずれか、
   または両方を行ってください:
   - `SKILL.md`のTroubleshootingに行を追加する(症状→修正)
   - `smoke.sh`を自動検出・自動回復するよう更新する
   - その上で下のエントリを"**Status:** graduated"にし、該当のcommit/PRへのリンクを付ける

すぐに"卒業"させた場合でも手順4を省略しないこと — このファイルはSKILLへの変更の背後にある*理由*です。

## Entry template

Copy this block to the top of "Entries" and fill it in:

```markdown
## YYYY-MM-DD — short title

**Environment:** macOS arm64 14.5 / Ubuntu 22.04 / Linux container / etc.
**Triggered by:** `./.agents/skills/run-rubygems-org/smoke.sh` (or the specific command, if narrower)
**Symptom:** one-line error + where it appeared (stderr, `tmp/run-skill/rails.log`, `tmp/run-skill/diagnostic.txt`)
**Root cause:** the actual reason (not just the surface error)
**Fix:** the exact command(s) that worked, in order
**Status:** in-the-wild  |  graduated (link to commit/PR)
```

## Entries

_Add new entries at the top so the most recent is easiest to find._

## 2026-06-11 — native Postgres rejects `db:prepare` with password auth

**Environment:** Linux container (Ubuntu 24.04, fresh cloud agent sandbox), Postgres 16 via apt
**Triggered by:** `bin/setup` (step `bin/rails db:prepare`)
**Symptom:** `PG::ConnectionBad: ... FATAL: password authentication failed for user "postgres"` on stderr — *after* the development DB had already been created successfully.
**Root cause:** `db:prepare` prepares dev *and* test, and `config/database.yml` expects different passwords for the same `postgres` role (`devpassword` vs `testpassword`). Only a server that doesn't check passwords can satisfy both. Docker's `POSTGRES_HOST_AUTH_METHOD=trust` and Homebrew's default local `trust` do; apt's default `scram-sha-256` doesn't.
**Fix:** set the local/`127.0.0.1` entries in `pg_hba.conf` to `trust` (`sed -i 's/scram-sha-256/trust/g; s/peer$/trust/' "$(sudo -u postgres psql -tAc 'SHOW hba_file')"`), restart Postgres, re-run `bin/setup`.
**Status:** graduated (#6546 — Prerequisites trap + Troubleshooting row in SKILL.md)

## 2026-06-11 — `mise install` Ruby not picked up in non-interactive shells

**Environment:** Linux container (Ubuntu 24.04, fresh cloud agent sandbox), system Ruby 3.3.6, mise 2026.6.2
**Triggered by:** `bin/setup` right after `mise install`
**Symptom:** bundler aborts with a Ruby version mismatch (Gemfile wants 4.0.x, got 3.3.6), even though `mise install` reported `ruby 4.0.5 ✓ installed`.
**Root cause:** mise only rewrites PATH in shells where it's activated (shell rc hook). Agent/CI shells are non-interactive, so the freshly installed Ruby never shadows the system one.
**Fix:** `eval "$(mise activate bash --shims)"` before running anything, or prefix each command with `mise exec --`.
**Status:** graduated (#6546 — Prerequisites trap + Troubleshooting row in SKILL.md)

## 2026-06-11 — Docker Hub unauthenticated pull rate limit

**Environment:** Linux container (Ubuntu 24.04, fresh cloud agent sandbox — shared egress IP)
**Triggered by:** `docker compose up -d db cache search`
**Symptom:** `error from registry: You have reached your unauthenticated pull rate limit` on stderr; persisted across retries with backoff.
**Root cause:** Docker Hub rate-limits anonymous pulls per source IP; cloud sandboxes and CI runners share egress IPs, so a fresh environment can be rate-limited before its first pull.
**Fix:** pulled OpenSearch from the AWS mirror instead — `docker run -d -p 127.0.0.1:9200:9200 -e discovery.type=single-node -e DISABLE_SECURITY_PLUGIN=true -e 'OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m' public.ecr.aws/opensearchproject/opensearch:2.13.0` — and installed Postgres + memcached via apt (see the Postgres trust-auth entry above).
**Status:** graduated (#6546 — Troubleshooting row in SKILL.md)
