# lint-guardrails

규칙을 문서가 아니라 lint로 강제하는 공통 ESLint 설정이다. 결정과 이유는
[ts-monorepo-template의 ADR](https://github.com/sangwook-dev/ts-monorepo-template/tree/main/docs/adr)에 있다.

- `base` — 타입 오류를 덮는 문법(단언·타입 서술·확정 대입·오버로드·앰비언트 선언·`@ts-expect-error`·인라인 eslint 주석) 금지,
  타입 정보를 쓰는 규칙, 상대 경로 import 금지
- `layers(packageDir, list)` — 폴더 단위 레이어 경계(기본 금지, 허용만 나열)
- `domainLayers(packageDir, modules)`·`noTopLevelEffects` — 명사 모듈 도메인과 부작용 없는 코드

## 쓰기

ESLint 설정은 TypeScript여야 하고 `jiti`가 있어야 한다(이 패키지는 빌드하지 않은 `index.mts`를 낸다).

```ts
// eslint.config.ts
import { base, layers } from 'lint-guardrails';

export default [
  ...base,
  ...layers(import.meta.dirname, [
    { type: 'domain', pattern: 'src/domain' },
    { type: 'ui', pattern: 'src/ui', allow: ['domain'] },
  ]),
];
```

저장소마다 다른 것(레이어·모듈 표)은 그 저장소의 설정 파일에 두고, 공통 규칙은 고치거나 끄지 않는다 — 맞지 않으면 이 패키지에서
고친다.
