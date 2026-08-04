# Organization layers

<!-- hy-mt2-i18n:start -->
[English](./README.md) | **日本語** | [Español](./README_es.md) | [中文](./README_zh-CN.md) | [한국어](./README_ko.md)
<!-- hy-mt2-i18n:end -->


このディレクトリは、プライベートなフォークからカスタマイズされた qm の場合に、組織独自のデプロイメント資料が格納される場所です。これは qm のクローンとして始まった独立したプライベートリポジトリであり、コア部分は元の upstream と同一のままで、組織固有のすべての要素は `deploy/layers/<org>/` 下に格納されます。

upstream の qm では、このディレクトリにはこのファイルしか存在せず、その状態は変わりません。レイヤーはある組織のプライベートフォークにのみ属し、決して upstream には送信されません。`upstream-pr` スキルがこの境界を厳守し、`update-qm` スキルがその周辺の upstream からの変更内容をマージします。

## レイヤーの作成

```bash
node cli/bin/qm.ts init deploy/layers/<org> --org <slug> --target <fly-or-aws>
```

`qm init` はデプロイメント設定、シークレット名の例、サンドボックスおよびプロバイダー用のスケルトン構造、オペレーター向けの運用マニュアル、そして `.env` ファイルの値や Terraform のステートを Git から除外する各ディレクトリ専用の `.gitignore` ファイルを作成します。手動で構築する代わりにレイヤーを生成することで、`.gitignore` も一緒に作成されます。ルートの `.gitignore` は予防策として同じファイルを対象にしています。

その結果は [`docs/deploy-directory.md`](../../docs/deploy-directory.md) に詳しく記載されています：

```text
deploy/layers/<org>/
  qm.config.jsonc          デプロイメント設定。コミットされるが、シークレット値は含まれない
 .gitignore               スケルトン構造が適用されており、.env および tfstate を Git から除外する
 .env.example             計算されたシークレット名のみ。実際の値は含まれない
 .env                     ローカルのシークレット値。コミットされることはない
  sandbox/                 エージェントコンピューター用の組織固有のツールやスキル
  plugins/<name>/          組織固有のサービスイメージ
  infra/                   AWS ベースのターゲットで使用されるプロバイダーのインフラストラクチャや tfvars
  slack-app-manifest.yml   生成されたボットのマニフェストファイル
  deployment.md            オペレーター向けの運用マニュアル
```

`--config` オプションを使って CLI を特定のレイヤーに向けます：

```bash
node cli/bin/qm.ts check --config deploy/layers/<org>/qm.config.jsonc
```

上記のようにツリー構造から CLI を実行します。ワークスペースのシンボリックリンクが未構築の `cli/` を指しているため、ソースコードをチェックアウトした状態では `npm exec qm` は動作しません。

## 近隣のディレクトリ

`deploy/stacks/` には Fly バックエンドのテストに使用される、アカウントに依存しない契約関連の固定ファイルが格納されており、`deploy/<service>/` には CLI がレンダリングするサービスイメージや Fly テンプレートが保存されています。どちらも組織固有の資料を格納する場所ではありません。

## 規則

`deploy/layers/` 下のどのファイルも upstream の qm には届けてはなりません。設定ファイルも、サンドボックスツールも、インフラストラクチャの情報も、またそこに記載されているシステム名や人物名も含まれます。シークレットは、このディレクトリであれ他のどの場所であれ、決して Git に入れてはなりません。それらはプロバイダーが管理する暗号化されたシークレットストアに保管され、ローカルの値のみが `.gitignore` に記載された `.env` ファイルに含まれます。
