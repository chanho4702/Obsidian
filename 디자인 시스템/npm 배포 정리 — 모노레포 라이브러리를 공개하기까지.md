# npm 배포 정리 — 모노레포 라이브러리를 공개하기까지

> 2026-07-06, Chanho Design System(@chanho/tokens, @chanho/react) 첫 배포 과정 기록.
> 블로그 초안용 — 실제로 밟은 순서와 실제로 겪은 함정 중심.

## 전체 그림

```
개발(src 직접 소비) ──┐
                      ├─ publishConfig로 이원화
배포(dist 소비)   ────┘

빌드(tsup / vite lib) → 버전(changesets) → pack 검증(tarball)
→ 소비자 앱 검증 → npm 계정/스코프 → publish
```

핵심 아이디어: **배포는 "빌드 산출물을 파는 것"** 이다. 개발할 때 편한 구조(소스 직접 import)와 소비자가 받는 구조(컴파일된 JS + 타입 + CSS)는 다르며, 이 둘을 한 package.json에서 공존시키는 장치가 `publishConfig`다.

## 1. exports 이원화 — 개발 경로와 배포 경로

```jsonc
{
  "exports": { ".": "./src/index.ts" },        // 개발: 워크스페이스 내부는 TS 소스 직접
  "files": ["dist"],                             // tarball에 dist만
  "publishConfig": {                             // pnpm이 pack/publish 시점에 덮어씀
    "access": "public",
    "exports": {
      ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" },
      "./styles.css": "./dist/styles.css"
    }
  }
}
```

- 테스트·Storybook은 계속 src를 보므로 **빌드 없이 개발 루프가 돈다**
- `pnpm publish`가 tarball을 만들 때 publishConfig의 exports가 본 exports를 교체
- `files: ["dist"]` — README/LICENSE/package.json은 자동 포함되므로 이걸로 끝

## 2. 빌드 — 패키지 성격 따라 도구가 다르다

**순수 TS 패키지(tokens)** → tsup 한 줄:
```
tsup src/index.ts --format esm --dts --clean && tsx src/build.ts   # CSS 생성은 자체 스크립트
```
⚠️ `--clean`이 dist를 비우므로 **CSS 생성을 tsup 뒤에** 놓아야 한다.

**CSS Modules 쓰는 React 패키지** → tsup으로는 안 되고 **Vite 라이브러리 모드**:
```ts
// vite.config.ts
build: {
  lib: { entry: "src/index.ts", formats: ["es"], fileName: "index", cssFileName: "styles" },
  rollupOptions: { external: ["react", "react-dom", "react/jsx-runtime", "radix-ui", "@chanho/tokens"] },
}
// + vite-plugin-dts({ tsconfigPath: "./tsconfig.build.json", rollupTypes: true })
```
- **external이 생명**: react를 번들에 넣으면 소비자 앱에 React가 두 개 생겨 훅이 터진다. peerDependencies에 있는 것 + 런타임 의존성은 전부 external
- CSS Modules는 클래스명이 해시로 컴파일되고 CSS는 `styles.css` 한 파일로 추출됨 → 소비자는 JS import + CSS import 두 줄
- 타입은 `tsconfig.build.json`에서 **테스트·스토리 파일을 exclude** 해야 d.ts에 안 섞인다
- `"sideEffects": ["**/*.css"]` — 이거 없으면 트리셰이킹이 CSS import를 날려버린다

## 3. 버전 관리 — Changesets

```bash
pnpm add -w -D @changesets/cli && pnpm changeset init
pnpm changeset          # 마크다운으로 변경 기록 (패키지별 major/minor/patch 선택)
pnpm changeset version  # package.json 버전 + CHANGELOG.md 자동 생성
```

- config에서 `access: "public"`, `baseBranch`, private 앱은 `ignore: ["docs"]`
- 모노레포 내부 의존(`workspace:*`)은 publish 때 pnpm이 실제 버전(`0.1.0`)으로 치환해준다
- ⚠️ `ignore`에 **존재하지 않는 패키지명**을 넣으면 에러

