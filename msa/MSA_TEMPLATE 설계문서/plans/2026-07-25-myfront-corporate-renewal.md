---
tags: [msa, myfront, 리뉴얼, 옵시디언, plan]
작성일: 2026-07-25
상태: 실행 대기
---

# myFront 기업형 리뉴얼 + 옵시디언 노트 게시 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** myFront를 원페이지 개인 포트폴리오에서 제품/기술/소개/연락으로 나뉜 1인 기술 스튜디오형 멀티페이지 사이트로 재편하고, 옵시디언 볼트의 `MSA_TEMPLATE 정리` 00~19 노트를 빌드타임에 동기화해 기술 페이지의 본문으로 게시한다.

**Architecture:** 순수 Node 스크립트가 볼트의 마크다운을 화이트리스트로 걸러 `src/site/content/notes/`에 정규화해 떨군다(산출물은 커밋되어 볼트 없는 CI에서도 빌드된다). 화면은 `src/site/ui/`의 헤어라인 프리미티브 5종만 조합해 만들고, 모든 텍스트 데이터는 `src/site/content/`로 분리해 페이지에서 상수를 없앤다. 라우팅은 기존 `src/main.tsx`의 `createBrowserRouter` 배열에 추가하며, 기존 개발용 라우트는 삭제하지 않고 푸터로 격리한다.

**Tech Stack:** React 19 · MUI v9 · Vite 6 · TypeScript 5.7 (strict) · react-markdown / remark-gfm (신규 2개) · Node `node --test` (내장, 신규 의존성 아님)

## Global Constraints

- 패키지 매니저는 **npm** (형제 리포 wiki-front/alm-front는 pnpm이지만 myFront는 npm이다).
- **MUI v9**: `Typography`/`Stack`/`styled(Stack)`에 `fontWeight`·`alignItems`·`justifyContent` 등 시스템 스타일 props를 직접 넘기면 빌드 에러(TS2769/TS2322). 전부 `sx={{ ... }}` 안으로. `Stack`의 `direction`/`spacing`/`useFlexGap`은 직접 props로 정상.
- **Grid**는 `size={{ xs: 12 }}` 형식. `item xs` 아님.
- 타입 게이트는 `npm run build`(= `tsc -b && vite build`). `strict` + `noUnusedLocals` + `noUnusedParameters`.
- **테스트 러너·린터·포매터가 없다.** 스크립트 로직은 Node 내장 `node --test`로 TDD하고, React 화면은 빌드 통과 + 명시된 수동 확인 절차가 게이트다. 새 테스트 러너를 도입하지 않는다.
- 팔레트·서체 변경 금지: 스틸 블루 `#1B66C9`(`brand[500]`) / 다크 `#3F84DC`(`brand[400]`), 서체 Pretendard. `themePrimitives.ts`를 수정하지 않는다.
- `src/context/templates/` 아래는 **수정 금지**(MUI 공식 템플릿 원본). 가져다 쓰기만 한다.
- **미확인 사실은 쓰지 않는다.** 제품 스펙 행의 값을 리포에서 확인하지 못하면 그 행을 통째로 뺀다. 빈 값·"미정"·"준비 중"을 렌더하지 않는다.
- 아이콘은 `@mui/icons-material`만 사용. 이모지를 아이콘으로 쓰지 않는다.
- **`src/site/content/*` 는 ui 배럴(`../ui`)을 import 하지 않는다.** 타입은 `../types`,
  `MONO` 같은 상수는 `../ui/tokens` 에서 **직접** 가져온다. 배럴을 경유하면 `NoteBody` 를 통해
  `react-markdown` 이 콘텐츠 모듈에 유입될 수 있다 — 트리쉐이킹이 걷어주길 기대하지 않는다.
  (페이지 컴포넌트는 이미 프리미티브를 쓰므로 배럴 경유가 정상이다.)
- 커밋 메시지: `feat(scope): 한국어 설명` / `chore(scope): ...` 형식.

---

## File Structure

**신규**

| 경로 | 책임 |
|------|------|
| `scripts/notes/transform.mjs` | 볼트 마크다운 → 사이트 마크다운 순수 변환 함수 모음. I/O 없음 |
| `scripts/notes/transform.test.mjs` | 위 함수들의 `node --test` 테스트 |
| `scripts/sync-notes.mjs` | 파일시스템 I/O + CLI. transform을 호출해 산출물을 쓴다 |
| `src/site/types.ts` | `SpecRow` · `StatItem` · `NoteMeta` 공용 타입. ui 와 content 양쪽이 여기서 가져온다(순환 방지) |
| `src/site/content/notes/index.generated.ts` | 노트 인덱스 (생성물, 커밋됨) |
| `src/site/content/notes/NN.md` × 20 | 노트 본문 (생성물, 커밋됨) |
| `src/site/content/notes.ts` | 노트 조회 API — 인덱스 + `import.meta.glob` 본문 로더 |
| `src/site/content/products.ts` | 제품 스펙 데이터 |
| `src/site/content/capabilities.ts` | 역량 4개 |
| `src/site/content/profile.ts` | 커리어·케이스스터디·수상·연락처 |
| `src/site/content/stack.ts` | 기술 스택 그룹 |
| `src/site/content/index.ts` | 배럴 |
| `src/site/ui/SectionLabel.tsx` | `SEC.02 / PRODUCTS` 모노 라벨 |
| `src/site/ui/GridSection.tsx` | 라벨 컬럼 + 콘텐츠 2열 셸 |
| `src/site/ui/SpecTable.tsx` | key/value 스펙 테이블 |
| `src/site/ui/StatBar.tsx` | 수치 4칸 |
| `src/site/ui/HairlineCard.tsx` | 그림자 없는 아웃라인 카드 |
| `src/site/ui/NoteBody.tsx` | 마크다운 렌더러 (react-markdown 래핑) |
| `src/site/ui/slug.ts` | 헤딩 → id 슬러그 함수. NoteBody 와 목차가 **같은 함수**를 써야 앵커가 맞는다 |
| `src/site/ui/tokens.ts` | 프리미티브·페이지가 공유하는 상수(`MONO` 모노스페이스 스택). 문자열 복제 금지 |
| `src/site/ui/index.ts` | 배럴 |
| `src/site/components/SiteHeader.tsx` | GNB (제품/기술/소개 + 문의 CTA, 활성 표시) |
| `src/site/components/SiteFooter.tsx` | 푸터 (개발 도구 그룹 격리) |
| `src/site/components/SitePage.tsx` | 공통 셸 (AppTheme + CssBaseline + Header + Footer) |
| `src/site/pages/ProductsPage.tsx` | `/products` |
| `src/site/pages/ProductDetailPage.tsx` | `/products/:slug` |
| `src/site/pages/TechPage.tsx` | `/tech` |
| `src/site/pages/NotesIndexPage.tsx` | `/tech/notes` |
| `src/site/pages/NoteDetailPage.tsx` | `/tech/notes/:id` |
| `src/site/pages/AboutPage.tsx` | `/about` |
| `src/site/pages/ContactPage.tsx` | `/contact` |
| `src/site/pages/ServiceRedirect.tsx` | `/services/:slug` → `/tech#cap-<slug>` |

**수정**

| 경로 | 변경 |
|------|------|
| `src/pages/Home.tsx` | 게이트웨이형 랜딩으로 재작성. 인라인 상수는 content로 이동 |
| `src/main.tsx` | 신규 라우트 등록, 구 상세 라우트 교체 |
| `package.json` | 의존성 3개 + `sync:notes` 스크립트 |

**삭제**

| 경로 | 사유 |
|------|------|
| `src/pages/landing/content.ts` | `src/site/content/`로 분해 이동 |
| `src/pages/landing/DetailPages.tsx` | "준비 중" 플레이스홀더. 실제 상세 페이지로 대체 |
| `src/pages/landing/LandingHeader.tsx` | `SiteHeader`로 대체 |

---

## Task 1: 노트 변환 순수 함수

**Files:**
- Create: `scripts/notes/transform.mjs`
- Test: `scripts/notes/transform.test.mjs`

**Interfaces:**
- Consumes: 없음 (첫 태스크)
- Produces:
  - `isWhitelisted(filename: string): boolean`
  - `noteIdOf(filename: string): string | null`
  - `parseFrontmatter(raw: string): { meta: Record<string, string | string[]>, body: string }`
  - `extractTitle(body: string, filename: string): { title: string, body: string }`
  - `transformWikiLinks(body: string, resolve: (target: string) => string | null): { body: string, broken: string[] }`
  - `transformCallouts(body: string): string`

- [ ] **Step 1: 실패하는 테스트를 쓴다**

`scripts/notes/transform.test.mjs`:

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import {
  isWhitelisted,
  noteIdOf,
  parseFrontmatter,
  extractTitle,
  transformWikiLinks,
  transformCallouts,
} from './transform.mjs';

test('화이트리스트는 두 자리 번호로 시작하는 파일만 통과시킨다', () => {
  assert.equal(isWhitelisted('00 개요 — 전체 구조.md'), true);
  assert.equal(isWhitelisted('19 게이트웨이 평가 + rate-limit XFF 수정 (2026-07-20).md'), true);
  assert.equal(isWhitelisted('auth-server 면접 방어 시트 — 쉬운 말 버전 (2026-07-19).md'), false);
  assert.equal(isWhitelisted('내 목표.md'), false);
  assert.equal(isWhitelisted('7 한자리.md'), false);
  assert.equal(isWhitelisted('00 확장자없음'), false);
});

test('노트 id 는 앞 두 자리다', () => {
  assert.equal(noteIdOf('07 보안 감사 + 하드닝 (2026-07-02).md'), '07');
  assert.equal(noteIdOf('내 목표.md'), null);
});

test('frontmatter 를 파싱하고 본문에서 떼어낸다', () => {
  const raw = [
    '---',
    'tags: [msa, template, 개요]',
    '작성일: 2026-06-30',
    '상태: 정리본',
    '---',
    '',
    '# 제목',
    '본문',
  ].join('\n');
  const { meta, body } = parseFrontmatter(raw);
  assert.deepEqual(meta.tags, ['msa', 'template', '개요']);
  assert.equal(meta['작성일'], '2026-06-30');
  assert.equal(meta['상태'], '정리본');
  assert.equal(body.startsWith('# 제목'), true);
});

test('CRLF 문서도 frontmatter 를 정상 파싱한다', () => {
  // 실제 볼트 22개 중 7개가 CRLF 다. \r 을 안 털면 meta 가 통째로 빈 객체가 된다.
  const raw = '---\r\ntags: [msa, 인증]\r\n작성일: 2026-07-19\r\n상태: 정리본\r\n---\r\n\r\n# 제목\r\n본문';
  const { meta, body } = parseFrontmatter(raw);
  assert.deepEqual(meta.tags, ['msa', '인증']);
  assert.equal(meta['작성일'], '2026-07-19');
  assert.equal(meta['상태'], '정리본');
  assert.equal(body.startsWith('# 제목'), true);
});

test('frontmatter 가 없으면 원문을 그대로 본문으로 돌려준다', () => {
  const { meta, body } = parseFrontmatter('# 제목\n본문');
  assert.deepEqual(meta, {});
  assert.equal(body, '# 제목\n본문');
});

test('본문 첫 H1 을 제목으로 뽑고 본문에서 제거한다', () => {
  const r = extractTitle('# 전체 구조 개요\n\n본문', '00 개요.md');
  assert.equal(r.title, '전체 구조 개요');
  assert.equal(r.body.trim(), '본문');
});

test('H1 이 없으면 파일명에서 번호와 확장자를 뗀 것이 제목이다', () => {
  const r = extractTitle('본문만 있다', '03 board-service (샘플).md');
  assert.equal(r.title, 'board-service (샘플)');
  assert.equal(r.body, '본문만 있다');
});

test('화이트리스트 안 위키링크는 내부 링크가 된다', () => {
  const resolve = (t) => (t === '05 API 게이트웨이 설계' ? '05' : null);
  const r = transformWikiLinks('앞 [[05 API 게이트웨이 설계]] 뒤', resolve);
  assert.equal(r.body, '앞 [05 API 게이트웨이 설계](/tech/notes/05) 뒤');
  assert.deepEqual(r.broken, []);
});

test('별칭과 헤딩 앵커를 처리한다', () => {
  const resolve = (t) => (t === '05 API 게이트웨이 설계' ? '05' : null);
  const r = transformWikiLinks('[[05 API 게이트웨이 설계#라우팅|게이트웨이]]', resolve);
  assert.equal(r.body, '[게이트웨이](/tech/notes/05)');
});

test('별칭 없이 앵커만 있으면 라벨에서 앵커를 뗀다', () => {
  const resolve = (t) => (t === '05 API 게이트웨이 설계' ? '05' : null);
  const r = transformWikiLinks('[[05 API 게이트웨이 설계#라우팅]]', resolve);
  assert.equal(r.body, '[05 API 게이트웨이 설계](/tech/notes/05)');
});

test('화이트리스트 밖 위키링크는 일반 텍스트로 평탄화하고 보고한다', () => {
  const r = transformWikiLinks('앞 [[내 목표]] 뒤', () => null);
  assert.equal(r.body, '앞 내 목표 뒤');
  assert.deepEqual(r.broken, ['내 목표']);
});

test('콜아웃은 라벨이 굵게 붙은 인용문이 된다', () => {
  const input = ['> [!warning] 주의', '> 본문 줄', '', '다음 문단'].join('\n');
  const out = transformCallouts(input);
  assert.equal(
    out,
    ['> **[WARNING] 주의**', '>', '> 본문 줄', '', '다음 문단'].join('\n'),
  );
});

test('제목 없는 콜아웃도 라벨만 붙인다', () => {
  assert.equal(transformCallouts('> [!note]\n> 본문'), '> **[NOTE]**\n>\n> 본문');
});

test('콜아웃이 아닌 인용문은 건드리지 않는다', () => {
  assert.equal(transformCallouts('> 그냥 인용'), '> 그냥 인용');
});
```

- [ ] **Step 2: 테스트를 돌려 실패를 확인한다**

Run: `node --test scripts/notes/transform.test.mjs`
Expected: FAIL — `Cannot find module '.../scripts/notes/transform.mjs'`

- [ ] **Step 3: 최소 구현을 쓴다**

`scripts/notes/transform.mjs`:

```js
// 옵시디언 볼트 마크다운 → 사이트용 마크다운. 순수 함수만 둔다(I/O 는 sync-notes.mjs).

const NUMBERED = /^(\d\d)\s.+\.md$/;

/** 파일명이 "NN 제목.md" 형태인가. 개인 노트를 구조적으로 배제하는 유일한 관문. */
export function isWhitelisted(filename) {
  return NUMBERED.test(filename);
}

/** 파일명 앞 두 자리 번호. 화이트리스트 밖이면 null. */
export function noteIdOf(filename) {
  const m = filename.match(NUMBERED);
  return m ? m[1] : null;
}

/** YAML frontmatter 의 평평한 key: value 와 [a, b] 배열만 다룬다(yaml 의존성 없음). */
export function parseFrontmatter(raw) {
  // CRLF \uC815\uADDC\uD654\uB294 \uD544\uC218\uB2E4. JS \uC815\uADDC\uC2DD\uC5D0\uC11C \r \uC740 \uC904\uC885\uACB0\uC790\uB77C `$` \uAC00 \uADF8 \uC55E\uC5D0\uC11C \uBA48\uCD94\uACE0,
  // `^([^:]+):\s*(.*)$` \uAC00 CR \uB85C \uB05D\uB098\uB294 \uC904\uC5D0\uC11C \uD1B5\uC9F8\uB85C null \uC744 \uBC18\uD658\uD574 meta \uAC00 \uC870\uC6A9\uD788 \uBE44\uC5B4\uBC84\uB9B0\uB2E4.
  // \uC2E4\uC81C \uBCFC\uD2B8 22\uAC1C \uC911 7\uAC1C\uAC00 CRLF \uB2E4. \uC774\uD6C4 \uB2E8\uACC4\uB294 \uBAA8\uB450 \uC774 \uD568\uC218 \uCD9C\uB825\uC744 \uBC1B\uC73C\uBBC0\uB85C \uC5EC\uAE30\uC11C \uD134\uB2E4.
  const normalized = raw.replace(/^\uFEFF/, '').replace(/\r\n/g, '\n');
  if (!normalized.startsWith('---')) return { meta: {}, body: normalized };
  const end = normalized.indexOf('\n---', 3);
  if (end === -1) return { meta: {}, body: normalized };

  const head = normalized.slice(3, end);
  // 닫는 --- 뒤의 남은 줄바꿈과 빈 줄을 전부 턴다. 한 줄만 지우면 본문이 \n 으로 시작한다.
  const body = normalized.slice(end + 4).replace(/^[^\n]*\r?\n/, '').replace(/^\s*\r?\n/, '');
  const meta = {};
  for (const line of head.split('\n')) {
    const m = line.match(/^([^:]+):\s*(.*)$/);
    if (!m) continue;
    const key = m[1].trim();
    const value = m[2].trim();
    if (value.startsWith('[') && value.endsWith(']')) {
      meta[key] = value
        .slice(1, -1)
        .split(',')
        .map((s) => s.trim())
        .filter(Boolean);
    } else if (value) {
      meta[key] = value;
    }
  }
  return { meta, body };
}

