# `qm`

<!-- hy-mt2-i18n:start -->
[English](./README.md) | **日本語** | [Español](./README_es.md) | [中文](./README_zh-CN.md) | [한국어](./README_ko.md)
<!-- hy-mt2-i18n:end -->


QM のスタンドアロンデプロイメント用 CLI です。標準的なディレクトリ構造、セキュリティ保証、目標とする動作、およびライフサイクルについては
[`docs/deploy-directory.md`](../docs/deploy-directory.md) をご覧ください。`qm init` は、エージェント側で使用されるパッケージの運用マニュアルをデプロイメントリポジトリに反映させます。

```bash
npm exec --yes --package=@yc-software/qm@latest -- \
  qm init. --org acme --target aws
npm install
npm exec qm -- check
npm exec qm -- infra render
npm exec qm -- doctor
npm exec qm -- infra build-image
npm exec qm -- plan
npm exec qm -- up --yes
npm exec qm -- check --live
```

このパッケージは `@yc-software/qm` という名前で npm に公開されており、npm のプロヴェナンス機能によってビルドワークフローが証明されます。リリースとは、`main` ブランチから `.github/workflows/release.yml` が実行されることを指します。このワークフローでは、第一者が作成したイメージに署名してプッシュし、それらのダイジェストを記録したパッケージピンを公開します。その後 `v<version>` というタグが付けられ、解決済みのダイジェストが添付された GitHub リリースが作成されます。バージョン番号は `cli/package.json` に記載されており、パッケージに含まれるコンテンツが変更されるたびに CI ではプルリクエストを提出する必要があります。既存のタグが存在する場合、バージョンアップは行われずリリースも停止されます。チェックインされたイメージマニフェストは、デプロイ時に実際のダイジェストで上書きされるためのセンチネルとして機能します。`packed-artifact` テストでは、消費者側のパスがローカルでテストされます。

この CLI は長時間実行される QM サービスをデプロイするためのものであり、実行環境自体ではありません。Docker はローカルでこれらのサービスを実行し、Fly は Fly Machines を使ってエージェントコンピュータとしてこれらを Fly アプリとして実行します。AWS では、Lambda MicroVM エージェントコンピュータを使用し、ECS Fargate 上でダイジェストで固定された ARM64 タスクが実行されます。

## デプロイメントディレクトリ

```text
qm.config.jsonc
package.json
package-lock.json
deployment.md
.codex/skills/deploy-qm/
.env.example
.env
slack-app-manifest.yml
slack-sso-manifest.yml
sandbox/
  tools/<id>/tool.json
  tools/<id>/<binary>
  skills/<id>/SKILL.md
  Dockerfile
plugins/<name>/Dockerfile
infra/
```

`qm.config.jsonc` はコミットされており、機密情報は含まれていません。`.env` ファイルは無視されます。`package.json` は、ディレクトリを作成した際の正確なバージョンで CLI パッケージを固定します。`contract: 1` は単なる互換性の下限値に過ぎず、常に同じインタプリタが使用されます。意図的にピンをアップグレードする必要があります。このディレクトリに `cd` すると、DEPLOY コマンドがそのディレクトリに対して動作します。`--config` / `--env-file` / `--sandbox-dir` を使用すると、特定の要素（例えば複数のデプロイメントで共有される `sandbox/`）の場所を変更できます。`check` コマンドは、ネットワーク接続なしで設定や計算されたシークレット名、ツール、スキル、プラグインを検証します。`up`、`plan`、`sandbox build` コマンドもまず同様の検証を行います。`doctor` コマンドは外部の前提条件を読み取るだけです。`plan` コマンドはデプロイメント内容を表示します。AWS での変更を行うには `up --yes` が必要です。

AWS 上では、`up` コマンドは最初の変更が行われる前にデプロイリース期間中の RDS インスタンスのスナップショットを作成し、そのスナップショットにはデプロイメントマニフェストの名前が付けられ、そのマニフェスト内に記録されます。`rollback` コマンドはコードと設定のみを復元するため、該当するスナップショットがデータ復元ポイントとして表示されます（`aws rds restore-db-instance-from-db-snapshot`）。デプロイ前のスナップショットは一定数に制限され、`aws.predeployDbSnapshot: false` を指定するとこの制限は適用されません。

`sandbox build` はローカルでの検証用ビルドです。`sandbox publish` は設定された OCI レジストリを経由してイメージをプッシュし、イメージとベースダイジェストを解決します。ベースピンとイメージピンは設定ファイルや docker/fly 用の設定ファイル、または永続的な AWS デプロイメントマニフェストに記録されます。コアにアクセスできる状態になったら永続的なデプロイメントレイヤーが同期され、実行中の Fly や AWS コアが再指向されます。AWS 上での実行には `sandbox.backend: "sprites"` が必要であり、何かをビルドする前には既存のデプロイメントマニフェストが存在し、`sandbox.image` によるオーバーライドは行われません。このオーバーライドは最初の `qm up` 時のみ有効で、その後は必ず削除する必要があります。通常の `up` コマンドでもレイヤーの同期が行われます。

`qm.config.jsonc` にプロバイダーラベル、HTTPS エンドポイント、および `shadow` または `enforce` 形式のロールアウト設定を持つ `securityScreen` プロキシが宣言されていない限り、Auto は内部のモデル分類器を使用します。プロキシトークンは `secretEnv.core.SECURITY_SCREEN_PROXY_TOKEN` を通じて別途ルーティングされます。

## コマンド一覧

```text
init [dir] [--org id] [--target docker|fly|aws]
check [--json] [--live]
doctor
infra render|build-image|delete-image|delete-task-definitions
conformance [dir] [--static]
plan
up [--yes] [--build-from[=repo]] [--image-label label]
slack render
outputs [--json]
proof scope-key <scope-id>
secrets push [--from file]
status
logs [service] [-f] [--tail n]
down [--purge]
rollback [--to revision-or-sha]
sandbox build [--from image] [--tag tag] [--dry-run]
sandbox publish [--from image] [--app registry/repo] [--tag tag] [--dry-run]
```

すべてのデプロイメントコマンドは `--config`、`--env-file`、`--sandbox-dir` を受け付けます。`dev` コマンドはコントリビューター向けのワークツリーループ用であり、汎用的なデプロイメント契約とは別物です。

## パッケージ契約

`@yc-software/qm/contract` のエクスポートは、コンフォーマンステストを行うためのサポートされているプログラムmaticなインターフェースです。このエクスポートには契約のバージョン、解析/レンダリング関数、およびプロバイダーIDが含まれており、任意のランタイムプラグインは登録されません。互換性のないディレクトリ変更があると契約のメジャーバージョンが上がり、メジャーバージョン内でオプションフィールドが追加されることもあります。

このパッケージにはランタイム依存関係はありません。Buildx、Flyctl、AWS CLI、Git を使用して Docker にコマンドを送信します。Terraform は `init` によって生成されたモジュールを使ってオペレーターによって実行されます。
