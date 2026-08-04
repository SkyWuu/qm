# `qm`

<!-- hy-mt2-i18n:start -->
[English](./README.md) | [日本語](./README_ja.md) | [Español](./README_es.md) | [中文](./README_zh-CN.md) | **한국어**
<!-- hy-mt2-i18n:end -->


QM을 위한 독립형 배포용 CLI입니다. 표준화된 디렉터리 구조, 보안 보장 사항, 목표 동작 방식 및 라이프사이클에 대한 내용은
[`docs/deploy-directory.md`](../docs/deploy-directory.md)에 있습니다. `qm init` 명령어는 에이전트가 사용하는 패키지 관련 가이드를 배포 리포지토리에 반영합니다.

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

이 패키지는 `@yc-software/qm`이라는 이름으로 npm에 게시되며, npm의 추적 기능을 통해 빌드 워크플로우가 검증됩니다. 릴리스란 `main` 브랜치에서 `.github/workflows/release.yml` 파일이 한 번 실행되는 것을 의미합니다. 이 과정에서 1차 이미지들이 서명되어 푸시되고, 해당 이미지들의 해시 값이 패키지에 고정되어 게시됩니다. 그 후 `v<version>` 태그가 부여되고, 해결된 해시 값들이 포함된 GitHub 릴리스가 생성됩니다. 버전 번호는 `cli/package.json`에서 가져오며, 패키지에 포함되는 내용이 변경될 때마다 CI에서는 이 파일을 수정하기 위한 풀 리퀘스트를 요구합니다. 이미 존재하는 태그가 있으면 버전 업그레이드가 중단되고 그대로 유지됩니다. 체크인된 이미지 매니페스트는 배포 시 실제 해시 값으로 대체될 수 있는 기준점 역할을 합니다. `packed-artifact` 테스트는 로컬 환경에서 소비자 경로가 제대로 작동하는지 확인합니다.

이 CLI는 장시간 실행되는 QM 서비스를 배포하는 데 사용되며, 서비스 자체의 런타임은 아닙니다. Docker는 로컬에서 이러한 서비스들을 실행하고, Fly는 Fly Machines를 통해 에이전트 컴퓨터로서 Fly 앱 형태로 이들을 실행합니다. AWS는 ECS Fargate와 Lambda MicroVM 에이전트 컴퓨터를 활용하여 해시 값이 고정된 ARM64 기반의 작업들을 실행합니다.

## 배포 디렉터리

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

`qm.config.jsonc` 파일은 커밋되며, 비밀 정보는 포함되어 있지 않습니다. `.env` 파일은 무시됩니다. `package.json` 파일은 디렉터리를 생성할 때 사용된 정확한 버전의 CLI 패키지를 고정시킵니다. `contract: 1`은 단지 최소 호환성 수준일 뿐이므로, 언제나 동일한 인터프리터가 사용됩니다. 필요에 따라 고정된 버전을 의도적으로 업그레이드해야 합니다. 해당 디렉터리로 `cd`를 한 후 DEPLOY 명령어들을 실행하면 됩니다. `--config` / `--env-file` / `--sandbox-dir` 옵션을 사용하면 특정 파일이나 디렉터리(예: 하나의 `sandbox/` 폴더를 여러 배포가 공유하는 경우)의 위치를 변경할 수 있습니다. `check` 명령어는 네트워크 연결 없이도 설정 파일, 계산된 비밀 정보 이름, 도구, 스킬, 플러그인 등을 검증합니다. `up`, `plan`, `sandbox build` 명령어들도 먼저 동일한 검증을 수행합니다. `doctor` 명령어는 외부의 사전 요구 사항들이 읽기 전용 상태인지 확인합니다. `plan` 명령어는 배포 계획을 생성하며, AWS 환경에서 변更을 적용하려면 `up --yes` 명령어가 필요합니다.

AWS 환경에서 `up` 명령어는 첫 번째 변更이 수행되기 전에 배포용 리스 계약 하에 있는 RDS 인스턴스의 스냅샷을 생성합니다. 이 스냅샷에는 배포 매니페스트의 이름이 붙으며, 해당 매니페스트에도 스냅샷 정보가 기록됩니다. `rollback` 명령어는 코드와 설정 파일만 복원하므로, 매칭되는 데이터 복원 지점으로 해당 스냅샷을 표시합니다(`aws rds restore-db-instance-from-db-snapshot`). 사전 배포용 스냅샷의 수는 일정한 한도 내로 제한되며, `aws.predeployDbSnapshot: false` 옵션을 사용하면 이 제한을 우회할 수 있습니다.

`sandbox build`는 로컬에서 수행되는 검증용 빌드입니다. `sandbox publish` 명령어는 설정된 OCI 레지스트리를 통해 이미지를 푸시하고, 이미지 및 기본 해시 값을 확인한 뒤, 설정 파일과 docker/fly 형식의 설정 파일 또는 AWS의 영구적인 배포 매니페스트에 해당 기본 해시 값과 이미지 해시 값을 기록합니다. 코어에 접근할 수 있게 되면 영구적인 배포 계층도 동기화되며, 실행 중인 Fly 또는 AWS 코어의 설정도 갱신됩니다. AWS 환경에서는 `sandbox.backend: "sprites"` 옵션이 필요하며, 무언가를 빌드하기 전에는 기존의 배포 매니페스트가 반드시 존재해야 하고 `sandbox.image` 오버라이드는 금지됩니다. `sandbox.image` 오버라이드는 첫 번째 `qm up` 실행 시에만 적용되며, 그 이후에는 반드시 제거되어야 합니다. 일반적인 `up` 명령어도 계층을 동기화합니다.

`qm.config.jsonc` 파일에 프로바이더 레이블, HTTPS 엔드포인트, 그리고 `shadow` 또는 `enforce` 롤아웃 옵션이 명시된 `securityScreen` 프록시가 정의되어 있지 않는 한, 이 CLI는 내장된 모델 분류기를 사용합니다. 프록시 토큰은 `secretEnv.core.SECURITY_SCREEN_PROXY_TOKEN` 경로를 통해 별도로 전달됩니다.

## 명령어

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

모든 배포 관련 명령어는 `--config`, `--env-file`, `--sandbox-dir` 옵션을 지원합니다. `dev` 명령어는 기여자가 사용하는 작업 트리를 관리하는 용도이며, 이동 가능한 배포 계약과는 별개의 개념입니다.

## 패키지 계약

`@yc-software/qm/contract` export 항목은 규격 검증 테스트를 위해 지원되는 프로그래밍 인터페이스입니다. 이 항목을 통해 계약 버전, 파싱/렌더링 함수, 그리고 프로바이더 ID들을 확인할 수 있으며, 임의의 런타임 플러그인은 등록되지 않습니다. 호환되지 않는 디렉터리 변경이 발생하면 계약의 메이저 버전이 증가하며, 메이저 버전 내에서는 선택적 필드들이 추가될 수 있습니다.

이 패키지는 런타임 의존성이 전혀 없습니다. Buildx, Flyctl, AWS CLI, Git 등의 도구들을 통해 Docker와 상호작용합니다. Terraform은 `init` 명령어를 통해 생성된 모듈을 기반으로 운영자가 직접 실행합니다.