/** 본문 첫 H1 을 제목으로 승격하고 본문에서 뺀다. 페이지 헤딩과 중복 h1 을 만들지 않기 위함. */
export function extractTitle(body, filename) {
  const m = body.match(/^#\s+(.+?)\s*$/m);
  if (m && body.slice(0, m.index).trim() === '') {
    return { title: m[1].trim(), body: body.slice(m.index + m[0].length).replace(/^\r?\n/, '') };
  }
  return { title: filename.replace(/^\d\d\s/, '').replace(/\.md$/, ''), body };
}

/**
 * [[대상]] / [[대상|별칭]] / [[대상#앵커]] 처리.
 * resolve 가 노트 id 를 주면 내부 링크, null 이면 텍스트로 평탄화한다(끊긴 링크 0).
 */
export function transformWikiLinks(body, resolve) {
  const broken = [];
  const out = body.replace(/\[\[([^\]]+)\]\]/g, (_all, inner) => {
    const [linkPart, alias] = inner.split('|').map((s) => s.trim());
    const target = linkPart.split('#')[0].trim();
    // 별칭이 없으면 앵커를 뗀 target 을 라벨로 쓴다. linkPart 를 쓰면 `#섹션` 이 링크 글자에 남는다.
    const label = alias || target;
    const id = resolve(target);
    if (id) return `[${label}](/tech/notes/${id})`;
    broken.push(target);
    return label;
  });
  return { body: out, broken };
}

/**
 * 옵시디언 콜아웃 -> 라벨이 굵게 붙은 표준 인용문.
 * MUI Alert 매핑은 의도적으로 하지 않는다 — 커스텀 AST 없이 안전하게 렌더되고,
 * 타입이 색이 아니라 글자로 남아 색 단독 정보전달 문제도 없다.
 */
export function transformCallouts(body) {
  // 제목 부분은 [ \t]* 로 받는다 — \s* 는 \n 을 삼켜 다음 줄까지 라벨 안으로 끌어온다.
  return body.replace(
    /^>[ \t]*\[!(\w+)\][ \t]*(.*)$/gm,
    (_all, type, title) => {
      const label = title.trim()
        ? `**[${type.toUpperCase()}] ${title.trim()}**`
        : `**[${type.toUpperCase()}]**`;
      return `> ${label}\n>`;
    },
  );
}
```

- [ ] **Step 4: 테스트를 돌려 통과를 확인한다**

Run: `node --test scripts/notes/transform.test.mjs`
Expected: PASS — `# pass 14`, `# fail 0`

- [ ] **Step 5: 커밋**

```bash
git add scripts/notes/transform.mjs scripts/notes/transform.test.mjs
git commit -m "feat(notes): 옵시디언 마크다운 변환 순수 함수 + 테스트"
```

---

## Task 2: 동기화 CLI 와 노트 산출물

**Files:**
- Create: `scripts/sync-notes.mjs`
- Create: `src/site/types.ts`
- Generate + commit: `src/site/content/notes/index.generated.ts`, `src/site/content/notes/NN.md` × 20
- Modify: `package.json` (scripts 에 `sync:notes` 추가)

**Interfaces:**
- Consumes: Task 1의 `transform.mjs` 전 함수
- Produces:
  - `NoteMeta = { id: string; title: string; tags: string[]; date: string; status: string }`
  - `SpecRow = { label: string; value: string }`
  - `StatItem = { value: string; label: string }`
  - `export const noteIndex: NoteMeta[]` (id 오름차순)
  - `src/site/content/notes/<id>.md` 본문 파일

- [ ] **Step 1: 공용 타입 파일을 만든다**

`src/site/types.ts`. `ui`와 `content`가 서로를 import 하면 순환이 생기므로,
양쪽이 공유하는 타입은 여기 한 곳에만 둔다.

```ts
/** SpecTable 한 행. 값이 빈 문자열이면 렌더하지 않는다. */
export interface SpecRow {
  label: string;
  value: string;
}

/** StatBar 한 칸. */
export interface StatItem {
  value: string;
  label: string;
}

/** 엔지니어링 노트 메타. scripts/sync-notes.mjs 가 index.generated.ts 를 이 타입으로 생성한다. */
export interface NoteMeta {
  /** 볼트 파일명 앞 두 자리. URL 세그먼트이자 본문 파일명. */
  id: string;
  title: string;
  tags: string[];
  /** 볼트 frontmatter 의 작성일 (YYYY-MM-DD). 없으면 빈 문자열. */
  date: string;
  /**
   * 배지용으로 줄인 상태 라벨(`statusLabel` 산출물, 최대 24자). 없으면 빈 문자열.
   * 볼트 원문 상태는 최대 237자에 커밋 SHA·마크다운·위키링크가 섞여 있어 싣지 않는다.
   */
  status: string;
}
```

- [ ] **Step 1.5: 제목·상태 정규화 함수를 `scripts/notes/transform.mjs` 에 추가한다**

실제 볼트 데이터를 보고 추가된 단계다. 두 가지 실측 문제가 있다.

1. **제목에 번호가 중복된다.** 20편 중 11편의 H1 이 `15 — ALM·Wiki 백엔드 …` 처럼 번호로
   시작한다. 화면은 `NO.15` 를 모노 라벨로 따로 렌더하므로 그대로 두면 번호가 두 번 나온다.
2. **`상태` 값이 배지로 쓸 수 없다.** 길이가 3자에서 **237자**까지 분포하고, 커밋 SHA·
   `**굵게**`·`[[위키링크]]` 가 섞여 있다. 특히 15번은 `[[17 Wave B …]]` 를 통째로 품고 있어
   그대로 렌더하면 링크 문법이 화면에 노출된다.

`transform.mjs` 끝에 다음 두 함수를 추가한다(순수 함수 계약 유지 — I/O 없음):

```js
/**
 * H1 앞에 붙은 노트 번호와 구분자를 뗀다. 화면이 NO.15 를 따로 렌더하므로 중복을 막는다.
 *
 * 두 겹으로 잠근다 — 둘 중 하나만 어긋나도 원문을 그대로 돌려준다.
 *  1) 앞 두 자리 **뒤에 공백이나 대시가 와야** 한다. `2026 회고` 가 `26 회고` 로 잘리는 것을 막는다.
 *  2) 그 두 자리가 **이 노트의 id 와 같아야** 한다. 남의 번호를 떼지 않는다.
 * 떼고 나서 남는 게 없으면(제목이 `00` 뿐) 원문을 유지한다 — 제목을 통째로 잃느니 중복이 낫다.
 */
export function stripNumberPrefix(title, id) {
  const m = title.match(/^(\d\d)(?=[\s—–-])\s*(?:[—–-]\s*)?([\s\S]*)$/);
  if (!m || m[1] !== id) return title.trim();
  return m[2].trim() || title.trim();
}

/**
 * frontmatter 의 `상태` 를 배지용 짧은 라벨로 줄인다.
 * 첫 구분자(괄호 · 가운뎃점 · 대시 · 플러스) 앞까지만 취하고 마크다운/위키링크 문법을 턴다.
 * 원문(커밋 SHA·잔여 작업 메모)은 사이트에 싣지 않는다 — 배지 자리에 들어갈 정보가 아니다.
 *
 * 구분자로 **시작하는** 값(`(진행중) 완료`)은 첫 조각이 빈 문자열이 되어 상태가 통째로
 * 사라진다. 그 경우 원문 전체를 정리해 쓴다 — 비어 있는 배지보다 긴 배지가 낫다.
 */
export function statusLabel(status) {
  const clean = (s) => s.replace(/\[\[.*?\]\]/g, '').replace(/\*\*/g, '').trim();
  const first = clean(status.split(/\s*\(|\s·\s|\s[—–-]\s|\s\+\s/)[0]);
  return (first || clean(status)).slice(0, 24);
}
```

`scripts/notes/transform.test.mjs` 에 테스트를 추가한다(import 목록에도 두 함수를 넣는다):

```js
test('제목 앞 번호와 구분자를 뗀다', () => {
  assert.equal(stripNumberPrefix('15 — ALM·Wiki 백엔드 요구사항', '15'), 'ALM·Wiki 백엔드 요구사항');
  assert.equal(stripNumberPrefix('05 API 게이트웨이 설계', '05'), 'API 게이트웨이 설계');
  assert.equal(stripNumberPrefix('번호 없는 제목', '07'), '번호 없는 제목');
});

test('의미 있는 숫자로 시작하는 제목은 건드리지 않는다', () => {
  // 두 자리 뒤에 공백/대시가 와야 번호로 본다. `2026` 은 `20` + `26` 으로 잘리면 안 된다.
  assert.equal(stripNumberPrefix('2026 회고', '20'), '2026 회고');
  // 남의 번호는 떼지 않는다.
  assert.equal(stripNumberPrefix('15 — 어떤 제목', '07'), '15 — 어떤 제목');
  // 떼면 아무것도 안 남는 제목은 원문을 유지한다. 중복이 제목 소실보다 낫다.
  assert.equal(stripNumberPrefix('00', '00'), '00');
});

test('상태를 배지용 짧은 라벨로 줄인다', () => {
  // 실제 볼트 15번 — 237자에 위키링크와 굵게 문법이 섞여 있다.
  assert.equal(
    statusLabel('설계 확정 + Wave A 완료(07-19) + **Wave B 완료(2026-07-21)** — 상세는 [[17 Wave B]] · 다음: Wave C'),
    '설계 확정',
  );
  assert.equal(statusLabel('구현 완료 (2026-07-01, gateway-server :8000)'), '구현 완료');
  assert.equal(statusLabel('구현·수정 완료 (커밋: my e02fdc2)'), '구현·수정 완료');
  assert.equal(statusLabel('완료 · 배포·E2E 검증 완료(2026-07-20)'), '완료');
  assert.equal(statusLabel('정리본'), '정리본');
});

test('구분자로 시작하는 상태도 라벨을 잃지 않는다', () => {
  // 첫 조각이 빈 문자열이 되는 입력. 예전 구현은 상태를 통째로 삼켰다.
  assert.equal(statusLabel('(진행중) 완료'), '(진행중) 완료');
  assert.equal(statusLabel('[[17 Wave B]]'), '');
  assert.equal(statusLabel(''), '');
});
```

Run: `node --test scripts/notes/transform.test.mjs`
Expected: `# pass 18`, `# fail 0`

- [ ] **Step 2: 동기화 스크립트를 쓴다**

`scripts/sync-notes.mjs`:

```js
#!/usr/bin/env node
// 옵시디언 볼트의 "MSA_TEMPLATE 정리" 00~19 노트를 사이트 콘텐츠로 동기화한다.
// 산출물은 커밋된다 — 볼트가 없는 CI 러너에서도 빌드가 성공해야 하기 때문.
//
// 사용: node scripts/sync-notes.mjs
//       OBSIDIAN_VAULT="D:/vault" node scripts/sync-notes.mjs

import { readdir, readFile, writeFile, mkdir, unlink } from 'node:fs/promises';
import { existsSync } from 'node:fs';
import path from 'node:path';
import { fileURLToPath } from 'node:url';
import {
  isWhitelisted,
  noteIdOf,
  parseFrontmatter,
  extractTitle,
  transformWikiLinks,
  transformCallouts,
  stripNumberPrefix,
  statusLabel,
} from './notes/transform.mjs';

const ROOT = path.resolve(path.dirname(fileURLToPath(import.meta.url)), '..');
const VAULT = process.env.OBSIDIAN_VAULT ?? 'C:/myBrain/내 로컬';
const SOURCE_DIR = path.join(VAULT, 'msa', 'MSA_TEMPLATE 정리');
const OUT_DIR = path.join(ROOT, 'src', 'site', 'content', 'notes');

async function main() {
  if (!existsSync(SOURCE_DIR)) {
    console.warn(`[sync-notes] 볼트를 찾지 못했습니다: ${SOURCE_DIR}`);
    console.warn('[sync-notes] 기존 산출물을 유지하고 종료합니다. (CI 에서는 정상)');
    return;
  }

  const all = await readdir(SOURCE_DIR);
  const skipped = all.filter((f) => f.endsWith('.md') && !isWhitelisted(f));
  const targets = all.filter(isWhitelisted).sort();

  if (targets.length === 0) throw new Error('[sync-notes] 화이트리스트에 걸린 노트가 0개입니다.');

  // 대상 → id 사전. 위키링크 해석에 쓴다(확장자 유무 양쪽 허용).
  const idByTarget = new Map();
  for (const file of targets) {
    const id = noteIdOf(file);
    idByTarget.set(file.replace(/\.md$/, ''), id);
    idByTarget.set(file, id);
  }
  const resolve = (target) => idByTarget.get(target) ?? idByTarget.get(`${target}.md`) ?? null;

  // 생성물만 지운다. `.md` 전체가 아니라 `NN.md` 형태만 — 이 폴더에 손으로 둔 README.md 같은
  // 파일이 경고도 없이 사라지면 안 된다. 볼트에서 삭제된 노트의 스테일 산출물은 이 규칙으로도 걷힌다.
  await mkdir(OUT_DIR, { recursive: true });
  for (const f of await readdir(OUT_DIR)) {
    if (/^\d\d\.md$/.test(f) || f === 'index.generated.ts') await unlink(path.join(OUT_DIR, f));
  }

  const index = [];
  const brokenAll = [];
  const missingMeta = [];
  const statusLost = [];

  for (const file of targets) {
    const id = noteIdOf(file);
    const raw = await readFile(path.join(SOURCE_DIR, file), 'utf8');
    const { meta, body: afterMeta } = parseFrontmatter(raw);
    const { title, body: afterTitle } = extractTitle(afterMeta, file);
    const { body: afterLinks, broken } = transformWikiLinks(afterTitle, resolve);
    const body = transformCallouts(afterLinks);

    const tags = Array.isArray(meta.tags) ? meta.tags : meta.tags ? [String(meta.tags)] : [];
    const date = typeof meta['작성일'] === 'string' ? meta['작성일'] : '';
    const rawStatus = typeof meta['상태'] === 'string' ? meta['상태'] : '';
    const status = rawStatus ? statusLabel(rawStatus) : '';
    // "상태가 원래 없던 노트" 와 "정규식이 상태를 통째로 삼킨 노트" 를 로그에서 구분한다.
    if (rawStatus && !status) statusLost.push(file);
    if (!tags.length || !date) missingMeta.push(file);
    broken.forEach((b) => brokenAll.push(`${file} → [[${b}]]`));

    await writeFile(path.join(OUT_DIR, `${id}.md`), `${body.trimEnd()}\n`, 'utf8');
    index.push({ id, title: stripNumberPrefix(title, id), tags, date, status });
  }

  const generated = [
    '// 생성 파일 — scripts/sync-notes.mjs 가 만든다. 직접 수정하지 말 것.',
    "import type { NoteMeta } from '../../types';",
    '',
    'export const noteIndex: NoteMeta[] = [',
    ...index.map((n) => `  ${JSON.stringify(n)},`),
    '];',
    '',
  ].join('\n');
  await writeFile(path.join(OUT_DIR, 'index.generated.ts'), generated, 'utf8');

  console.log(`[sync-notes] 노트 ${index.length}개 동기화 완료 → ${OUT_DIR}`);
  if (skipped.length) console.log(`[sync-notes] 화이트리스트 제외 ${skipped.length}개: ${skipped.join(', ')}`);
  if (missingMeta.length) console.warn(`[sync-notes] 경고 — frontmatter 누락: ${missingMeta.join(', ')}`);
  if (statusLost.length) console.warn(`[sync-notes] 경고 — 상태 라벨이 비었다(원문은 있음): ${statusLost.join(', ')}`);
  if (brokenAll.length) {
    console.warn(`[sync-notes] 경고 — 평탄화된 외부 위키링크 ${brokenAll.length}건:`);
    brokenAll.forEach((b) => console.warn(`  - ${b}`));
  }
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

- [ ] **Step 3: package.json 에 스크립트를 추가한다**

`package.json`의 `scripts`를 다음으로 바꾼다:

```json
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "sync:notes": "node scripts/sync-notes.mjs",
    "test:scripts": "node --test scripts/notes/transform.test.mjs"
  },
```

- [ ] **Step 4: 동기화를 실행한다**

Run: `npm run sync:notes`
Expected: `[sync-notes] 노트 20개 동기화 완료 → ...src/site/content/notes`
그리고 `화이트리스트 제외 2개: auth-server RT 하드닝 ..., auth-server 면접 방어 시트 ...`

`평탄화된 외부 위키링크` 경고는 나올 수 있다 — 그건 정상(볼트 내 비공개 노트를 가리키는 링크가
텍스트로 내려앉았다는 뜻). **`frontmatter 누락` 경고가 나오면** 해당 볼트 파일의 frontmatter를
고친 뒤 다시 실행한다.

- [ ] **Step 5: 산출물을 눈으로 확인한다**

Run: `ls src/site/content/notes/` → `00.md` … `19.md` + `index.generated.ts` (총 21개)
Run: `head -5 src/site/content/notes/00.md` → frontmatter 와 H1 이 **없어야** 한다
Run: `grep -c "\[\[" src/site/content/notes/*.md` → 전부 `0` (남은 위키링크 없음)

- [ ] **Step 6: 커밋**

```bash
git add scripts/sync-notes.mjs src/site/types.ts src/site/content/notes package.json
git commit -m "feat(notes): 볼트 동기화 CLI + 노트 20건 산출물"
```

---

## Task 3: 노트 조회 API 와 마크다운 렌더러

**Files:**
- Create: `src/site/content/notes.ts`
- Create: `src/site/ui/slug.ts`
- Create: `src/site/ui/NoteBody.tsx`
- Modify: `package.json` (의존성 2개)

**Interfaces:**
- Consumes: Task 2의 `noteIndex`, `NoteMeta`, `notes/NN.md`
- Produces:
  - `notes: NoteMeta[]` (id 오름차순)
  - `getNote(id?: string): NoteMeta | undefined`
  - `getNoteBody(id: string): string | undefined`
  - `allNoteTags(): string[]`
  - `slugify(text: string): string`
  - `tableOfContents(markdown: string): { level: 2 | 3; text: string; id: string }[]`
  - `<NoteBody markdown={string} />`

- [ ] **Step 1: 의존성을 설치한다**

Run: `npm install react-markdown@^10 remark-gfm@^4`
Expected: 2개 추가, 취약점 0. (형제 리포 `wiki-front`가 같은 조합을 쓴다.)

> `rehype-slug`는 **쓰지 않는다.** 목차 링크와 헤딩 id 를 서로 다른 구현이 만들면
> 한글 헤딩에서 앵커가 어긋난다. 아래 `slug.ts` 하나로 양쪽을 모두 생성한다.

- [ ] **Step 2: 조회 API 를 쓴다**

`src/site/content/notes.ts`:

```ts
import { noteIndex } from './notes/index.generated';
import type { NoteMeta } from '../types';

export type { NoteMeta };

/** 본문은 빌드타임에 raw 문자열로 번들된다. 20개 · 228KB 규모라 eager 로 충분하다. */
const bodies = import.meta.glob('./notes/*.md', {
  query: '?raw',
  import: 'default',
  eager: true,
}) as Record<string, string>;

