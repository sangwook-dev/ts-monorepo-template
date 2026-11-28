# lint-guardrails

규칙을 문서가 아니라 lint로 강제하는 공통 ESLint 설정이다. 결정과 이유는
[ts-monorepo-template의 ADR](https://github.com/sangwook-dev/ts-monorepo-template/tree/main/docs/adr)에 있다.

- `base` — 타입 오류를 덮는 문법(단언·타입 서술·확정 대입·오버로드·앰비언트 선언·`@ts-expect-error`·인라인 eslint 주석) 금지,
  타입 정보를 쓰는 규칙, 상대 경로 import 금지
- `layers(packageDir, list)` — 폴더 단위 레이어 경계(기본 금지, 허용만 나열)
- `domainLayers(packageDir, modules)`·`noTopLevelEffects` — 명사 모듈 도메인과 부작용 없는 코드

## 쓰기

배포 판은 JS(`dist/index.mjs`)와 타입(`dist/index.d.mts`)이라 설정 파일은 JS·TS 어느 쪽이든 된다. 설정 파일을 TypeScript(`eslint.config.ts`)로 쓰면 ESLint가 그것을 읽을 `jiti`가 필요하다.

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

저장소마다 다른 것(레이어·모듈 표)은 그 저장소의 설정 파일에 둔다. 공통 규칙은 끄거나 느슨하게 하지 않는다 — 맞지 않으면 이
패키지에서 고친다. 저장소가 공통 규칙보다 엄격하게 새로 정한 규칙은 그 저장소 설정의 표 하나에 모아 `base` 뒤에 건다(저장소가 새로
정한 규칙이 이긴다):

```ts
import type { Linter } from 'eslint';
import { defineConfig } from 'eslint/config';
import { base } from 'lint-guardrails';
import tseslint from 'typescript-eslint';

// 공통 규칙보다 엄격하게 새로 정한 규칙 — base 뒤에 건다
const STRICTER_THAN_SHARED = {
  // 옵션까지 적는다 — 심각도만 적으면 ESLint가 base의 옵션(`_` 이름 허용)을 그대로 둔다
  '@typescript-eslint/no-unused-vars': ['error', {}],
} satisfies Linter.RulesRecord;

// extends는 defineConfig 안에서만 풀린다
export default defineConfig([
  // 더 엄격한 프리셋(typescript-eslint의 strictTypeChecked·stylisticTypeChecked)은 base 앞에 둔다 — 뒤에 두면 프리셋이 base가
  // 고른 옵션을 프리셋 값으로 되돌린다(strict는 `@ts-expect-error` 금지, stylistic은 단언 금지). base가 타입 정보를 켜는 파일(선언 파일을 뺀 TS)로 범위를 맞춘다 — 범위 없이
  // 펼치면 JS 파일·선언 파일에도 타입 정보 규칙이 켜져 lint가 멈춘다
  {
    files: ['**/*.{ts,tsx,mts,cts}'],
    ignores: ['**/*.d.{ts,mts,cts}', '**/*.d.*.ts'],
    extends: [tseslint.configs.strictTypeChecked],
  },
  ...base,
  { rules: STRICTER_THAN_SHARED },
]);
```
