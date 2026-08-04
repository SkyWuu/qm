# 조직별 레이어

<!-- hy-mt2-i18n:start -->
[English](./README.md) | [日本語](./README_ja.md) | [Español](./README_es.md) | [中文](./README_zh-CN.md) | **한국어**
<!-- hy-mt2-i18n:end -->


qm을 개인 포크에서 커스터마이즈할 때, 해당 조직만의 배포 자료는 이 디렉터리에 저장됩니다. 이곳은 qm의 클론으로 시작된 독립적인 프라이빗 리포지토리로, 핵심 부분은 상위 버전과 동일하게 유지되며 조직에 특화된 모든 내용은 `deploy/layers/<org>/` 아래에 수록됩니다.

상위 버전의 qm에서는 이 디렉터리에 이 파일 외에는 아무것도 존재하지 않으며, 그 상태가 그대로 유지됩니다. 레이어는 특정 조직의 프라이빗 포크에 속해 있으며 결코 상위 버전으로 전파되지 않습니다. `upstream-pr` 스킬이 이러한 경계를 엄격히 준수하며, `update-qm` 스킬은 주변의 상위 버전 변경 사항들을 병합합니다.

## 레이어 생성하기

```bash
node cli/bin/qm.ts init deploy/layers/<org> --org <slug> --target <fly-or-aws>
```

`qm init` 명령어는 배포 설정 파일, 비밀 정보 이름 예시, 샌드박스 및 프로바이더 관련 스캐폴딩 파일, 운영자용 런북, 그리고 `.env` 파일의 값과 Terraform 상태 정보를 Git에서 제외하는 디렉터리별 `.gitignore` 파일을 생성합니다. 수동으로 레이어를 구축하는 대신 자동으로 생성하면 `.gitignore` 파일도 함께 만들어지므로 보다 편리합니다. 루트에 있는 `.gitignore` 파일도 마찬가지로 해당 파일들을 차단하는 역할을 합니다.

자세한 내용은 [`docs/deploy-directory.md`](../../docs/deploy-directory.md)를 참조하십시오:

```text
deploy/layers/<org>/
  qm.config.jsonc          배포 설정 파일; 커밋되지만 비밀 정보 값은 포함되지 않음
 .gitignore               스캐폴딩된 파일;.env 및 tfstate 파일을 Git에서 제외함
 .env.example             계산된 비밀 정보 이름만 저장됨, 실제 값은 저장되지 않음
 .env                     로컬 비밀 정보 값; 절대 커밋되지 않음
  sandbox/                 에이전트 컴퓨터용 조직 툴 및 스킬 파일들
  plugins/<name>/          조직에 특화된 서비스 이미지 파일들
  infra/                   AWS 환경용 프로바이더 인프라 및 tfvars 파일들
  slack-app-manifest.yml   자동으로 생성된 봇 매니페스트 파일
  deployment.md            운영자용 런북
```

`--config` 옵션을 사용하여 CLI를 특정 레이어에 지정할 수 있습니다:

```bash
node cli/bin/qm.ts check --config deploy/layers/<org>/qm.config.jsonc
```

위와 같이 트리 구조 내에서 CLI를 실행하십시오. 워크스페이스의 심볼릭 링크가 아직 구축되지 않은 `cli/`를 가리키기 때문에 `npm exec qm` 명령어는 소스 코드를 체크아웃한 상태에서는 작동하지 않습니다.

## 주변 디렉터리

`deploy/stacks/` 디렉터리에는 Fly 백엔드를 테스트하는 데 사용되는 계정과 무관한 계약 관련 픽스처 파일들이 저장되며, `deploy/<service>/` 디렉터리에는 CLI가 렌더링하는 서비스 이미지와 Fly 템플릿 파일들이 있습니다. 이 두 곳 모두 조직 관련 자료를 저장하는 곳이 아닙니다.

## 규칙

`deploy/layers/` 아래에 있는 어떤 파일도 상위 버전의 qm으로 전달되어서는 안 됩니다. 배포 설정 파일도, 샌드박스 툴도, 인프라 관련 정보도, 그리고 그 안에 포함된 시스템이나 담당자의 이름도 마찬가지입니다. 비밀 정보는 이 디렉터리나 다른 어떤 곳에서도 절대 Git에 들어가지 않습니다. 비밀 정보는 프로바이더의 암호화된 비밀 저장소에 보관되며, 로컬 값만이 `.gitignore` 파일에 포함됩니다.