export const notes: NoteMeta[] = [...noteIndex].sort((a, b) => a.id.localeCompare(b.id));

export const getNote = (id?: string): NoteMeta | undefined =>
  id ? notes.find((n) => n.id === id) : undefined;

export const getNoteBody = (id: string): string | undefined => bodies[`./notes/${id}.md`];

/** 인덱스 필터용 태그 목록. 사용 빈도 내림차순, 동률이면 가나다순. */
export function allNoteTags(): string[] {
  const count = new Map<string, number>();
  for (const n of notes) for (const t of n.tags) count.set(t, (count.get(t) ?? 0) + 1);
  return [...count.entries()]
    .sort((a, b) => b[1] - a[1] || a[0].localeCompare(b[0], 'ko'))
    .map(([tag]) => tag);
}
```

- [ ] **Step 3: 슬러그 모듈을 쓴다**

`src/site/ui/slug.ts`:

```ts
/**
 * 헤딩 텍스트 → 앵커 id. 한글을 살리고 공백만 하이픈으로 바꾼다.
 * NoteBody(헤딩 id 부여)와 목차(링크 생성)가 이 함수 하나를 공유해야 앵커가 맞는다.
 */
export function slugify(text: string): string {
  return text
    .trim()
    .toLowerCase()
    .replace(/[^\p{L}\p{N}\s-]/gu, '')
    .replace(/\s+/g, '-');
}

export interface TocEntry {
  level: 2 | 3;
  text: string;
  id: string;
}

/**
 * 본문에서 h2/h3 만 뽑아 목차를 만든다. 코드블록 안의 `#` 은 건너뛴다.
 *
 * id 는 `slugify(text)` 뿐이다 — 등장 순서 카운터를 쓰지 않는다. 카운터를 쓰면 렌더 시점의
 * 상태에 id 가 의존하게 되고, `NoteBody` 쪽 카운터가 리렌더마다 이어져 앵커가 밀린다.
 * 실측: 노트 20편 196개 헤딩 중 슬러그 중복 0건. 훗날 중복이 생기면 목차 링크가 첫 번째
 * 헤딩으로 가는 정도의 열화만 남는다(앵커가 통째로 깨지는 것보다 낫다).
 */
export function tableOfContents(markdown: string): TocEntry[] {
  const out: TocEntry[] = [];
  let inFence = false;

  for (const line of markdown.split('\n')) {
    if (/^\s*```/.test(line)) {
      inFence = !inFence;
      continue;
    }
    if (inFence) continue;
    const m = line.match(/^(##|###)\s+(.+?)\s*$/);
    if (!m) continue;
    const text = m[2].replace(/[*_`]/g, '').trim();
    out.push({ level: m[1].length as 2 | 3, text, id: slugify(text) });
  }
  return out;
}
```

- [ ] **Step 4: 렌더러를 쓴다**

`src/site/ui/NoteBody.tsx`:

```tsx
import { useMemo } from 'react';
import Markdown from 'react-markdown';
import remarkGfm from 'remark-gfm';
import Box from '@mui/material/Box';
import { slugify } from './slug';
import { MONO, ANCHOR_OFFSET } from './tokens';

/** 자식 노드에서 순수 텍스트만 뽑는다 — 헤딩 id 계산용. */
function textOf(node: React.ReactNode): string {
  if (typeof node === 'string' || typeof node === 'number') return String(node);
  if (Array.isArray(node)) return node.map(textOf).join('');
  if (node && typeof node === 'object' && 'props' in node) {
    return textOf((node as { props: { children?: React.ReactNode } }).props.children);
  }
  return '';
}

/**
 * 노트 마크다운 렌더러. raw HTML 은 렌더하지 않는다(react-markdown 기본값 — XSS 방어).
 * 코드블록은 신택스 하이라이터 없이 모노 + 가로 스크롤로만 처리한다(번들 절약).
 * 헤딩 id 는 slug.ts 의 slugify 로 붙인다 — 목차와 같은 규칙이어야 앵커가 맞는다.
 */
export default function NoteBody({ markdown }: { markdown: string }) {
  // 헤딩 id 는 텍스트만의 순수 함수다. 등장 순서 카운터를 두면 그 카운터가 리렌더 사이에
  // 살아남아(useMemo 가 클로저를 캐시한다) 다크모드 토글 한 번에 모든 앵커가 밀린다.
  // deps 를 [] 로 둬서 컴포넌트 타입도 렌더마다 새로 만들지 않는다(불필요한 리마운트 방지).
  const components = useMemo(
    () => ({
      h2: ({ children }: { children?: React.ReactNode }) => <h2 id={slugify(textOf(children))}>{children}</h2>,
      h3: ({ children }: { children?: React.ReactNode }) => <h3 id={slugify(textOf(children))}>{children}</h3>,
    }),
    [],
  );

  return (
    <Box
      sx={{
        color: 'text.primary',
        lineHeight: 1.8,
        '& h2': { fontSize: '1.5rem', fontWeight: 700, mt: 6, mb: 2, letterSpacing: '-0.01em', scrollMarginTop: ANCHOR_OFFSET },
        '& h3': { fontSize: '1.15rem', fontWeight: 700, mt: 4, mb: 1.5, scrollMarginTop: ANCHOR_OFFSET },
        '& p': { my: 2 },
        '& a': { color: 'primary.main', textDecorationColor: 'currentColor' },
        '& ul, & ol': { pl: 3, my: 2 },
        '& li': { my: 0.5 },
        '& blockquote': {
          my: 3,
          ml: 0,
          pl: 2.5,
          borderLeft: '3px solid',
          borderColor: 'primary.main',
          color: 'text.secondary',
        },
        '& code': {
          fontFamily: MONO,
          fontSize: '0.875em',
          bgcolor: 'action.hover',
          px: 0.75,
          py: 0.25,
          borderRadius: 0.5,
        },
        '& pre': {
          overflowX: 'auto',
          p: 2,
          borderRadius: '4px',
          border: '1px solid',
          borderColor: 'divider',
          bgcolor: 'action.hover',
        },
        '& pre code': { bgcolor: 'transparent', p: 0, fontSize: '0.8125rem', lineHeight: 1.7 },
        '& table': { width: '100%', borderCollapse: 'collapse', my: 3, display: 'block', overflowX: 'auto' },
        '& th, & td': { border: '1px solid', borderColor: 'divider', px: 1.5, py: 1, textAlign: 'left' },
        '& th': { fontWeight: 700, bgcolor: 'action.hover' },
        '& img': { maxWidth: '100%' },
        '& hr': { border: 0, borderTop: '1px solid', borderColor: 'divider', my: 5 },
      }}
    >
      <Markdown remarkPlugins={[remarkGfm]} components={components}>
        {markdown}
      </Markdown>
    </Box>
  );
}
```

- [ ] **Step 5: 타입 게이트를 통과시킨다**

Run: `npm run build`
Expected: 성공. 실패하면 `import.meta.glob` 타입 문제일 수 있으니 `src/vite-env.d.ts`에
`/// <reference types="vite/client" />`가 있는지 확인한다.

- [ ] **Step 6: 커밋**

```bash
git add package.json package-lock.json src/site/content/notes.ts src/site/ui/slug.ts src/site/ui/NoteBody.tsx
git commit -m "feat(notes): 노트 조회 API + 마크다운 렌더러"
```

---

## Task 4: 시각 프리미티브 5종

**Files:**
- Create: `src/site/ui/SectionLabel.tsx`, `GridSection.tsx`, `SpecTable.tsx`, `StatBar.tsx`, `HairlineCard.tsx`, `index.ts`

**Interfaces:**
- Consumes: 없음 (MUI 만)
- Produces:
  - `<SectionLabel index="02" label="PRODUCTS" />`
  - `<GridSection index="02" label="PRODUCTS" title?: string caption?: string>{children}</GridSection>`
  - `<SpecTable rows={{ label: string; value: string }[]} />`
  - `<StatBar items={{ value: string; label: string }[]} />`
  - `<HairlineCard to?: string href?: string>{children}</HairlineCard>`
  - 배럴 `src/site/ui/index.ts`가 위 5개 + `NoteBody`를 재수출

- [ ] **Step 0: 공유 토큰**

`src/site/ui/tokens.ts`:

```ts
/**
 * 모노스페이스 스택. 프리미티브와 페이지가 **전부 여기서 가져온다.**
 * 이 문자열을 파일마다 복제하면 페이지를 늘릴 때 하나씩 어긋나기 시작한다 —
 * 실제로 초안에서는 8개 파일에 11번 복제돼 있었다.
 */
export const MONO = 'ui-monospace, SFMono-Regular, Menlo, monospace';

/** 스티키 헤더 높이(px). SiteHeader 가 이 값으로 렌더한다. */
export const HEADER_H = 56;

/**
 * 해시 앵커로 이동했을 때 헤더 아래로 확보할 여백.
 * 헤더 높이에서 파생시킨다 — 60/80 처럼 손으로 적은 숫자가 흩어지면
 * 헤더 높이를 바꾸는 순간 어떤 앵커는 헤더에 가리고 어떤 앵커는 안 가린다.
 */
export const ANCHOR_OFFSET = `${HEADER_H + 24}px`;
```

- [ ] **Step 1: SectionLabel**

`src/site/ui/SectionLabel.tsx`:

```tsx
import Stack from '@mui/material/Stack';
import Typography from '@mui/material/Typography';
import { MONO } from './tokens';

/** `SEC.02 / PRODUCTS` 모노 라벨. 엔지니어링 그리드의 기본 표식. */
export default function SectionLabel({ index, label }: { index: string; label: string }) {
  return (
    <Stack direction="row" spacing={1} sx={{ alignItems: 'baseline' }}>
      <Typography
        component="span"
        sx={{ fontFamily: MONO, fontSize: '0.75rem', color: 'primary.main', fontWeight: 700, letterSpacing: '0.08em' }}
      >
        SEC.{index}
      </Typography>
      <Typography
        component="span"
        sx={{ fontFamily: MONO, fontSize: '0.75rem', color: 'text.secondary', letterSpacing: '0.14em' }}
      >
        {label}
      </Typography>
    </Stack>
  );
}
```

- [ ] **Step 2: GridSection**

`src/site/ui/GridSection.tsx`:

```tsx
import Box from '@mui/material/Box';
import Container from '@mui/material/Container';
import Stack from '@mui/material/Stack';
import Typography from '@mui/material/Typography';
import SectionLabel from './SectionLabel';
import { ANCHOR_OFFSET } from './tokens';

/**
 * 좌측 고정폭 라벨 컬럼 + 우측 콘텐츠. 상단 헤어라인으로 섹션을 나눈다.
 * 모바일에서는 라벨이 콘텐츠 위로 적층된다.
 */
export default function GridSection({
  id,
  index,
  label,
  title,
  caption,
  children,
}: {
  id?: string;
  index: string;
  label: string;
  title?: string;
  caption?: string;
  children: React.ReactNode;
}) {
  return (
    <Box component="section" id={id} sx={{ borderTop: '1px solid', borderColor: 'divider', scrollMarginTop: ANCHOR_OFFSET }}>
      <Container maxWidth="lg" sx={{ py: { xs: 7, md: 12 } }}>
        <Stack direction={{ xs: 'column', md: 'row' }} spacing={{ xs: 3, md: 6 }}>
          {/* 고정폭 라벨 컬럼. flexShrink 0 이라 긴 라벨이 콘텐츠 쪽으로 넘치지 않게 줄바꿈을 허용한다. */}
          <Box sx={{ width: { md: 180 }, flexShrink: 0, pt: { md: 0.5 }, overflowWrap: 'anywhere' }}>
            <SectionLabel index={index} label={label} />
          </Box>
          <Box sx={{ flexGrow: 1, minWidth: 0 }}>
            {title && (
              <Typography
                variant="h4"
                component="h2"
                sx={{ fontWeight: 700, letterSpacing: '-0.02em', fontSize: 'clamp(1.6rem, 3.2vw, 2.1rem)', mb: caption ? 1.5 : 4 }}
              >
                {title}
              </Typography>
            )}
            {caption && <Typography sx={{ color: 'text.secondary', maxWidth: 620, mb: 4 }}>{caption}</Typography>}
            {children}
          </Box>
        </Stack>
      </Container>
    </Box>
  );
}
```

- [ ] **Step 3: SpecTable**

`src/site/ui/SpecTable.tsx`:

```tsx
import Box from '@mui/material/Box';
import Stack from '@mui/material/Stack';
import Typography from '@mui/material/Typography';
import type { SpecRow } from '../types';
import { MONO } from './tokens';

/** key/value 스펙 테이블. 값이 빈 행은 렌더하지 않는다(미확인 사실 금지 규칙). */
export default function SpecTable({ rows }: { rows: SpecRow[] }) {
  const visible = rows.filter((r) => r.value.trim() !== '');
  if (visible.length === 0) return null;
  return (
    <Box sx={{ borderTop: '1px solid', borderColor: 'divider' }}>
      {visible.map((row) => (
        <Stack
          key={row.label}
          direction={{ xs: 'column', sm: 'row' }}
          spacing={{ xs: 0.5, sm: 3 }}
          sx={{ py: 1.75, borderBottom: '1px solid', borderColor: 'divider', alignItems: { sm: 'baseline' } }}
        >
          <Typography
            sx={{
              width: { sm: 148 },
              flexShrink: 0,
              fontFamily: MONO,
              fontSize: '0.75rem',
              letterSpacing: '0.08em',
              color: 'text.secondary',
              textTransform: 'uppercase',
            }}
          >
            {row.label}
          </Typography>
          <Typography variant="body2" sx={{ color: 'text.primary', lineHeight: 1.7 }}>
            {row.value}
          </Typography>
        </Stack>
      ))}
    </Box>
  );
}
```

- [ ] **Step 4: StatBar**

`src/site/ui/StatBar.tsx`:

```tsx
import Box from '@mui/material/Box';
import Typography from '@mui/material/Typography';
import type { StatItem } from '../types';

/**
 * 수치 전면 노출 바. 숫자는 tabular-nums 로 자리를 고정한다.
 *
 * 테두리는 컨테이너가 top·left, 각 칸이 right·bottom 을 맡아 내부 seam 을 한 요소만 그린다
 * (이중선 방지). xs 는 2열 고정이므로 항목이 홀수면 마지막 행에 한 칸만 남아 우측·하단
 * 모서리가 뚫린다 — 마지막 칸을 2열로 늘려 행을 채운다. md 는 항목 수만큼 열을 만들어
 * 항상 정확히 나누어떨어진다.
 */
export default function StatBar({ items }: { items: StatItem[] }) {
  const oddOnMobile = items.length % 2 === 1;
  return (
    <Box
      sx={{
        display: 'grid',
        gridTemplateColumns: { xs: '1fr 1fr', md: `repeat(${items.length}, 1fr)` },
        borderTop: '1px solid',
        borderLeft: '1px solid',
        borderColor: 'divider',
      }}
    >
      {items.map((item, i) => (
        <Box
          key={item.label}
          sx={{
            borderRight: '1px solid',
            borderBottom: '1px solid',
            borderColor: 'divider',
            px: { xs: 2, md: 3 },
            py: { xs: 2.5, md: 3.5 },
            gridColumn: { xs: oddOnMobile && i === items.length - 1 ? 'span 2' : 'auto', md: 'auto' },
          }}
        >
          <Typography
            sx={{ fontWeight: 700, letterSpacing: '-0.02em', fontVariantNumeric: 'tabular-nums', fontSize: 'clamp(1.6rem, 3.5vw, 2.25rem)', lineHeight: 1.1 }}
          >
            {item.value}
          </Typography>
          <Typography variant="body2" sx={{ color: 'text.secondary', mt: 1 }}>
            {item.label}
          </Typography>
        </Box>
      ))}
    </Box>
  );
}
```

- [ ] **Step 5: HairlineCard**

`src/site/ui/HairlineCard.tsx`:

