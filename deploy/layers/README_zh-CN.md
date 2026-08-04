# 组织层

<!-- hy-mt2-i18n:start -->
[English](./README.md) | [日本語](./README_ja.md) | [Español](./README_es.md) | **中文** | [한국어](./README_ko.md)
<!-- hy-mt2-i18n:end -->


当从私有分支定制 qm 时，该目录用于存放组织自身的部署相关资料：它是一个独立的私有仓库，其历史记录始于对 qm 的克隆，其中核心部分与上游版本保持一致，而所有特定于组织的配置则都存放在 `deploy/layers/<org>/` 下。

在上游的 qm 中，该目录仅包含此文件，且始终如此。一个层属于某个组织的私有分支，绝不会上传到上游。`upstream-pr` 技能会强制维护这一边界；`update-qm` 技能则负责合并上游的变更。

## 创建层

```bash
node cli/bin/qm.ts init deploy/layers/<org> --org <slug> --target <fly-or-aws>
```

`qm init` 会生成部署配置、密钥名称示例、沙箱及提供商模板、操作员操作手册，以及针对每个目录的 `.gitignore` 文件，后者用于防止 `.env` 文件中的值和 Terraform 状态被纳入 Git。建议直接生成层而非手动构建，这样 `.gitignore` 文件就会随之产生；根目录下的 `.gitignore` 文件则作为备用，覆盖相同的文件。

其完整内容可见 [`docs/deploy-directory.md`](../../docs/deploy-directory.md)：

```text
deploy/layers/<org>/
  qm.config.jsonc          部署配置；已提交，不包含任何密钥值
 .gitignore               模板生成的文件；防止.env 和 tfstate 被纳入 Git
 .env.example             计算出的密钥名称，不含实际值
 .env                     本地的密钥值；永不提交
  sandbox/                 供代理计算机使用的组织工具和技能
  plugins/<name>/          组织特定的服务镜像
  infra/                   AWS 目标环境下的提供商基础设施及 tfvars 文件
  slack-app-manifest.yml   生成的机器人清单文件
  deployment.md            操作员操作手册
```

可以使用 `--config` 参数让 CLI 指向某个层：

```bash
node cli/bin/qm.ts check --config deploy/layers/<org>/qm.config.jsonc
```

按照所示方式从该目录结构下运行 CLI。在源码检出模式下，`npm exec qm` 是无法使用的，因为工作区符号链接指向的是尚未构建的 `cli/` 目录。

## 相关目录

`deploy/stacks/` 目录存放用于测试 Fly 后端的、与账户无关的合约固定配置，而 `deploy/<service>/` 目录则存放服务镜像以及 CLI 用于渲染的 Fly 模板。这两个目录均不适合存放组织相关资料。

## 规则

`deploy/layers/` 下的任何内容都不得上传到上游的 qm：无论是配置文件、沙箱工具、基础设施坐标，还是其中出现的系统或人员名称都不行。无论是在该目录还是其他任何地方，密钥信息都绝不能进入 Git。它们应存储在提供商的加密密钥存储库中，仅在被加入 `.gitignore` 的 `.env` 文件中保留本地值。