## 4. 배포 전 검증 — 이 두 개가 진짜 테스트다

**① tarball 내용물 검사** — 소스나 테스트가 새는지:
```bash
pnpm pack --pack-destination ../../artifacts
tar -tzf artifacts/chanho-react-0.1.0.tgz    # dist + README + LICENSE + package.json만 있어야 함
```

**② 소비자 앱 스모크** — 레지스트리 없이 tarball을 직접 설치:
```jsonc
// examples/consumer/package.json (워크스페이스 밖 독립 앱)
"dependencies": {
  "@chanho/react": "file:../../artifacts/chanho-react-0.1.0.tgz",
  "@chanho/tokens": "file:../../artifacts/chanho-tokens-0.1.0.tgz"
}
```
⚠️ 함정 셋:
- react tarball의 의존성 `@chanho/tokens@0.1.0`은 **아직 레지스트리에 없어서** 해석 실패 → `overrides`로 tokens도 tarball로 강제
- pnpm 11은 `package.json`의 `pnpm.overrides`를 안 읽는다 → **자체 `pnpm-workspace.yaml`** 에 넣어야 함 (부모 모노레포 워크스페이스와의 격리도 겸함)
- 검증은 `pnpm install && tsc --noEmit && vite build` 3종 — **tsc가 번들된 d.ts를, vite build가 exports 맵과 CSS를** 각각 검증한다

## 5. npm 계정·스코프 — 배포보다 여기서 막힌다

- `@chanho/react`의 `@chanho`는 아무나 못 쓴다 — **npm에서 그 이름의 사용자 또는 조직(Organization)을 소유**해야 함
- 조직은 npmjs.com → Add Organization에서 무료 생성 가능 (**공개 패키지는 완전 무료**, 비공개만 유료 $7/월)
- 참고: **GitHub Packages는 스코프=GitHub 계정명 강제**라 `@chanho`를 쓰려면 GitHub에도 chanho 조직이 필요 → 그래서 npmjs 선택
- 공개 배포 후 72시간 지나면 unpublish 제한 — 실수하면 지우지 말고 새 버전으로 고친다

## 6. publish — 마지막 함정 두 개

```bash
npm login                # 브라우저 인증
pnpm -r build && pnpm -r publish --access public --publish-branch main
```

- ⚠️ **pnpm publish의 기본 허용 브랜치는 master** — main 브랜치면 `--publish-branch main` 필요
- ⚠️ **403 "Two-factor authentication ... required"** — npm은 이제 2FA 없인 publish를 막는다. 해결 둘 중 하나:
  - 계정에 2FA 활성화(npmjs.com → Account → Two-Factor Authentication, 인증 앱) 후 `--otp=123456`으로 publish
  - 또는 Granular Access Token(publish 권한 + bypass 2FA) 발급해서 `.npmrc`에 설정
- private 패키지(docs 앱 등)는 `pnpm -r publish`가 알아서 건너뜀

## 배포 후

- `npm view @chanho/react version`으로 확인
- 소비자 앱의 `file:` 의존을 `^0.1.0`으로 바꿔 레지스트리 설치로 재검증
- package.json에 `repository`(+ `directory`), `homepage`, `bugs`를 넣으면 npm 페이지가 GitHub와 연결됨 — **배포 전에** 넣는 게 좋다 (우리는 이거 까먹어서 0.1.0 publish 직전에 급히 추가)

## 요약 체크리스트

- [ ] exports 이원화 (개발 src / publishConfig dist)
- [ ] 빌드: external 완비, d.ts에서 테스트 제외, sideEffects CSS
- [ ] `tar -tzf`로 tarball 내용물 확인
- [ ] 워크스페이스 밖 소비자 앱에서 tarball 설치 + tsc + build
- [ ] npm 조직/스코프 확보, repository 필드
- [ ] 2FA 켜기, `--publish-branch`, `--access public`