```tsx
import { Link as RouterLink } from 'react-router-dom';
import Card from '@mui/material/Card';
import CardActionArea from '@mui/material/CardActionArea';
import CardContent from '@mui/material/CardContent';

/**
 * 그림자 없는 아웃라인 카드. hover 는 border-color 만 바꾼다 —
 * transform 이동을 쓰지 않아 레이아웃이 흔들리지 않는다.
 *
 * hover 규칙은 링크일 때만 건다. to/href 가 없으면 클릭도 포커스도 안 되는 정적 카드인데,
 * hover 에 반응하면 마우스 사용자에게 누를 수 있다는 잘못된 신호를 준다.
 */
export default function HairlineCard({
  to,
  href,
  children,
}: {
  to?: string;
  href?: string;
  children: React.ReactNode;
}) {
  const interactive = Boolean(to || href);
  const body = <CardContent sx={{ p: { xs: 2.5, md: 3 }, height: '100%' }}>{children}</CardContent>;
  return (
    <Card
      variant="outlined"
      sx={{
        height: '100%',
        borderRadius: '4px', // 테마 shape.borderRadius 는 8 — 엔지니어링 그리드는 4 로 조인다
        boxShadow: 'none',
        borderColor: 'divider',
        transition: (theme) => theme.transitions.create('border-color', { duration: 150 }),
        ...(interactive && { '&:hover': { borderColor: 'primary.main' } }),
        '@media (prefers-reduced-motion: reduce)': { transition: 'none' },
      }}
    >
      {to ? (
        <CardActionArea component={RouterLink} to={to} sx={{ height: '100%' }}>
          {body}
        </CardActionArea>
      ) : href ? (
        <CardActionArea component="a" href={href} target="_blank" rel="noopener" sx={{ height: '100%' }}>
          {body}
        </CardActionArea>
      ) : (
        body
      )}
    </Card>
  );
}
```

- [ ] **Step 6: 배럴**

`src/site/ui/index.ts`:

```ts
export { default as SectionLabel } from './SectionLabel';
export { default as GridSection } from './GridSection';
export { default as SpecTable } from './SpecTable';
export { default as StatBar } from './StatBar';
export { default as HairlineCard } from './HairlineCard';
export { default as NoteBody } from './NoteBody';
export { slugify, tableOfContents } from './slug';
export { MONO, HEADER_H, ANCHOR_OFFSET } from './tokens';
export type { TocEntry } from './slug';
```

> 타입 `SpecRow` / `StatItem` / `NoteMeta` 는 `src/site/types.ts` 에서 직접 가져온다.
> ui 배럴로 재수출하지 않는다 — content 가 ui 를 import 하면 react-markdown 이 딸려 온다.

**전역 규칙 — radius**: 이 리뉴얼에서 새로 만드는 `Button`·`Chip`·`Card`는 전부
`sx={{ borderRadius: '4px' }}`를 명시한다. 테마 기본값이 8 이므로 생략하면 8px 가 된다.
아래 태스크들의 코드에 `borderRadius: 1` 로 적힌 곳이 있으면 **`'4px'` 로 바꿔 쓴다.**

- [ ] **Step 7: 빌드 확인 후 커밋**

Run: `npm run build`
Expected: 성공. (`noUnusedLocals` 때문에 아직 아무도 안 쓰는 export 는 문제되지 않는다 — 미사용 *지역* 변수만 잡는다.)

```bash
git add src/site/ui
git commit -m "feat(site): 엔지니어링 그리드 프리미티브 5종"
```

> `borderRadius: '4px'` 명시가 누락된 곳이 없는지 `grep -n "borderRadius: '4px'" src/site` 로 훑는다.
> 결과가 비어 있어야 한다.

---

## Task 5: 콘텐츠 계층 분리

**Files:**
- Create: `src/site/content/products.ts`, `capabilities.ts`, `profile.ts`, `stack.ts`, `index.ts`
- Delete: `src/pages/landing/content.ts`

**Interfaces:**
- Consumes: 없음
- Produces (배럴 `src/site/content` 경유):
  - `GITHUB_URL`, `CONTACT_EMAIL`, `PORTFOLIO_URL`
  - `products: Product[]`, `ossProducts`, `companyProducts`, `getProduct(slug?)`
  - `capabilities: Capability[]`, `getCapability(slug?)`
  - `career: CareerEntry[]`, `caseStudies: CaseStudy[]`, `stats: StatItem[]`
  - `techGroups: TechGroup[]`, `platformSpec: SpecRow[]`

- [ ] **Step 1: 사실을 리포에서 확인한다**

다음을 읽고 `products.ts`의 각 값을 **직접 대조**한다. 확인 못 한 행은 배열에서 뺀다.

```bash
head -60 /c/MSA_TEMPLATE/CLAUDE.md
head -60 /c/MSA_TEMPLATE/wiki-front/CLAUDE.md
head -40 /c/MSA_TEMPLATE/alm-front/CLAUDE.md
ls /c/MSA_TEMPLATE/design-system/packages
git -C /c/MSA_TEMPLATE log --oneline -15
```

- [ ] **Step 2: products.ts 를 쓴다**

`src/site/content/products.ts`:

```ts
import type { SpecRow } from '../types';

export interface Product {
  slug: string;
  name: string;
  category: 'oss' | 'company';
  /** 카드 한 줄 설명 */
  tagline: string;
  /** 상세 페이지 리드 문단 */
  summary: string;
  spec: SpecRow[];
  highlights: string[];
  repoUrl?: string;
  /** nginx 단일 오리진(/wiki/, /alm/)에서만 유효한 구동 링크 */
  liveUrl?: string;
  badge?: string;
}

export const products: Product[] = [
  {
    slug: 'wiki',
    name: 'WIKI',
    category: 'oss',
    tagline: 'Confluence 스타일 문서·위키',
    summary:
      '스페이스 단위로 문서를 쓰고 관리하는 위키. 편집은 TipTap 리치 에디터지만 저장 포맷은 마크다운 문자열이라, 문서가 특정 에디터에 갇히지 않는다.',
    spec: [
      { label: 'Frontend', value: 'React · TipTap · react-router' },
      { label: 'UI', value: '@chanho/react + @chanho/tokens (자체 디자인 시스템)' },
      { label: 'Backend', value: 'wiki-backend (Spring Boot · PostgreSQL)' },
      { label: '저장 포맷', value: '마크다운 문자열 (serializeMarkdown 직렬화)' },
      { label: '서빙', value: 'nginx 단일 오리진 /wiki/ (Vite base + router basename 쌍)' },
      { label: '데이터 경로', value: 'wikiStore async 함수 단일 경유 — 백엔드 교체 시 화면 무수정' },
    ],
    highlights: [
      '에디터 스키마의 단일 원천을 확장 목록 파일 하나로 고정해, 화면 에디터와 헤드리스 마크다운 변환기가 같은 스키마를 공유한다. 마크다운 왕복이 깨지지 않는 근거.',
      '보기 렌더와 에디터 양쪽 모두 raw HTML 을 렌더하지 않는다 — XSS 방어를 한쪽만 걸지 않았다.',
      '도메인 데이터 접근을 스토어 파일 하나로 좁혀, localStorage 목업에서 실제 백엔드로 옮길 때 바꿀 파일이 하나다.',
    ],
    repoUrl: 'https://github.com/chanho4702/WIKI',
    liveUrl: '/wiki/',
  },
  {
    slug: 'alm',
    name: 'ALM',
    category: 'oss',
    tagline: 'Jira 스타일 이슈·스프린트 관리',
    summary:
      '지라의 검증된 구조 위에 한국어 스마트 검색과 필터 URL 공유를 얹은 이슈 트래커. 지라 클론이 아니라, 지라가 잘한 것을 가져오고 다른 지점을 의도적으로 다르게 만들었다.',
    spec: [
      { label: 'Frontend', value: 'React · react-router' },
      { label: 'UI', value: '@chanho/react + @chanho/tokens (자체 디자인 시스템)' },
      { label: '서빙', value: 'nginx 단일 오리진 /alm/' },
      { label: '데이터 경로', value: 'jiraStore async 함수 단일 경유' },
      { label: '특색', value: '한국어 스마트 검색 · 필터 URL 공유 · 저장 필터 사이드바 · 시간추적' },
    ],
    highlights: [
      '필터 상태를 URL 에 실어 공유 가능하게 만들었다 — 협업 도구에서 "내가 보는 화면"을 그대로 넘길 수 있어야 한다는 판단.',
      '백엔드 없이 못 만드는 기능은 구현하지 않고 백로그에 남긴다. 목업으로 흉내 낸 기능이 나중에 계약과 어긋나는 것을 막는다.',
    ],
    repoUrl: 'https://github.com/chanho4702/ALM',
    liveUrl: '/alm/',
  },
  {
    slug: 'design-system',
    name: 'Chanho Design System',
    category: 'oss',
    tagline: '디자인 토큰과 React 컴포넌트 라이브러리',
    summary:
      '세 개의 프론트가 같은 얼굴을 갖게 하는 공유 레이어. 토큰 패키지와 React 컴포넌트 패키지로 나뉘어, 색·간격 같은 값과 그 값을 쓰는 컴포넌트를 따로 버전 관리한다.',
    spec: [
      { label: '패키지', value: '@chanho/tokens (디자인 토큰) · @chanho/react (컴포넌트)' },
      { label: '팔레트', value: 'Atlassian 정렬 블루 #0C66E4 · 뉴트럴 스케일 (tokens 0.3.0)' },
      { label: '소비처', value: 'wiki-front · alm-front' },
      { label: '배포', value: 'artifacts/*.tgz tarball — 소비 리포가 로컬 체크아웃으로 설치' },
    ],
    highlights: [
      '토큰을 CSS 변수(--chanho-*)로 노출해, 컴포넌트를 안 쓰는 커스텀 마크업도 같은 값을 쓰게 만들었다.',
      '토큰 패키지를 버전으로 배포해, 소비 리포가 각자의 시점에 팔레트 변경을 받아들일 수 있게 했다.',
    ],
    repoUrl: 'https://github.com/chanho4702/design-system',
  },
  {
    slug: 'msa-platform-template',
    name: 'MSA Platform Template',
    category: 'oss',
    tagline: 'Keycloak BFF · 게이트웨이 · 이벤트 기반 MSA 스타터',
    summary:
      '새 서비스를 시작할 때마다 인증·게이트웨이·UI 를 처음부터 세팅하던 반복을 없애려고 만든 플랫폼 골격. 이 소개 사이트도 그 위에서 돌아간다.',
    spec: [
      { label: '인증', value: 'Keycloak OIDC 리다이렉트 + auth-server 자체 RS256 JWT (BFF)' },
      { label: '게이트웨이', value: 'Spring Cloud Gateway · JWT 조기차단 · rate-limit' },
      { label: '디스커버리', value: 'Eureka' },
      { label: '이벤트', value: 'Redis Streams' },
      { label: '데이터', value: 'PostgreSQL' },
      { label: '관측', value: 'stdout JSON → Alloy → Loki → Grafana' },
      { label: 'CI/CD', value: 'GitHub Actions → GHCR → 셀프호스티드 러너 배포 (auth·gateway·eureka·board 적용)' },
      { label: '서비스', value: '백엔드 6 · 프론트 3 + 공유 디자인 시스템' },
    ],
    highlights: [
      '로그 수집을 앱에서 디커플했다 — 앱은 stdout 에 JSON 만 쓰고 수집기가 가져간다. 로그 백엔드를 바꿔도 앱을 안 고친다.',
      '게이트웨이가 JWT 를 앞단에서 검증해 잘못된 요청이 서비스까지 내려가지 않는다.',
      'rate-limit 키를 nginx 뒤 실제 클라이언트 IP 로 잡도록 신뢰 프록시 깊이를 고정했다 — 안 그러면 전체 트래픽이 한 IP 로 묶인다.',
    ],
    repoUrl: 'https://github.com/chanho4702/infra-settings',
  },
  {
    slug: 'moves-workforce',
    name: 'Moves Workforce',
    category: 'company',
    badge: '디무브',
    tagline: 'Jira Cloud 연동 인력·자원 관리 SaaS',
    summary:
      '엑셀과 수작업에 의존하던 리소스 관리를 여러 조직이 함께 쓰는 제품으로 만든 서버리스 SaaS. 수백 명이 사용한다.',
    spec: [
      { label: 'Platform', value: 'Atlassian Forge (Node.js 서버리스)' },
      { label: 'Frontend', value: 'React · TanStack Query' },
      { label: '연동', value: 'Jira Cloud' },
      { label: '권한', value: 'RBAC' },
      { label: '관측', value: 'Elastic Stack' },
    ],
    highlights: [
      'Excel 대량 등록을 Async Events + Queue 로 분할 처리해 서버리스 실행시간 제한을 넘겼다.',
    ],
  },
  {
    slug: 'moves-eye',
    name: 'Moves Eye',
    category: 'company',
    badge: '디무브',
    tagline: 'Elastic Stack 기반 로그 수집·관측 플랫폼',
    summary: '서비스 로그를 모아 관제하는 플랫폼.',
    spec: [{ label: 'Stack', value: 'Elasticsearch · Kibana · Logstash · Beats' }],
    highlights: [],
  },
];

export const ossProducts = products.filter((p) => p.category === 'oss');
export const companyProducts = products.filter((p) => p.category === 'company');
export const getProduct = (slug?: string): Product | undefined => products.find((p) => p.slug === slug);
```

- [ ] **Step 3: capabilities.ts 를 쓴다**

`src/site/content/capabilities.ts`:

```ts
export interface Capability {
  slug: string;
  title: string;
  lead: string;
  evidence: string;
}

export const capabilities: Capability[] = [
  {
    slug: 'platform-architecture',
    title: '플랫폼 아키텍처',
    lead: '서비스가 설 토대를 설계합니다.',
    evidence: '서버리스 SaaS(Atlassian Forge)와 Spring Cloud 기반 MSA를 직접 설계·구현.',
  },
  {
    slug: 'data-engineering',
    title: '데이터 엔지니어링 · 관측',
    lead: '결정을 데이터 위에 세웁니다.',
    evidence: 'Elasticsearch 수집·적재, Beats/Logstash 로그 파이프라인, Kibana·Grafana 대시보드.',
  },
  {
    slug: 'operations-reliability',
    title: '운영 · 안정성',
    lead: '멈추지 않게 운영합니다.',
    evidence: 'SLA 기반 장애 대응, 보안 솔루션 30개 사이트 구축·운영.',
  },
  {
    slug: 'ai-dev-env',
    title: 'AI 개발 환경',
    lead: '팀이 더 빠르게 만들게 합니다.',
    evidence: 'Claude Code 에이전트·스킬·MCP 직접 구성, 문서 기반 AI 협업 프로세스.',
  },
];

export const getCapability = (slug?: string): Capability | undefined =>
  capabilities.find((c) => c.slug === slug);
```

- [ ] **Step 4: profile.ts 를 쓴다**

`src/site/content/profile.ts` — 아래를 그대로 쓴다. 케이스스터디 문구는 이력서에서 확정된
문장이므로 재작성하지 않는다.

```ts
import type { StatItem } from '../types';

export const GITHUB_URL = 'https://github.com/chanho4702';
export const CONTACT_EMAIL = 'chanho470@naver.com';
export const PORTFOLIO_URL = 'https://oxidized-tile-0f2.notion.site/9bd7653948f34869ac67163d4bf40a89';

export interface CareerEntry {
  year: string;
  text: string;
}

export const career: CareerEntry[] = [
  { year: '2025.12 ~ 재직중', text: '디무브 — 서버리스 SaaS RMS 플랫폼 설계·구현' },
  { year: '2022.05 ~ 2025.12', text: '마크애니 — 보안 솔루션 구축·운영, 레거시 SPA 전면 전환' },
];

/** 홈 히어로 아래 신뢰 바. 전부 케이스스터디·역량 근거에서 나온 수치다. */
export const stats: StatItem[] = [
  { value: '2', label: '장관상 수상 (A-RMS)' },
  { value: '30', label: '보안 솔루션 구축 사이트' },
  { value: '3,000', label: '업무 이력 자동 이관 건수' },
  { value: '6', label: '플랫폼 백엔드 서비스' },
];

export interface CaseStudy {
  eyebrow: string;
  title: string;
  problem: string;
  solution: string;
  result: string;
  tags: string[];
  /**
   * `w`/`h` 는 원본 픽셀 크기다. 이걸 넘겨야 프레임이 비율대로 공간을 미리 잡아
   * 로딩 중 레이아웃이 밀리지 않고, 고정 높이 레터박싱도 생기지 않는다.
   */
  images?: { src: string; alt: string; w: number; h: number }[];
}

export const caseStudies: CaseStudy[] = [
  {
    eyebrow: 'A-RMS · 장관상 2회 수상작',
    title: '기억에 의존하던 업무 이력을 데이터 기반 의사결정으로',
    problem:
      '프로젝트 이력이 담당자의 기억과 흩어진 문서에 남아, 진행률·인력·성과를 한눈에 볼 수 없었습니다.',
    solution:
      '사내 업무 이력 3,000건을 Jira로 자동 이관하고, ALM/Jira 데이터를 Elasticsearch로 수집·적재해 시계열 인덱스와 집계 쿼리로 분석 체계를 세웠습니다.',
    result:
      '진행률·인력 활용률·ROI를 실시간 대시보드로 시각화. 두 개의 장관상을 수상하고, 데이터 기반 의사결정 체계로 사내에 정착시켰습니다.',
    tags: ['Elasticsearch', 'Kibana', 'Spring', 'ALM 데이터 분석'],
    images: [
      { src: '/arms-architecture.png', alt: 'A-RMS 시스템 아키텍처 다이어그램', w: 1549, h: 1524 },
      { src: '/arms-award.png', alt: 'A-RMS 수상 발표 공고 — SW기술 대상 · 공개SW 개발자대회 대상', w: 800, h: 850 },
    ],
  },
  {
    eyebrow: 'RMS SaaS · 디무브',
    title: '수작업 인력 관리를 수백 명이 쓰는 SaaS로',
    problem: '엑셀과 수작업에 의존하던 리소스 관리 업무를, 여러 조직이 함께 쓰는 제품으로 만들어야 했습니다.',
    solution:
      'Atlassian Forge(Node.js) + React로 서버리스 SaaS를 설계하고, Excel 대량 등록을 Async Events + Queue로 분할 처리해 서버리스 실행시간 제한을 극복했습니다.',
    result: 'Jira Cloud와 연동되는 RBAC 기반 플랫폼으로 수백 명이 사용합니다. Elastic Stack으로 로그를 관제합니다.',
    tags: ['Atlassian Forge', 'React', 'TanStack Query', 'Elastic Stack'],
  },
  {
    eyebrow: 'MSA 플랫폼 템플릿',
    title: '서비스를 만드는 게 아니라, 빠르게 만들 수 있는 환경을',
    problem: '새 서비스를 시작할 때마다 인증·게이트웨이·UI를 처음부터 세팅하는 반복이 있었습니다.',
    solution: 'Keycloak BFF 인증과 API 게이트웨이, 자체 디자인 시스템을 하나의 스타터로 묶었습니다.',
    result:
      '퇴근 후·주말 2주 만에 재사용 가능한 MSA 플랫폼 골격을 완성했습니다. 이 소개 페이지도 그 위에서 만들었습니다.',
    tags: ['Keycloak', 'Spring Cloud Gateway', '디자인 시스템', 'MSA'],
    images: [{ src: '/msa-architecture.jpg', alt: 'MSA 스타터 템플릿 아키텍처 다이어그램', w: 1541, h: 998 }],
  },
];
```

