# `qm`

<!-- hy-mt2-i18n:start -->
[English](./README.md) | [日本語](./README_ja.md) | [Español](./README_es.md) | **中文** | [한국어](./README_ko.md)
<!-- hy-mt2-i18n:end -->


QM 的独立部署 CLI 工具。关于标准目录结构、安全保障机制、目标行为以及生命周期的详细说明，请参阅 [`docs/deploy-directory.md`](../docs/deploy-directory.md)。`qm init` 命令会将供代理程序使用的包运行手册转换为部署仓库中的实际内容。

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

该包在 npm 上以 `@yc-software/qm` 的名称发布，npm 的溯源功能可验证其构建流程。一次版本发布意味着从 `main` 分支触发 `.github/workflows/release.yml` 脚本：该脚本会对自建镜像进行签名并推送，同时发布锁定这些镜像哈希值的包信息，随后打上 `v<版本号>` 标签，并附带已解析的哈希值在 GitHub 上创建发布记录。版本号来自 `cli/package.json`，每当该文件中指定的打包内容发生变化时，CI 系统就会要求通过 Pull Request 来提升版本号；如果已存在相同版本的标签，则会阻止再次发布而非直接升级。检查后保存的镜像清单仅作为参考，实际的部署会使用真实的哈希值来覆盖它。`packed-artifact` 测试则用于在本地验证消费者的使用路径是否正常。

该 CLI 用于部署长时间运行的 QM 服务，但它本身并非运行时环境。Docker 会在本地运行这些服务，Fly 则利用 Fly Machines 作为代理节点来运行它们，而 AWS 则在 ECS Fargate 上通过 Lambda MicroVM 代理节点来运行基于哈希值锁定的 ARM64 任务。

## 部署目录结构

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

`qm.config.jsonc` 会被提交到版本控制中，且其中不包含任何敏感信息。`.env` 文件会被忽略。`package.json` 会将 CLI 包锁定在创建该目录时所使用的确切版本——`contract: 1` 仅作为最低兼容性标准——因此每次检出都会使用相同的解释器；如需升级版本，请刻意修改该锁定值。进入该目录后，DEPLOY 相关命令就会对其生效；`--config` / `--env-file` / `--sandbox-dir` 参数可用于调整相关配置的位置（例如让多个部署共享同一个 `sandbox/` 目录）。`check` 命令可在无网络连接的情况下验证配置、计算出的敏感信息名称、工具、技能以及插件是否有效；`up`、`plan` 和 `sandbox build` 命令在运行前也会先执行相同的验证步骤。`doctor` 命令仅以只读方式检查外部依赖项是否满足要求。`plan` 命令用于生成部署方案；若要在 AWS 上执行实际部署，则需要使用 `up --yes`。

在 AWS 环境中，`up` 命令会在首次执行变更操作之前对 RDS 实例创建快照，该快照的名称会取自其前面的部署清单，并且该快照信息也会被记录在部署清单中。`rollback` 命令仅能恢复代码和配置，它会将该快照作为对应的数据恢复点显示出来（命令为 `aws rds restore-db-instance-from-db-snapshot`）。预部署阶段的快照数量是有限制的；若要将此限制关闭，可设置 `aws.predeployDbSnapshot: false`。

`sandbox build` 是一种本地验证构建过程。`sandbox publish` 命令则会通过配置好的 OCI 注册表推送镜像，解析镜像及其基础哈希值，将基础版本锁定信息记录在配置文件中，同时将镜像版本锁定信息记录在配置文件（docker/fly）或持久化的 AWS 部署清单中。当核心服务可访问时，它会同步持久化的部署层，并重新指向正在运行的 Fly 或 AWS 核心服务。在 AWS 上使用时，需要设置 `sandbox.backend: "sprites"`，并且在开始构建之前必须已有现有的部署清单，同时不能有 `sandbox.image` 的自定义设置——因为该自定义设置仅用于首次执行 `qm up` 操作，之后必须将其移除。每次普通的 `up` 操作也会同步该部署层。

除非 `qm.config.jsonc` 中指定了带有提供者标签、HTTPS 接入点以及 `shadow` 或 `enforce` 部署策略的 `securityScreen` 代理，否则 Auto 会使用其内置的模型分类器。代理令牌会通过 `secretEnv.core.SECURITY_SCREEN_PROXY_TOKEN` 途径单独传递。

## 命令列表

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

所有的部署命令都支持 `--config`、`--env-file` 和 `--sandbox-dir` 参数。`dev` 命令仅用于维护者的工作树循环，与通用的部署契约是分开的。

## 包契约

`@yc-software/qm/contract` 导出的是用于一致性测试的标准化编程接口。它公开了契约版本、解析/渲染函数以及提供者 ID，而无需注册任意运行时插件。如果目录结构发生不兼容的变更，契约的主版本号将会递增；在同一主版本号下还可以添加可选字段。

该包没有运行时依赖项。它在后台调用 Docker、Buildx、Flyctl、AWS CLI 以及 Git 等工具。Terraform 则由操作员针对 `init` 命令生成的模块来进行配置和使用。
