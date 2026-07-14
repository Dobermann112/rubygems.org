# RubyGems.org (旧称 Gemcutter)
Rubyコミュニティのgemホスティングサービス。

## 目的

* gemを扱うためのより良いAPIを提供する
* より透明で利用しやすいプロジェクトページを作る
* コミュニティがサイトを改善・拡張できるようにする

## サポート

<a href="https://rubycentral.org/"><img src="doc/ruby_central_logo.png" height=110></a><br>

[RubyGems.org](https://rubygems.org) は非営利団体[Ruby Central](https://rubycentral.org)によって運営されています。Ruby Centralは、このプロジェクトのほか、[RubyConf](https://rubyconf.org)、[RailsConf](https://railsconf.org)、[Bundler](https://bundler.io)といったプロジェクトを通じてRubyコミュニティを支援しています。カンファレンスに参加・[協賛](sponsors@rubycentral.org)したり、[サポートメンバーとして参加](https://rubycentral.org/#/portal/signup)することで、Ruby Centralを支援できます。

ホスティングは[Amazon Web Services](https://aws.amazon.com)から、CDNサービスは[Fastly](https://fastly.com)から提供されています。

[スポンサーとその協力関係について詳しくはこちら。](https://rubygems.org/pages/sponsors)

## リンク

* [RFC(仕様提案)](https://github.com/rubygems/rfcs)
* [サポート](mailto:support@rubygems.org)
* [GitHub Workflow][]: [![test workflow](https://github.com/rubygems/rubygems.org/actions/workflows/test.yml/badge.svg)](https://github.com/rubygems/rubygems.org/actions/workflows/test.yml) [![lint workflow](https://github.com/rubygems/rubygems.org/actions/workflows/lint.yml/badge.svg)](https://github.com/rubygems/rubygems.org/actions/workflows/lint.yml) [![docker workflow](https://github.com/rubygems/rubygems.org/actions/workflows/docker.yml/badge.svg)](https://github.com/rubygems/rubygems.org/actions/workflows/docker.yml)
[![codecov](https://codecov.io/github/rubygems/rubygems.org/graph/badge.svg)](https://codecov.io/github/rubygems/rubygems.org)

[github workflow]: https://github.com/rubygems/rubygems.org/actions/

## コントリビューション

[コントリビューションガイドライン][contribution guidelines]に従ってください。

[contribution guidelines]: https://github.com/rubygems/rubygems.org/blob/master/CONTRIBUTING.md

セットアップ方法は[Development Setup][]を確認してください。

[development setup]: https://github.com/rubygems/rubygems.org/blob/master/CONTRIBUTING.md#development-setup

デプロイ手順もwikiにドキュメント化されており、複数ステップからなる[チェックリスト][Checklist]に沿って進めます。

[Checklist]: https://github.com/rubygems/rubygems-infrastructure/wiki/Deploys

また、[行動規範(Code of Conduct)](https://github.com/rubygems/rubygems.org/blob/master/CODE_OF_CONDUCT.md)にもご留意ください。

セットアップでお困りの場合やご質問がある場合は、このリポジトリでIssueを作成してください。喜んでお手伝いします!

## 構成

RubyGems.orgは主に以下の要素から構成されています。

* Railsアプリ: ユーザー管理や、他のユーザーがgemを閲覧できるようにする等の機能を担当
* Gemプロセッサー: アップロードされたgemを受け取り、Amazon S3(本番環境)またはファイルシステム上の`server/`(開発環境)に保存する処理を担当

## ライセンス

RubyGems.orgはMITライセンスを使用しています。詳細は[LICENSE][license]ファイルを確認してください。

[license]: https://github.com/rubygems/rubygems.org/blob/master/MIT-LICENSE