- [ ] **Step 5: stack.ts 를 쓴다**

`src/site/content/stack.ts` — 아래를 그대로 쓴다. 스택 목록은 이력서 스킬 기준이다.

```ts
import type { SpecRow } from '../types';

export interface TechGroup {
  category: string;
  items: string[];
}

export const techGroups: TechGroup[] = [
  {
    category: 'Backend',
    items: [
      'Spring Boot',
      'Spring Data JPA',
      'Spring Batch',
      'Spring Security',
      'WebFlux',
      'Spring Cloud',
      'Node.js (Forge)',
      'Kafka',
      'Redis',
      'Quartz',
    ],
  },
  {
    category: 'Data · Search',
    items: ['Elasticsearch', 'Kibana', 'Logstash', 'Beats / Fluentd', 'Grafana', 'MySQL', 'MSSQL'],
  },
  {
    category: 'DevOps · Infra',
    items: ['Docker', 'Kubernetes', 'Jenkins', 'ArgoCD', 'Spinnaker', 'Nexus', 'SonarQube', 'Keycloak'],
  },
  { category: 'Frontend', items: ['React', 'Vue.js', 'TypeScript', 'MUI', 'TanStack Query'] },
  { category: 'AI', items: ['Spring AI', 'Ollama', 'Claude Code 워크플로'] },
];

/** /tech 상단 — 이 플랫폼이 실제로 어떻게 구성돼 있는지. products.ts 의 MSA 템플릿 스펙과 같은 사실. */
export const platformSpec: SpecRow[] = [
  { label: '인증', value: 'Keycloak OIDC 리다이렉트 + auth-server 자체 RS256 JWT (BFF)' },
  { label: '게이트웨이', value: 'Spring Cloud Gateway · JWT 조기차단 · rate-limit' },
  { label: '디스커버리', value: 'Eureka' },
  { label: '이벤트', value: 'Redis Streams' },
  { label: '데이터', value: 'PostgreSQL' },
  { label: '관측', value: 'stdout JSON → Alloy → Loki → Grafana' },
  { label: 'CI/CD', value: 'GitHub Actions → GHCR → 셀프호스티드 러너 배포' },
  { label: '프론트', value: 'React 19 · 공유 디자인 시스템 · nginx 단일 오리진' },
];
```

- [ ] **Step 6: 배럴을 만들고 구 파일을 지운다**

`src/site/content/index.ts`:

```ts
export * from './products';
export * from './capabilities';
export * from './profile';
export * from './stack';
export * from './notes';
```

Run: `rm src/pages/landing/content.ts`

- [ ] **Step 7: 빌드로 깨진 import 를 잡는다**

Run: `npm run build`
Expected: `src/pages/Home.tsx`와 `src/pages/landing/DetailPages.tsx`에서 `./landing/content`를 못 찾는 에러.
**이 시점에는 정상이다** — Task 6~11에서 두 파일을 교체한다. 임시로 두 파일의 import 를
`../site/content` / `../../site/content`로 바꿔 빌드를 통과시킨 뒤 커밋한다.

- [ ] **Step 8: 커밋**

```bash
git add src/site/content src/pages
git commit -m "feat(site): 콘텐츠 계층을 src/site/content 로 분리"
```

---

## Task 6: 사이트 셸 — 헤더 · 푸터 · 페이지 래퍼

**Files:**
- Create: `src/site/components/SiteHeader.tsx`, `SiteFooter.tsx`, `SitePage.tsx`
- Delete: `src/pages/landing/LandingHeader.tsx`

**Interfaces:**
- Consumes: Task 5의 `GITHUB_URL`, `CONTACT_EMAIL`
- Produces:
  - `HEADER_H` / `ANCHOR_OFFSET` 은 `src/site/ui/tokens.ts` 가 소유한다
  - `<SitePage>{children}</SitePage>` — AppTheme + CssBaseline + Header + main + Footer

- [ ] **Step 1: SiteHeader**

`src/site/components/SiteHeader.tsx`:

```tsx
import { useState } from 'react';
import { Link as RouterLink, useLocation } from 'react-router-dom';
import { alpha } from '@mui/material/styles';
import Box from '@mui/material/Box';
import Container from '@mui/material/Container';
import Stack from '@mui/material/Stack';
import Link from '@mui/material/Link';
import Button from '@mui/material/Button';
import IconButton from '@mui/material/IconButton';
import Drawer from '@mui/material/Drawer';
import MenuRoundedIcon from '@mui/icons-material/MenuRounded';
import CloseRoundedIcon from '@mui/icons-material/CloseRounded';
import ColorModeIconDropdown from '../../context/templates/shared-theme/ColorModeIconDropdown';
import { HEADER_H } from '../ui';


const navItems = [
  { to: '/products', label: '제품' },
  { to: '/tech', label: '기술' },
  { to: '/about', label: '소개' },
];

export default function SiteHeader() {
  const [open, setOpen] = useState(false);
  const { pathname } = useLocation();
  const isActive = (to: string) => pathname === to || pathname.startsWith(`${to}/`);

  const navLink = (to: string, label: string, onClick?: () => void, big?: boolean) => (
    <Link
      key={to}
      component={RouterLink}
      to={to}
      underline="none"
      onClick={onClick}
      aria-current={isActive(to) ? 'page' : undefined}
      sx={{
        fontSize: big ? '1.05rem' : '0.875rem',
        fontWeight: isActive(to) ? 700 : 500,
        color: isActive(to) ? 'text.primary' : 'text.secondary',
        '&:hover': { color: 'text.primary' },
      }}
    >
      {label}
    </Link>
  );

  return (
    <>
      <Box
        component="header"
        sx={{
          position: 'sticky',
          top: 0,
          zIndex: (theme) => theme.zIndex.appBar,
          backdropFilter: 'saturate(180%) blur(20px)',
          WebkitBackdropFilter: 'saturate(180%) blur(20px)',
          bgcolor: (theme) => alpha(theme.palette.background.default, 0.78),
          borderBottom: '1px solid',
          borderColor: 'divider',
        }}
      >
        <Container maxWidth="lg">
          <Stack direction="row" sx={{ height: HEADER_H, alignItems: 'center', justifyContent: 'space-between' }}>
            <Link
              component={RouterLink}
              to="/"
              underline="none"
              sx={{ fontWeight: 700, letterSpacing: '-0.02em', color: 'text.primary', fontSize: '1rem' }}
            >
              chanho.dev
            </Link>

            <Stack direction="row" spacing={3.5} sx={{ display: { xs: 'none', md: 'flex' }, alignItems: 'center' }}>
              {navItems.map((i) => navLink(i.to, i.label))}
              <ColorModeIconDropdown size="small" />
              <Button component={RouterLink} to="/contact" variant="contained" size="small" sx={{ borderRadius: '4px' }}>
                문의하기
              </Button>
            </Stack>

            <Stack direction="row" spacing={0.5} sx={{ display: { xs: 'flex', md: 'none' }, alignItems: 'center' }}>
              <ColorModeIconDropdown size="small" />
              <IconButton onClick={() => setOpen(true)} aria-label="메뉴 열기" size="small">
                <MenuRoundedIcon />
              </IconButton>
            </Stack>
          </Stack>
        </Container>
      </Box>

      <Drawer anchor="right" open={open} onClose={() => setOpen(false)}>
        <Box sx={{ width: 264, p: 2 }}>
          <Stack direction="row" sx={{ justifyContent: 'flex-end' }}>
            <IconButton onClick={() => setOpen(false)} aria-label="메뉴 닫기">
              <CloseRoundedIcon />
            </IconButton>
          </Stack>
          <Stack spacing={2.5} sx={{ p: 2, pt: 1 }}>
            {navItems.map((i) => navLink(i.to, i.label, () => setOpen(false), true))}
            <Button component={RouterLink} to="/contact" variant="contained" onClick={() => setOpen(false)} sx={{ borderRadius: '4px' }}>
              문의하기
            </Button>
          </Stack>
        </Box>
      </Drawer>
    </>
  );
}
```

- [ ] **Step 2: SiteFooter**

`src/site/components/SiteFooter.tsx`:

```tsx
import { Link as RouterLink } from 'react-router-dom';
import Box from '@mui/material/Box';
import Container from '@mui/material/Container';
import Stack from '@mui/material/Stack';
import Typography from '@mui/material/Typography';
import Link from '@mui/material/Link';
import { MONO } from '../ui';
import { GITHUB_URL, CONTACT_EMAIL } from '../content';

const siteLinks = [
  { to: '/products', label: '제품' },
  { to: '/tech', label: '기술' },
  { to: '/tech/notes', label: '엔지니어링 노트' },
  { to: '/about', label: '소개' },
  { to: '/contact', label: '연락' },
];

/** 개발용 진입점. GNB 에서 빼고 여기로 격리한다. */
const devLinks = [
  { to: '/app', label: '서비스 데모' },
  { to: '/designs', label: '설계 문서' },
  { to: '/profile', label: '프로필' },
  { to: '/templates', label: 'MUI 템플릿' },
  { to: '/components', label: '컴포넌트 카탈로그' },
  { to: '/showcase', label: '컴포넌트 쇼케이스' },
];

function LinkGroup({ title, items }: { title: string; items: { to: string; label: string }[] }) {
  return (
    <Box>
      <Typography sx={{ fontFamily: MONO, fontSize: '0.7rem', letterSpacing: '0.14em', color: 'text.secondary', mb: 1.5 }}>
        {title}
      </Typography>
      <Stack spacing={1}>
        {items.map((l) => (
          <Link
            key={l.to}
            component={RouterLink}
            to={l.to}
            underline="hover"
            variant="body2"
            sx={{ color: 'text.secondary', '&:hover': { color: 'primary.main' } }}
          >
            {l.label}
          </Link>
        ))}
      </Stack>
    </Box>
  );
}

export default function SiteFooter() {
  return (
    <Box component="footer" sx={{ borderTop: '1px solid', borderColor: 'divider' }}>
      <Container maxWidth="lg" sx={{ py: { xs: 5, md: 7 } }}>
        <Stack direction={{ xs: 'column', sm: 'row' }} spacing={{ xs: 4, sm: 8 }}>
          <LinkGroup title="SITE" items={siteLinks} />
          <LinkGroup title="DEV" items={devLinks} />
          <Box>
            <Typography sx={{ fontFamily: MONO, fontSize: '0.7rem', letterSpacing: '0.14em', color: 'text.secondary', mb: 1.5 }}>
              CONTACT
            </Typography>
            <Stack spacing={1}>
              <Link href={`mailto:${CONTACT_EMAIL}`} underline="hover" variant="body2" sx={{ color: 'text.secondary' }}>
                {CONTACT_EMAIL}
              </Link>
              <Link href={GITHUB_URL} target="_blank" rel="noopener" underline="hover" variant="body2" sx={{ color: 'text.secondary' }}>
                GitHub
              </Link>
            </Stack>
          </Box>
        </Stack>
        <Typography variant="caption" sx={{ display: 'block', color: 'text.secondary', mt: { xs: 4, md: 6 } }}>
          © {new Date().getFullYear()} 김찬호 · Built with React · MUI
        </Typography>
      </Container>
    </Box>
  );
}
```

- [ ] **Step 3: SitePage**

`src/site/components/SitePage.tsx`:

```tsx
import { useEffect } from 'react';
import { useLocation } from 'react-router-dom';
import Box from '@mui/material/Box';
import CssBaseline from '@mui/material/CssBaseline';
import Link from '@mui/material/Link';
import AppTheme from '../../context/templates/shared-theme/AppTheme';
import SiteHeader from './SiteHeader';
import SiteFooter from './SiteFooter';

/**
 * 라우트가 바뀌며 들어온 해시(`/tech#cap-...`)로 스크롤한다.
 *
 * React Router 의 데이터 라우터는 URL 해시를 **스스로 처리하지 않는다.** `<ScrollRestoration>`
 * 을 두거나 이렇게 직접 처리하지 않으면, `<Navigate to="/tech#cap-x">` 는 주소만 바꾸고
 * 사용자는 아무 에러 없이 페이지 최상단에 떨어진다.
 * (같은 문서 안의 `<a href="#...">` 목차 링크는 브라우저가 알아서 처리하므로 이 훅과 무관하다.)
 *
 * `querySelector` 가 아니라 `getElementById` 를 쓰는 이유: 노트 헤딩 슬러그는 한글이고
 * 숫자로 시작할 수도 있어서(`#1단계`) CSS 선택자로는 파싱 에러가 난다.
 */
function useHashScroll() {
  const { hash } = useLocation();
  useEffect(() => {
    if (!hash) return;
    const el = document.getElementById(decodeURIComponent(hash.slice(1)));
    if (!el) return;
    el.scrollIntoView(); // scroll-margin-top(ANCHOR_OFFSET)을 존중한다
    // 스크롤만 하면 스크린리더 사용자는 도착 사실을 모른다. 포커스도 함께 옮긴다.
    if (!el.hasAttribute('tabindex')) el.setAttribute('tabindex', '-1');
    el.focus({ preventScroll: true });
  }, [hash]);
}

/**
 * 공개 사이트 공통 셸. 테마·헤더·푸터를 한 곳에서 감싼다.
 *
 * 스킵 링크가 여기 있는 이유: 스티키 헤더가 모든 페이지에 얹히므로, 없으면 키보드·스크린리더
 * 사용자가 페이지를 옮길 때마다 GNB 3개 + CTA + 컬러모드 드롭다운을 매번 통과해야 한다.
 * `display: none` 은 포커스를 받지 못하므로 화면 밖으로 밀어 두고 포커스 시 끌어온다.
 */
export default function SitePage({ children }: { children: React.ReactNode }) {
  useHashScroll();
  return (
    <AppTheme>
      <CssBaseline enableColorScheme />
      <Link
        href="#main-content"
        sx={{
          position: 'fixed',
          left: 8,
          top: -80,
          zIndex: (theme) => theme.zIndex.appBar + 1,
          px: 2,
          py: 1,
          borderRadius: '4px',
          bgcolor: 'background.paper',
          border: '1px solid',
          borderColor: 'divider',
          color: 'text.primary',
          textDecoration: 'none',
          '&:focus-visible': { top: 8 },
        }}
      >
        본문으로 건너뛰기
      </Link>
      <SiteHeader />
      {/*
        tabIndex={-1} 은 선택이 아니라 필수다 — <main> 은 네이티브로 포커스를 못 받아서,
        해시로 이동해도 이게 없으면 브라우저가 포커스를 옮기지 않는다.
        페이지 높이만 한 랜드마크에 기본 아웃라인을 두르는 건 의미가 없어 죽이되,
        스킵 링크로 건너뛴 직후 "포커스가 어디 있는지" 단서가 아예 없으면 곤란하므로
        포커스 시에만 상단에 옅은 표시를 남긴다.
      */}
      <Box
        component="main"
        id="main-content"
        tabIndex={-1}
        sx={{
          outline: 'none',
          '&:focus-visible': { boxShadow: (theme) => `inset 0 3px 0 0 ${theme.palette.primary.main}` },
        }}
      >
        {children}
      </Box>
      <SiteFooter />
    </AppTheme>
  );
}
```

- [ ] **Step 4: 구 헤더를 지우고 빌드**

Run: `rm src/pages/landing/LandingHeader.tsx`
Run: `npm run build`
Expected: `Home.tsx` / `DetailPages.tsx`가 `LandingHeader`를 찾지 못하는 에러 — Task 7·11에서 해소한다.
지금은 두 파일의 `LandingHeader` 사용을 `SitePage`로 임시 교체해 빌드를 통과시킨다.

- [ ] **Step 5: 커밋**

```bash
git add src/site/components src/pages
git commit -m "feat(site): 사이트 셸(헤더·푸터·페이지 래퍼)"
```

---

## Task 7: 제품 페이지

**Files:**
- Create: `src/site/pages/ProductsPage.tsx`, `src/site/pages/ProductDetailPage.tsx`
- Modify: `src/main.tsx`
- Delete: `src/pages/landing/DetailPages.tsx`

**Interfaces:**
- Consumes: Task 4 프리미티브, Task 5 `products`/`ossProducts`/`companyProducts`/`getProduct`, Task 6 `SitePage`
- Produces: 라우트 `/products`, `/products/:slug`

- [ ] **Step 1: 제품 인덱스**

`src/site/pages/ProductsPage.tsx`:

```tsx
import Box from '@mui/material/Box';
import Container from '@mui/material/Container';
import Grid from '@mui/material/Grid';
import Stack from '@mui/material/Stack';
import Typography from '@mui/material/Typography';
import Chip from '@mui/material/Chip';
import ArrowForwardRoundedIcon from '@mui/icons-material/ArrowForwardRounded';
import SitePage from '../components/SitePage';
import { GridSection, HairlineCard, MONO } from '../ui';
import { ossProducts, companyProducts, type Product } from '../content';

