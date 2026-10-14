# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 개요

GitHub Actions → Amazon ECR → EC2 배포 파이프라인을 시연하기 위한 최소 Express 앱입니다. 애플리케이션 코드는 [server.js](server.js) 하나(`GET /` 응답 하나)뿐이고, 이 저장소의 핵심은 [.github/workflows/build-and-deploy.yml](.github/workflows/build-and-deploy.yml)의 배포 흐름입니다. 커밋은 대부분 배포 시연용 응답 문자열 변경(예: `Hello World(10/13 18:34)`)입니다.

## 명령어

- 로컬 실행: `npm install && npm start` (http://localhost:8080)
- 컨테이너 실행: `docker build -t mydemo-nodejs . && docker run -p 8080:8080 mydemo-nodejs`
- 테스트, 린트, 빌드 단계는 없습니다.

## 배포 흐름

`main`에 push하거나 `workflow_dispatch`로 수동 실행하면 아래 두 job이 순서대로 돕니다.

1. **build-and-push-to-ecr**: 이미지 태그를 `YYYYMMDDHHMM(KST)-<sha 7자리>` 형식으로 만들고, Buildx + GHA 캐시(`type=gha,mode=max`)로 빌드해 ECR(`ap-northeast-2`, `mydemo-nodejs-ecr`)에 push합니다. `latest` 태그는 쓰지 않습니다.
2. **deploy-to-ec2**: `production` environment(승인 필요 가능)와 `deploy-production` concurrency 그룹으로 실행됩니다. SSH 대신 **SSM Run Command**(`AWS-RunShellScript`)로 EC2에서 `docker pull` → 기존 컨테이너 stop/rm → `docker run -d --restart unless-stopped -p 80:8080`을 수행하고, 최대 10분 동안 상태를 폴링합니다.

설정과 전제 조건:
- AWS 인증은 OIDC(`id-token: write`)와 `secrets.AWS_ROLE_ARN`으로 합니다. 액세스 키는 쓰지 않습니다.
- 대상 인스턴스는 `vars.EC2_INSTANCE_ID`로 지정합니다. EC2의 ECR pull 권한은 인스턴스 프로파일에서 옵니다.
- `vars.DEPLOY_FREEZE == 'true'`이면 push로 시작된 배포는 건너뛰고, 수동 실행(`workflow_dispatch`)만 배포됩니다.
- 롤백 경로는 따로 없습니다. 이전 이미지 태그로 다시 배포하려면 워크플로를 수정하거나 EC2에서 직접 실행해야 합니다.

## 주의 사항

- 포트는 컨테이너 내부 8080(`server.js`의 `PORT`, Dockerfile의 `EXPOSE`), 호스트 80입니다. 포트를 바꾸려면 세 곳을 함께 수정해야 합니다.
- `.dockerignore`와 `.gitignore`가 없습니다. Dockerfile의 `COPY . .`는 로컬 `node_modules`까지 이미지에 덮어씁니다.
- 베이스 이미지는 `node:16`(EOL)입니다.
- `package.json`에 `react`, `react-dom`, `typescript`, `lodash`가 들어 있지만 `server.js`는 `express`만 사용합니다.
