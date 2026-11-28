---
date: 2026-11-28
status: accepted
---

# 공통 lint 규칙은 npm 패키지 lint-guardrails 하나로 배포하고, 템플릿으로 만든 저장소도 복사본 대신 그 패키지에 의존한다

## 결정

- 공통 lint 규칙(`base`·`layers`·`domainLayers`·`noTopLevelEffects`와 가드 테스트)은 템플릿 저장소의 packages/guardrails에만 있다. 패키지 이름은
  `lint-guardrails`이고 npm에 공개로 배포한다. 이름에 도구(ESLint)를 넣지 않는다 — 규칙을 강제하는 도구가 바뀌어도 이름이 맞게. 템플릿 안의 앱·패키지는 워크스페이스 의존성(`workspace:*`)으로 쓴다.
- `init`은 만든 저장소에서 그 폴더를 지우고, 의존성을 그때의 배포 판(정확한 버전)으로 바꾼다. 만든 저장소는 규칙을
  고치지 않고 의존성을 올려 받는다. 따로 시작한 저장소도 같은 패키지에 의존한다.
- 저장소마다 다른 것은 그 저장소의 `eslint.config.ts` 한 곳에만 둔다 — 레이어·모듈·예외 목록 같은 표다. 공통 규칙을 저장소에서 고치거나
  끄지 않는다. 한 저장소의 특성 때문에 공통 규칙을 바꿔야 하면 그 규칙을 공통에서 빼서 그 저장소의 규칙으로 돌린다. 공통에는 둘
  이상의 저장소에 실제로 있는 규칙만 둔다.
- 배포는 `publish-guardrails.yml`이 한다 — `main`에 들어온 `package.json`의 `version`이 npm에 없을 때만, npm trusted publishing
  (OIDC)으로 provenance와 함께 올린다(npm CLI 11.5 이상이 필요하다 — https://docs.npmjs.com/trusted-publishers, 2026-11-28 확인).
  저장소에 npm 토큰을 두지 않는다. 첫 판(0.1.0)은 손으로 올리고, 그 뒤 npmjs.com의 패키지 설정에서 이 워크플로를 신뢰한 출판자로
  등록한다.
- 버전: 규칙을 더하거나 넓혀 지금 통과하던 코드가 오류가 될 수 있으면 가운데, 고치기만 하면 끝자리를 올린다(1.0 전이다).
- 저장소 루트는 루트 표지 파일(`pnpm-workspace.yaml`, pnpm·npm·yarn·bun lockfile)을 위로 찾아 정한다. `layers()`는 레이어를 적는
  패키지에서, `base`의 `tsconfigRootDir`(ADR 0019)는 워크스페이스 안에서는 이 패키지 위치에서, 설치된 패키지로 쓰면 ESLint를 띄운
  폴더(`process.cwd()`)에서 찾는다. `link:`·`file:`로 다른 저장소에 연결해 쓰는 것은 지원하지 않는다 — 그때는 tsconfig 기준이 이
  저장소가 된다.
- 배포 판은 JS(`dist/index.mjs`)와 타입(`dist/index.d.mts`)이다 — `pnpm pack`이 `prepack`으로 빌드하고 `publishConfig.exports`가
  그쪽을 가리킨다. 템플릿 안에서는 소스(`index.mts`)를 그대로 쓴다. Node는 `node_modules` 안의 TypeScript를 타입 제거로 읽지 않아,
  소스를 그대로 낸 0.1.0은 소비 저장소에서 설정을 불러오다 멈췄다(`ERR_UNSUPPORTED_NODE_MODULES_TYPE_STRIPPING`).
- 이 패키지만 pnpm `minimumReleaseAge`(갓 나온 판 설치 보류, ADR 0006)의 예외다 — 우리가 템플릿 `main`에서 provenance와 함께
  배포하고, `init`이 그때의 판을 정확히 적어 넣어서 막으면 배포 직후 하루 동안 템플릿으로 저장소를 만들 수 없다. 소비 저장소도
  `pnpm-workspace.yaml`의 `minimumReleaseAgeExclude`에 같은 이름을 둔다.

## 이유

- 같은 하네스가 손 복사본으로 퍼져 저장소마다 따로 자랐다 — 맞춘 규칙이 다시 어긋나고, 템플릿에서 더한 규칙(ADR 0016·0018·0020)이
  다른 저장소에 닿지 않았고, `init`이 원본 커밋을 남기지 않아 만든 뒤 갱신받을 길이 없었다.
- npm 공개를 고른 것: 숨길 내용이 없고(템플릿이 이미 공개 저장소다), 설치하는 모든 곳(로컬·CI·배포 빌드·Renovate)에 토큰이
  필요 없다. GitHub Packages는 공개 패키지도 설치에 토큰이 들고 그 토큰은 classic PAT만 된다(Actions 안의 `GITHUB_TOKEN`은
  예외) — Actions 밖(배포 빌드·Renovate)에서 만료되는 토큰을 관리해야 한다
  (https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-npm-registry, 2026-11-28 확인).
- git 하위 폴더 의존성은 `catalog:` 표기를 풀지 못해 설치되지 않았다(pnpm 12.6, 2026-11-28 확인). `pnpm pack`한 tarball은 설치되고
  jiti가 `.mts`를 읽는 것을 단일 패키지 저장소에서 확인했다.

## 끝과 손절선

이 일의 끝(규칙 구현이 한 곳에만 있다 · 저장소마다 다른 것은 설정 파일 한 곳에만 있다 · 갱신이 한 번 실제로 흐른다)과 손절선은
이슈 #39에 있다.