function ProductCard({ product }: { product: Product }) {
  return (
    <HairlineCard to={`/products/${product.slug}`}>
      <Stack sx={{ height: '100%' }}>
        <Stack direction="row" spacing={1} sx={{ alignItems: 'center', justifyContent: 'space-between', mb: 1 }}>
          <Typography variant="h6" component="h3" sx={{ fontWeight: 700, letterSpacing: '-0.01em' }}>
            {product.name}
          </Typography>
          {product.badge && <Chip label={product.badge} size="small" color="primary" variant="outlined" />}
        </Stack>
        <Typography variant="body2" sx={{ color: 'text.secondary', lineHeight: 1.7, flexGrow: 1 }}>
          {product.tagline}
        </Typography>
        <Stack direction="row" spacing={0.5} sx={{ alignItems: 'center', mt: 2, color: 'text.secondary' }}>
          <Typography
            variant="caption"
            sx={{ fontFamily: MONO, letterSpacing: '0.08em' }}
          >
            {product.repoUrl ? 'OPEN SOURCE' : '사내·고객사 제품 · 비공개'}
          </Typography>
          <ArrowForwardRoundedIcon sx={{ fontSize: 14 }} />
        </Stack>
      </Stack>
    </HairlineCard>
  );
}

export default function ProductsPage() {
  return (
    <SitePage>
      <Container maxWidth="lg" sx={{ pt: { xs: 7, md: 12 }, pb: { xs: 5, md: 8 } }}>
        <Typography component="h1" sx={{ fontWeight: 800, letterSpacing: '-0.03em', fontSize: 'clamp(2.2rem, 6vw, 3.4rem)', lineHeight: 1.08 }}>
          제품
        </Typography>
        <Typography sx={{ mt: 2.5, color: 'text.secondary', maxWidth: 620, fontSize: '1.1rem', lineHeight: 1.7 }}>
          직접 만들어 공개한 오픈소스와, 재직 중 만든 사내 제품.
        </Typography>
      </Container>

      <GridSection index="01" label="OPEN SOURCE" title="공개한 제품">
        <Grid container spacing={{ xs: 2, md: 2.5 }}>
          {ossProducts.map((p) => (
            <Grid key={p.slug} size={{ xs: 12, sm: 6, md: 3 }}>
              <ProductCard product={p} />
            </Grid>
          ))}
        </Grid>
      </GridSection>

      <GridSection index="02" label="COMPANY" title="재직 중 개발">
        <Grid container spacing={{ xs: 2, md: 2.5 }}>
          {companyProducts.map((p) => (
            <Grid key={p.slug} size={{ xs: 12, sm: 6 }}>
              <ProductCard product={p} />
            </Grid>
          ))}
        </Grid>
      </GridSection>
      <Box sx={{ borderTop: '1px solid', borderColor: 'divider' }} />
    </SitePage>
  );
}
```

- [ ] **Step 2: 제품 상세**

`src/site/pages/ProductDetailPage.tsx`:

```tsx
import { Link as RouterLink, useParams } from 'react-router-dom';
import Box from '@mui/material/Box';
import Container from '@mui/material/Container';
import Stack from '@mui/material/Stack';
import Typography from '@mui/material/Typography';
import Button from '@mui/material/Button';
import Chip from '@mui/material/Chip';
import GitHubIcon from '@mui/icons-material/GitHub';
import LaunchRoundedIcon from '@mui/icons-material/LaunchRounded';
import ArrowBackRoundedIcon from '@mui/icons-material/ArrowBackRounded';
import SitePage from '../components/SitePage';
import { GridSection, SpecTable } from '../ui';
import { getProduct } from '../content';
import NotFoundPage from '../../app/pages/NotFoundPage';

export default function ProductDetailPage() {
  const { slug } = useParams<{ slug: string }>();
  const product = getProduct(slug);
  if (!product) return <NotFoundPage />;

  return (
    <SitePage>
      <Container maxWidth="lg" sx={{ pt: { xs: 6, md: 10 }, pb: { xs: 5, md: 8 } }}>
        <Button component={RouterLink} to="/products" startIcon={<ArrowBackRoundedIcon />} sx={{ color: 'text.secondary', mb: 3, ml: -1 }}>
          제품
        </Button>
        <Stack direction="row" spacing={1.5} sx={{ alignItems: 'center', mb: 2 }}>
          <Typography component="h1" sx={{ fontWeight: 800, letterSpacing: '-0.03em', fontSize: 'clamp(2rem, 5.5vw, 3.2rem)', lineHeight: 1.1 }}>
            {product.name}
          </Typography>
          {product.badge && <Chip label={product.badge} size="small" color="primary" variant="outlined" />}
        </Stack>
        <Typography sx={{ color: 'text.secondary', maxWidth: 680, fontSize: '1.1rem', lineHeight: 1.7 }}>
          {product.summary}
        </Typography>
        {/*
          링크가 하나도 없는 사내 제품(moves-*)은 버튼 영역이 비어 여백만 남는다.
          구 상세 페이지가 이 자리에 "비공개" 표기를 렌더했었으므로 그 정보를 잃지 않는다.
        */}
        {product.liveUrl || product.repoUrl ? (
          <Stack direction="row" spacing={1.5} sx={{ flexWrap: 'wrap', gap: 1.5, mt: 4 }}>
            {product.liveUrl && (
              <Button variant="contained" href={product.liveUrl} startIcon={<LaunchRoundedIcon />} sx={{ borderRadius: '4px' }}>
                라이브로 열기
              </Button>
            )}
            {product.repoUrl && (
              <Button
                variant={product.liveUrl ? 'outlined' : 'contained'}
                href={product.repoUrl}
                target="_blank"
                rel="noopener"
                startIcon={<GitHubIcon />}
                sx={{ borderRadius: '4px' }}
              >
                소스 보기
              </Button>
            )}
          </Stack>
        ) : (
          <Typography variant="body2" sx={{ mt: 4, color: 'text.secondary' }}>
            사내·고객사 제품 — 소스와 데모를 공개하지 않습니다.
          </Typography>
        )}
      </Container>

      <GridSection index="01" label="SPEC" title="구성">
        <SpecTable rows={product.spec} />
      </GridSection>

      {product.highlights.length > 0 && (
        <GridSection index="02" label="DECISIONS" title="설계에서 판단한 것들">
          <Stack spacing={2.5}>
            {product.highlights.map((h) => (
              <Stack key={h} direction="row" spacing={2} sx={{ alignItems: 'flex-start' }}>
                <Box sx={{ width: 3, alignSelf: 'stretch', bgcolor: 'primary.main', flexShrink: 0, mt: 0.5 }} />
                <Typography variant="body2" sx={{ lineHeight: 1.85, color: 'text.primary' }}>
                  {h}
                </Typography>
              </Stack>
            ))}
          </Stack>
        </GridSection>
      )}
      <Box sx={{ borderTop: '1px solid', borderColor: 'divider' }} />
    </SitePage>
  );
}
```

- [ ] **Step 3: 라우트를 등록한다**

`src/main.tsx`에서 다음 import 를 지운다:

```tsx
import { ServiceDetailPage, ProductDetailPage } from './pages/landing/DetailPages';
```

대신 추가한다:

```tsx
import ProductsPage from './site/pages/ProductsPage';
import ProductDetailPage from './site/pages/ProductDetailPage';
```

라우터 배열에서 `{ path: '/services/:slug', ... }`와 `{ path: '/products/:slug', ... }` 두 줄을 다음으로 교체한다:

```tsx
  { path: '/products', element: <ProductsPage />, errorElement: <RouteErrorPage /> },
  { path: '/products/:slug', element: <ProductDetailPage />, errorElement: <RouteErrorPage /> },
```

Run: `rm src/pages/landing/DetailPages.tsx`

- [ ] **Step 4: 빌드하고 눈으로 확인한다**

Run: `npm run build`
Expected: 성공

Run: `npm run dev` 후 브라우저에서 확인
- `http://localhost:5173/products` — 오픈소스 4장 + 디무브 2장, 그림자 없음, hover 시 테두리만 파랗게
- `http://localhost:5173/products/wiki` — 스펙 테이블 6행, DECISIONS 3건, "라이브로 열기"·"소스 보기" 버튼
- `http://localhost:5173/products/moves-eye` — highlights 가 빈 배열이므로 **DECISIONS 섹션이 아예 없어야** 한다
- `http://localhost:5173/products/없는슬러그` — NotFound
- 라이트/다크 토글 양쪽에서 헤어라인이 보이는지

- [ ] **Step 5: 커밋**

```bash
git add src/site/pages src/main.tsx src/pages
git commit -m "feat(site): 제품 인덱스·상세 페이지 (준비 중 플레이스홀더 제거)"
```

---

## Task 8: 기술 페이지와 구 라우트 리다이렉트

**Files:**
- Create: `src/site/pages/TechPage.tsx`, `src/site/pages/ServiceRedirect.tsx`
- Modify: `src/main.tsx`

**Interfaces:**
- Consumes: Task 4 프리미티브, Task 5 `platformSpec`/`capabilities`/`techGroups`, Task 3 `notes`
- Produces: 라우트 `/tech`, `/services/:slug`(리다이렉트)

- [ ] **Step 1: TechPage**

`src/site/pages/TechPage.tsx`:

```tsx
import { Link as RouterLink } from 'react-router-dom';
import Box from '@mui/material/Box';
import Container from '@mui/material/Container';
import Grid from '@mui/material/Grid';
import Stack from '@mui/material/Stack';
import Typography from '@mui/material/Typography';
import Button from '@mui/material/Button';
import Chip from '@mui/material/Chip';
import ArrowForwardRoundedIcon from '@mui/icons-material/ArrowForwardRounded';
import SitePage from '../components/SitePage';
import { GridSection, SpecTable, HairlineCard, MONO, ANCHOR_OFFSET } from '../ui';
import { platformSpec, capabilities, techGroups, notes } from '../content';

export default function TechPage() {
  const latest = notes.slice(-3).reverse();

  return (
    <SitePage>
      <Container maxWidth="lg" sx={{ pt: { xs: 7, md: 12 }, pb: { xs: 5, md: 8 } }}>
        <Typography component="h1" sx={{ fontWeight: 800, letterSpacing: '-0.03em', fontSize: 'clamp(2.2rem, 6vw, 3.4rem)', lineHeight: 1.08 }}>
          기술
        </Typography>
        <Typography sx={{ mt: 2.5, color: 'text.secondary', maxWidth: 640, fontSize: '1.1rem', lineHeight: 1.7 }}>
          이 사이트가 올라가 있는 플랫폼의 실제 구성과, 그것을 만들면서 남긴 기록.
        </Typography>
      </Container>

      <GridSection index="01" label="PLATFORM" title="플랫폼 구성" caption="chanho.dev · WIKI · ALM 이 공유하는 골격.">
        <SpecTable rows={platformSpec} />
      </GridSection>

      <GridSection index="02" label="CAPABILITIES" title="무엇을 해드리는가">
        <Grid container spacing={{ xs: 2, md: 2.5 }}>
          {capabilities.map((c) => (
            <Grid key={c.slug} size={{ xs: 12, sm: 6 }}>
              <Box id={`cap-${c.slug}`} sx={{ height: '100%', scrollMarginTop: ANCHOR_OFFSET }}>
                <HairlineCard>
                  <Typography
                    variant="caption"
                    sx={{ fontFamily: MONO, letterSpacing: '0.1em', color: 'text.secondary' }}
                  >
                    {c.title}
                  </Typography>
                  <Typography variant="h6" component="h3" sx={{ fontWeight: 700, mt: 0.5, mb: 1.5 }}>
                    {c.lead}
                  </Typography>
                  <Typography variant="body2" sx={{ color: 'text.secondary', lineHeight: 1.75 }}>
                    {c.evidence}
                  </Typography>
                </HairlineCard>
              </Box>
            </Grid>
          ))}
        </Grid>
      </GridSection>

      <GridSection
        index="03"
        label="NOTES"
        title="엔지니어링 노트"
        caption={`플랫폼을 만들며 남긴 기록 ${notes.length}편. 인증·게이트웨이·통합배포·관측까지 시간순으로 이어진다.`}
      >
        <Stack spacing={1.5}>
          {latest.map((n) => (
            <HairlineCard key={n.id} to={`/tech/notes/${n.id}`}>
              <Stack direction={{ xs: 'column', sm: 'row' }} spacing={{ xs: 0.5, sm: 2 }} sx={{ alignItems: { sm: 'baseline' } }}>
                <Typography
                  sx={{
                    fontFamily: MONO,
                    fontSize: '0.75rem',
                    color: 'primary.main',
                    fontVariantNumeric: 'tabular-nums',
                    width: { sm: 64 },
                    flexShrink: 0,
                  }}
                >
                  NO.{n.id}
                </Typography>
                <Typography sx={{ fontWeight: 600, flexGrow: 1 }}>{n.title}</Typography>
                <Typography variant="caption" sx={{ color: 'text.secondary', fontVariantNumeric: 'tabular-nums' }}>
                  {n.date}
                </Typography>
              </Stack>
            </HairlineCard>
          ))}
        </Stack>
        <Button component={RouterLink} to="/tech/notes" endIcon={<ArrowForwardRoundedIcon />} sx={{ mt: 2.5, ml: -1 }}>
          노트 전체 보기
        </Button>
      </GridSection>

      <GridSection index="04" label="STACK" title="기술 스택">
        <Grid container spacing={{ xs: 3.5, md: 4 }}>
          {techGroups.map((g) => (
            <Grid key={g.category} size={{ xs: 12, sm: 6, md: 4 }}>
              <Typography variant="subtitle2" sx={{ color: 'text.secondary', fontWeight: 700, mb: 1.5 }}>
                {g.category}
              </Typography>
              <Box sx={{ display: 'flex', flexWrap: 'wrap', gap: 0.75 }}>
                {g.items.map((item) => (
                  <Chip key={item} label={item} size="small" variant="outlined" sx={{ borderRadius: '4px' }} />
                ))}
              </Box>
            </Grid>
          ))}
        </Grid>
      </GridSection>
      <Box sx={{ borderTop: '1px solid', borderColor: 'divider' }} />
    </SitePage>
  );
}
```

- [ ] **Step 2: 구 `/services/:slug` 리다이렉트**

`src/site/pages/ServiceRedirect.tsx`:

```tsx
import { Navigate, useParams } from 'react-router-dom';
import { getCapability } from '../content';

/** 구 랜딩의 /services/:slug 링크를 살려둔다 — /tech 의 해당 역량 앵커로 보낸다. */
export default function ServiceRedirect() {
  const { slug } = useParams<{ slug: string }>();
  const target = getCapability(slug) ? `/tech#cap-${slug}` : '/tech';
  return <Navigate to={target} replace />;
}
```

- [ ] **Step 3: 라우트 등록**

`src/main.tsx`에 추가:

```tsx
import TechPage from './site/pages/TechPage';
import ServiceRedirect from './site/pages/ServiceRedirect';
```

```tsx
  { path: '/tech', element: <TechPage />, errorElement: <RouteErrorPage /> },
  { path: '/services/:slug', element: <ServiceRedirect /> },
```

- [ ] **Step 4: 빌드하고 확인한다**

Run: `npm run build`
Expected: 성공

브라우저 확인:
- `/tech` — 플랫폼 스펙 8행, 역량 4장, 최신 노트 3건(NO.19 / 18 / 17), 스택 5그룹
- `/services/platform-architecture` → `/tech#cap-platform-architecture`로 이동하고 해당 카드가 헤더에 가리지 않는 위치에 온다
- `/services/없는것` → `/tech`

- [ ] **Step 5: 커밋**

```bash
git add src/site/pages src/main.tsx
git commit -m "feat(site): 기술 페이지 + 구 services 라우트 리다이렉트"
```

---

## Task 9: 엔지니어링 노트 페이지

**Files:**
- Create: `src/site/pages/NotesIndexPage.tsx`, `src/site/pages/NoteDetailPage.tsx`
- Modify: `src/main.tsx`

**Interfaces:**
- Consumes: Task 3 `notes`/`getNote`/`getNoteBody`/`allNoteTags`/`NoteBody`/`tableOfContents`, Task 4 프리미티브, Task 6 `SitePage`
- Produces: 라우트 `/tech/notes`, `/tech/notes/:id`

- [ ] **Step 1: 노트 인덱스**

`src/site/pages/NotesIndexPage.tsx`:

```tsx
import { useMemo, useState } from 'react';
import { Link as RouterLink } from 'react-router-dom';
import Box from '@mui/material/Box';
import Container from '@mui/material/Container';
import Stack from '@mui/material/Stack';
import Typography from '@mui/material/Typography';
import TextField from '@mui/material/TextField';
import Chip from '@mui/material/Chip';
import Button from '@mui/material/Button';
import ArrowBackRoundedIcon from '@mui/icons-material/ArrowBackRounded';
import SitePage from '../components/SitePage';
import { HairlineCard, MONO } from '../ui';
import { notes, allNoteTags } from '../content';

export default function NotesIndexPage() {
  const [query, setQuery] = useState('');
  const [tag, setTag] = useState<string | null>(null);
  const tags = useMemo(() => allNoteTags(), []);

  const visible = useMemo(() => {
    const q = query.trim().toLowerCase();
    return notes
      .filter((n) => (tag ? n.tags.includes(tag) : true))
      .filter((n) => (q ? n.title.toLowerCase().includes(q) || n.tags.some((t) => t.toLowerCase().includes(q)) : true))
      .slice()
      .reverse();
  }, [query, tag]);

  return (
    <SitePage>
      <Container maxWidth="md" sx={{ pt: { xs: 6, md: 10 }, pb: { xs: 8, md: 12 } }}>
        <Button component={RouterLink} to="/tech" startIcon={<ArrowBackRoundedIcon />} sx={{ color: 'text.secondary', mb: 3, ml: -1 }}>
          기술
        </Button>
        <Typography component="h1" sx={{ fontWeight: 800, letterSpacing: '-0.03em', fontSize: 'clamp(2rem, 5.5vw, 3rem)', lineHeight: 1.1 }}>
          엔지니어링 노트
        </Typography>
        <Typography sx={{ mt: 2, color: 'text.secondary', maxWidth: 620, lineHeight: 1.7 }}>
          플랫폼을 만들며 남긴 기록 {notes.length}편. 인증부터 관측까지 시간순으로 이어진다.
        </Typography>

        <Stack spacing={2} sx={{ mt: 5, mb: 4 }}>
          <TextField
            label="노트 검색"
            placeholder="제목·태그"
            size="small"
            value={query}
            onChange={(e) => setQuery(e.target.value)}
            sx={{ maxWidth: 360 }}
          />
          <Box sx={{ display: 'flex', flexWrap: 'wrap', gap: 0.75 }}>
            <Chip
              label="전체"
              size="small"
              variant={tag === null ? 'filled' : 'outlined'}
              color={tag === null ? 'primary' : 'default'}
              onClick={() => setTag(null)}
              sx={{ borderRadius: '4px' }}
            />
            {tags.map((t) => (
              <Chip
                key={t}
                label={t}
                size="small"
                variant={tag === t ? 'filled' : 'outlined'}
                color={tag === t ? 'primary' : 'default'}
                onClick={() => setTag(tag === t ? null : t)}
                sx={{ borderRadius: '4px' }}
              />
            ))}
          </Box>
        </Stack>

        {visible.length === 0 ? (
          <Typography sx={{ color: 'text.secondary', py: 6, textAlign: 'center' }}>
            조건에 맞는 노트가 없습니다. 검색어나 태그를 바꿔 보세요.
          </Typography>
        ) : (
          <Stack spacing={1.5}>
            {visible.map((n) => (
              <HairlineCard key={n.id} to={`/tech/notes/${n.id}`}>
                <Stack direction="row" spacing={2} sx={{ alignItems: 'baseline' }}>
                  <Typography
                    sx={{ fontFamily: MONO, fontSize: '0.75rem', color: 'primary.main', fontVariantNumeric: 'tabular-nums', flexShrink: 0 }}
                  >
                    NO.{n.id}
                  </Typography>
                  <Box sx={{ flexGrow: 1, minWidth: 0 }}>
                    <Typography sx={{ fontWeight: 600, lineHeight: 1.5 }}>{n.title}</Typography>
                    <Stack direction="row" spacing={1} sx={{ mt: 1, flexWrap: 'wrap', gap: 0.5, alignItems: 'center' }}>
                      <Typography variant="caption" sx={{ color: 'text.secondary', fontVariantNumeric: 'tabular-nums' }}>
                        {n.date}
                      </Typography>
                      {n.tags.slice(0, 4).map((t) => (
                        <Chip key={t} label={t} size="small" variant="outlined" sx={{ borderRadius: '4px', height: 20, fontSize: '0.7rem' }} />
                      ))}
                    </Stack>
                  </Box>
                </Stack>
              </HairlineCard>
            ))}
          </Stack>
        )}
      </Container>
    </SitePage>
  );
}
```

- [ ] **Step 2: 노트 본문**

`src/site/pages/NoteDetailPage.tsx`:

```tsx
import { Link as RouterLink, useParams } from 'react-router-dom';
import Box from '@mui/material/Box';
import Container from '@mui/material/Container';
import Divider from '@mui/material/Divider';
import Stack from '@mui/material/Stack';
import Typography from '@mui/material/Typography';
import Button from '@mui/material/Button';
import Chip from '@mui/material/Chip';
import Link from '@mui/material/Link';
import ArrowBackRoundedIcon from '@mui/icons-material/ArrowBackRounded';
import ArrowForwardRoundedIcon from '@mui/icons-material/ArrowForwardRounded';
import SitePage from '../components/SitePage';
import { NoteBody, tableOfContents, MONO } from '../ui';
import { notes, getNote, getNoteBody } from '../content';
import NotFoundPage from '../../app/pages/NotFoundPage';

/** 데스크톱 전용 목차. 모바일에서는 숨긴다(본문 위 긴 링크 목록이 더 방해된다). */
function Toc({ markdown }: { markdown: string }) {
  const entries = tableOfContents(markdown);
  if (entries.length < 3) return null; // 항목이 적으면 목차가 소음이다
  return (
    <Box
      component="nav"
      aria-label="이 노트의 목차"
      sx={{ display: { xs: 'none', lg: 'block' }, position: 'sticky', top: 88, width: 220, flexShrink: 0 }}
    >
      <Typography sx={{ fontFamily: MONO, fontSize: '0.7rem', letterSpacing: '0.14em', color: 'text.secondary', mb: 1.5 }}>
        CONTENTS
      </Typography>
      <Stack spacing={0.75}>
        {entries.map((e) => (
          <Link
            key={e.id}
            href={`#${e.id}`}
            underline="hover"
            variant="body2"
            sx={{ color: 'text.secondary', pl: e.level === 3 ? 1.5 : 0, lineHeight: 1.5, '&:hover': { color: 'primary.main' } }}
          >
            {e.text}
          </Link>
        ))}
      </Stack>
    </Box>
  );
}

export default function NoteDetailPage() {
  const { id } = useParams<{ id: string }>();
  const note = getNote(id);
  const body = id ? getNoteBody(id) : undefined;
  if (!note || body === undefined) return <NotFoundPage />;

  const at = notes.findIndex((n) => n.id === note.id);
  const prev = at > 0 ? notes[at - 1] : undefined;
  const next = at < notes.length - 1 ? notes[at + 1] : undefined;

  return (
    <SitePage>
      <Container maxWidth="lg" sx={{ pt: { xs: 5, md: 8 }, pb: { xs: 8, md: 12 }, maxWidth: { lg: 1120 } }}>
        <Stack direction="row" spacing={6} sx={{ alignItems: 'flex-start' }}>
        <Box sx={{ flexGrow: 1, minWidth: 0, maxWidth: 720 }}>
        <Button component={RouterLink} to="/tech/notes" startIcon={<ArrowBackRoundedIcon />} sx={{ color: 'text.secondary', mb: 3, ml: -1 }}>
          노트 목록
        </Button>

        <Typography sx={{ fontFamily: MONO, fontSize: '0.75rem', color: 'primary.main', letterSpacing: '0.1em' }}>
          NO.{note.id}
        </Typography>
        <Typography component="h1" sx={{ fontWeight: 800, mt: 1.5, letterSpacing: '-0.025em', lineHeight: 1.2, fontSize: 'clamp(1.8rem, 4.5vw, 2.6rem)' }}>
          {note.title}
        </Typography>
        <Stack direction="row" spacing={1} sx={{ mt: 2.5, flexWrap: 'wrap', gap: 0.75, alignItems: 'center' }}>
          <Typography variant="body2" sx={{ color: 'text.secondary', fontVariantNumeric: 'tabular-nums' }}>
            {note.date}
          </Typography>
          {note.status && <Chip label={note.status} size="small" variant="outlined" sx={{ borderRadius: '4px' }} />}
          {note.tags.map((t) => (
            <Chip key={t} label={t} size="small" variant="outlined" sx={{ borderRadius: '4px' }} />
          ))}
        </Stack>

        <Divider sx={{ my: { xs: 4, md: 5 } }} />

        <NoteBody markdown={body} />

        <Divider sx={{ my: { xs: 5, md: 7 } }} />

        <Stack direction={{ xs: 'column', sm: 'row' }} spacing={2} sx={{ justifyContent: 'space-between' }}>
          <Box>
            {prev && (
              <Button component={RouterLink} to={`/tech/notes/${prev.id}`} startIcon={<ArrowBackRoundedIcon />} sx={{ textAlign: 'left' }}>
                NO.{prev.id} {prev.title}
              </Button>
            )}
          </Box>
          <Box>
            {next && (
              <Button component={RouterLink} to={`/tech/notes/${next.id}`} endIcon={<ArrowForwardRoundedIcon />} sx={{ textAlign: 'right' }}>
                NO.{next.id} {next.title}
              </Button>
            )}
          </Box>
        </Stack>
        </Box>

        <Toc markdown={body} />
        </Stack>
      </Container>
    </SitePage>
  );
}
```

- [ ] **Step 3: 라우트 등록**

`src/main.tsx`에 추가:

```tsx
import NotesIndexPage from './site/pages/NotesIndexPage';
import NoteDetailPage from './site/pages/NoteDetailPage';
```

```tsx
  { path: '/tech/notes', element: <NotesIndexPage />, errorElement: <RouteErrorPage /> },
  { path: '/tech/notes/:id', element: <NoteDetailPage />, errorElement: <RouteErrorPage /> },
```

> 주의: `/tech/notes`는 `/tech`보다 **뒤에** 와도 무방하다 (`createBrowserRouter`는 정확 매칭).
> 단 `/tech/notes/:id`가 `/tech/notes`보다 먼저 오면 안 된다.

- [ ] **Step 4: 빌드하고 전수 확인한다**

Run: `npm run build`
Expected: 성공

브라우저 확인:
- `/tech/notes` — 20건이 NO.19 → NO.00 역순. 태그 칩 클릭 시 필터, 검색어 입력 시 좁혀짐, 결과 0건일 때 안내 문구
- `/tech/notes/00` — 본문 렌더. **코드블록이 가로로 넘치지 않고 자체 스크롤**되는지, 표가 렌더되는지
- `/tech/notes/12` — 본문 안 내부 링크(`/tech/notes/NN`)를 눌러 이동되는지
- 1280px 이상에서 우측 목차가 뜨고, **항목을 눌렀을 때 해당 헤딩으로 정확히 스크롤**되는지
  (한글 헤딩 앵커가 맞는지 확인하는 지점 — 어긋나면 `slug.ts`와 `NoteBody`의 중복 카운팅이 갈린 것)
- 콜아웃이 있는 노트(예: `grep -l "\[NOTE\]\|\[WARNING\]" src/site/content/notes/*.md`로 찾는다) — 왼쪽 파란 세로선 인용문에 `[TYPE] 제목`이 굵게
- `/tech/notes/99` — NotFound
- 다크모드에서 코드블록·표 배경이 본문과 구분되는지

- [ ] **Step 5: 커밋**

```bash
git add src/site/pages src/main.tsx
git commit -m "feat(site): 엔지니어링 노트 인덱스·본문 페이지"
```

---

## Task 10: 소개 · 연락 페이지

**Files:**
- Create: `src/site/pages/AboutPage.tsx`, `src/site/pages/ContactPage.tsx`
- Modify: `src/main.tsx`

**Interfaces:**
- Consumes: Task 5 `career`/`caseStudies`/`stats`/`CONTACT_EMAIL`/`GITHUB_URL`/`PORTFOLIO_URL`, Task 4 프리미티브
- Produces: 라우트 `/about`, `/contact`

- [ ] **Step 1: AboutPage**

`src/site/pages/AboutPage.tsx`:

```tsx
import Avatar from '@mui/material/Avatar';
import Box from '@mui/material/Box';
import CardMedia from '@mui/material/CardMedia';
import Chip from '@mui/material/Chip';
import Link from '@mui/material/Link';
import Container from '@mui/material/Container';
import Grid from '@mui/material/Grid';
import Stack from '@mui/material/Stack';
import Typography from '@mui/material/Typography';
import SitePage from '../components/SitePage';
import { GridSection, StatBar, MONO } from '../ui';
import { career, caseStudies, stats } from '../content';

/**
 * 다이어그램 프레임.
 *
 * 고정 높이를 쓰지 않는다 — 1549×1524 다이어그램을 200px 높이에 `contain` 으로 넣으면
 * 실제 렌더 폭이 203px, 원본의 13% 로 줄어 라벨을 아예 읽을 수 없다.
 * 대신 원본 비율로 컬럼 폭을 꽉 채우고, 비율을 미리 선언해 로딩 중 레이아웃이 밀리지 않게 한다.
 *
 * 배경도 `common.white` 고정을 쓰지 않는다 — 수상 공고 이미지는 자체 배경이 어두워서,
 * 다크모드에서 그 주위에만 순백 여백이 남아 오히려 눈에 튄다. 비율을 맞추면 여백 자체가
 * 거의 없어지고, 남는 부분은 `background.paper` 라 양쪽 모드에서 자연스럽다.
 *
 * 조밀한 다이어그램은 컬럼 폭으로도 부족하므로 원본을 새 탭에서 열 수 있게 한다.
 */
function DiagramFrame({ src, alt, w, h }: { src: string; alt: string; w: number; h: number }) {
  return (
    <Box sx={{ border: '1px solid', borderColor: 'divider', borderRadius: '4px', overflow: 'hidden', bgcolor: 'background.paper' }}>
      <CardMedia
        component="img"
        image={src}
        alt={alt}
        loading="lazy"
        // objectFit 은 안전망이다 — 지금은 비율이 정확히 맞아 차이가 없지만,
        // 이미지를 교체하거나 w/h 를 잘못 적으면 기본값(fill)이 그림을 늘려 버린다.
        sx={{ width: '100%', height: 'auto', aspectRatio: `${w} / ${h}`, objectFit: 'contain', display: 'block' }}
      />
      <Box sx={{ px: 1.5, py: 0.75, borderTop: '1px solid', borderColor: 'divider' }}>
        {/*
          링크 텍스트가 세 이미지에서 같아서, 스크린리더의 링크 목록으로 훑으면 어느 이미지의
          원본인지 구분되지 않는다. alt 를 접근성 이름에 붙여 고유하게 만든다.
          display:'block' 은 부모 패딩까지 탭 타깃으로 넓히기 위한 것.
        */}
        <Link
          href={src}
          target="_blank"
          rel="noopener"
          variant="caption"
          aria-label={`${alt} 원본 크기로 보기`}
          sx={{ display: 'block', color: 'text.secondary' }}
        >
          원본 크기로 보기
        </Link>
      </Box>
    </Box>
  );
}

export default function AboutPage() {
  return (
    <SitePage>
      <Container maxWidth="lg" sx={{ pt: { xs: 7, md: 12 }, pb: { xs: 5, md: 8 } }}>
        <Stack direction="row" spacing={1.5} sx={{ alignItems: 'center', mb: 3.5 }}>
          <Avatar alt="김찬호 프로필 사진" src="/profile.png" sx={{ width: 44, height: 44 }} />
          <Typography variant="body2" sx={{ color: 'text.secondary' }}>
            김찬호 · 플랫폼 백엔드 엔지니어
          </Typography>
        </Stack>
        <Typography component="h1" sx={{ fontWeight: 800, letterSpacing: '-0.03em', fontSize: 'clamp(2.2rem, 6vw, 3.4rem)', lineHeight: 1.08 }}>
          소개
        </Typography>
        <Typography sx={{ mt: 2.5, color: 'text.secondary', maxWidth: 640, fontSize: '1.1rem', lineHeight: 1.7 }}>
          플랫폼을 설계하고, 데이터로 굴러가게 만들고, 팀이 더 빠르게 만들 환경까지 함께 세웁니다.
        </Typography>
      </Container>

      <GridSection index="01" label="RECORD" title="숫자로 본 기록">
        <StatBar items={stats} />
      </GridSection>

      <GridSection index="02" label="CAREER" title="커리어">
        <Box>
          {career.map((h, i) => (
            <Stack
              key={h.year}
              direction={{ xs: 'column', sm: 'row' }}
              spacing={{ xs: 0.5, sm: 3 }}
              sx={{ py: 2.5, borderTop: i === 0 ? '1px solid' : 0, borderBottom: '1px solid', borderColor: 'divider', alignItems: { sm: 'baseline' } }}
            >
              <Typography sx={{ minWidth: 172, fontWeight: 700, color: 'primary.main', fontVariantNumeric: 'tabular-nums' }}>
                {h.year}
              </Typography>
              <Typography sx={{ color: 'text.primary' }}>{h.text}</Typography>
            </Stack>
          ))}
        </Box>
      </GridSection>

      <GridSection index="03" label="CASE STUDIES" title="문제를 어떻게 시스템으로 바꿨는가" caption="문제 → 해결 → 성과.">
        <Stack spacing={{ xs: 6, md: 9 }}>
          {caseStudies.map((c) => (
            <Box key={c.title}>
              <Typography
                variant="overline"
                sx={{ color: 'primary.main', fontWeight: 700, letterSpacing: '0.18em', display: 'block' }}
              >
                {c.eyebrow}
              </Typography>
              <Typography variant="h5" component="h3" sx={{ fontWeight: 700, mt: 1.5, mb: 3, lineHeight: 1.35, letterSpacing: '-0.01em' }}>
                {c.title}
              </Typography>
              <Grid container spacing={{ xs: 3, md: 5 }}>
                <Grid size={{ xs: 12, md: c.images ? 7 : 12 }}>
                  <Stack spacing={2}>
                    {([['문제', c.problem], ['해결', c.solution], ['성과', c.result]] as const).map(([label, text]) => (
                      <Stack key={label} direction="row" spacing={2} sx={{ alignItems: 'flex-start' }}>
                        <Typography
                          variant="caption"
                          sx={{
                            fontFamily: MONO,
                            fontWeight: 700,
                            color: label === '성과' ? 'primary.main' : 'text.secondary',
                            minWidth: 40,
                            pt: 0.4,
                            flexShrink: 0,
                          }}
                        >
                          {label}
                        </Typography>
                        <Typography
                          variant="body2"
                          sx={{
                            lineHeight: 1.8,
                            color: label === '성과' ? 'text.primary' : 'text.secondary',
                            fontWeight: label === '성과' ? 500 : 400,
                          }}
                        >
                          {text}
                        </Typography>
                      </Stack>
                    ))}
                  </Stack>
                  <Box sx={{ display: 'flex', flexWrap: 'wrap', gap: 0.75, mt: 3 }}>
                    {c.tags.map((tag) => (
                      <Chip key={tag} label={tag} size="small" color="primary" variant="outlined" sx={{ borderRadius: '4px' }} />
                    ))}
                  </Box>
                </Grid>
                {c.images && (
                  <Grid size={{ xs: 12, md: 5 }}>
                    <Stack spacing={2}>
                      {c.images.map((img) => (
                        <DiagramFrame key={img.src} src={img.src} alt={img.alt} w={img.w} h={img.h} />
                      ))}
                    </Stack>
                  </Grid>
                )}
              </Grid>
            </Box>
          ))}
        </Stack>
      </GridSection>
      <Box sx={{ borderTop: '1px solid', borderColor: 'divider' }} />
    </SitePage>
  );
}
```

- [ ] **Step 2: ContactPage**

`src/site/pages/ContactPage.tsx`:

```tsx
import Container from '@mui/material/Container';
import Stack from '@mui/material/Stack';
import Typography from '@mui/material/Typography';
import Button from '@mui/material/Button';
import GitHubIcon from '@mui/icons-material/GitHub';
import EmailRoundedIcon from '@mui/icons-material/EmailRounded';
import LaunchRoundedIcon from '@mui/icons-material/LaunchRounded';
import SitePage from '../components/SitePage';
import { SpecTable } from '../ui';
import { CONTACT_EMAIL, GITHUB_URL, PORTFOLIO_URL, career } from '../content';

export default function ContactPage() {
  return (
    <SitePage>
      <Container maxWidth="md" sx={{ py: { xs: 8, md: 14 } }}>
        <Typography component="h1" sx={{ fontWeight: 800, letterSpacing: '-0.03em', fontSize: 'clamp(2rem, 5.5vw, 3rem)', lineHeight: 1.12 }}>
          함께 일할 사람을 찾고 계신가요?
        </Typography>
        <Typography sx={{ mt: 2.5, color: 'text.secondary', maxWidth: 560, fontSize: '1.1rem', lineHeight: 1.7 }}>
          플랫폼을 설계하고, 데이터로 굴러가게 만들고, 팀이 더 빠르게 만들 환경까지 함께 세울 사람입니다.
        </Typography>

        <Stack direction="row" spacing={1.5} sx={{ flexWrap: 'wrap', gap: 1.5, mt: 5, mb: 7 }}>
          <Button variant="contained" size="large" href={`mailto:${CONTACT_EMAIL}`} startIcon={<EmailRoundedIcon />} sx={{ borderRadius: '4px' }}>
            이메일 보내기
          </Button>
          <Button variant="outlined" size="large" href={PORTFOLIO_URL} target="_blank" rel="noopener" startIcon={<LaunchRoundedIcon />} sx={{ borderRadius: '4px' }}>
            포트폴리오
          </Button>
          <Button variant="outlined" size="large" href={GITHUB_URL} target="_blank" rel="noopener" startIcon={<GitHubIcon />} sx={{ borderRadius: '4px' }}>
            GitHub
          </Button>
        </Stack>

        <SpecTable
          rows={[
            { label: 'Email', value: CONTACT_EMAIL },
            { label: 'GitHub', value: GITHUB_URL.replace(/^https?:\/\//, '') },
            // career[0] 을 그대로 쓴다 — 여기서 문구를 다시 쓰면 이직·직함 변경 때 조용히 낡는다.
            { label: '현재', value: career[0].text },
          ]}
        />
      </Container>
    </SitePage>
  );
}
```

- [ ] **Step 3: 라우트 등록 후 빌드·확인**

`src/main.tsx`에 추가:

```tsx
import AboutPage from './site/pages/AboutPage';
import ContactPage from './site/pages/ContactPage';
```

```tsx
  { path: '/about', element: <AboutPage />, errorElement: <RouteErrorPage /> },
  { path: '/contact', element: <ContactPage />, errorElement: <RouteErrorPage /> },
```

Run: `npm run build` → 성공
브라우저: `/about` — 스탯바 4칸, 커리어 2행, 케이스 3건(A-RMS 이미지 2장 · MSA 이미지 1장 표시).
`/contact` — 버튼 3개 + 스펙 3행.

- [ ] **Step 4: 커밋**

```bash
git add src/site/pages src/main.tsx
git commit -m "feat(site): 소개·연락 페이지"
```

---

## Task 11: 홈을 게이트웨이형 랜딩으로 재작성

**Files:**
- Modify: `src/pages/Home.tsx` (전면 재작성)

**Interfaces:**
- Consumes: Task 4 프리미티브, Task 5 콘텐츠 전부, Task 6 `SitePage`
- Produces: 없음 (최종 소비자)

- [ ] **Step 1: Home 을 재작성한다**

`src/pages/Home.tsx` 전체를 다음으로 교체한다. 인라인 상수(`caseStudies`, `history`, `techGroups`, `devLinks`)는 Task 5에서 이미 옮겼으므로 여기서 전부 사라진다.

```tsx
import { Link as RouterLink } from 'react-router-dom';
import { alpha } from '@mui/material/styles';
import Avatar from '@mui/material/Avatar';
import Box from '@mui/material/Box';
import Container from '@mui/material/Container';
import Grid from '@mui/material/Grid';
import Stack from '@mui/material/Stack';
import Typography from '@mui/material/Typography';
import Button from '@mui/material/Button';
import ArrowForwardRoundedIcon from '@mui/icons-material/ArrowForwardRounded';
import SitePage from '../site/components/SitePage';
import { GridSection, StatBar, HairlineCard, MONO } from '../site/ui';
import { stats, ossProducts, capabilities, notes } from '../site/content';

export default function Home() {
  const latest = notes.slice(-3).reverse();

  return (
    <SitePage>
      {/* Hero — 가치 제안 */}
      <Box
        component="section"
        sx={{
          background: (theme) =>
            `radial-gradient(ellipse 90% 55% at 12% -10%, ${alpha(
              theme.palette.primary.main,
              theme.palette.mode === 'dark' ? 0.18 : 0.07,
            )}, transparent)`,
        }}
      >
        <Container maxWidth="lg" sx={{ pt: { xs: 9, md: 16 }, pb: { xs: 7, md: 12 } }}>
          <Stack direction="row" spacing={1.5} sx={{ alignItems: 'center', mb: 4 }}>
            <Avatar alt="김찬호 프로필 사진" src="/profile.png" sx={{ width: 40, height: 40 }} />
            <Typography variant="body2" sx={{ color: 'text.secondary' }}>
              김찬호 · 플랫폼 백엔드 엔지니어
            </Typography>
          </Stack>

          <Typography component="h1" sx={{ fontWeight: 800, letterSpacing: '-0.04em', lineHeight: 1.02, fontSize: 'clamp(2.8rem, 8.5vw, 5.5rem)' }}>
            수작업을{' '}
            <Box component="span" sx={{ color: 'primary.main' }}>
              시스템
            </Box>
            으로
            <br />
            바꿉니다
          </Typography>
          <Typography sx={{ mt: 4, maxWidth: 600, color: 'text.secondary', fontSize: 'clamp(1.05rem, 2.2vw, 1.3rem)', lineHeight: 1.6 }}>
            플랫폼 설계부터 운영까지 — 데이터 기반 의사결정 체계를 만드는 엔지니어링.
          </Typography>
          <Stack direction="row" spacing={1.5} sx={{ flexWrap: 'wrap', gap: 1.5, mt: 5 }}>
            <Button component={RouterLink} to="/products" variant="contained" size="large" endIcon={<ArrowForwardRoundedIcon />} sx={{ borderRadius: '4px' }}>
              제품 보기
            </Button>
            <Button component={RouterLink} to="/contact" variant="outlined" size="large" sx={{ borderRadius: '4px' }}>
              문의하기
            </Button>
          </Stack>
        </Container>
      </Box>

      {/* 신뢰 바 */}
      <Container maxWidth="lg" sx={{ pb: { xs: 2, md: 4 } }}>
        <StatBar items={stats} />
      </Container>

      <GridSection index="01" label="PRODUCTS" title="만든 것들" caption="직접 설계하고 공개한 제품들.">
        <Grid container spacing={{ xs: 2, md: 2.5 }}>
          {ossProducts.map((p) => (
            <Grid key={p.slug} size={{ xs: 12, sm: 6, md: 3 }}>
              <HairlineCard to={`/products/${p.slug}`}>
                <Typography variant="h6" component="h3" sx={{ fontWeight: 700, letterSpacing: '-0.01em', mb: 1 }}>
                  {p.name}
                </Typography>
                <Typography variant="body2" sx={{ color: 'text.secondary', lineHeight: 1.7 }}>
                  {p.tagline}
                </Typography>
              </HairlineCard>
            </Grid>
          ))}
        </Grid>
        <Button component={RouterLink} to="/products" endIcon={<ArrowForwardRoundedIcon />} sx={{ mt: 2.5, ml: -1 }}>
          제품 전체 보기
        </Button>
      </GridSection>

      <GridSection index="02" label="CAPABILITIES" title="무엇을 해드리는가">
        <Grid container spacing={{ xs: 2, md: 2.5 }}>
          {capabilities.map((c) => (
            <Grid key={c.slug} size={{ xs: 12, sm: 6 }}>
              <HairlineCard to={`/tech#cap-${c.slug}`}>
                <Typography variant="caption" sx={{ fontFamily: MONO, letterSpacing: '0.1em', color: 'text.secondary' }}>
                  {c.title}
                </Typography>
                <Typography variant="h6" component="h3" sx={{ fontWeight: 700, mt: 0.5, mb: 1.5 }}>
                  {c.lead}
                </Typography>
                <Typography variant="body2" sx={{ color: 'text.secondary', lineHeight: 1.75 }}>
                  {c.evidence}
                </Typography>
              </HairlineCard>
            </Grid>
          ))}
        </Grid>
      </GridSection>

      <GridSection index="03" label="NOTES" title="엔지니어링 노트" caption={`플랫폼을 만들며 남긴 기록 ${notes.length}편.`}>
        <Stack spacing={1.5}>
          {latest.map((n) => (
            <HairlineCard key={n.id} to={`/tech/notes/${n.id}`}>
              <Stack direction={{ xs: 'column', sm: 'row' }} spacing={{ xs: 0.5, sm: 2 }} sx={{ alignItems: { sm: 'baseline' } }}>
                <Typography sx={{ fontFamily: MONO, fontSize: '0.75rem', color: 'primary.main', fontVariantNumeric: 'tabular-nums', width: { sm: 64 }, flexShrink: 0 }}>
                  NO.{n.id}
                </Typography>
                <Typography sx={{ fontWeight: 600, flexGrow: 1 }}>{n.title}</Typography>
                <Typography variant="caption" sx={{ color: 'text.secondary', fontVariantNumeric: 'tabular-nums' }}>
                  {n.date}
                </Typography>
              </Stack>
            </HairlineCard>
          ))}
        </Stack>
        <Button component={RouterLink} to="/tech/notes" endIcon={<ArrowForwardRoundedIcon />} sx={{ mt: 2.5, ml: -1 }}>
          노트 전체 보기
        </Button>
      </GridSection>

      <GridSection index="04" label="CONTACT" title="함께 일할 사람을 찾고 계신가요?" caption="플랫폼을 설계하고, 데이터로 굴러가게 만들고, 팀이 더 빠르게 만들 환경까지 함께 세울 사람입니다.">
        <Button component={RouterLink} to="/contact" variant="contained" size="large" endIcon={<ArrowForwardRoundedIcon />} sx={{ borderRadius: '4px' }}>
          연락처 보기
        </Button>
      </GridSection>
      <Box sx={{ borderTop: '1px solid', borderColor: 'divider' }} />
    </SitePage>
  );
}
```

> `Home`이 받던 `disableCustomTheme` prop 은 더 이상 쓰지 않는다 — `SitePage`가 테마를 감싼다.
> `src/main.tsx`에서 `<Home />`로 호출하고 있으므로 시그니처 변경만으로 충분하다.

- [ ] **Step 2: 잔여물을 지운다**

Run: `ls src/pages/landing/`
Expected: 비어 있음. 비어 있으면 `rmdir src/pages/landing`

- [ ] **Step 3: 빌드하고 확인한다**

Run: `npm run build`
Expected: 성공. `noUnusedLocals` 위반이 있으면 미사용 import 를 지운다.

브라우저 `/`:
- 히어로 → 스탯바 → PRODUCTS → CAPABILITIES → NOTES → CONTACT
- 스탯바 숫자가 tabular-nums 로 정렬
- 카드 hover 시 이동 없이 테두리만 변함

- [ ] **Step 4: 커밋**

```bash
git add src/pages src/main.tsx
git commit -m "feat(site): 홈을 게이트웨이형 랜딩으로 재작성"
```

---

## Task 12: 회귀 · 접근성 · 반응형 검수

**Files:**
- Modify: 검수에서 발견된 파일만

**Interfaces:**
- Consumes: Task 1~11 전부
- Produces: 없음

- [ ] **Step 1: 스크립트 테스트와 빌드를 다시 돌린다**

```bash
npm run test:scripts
npm run build
```
Expected: 테스트 `# fail 0`, 빌드 성공

- [ ] **Step 2: 링크 전수 확인**

`npm run dev` 후 아래를 **전부** 클릭한다. 하나라도 404·빈 화면이면 고치고 다시 돈다.

- GNB: 제품 / 기술 / 소개 / 문의하기 — 각 페이지에서 현재 항목이 굵게(활성 표시)
- 로고 `chanho.dev` → `/`
- 푸터 SITE 5개 · DEV 6개 · CONTACT 2개
- `/products` 카드 6장 → 각 상세
- `/tech` 역량 카드 4 · 노트 3 · "노트 전체 보기"
- `/tech/notes` 20건 → 각 본문, 본문 내부 위키링크
- 구 링크: `/services/platform-architecture`, `/services/data-engineering`, `/services/operations-reliability`, `/services/ai-dev-env`
  — 주소만 `/tech`로 바뀌는 게 아니라 **해당 역량 카드까지 실제로 스크롤**되고, 카드가 헤더에
  가리지 않아야 한다. 최상단에 머무르면 `SitePage`의 해시 스크롤 훅이 동작하지 않는 것이다

- [ ] **Step 3: 기존 앱 라우트 회귀**

- `/app` → 로그인 안 됐으면 `/login`, 됐으면 대시보드. 사이드메뉴·게시판 CRUD 정상
- `/designs` 목록·상세·작성
- `/profile` 목록·상세
- `/templates` `/components` `/showcase` `/sign-in` `/dashboard` 진입
- `/없는경로` → NotFound

- [ ] **Step 4: 반응형**

브라우저 devtools 로 **375 / 768 / 1024 / 1440** 각 폭에서 `/`, `/products/wiki`, `/tech`, `/tech/notes/14`를 본다.

- 가로 스크롤이 **없어야** 한다 (코드블록·표는 자기 안에서만 스크롤)
- 375px 에서 GridSection 라벨이 콘텐츠 위로 적층
- 375px 에서 StatBar 가 2×2
- 모바일 드로어 열기/닫기, 항목 클릭 시 닫힘

- [ ] **Step 5: 접근성**

- 각 페이지에서 **Tab 을 한 번만 누르면 "본문으로 건너뛰기" 링크가 화면에 나타나고**, Enter 로
  본문(`#main-content`)에 포커스가 옮겨간다. 헤더를 통과하지 않고 본문에 닿을 수 있어야 한다
- 키보드 Tab 만으로 GNB → 본문 카드 → 푸터 순회, **포커스 링이 모든 단계에서 보인다**
- 각 페이지 h1 이 정확히 하나 (devtools 에서 `document.querySelectorAll('h1').length` 로 확인)
- 아이콘 전용 버튼(메뉴 열기/닫기, 컬러모드)에 `aria-label` 존재
- OS 설정에서 "동작 줄이기"를 켠 뒤 카드 hover — 전이가 즉시 적용된다
- 라이트·다크 각각에서 본문 텍스트 대비 확인 (devtools > Elements > Accessibility > Contrast)

- [ ] **Step 6: 최종 커밋**

```bash
git add -A
git commit -m "fix(site): 리뉴얼 검수 반영"
```

---

## 부록: 노트 갱신 절차

볼트에 `20 ...md`를 추가한 뒤:

```bash
npm run sync:notes     # 산출물 갱신 + 경고 확인
npm run build          # 타입 게이트
git add src/site/content/notes
git commit -m "chore(notes): 노트 동기화"
```

사이트 코드는 손대지 않는다 — 인덱스·라우트·목록이 전부 `noteIndex`에서 파생된다.
