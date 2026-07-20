---
tags: [msa, template, wiki, redis-streams, grpc, 낙관적잠금, 리비전, 구현기록, 해설]
작성일: 2026-07-21
상태: Wave B 완료 (14태스크 + 최종 fix 3건, 29/29 테스트, 병합 Ready) · 수동 체크리스트 5건 대기 · 다음 Wave C(search/ES)
---

# 17 — Wave B: wiki-backend 구현기록 — 과정과 원리 해설 (2026-07-21)

상위: [[15 ALM·Wiki 백엔드 요구사항 + 서비스 분할 설계 (2026-07-19)]] · 선행: [[16 Wave A — platform-backend·common-proto·org-service 구현기록 (2026-07-19)]]

## 한눈에 보기

**첫 번째 제품 서비스 wiki-backend가 완성됐다.** 스페이스·페이지(계층 트리)·버전 히스토리·첨부파일을 갖춘 위키 코어로, Wave A가 만든 권한 서비스(org-service)를 gRPC로 소비하는 첫 사례이자, Kafka 대신 채택한 Redis Streams로 이벤트를 발행하는 첫 사례다.

- repo: [chanho4702/wiki-backend](https://github.com/chanho4702/wiki-backend) (REST :9110, 게이트웨이 `/api/wiki/**`)
- 테스트 29/29 · CI green(첫 시도) · compose 13컨테이너 healthy · 401 스모크 실측
- 문서 체계: 스펙 `specs/2026-07-20-wave-b-wiki-backend-design.md` · 플랜 `plans/2026-07-20-wave-b-wiki-backend.md`(14태스크)

---

## 1부. 무엇을 만들었나 — 쉽게 설명하면

### 위키의 뼈대: 스페이스와 페이지 트리

**스페이스**는 위키의 "책장"이다 — 팀이나 프로젝트마다 하나씩 만든다(예: 개발팀 스페이스, 운영 스페이스). **페이지**는 그 책장 안의 문서인데, 폴더처럼 페이지 아래에 페이지를 넣을 수 있다(계층 트리). 트리라서 두 가지 규칙을 코드로 지킨다:

1. **부모는 같은 스페이스여야 한다** — 개발 스페이스의 페이지를 운영 스페이스 페이지 아래로 못 붙인다
2. **자기 자신이나 자기 자손 아래로 이동 금지** — 허용하면 "A의 부모가 B, B의 부모가 A"라는 고리(순환)가 생겨 트리가 망가진다. 이동할 때마다 새 부모의 조상을 끝까지 따라 올라가며 자기 자신이 나오는지 검사한다.

### 버전 히스토리: "저장할 때마다 전체 사진을 찍는다"

편집 이력은 **diff(변경분)가 아니라 전체 스냅샷**으로 저장한다. 저장 버튼을 누를 때마다 그 시점의 제목·본문 전체를 `page_revision` 테이블에 한 행으로 복사하는 것 — 사진첩에 매번 전체 사진을 넣는 방식이다.

- 왜? diff 방식은 저장공간을 아끼지만 "v3로 복원"하려면 v1부터 변경분을 차례로 재생해야 해서 복잡하고 깨지기 쉽다. 스냅샷은 그 행을 그대로 읽으면 끝. 문서 수십만 건 규모(M 티어)에서 텍스트 스토리지는 충분히 싸다.
- **불변식 하나로 정리**: "존재했던 모든 버전은 리비전 테이블에 있다." 페이지 생성 시 v1도 즉시 리비전에 넣는 이유다.
- **복원은 롤백이 아니다**: v1로 복원하면 이력이 v1로 되감기는 게 아니라, v1의 내용을 가진 **새 버전(v3)**이 생긴다. "누가 언제 복원했는지"까지 이력에 남는다.

### 동시 편집 충돌: 낙관적 잠금(optimistic lock)

두 사람이 같은 문서를 동시에 열어 고치면? 문서를 잠그는 대신(비관적 잠금), **"충돌은 드물 것"이라 낙관하고 저장 시점에만 검사**한다. 클라이언트는 저장할 때 "내가 v3을 보고 고쳤어(`expectedVersion: 3`)"를 함께 보낸다. 서버의 현재 버전이 3이면 저장(→v4), 이미 누가 v4로 만들어놨으면 **409 Conflict**로 거절 — 프론트가 "다른 사람이 먼저 저장했어요"를 띄우고 병합을 안내한다. 잠금 관리 서버도, 소켓도 필요 없는 가장 단순한 안전장치다.

### 첨부파일: 파일명을 믿지 않는다

업로드된 파일은 디스크에 **UUID 이름**으로 저장하고, 원본 파일명은 DB에만 둔다. `../../etc/passwd` 같은 이름으로 경로를 조작하는 공격이 원천 차단되고, 같은 이름 파일 충돌도 없다. 다운로드 응답에는 `Content-Disposition: attachment`를 고정해 브라우저가 HTML 첨부를 "실행"하지 않고 무조건 내려받게 한다(XSS 차단).

### 권한: 위키는 판정하지 않고 "질문"만 한다

위키에는 권한 테이블이 없다. 모든 판정은 Wave A의 org-service에 gRPC로 물어본다: "사용자 5가 스페이스 7을 EDIT할 수 있어?" — 이 구조 덕에 권한 정책(팀 상속, GLOBAL 관리자)이 바뀌어도 위키는 재배포가 필요 없다. 실무 장치 세 가지:

- **Caffeine 캐시 30초**: 같은 질문을 30초간 기억해 gRPC 왕복을 아낀다. 대가는 "권한 회수가 최대 30초 늦게 반영" — 수용하고 문서화했다.
- **fail-closed**: org-service가 죽으면 "모름 → 허용"이 아니라 **"모름 → 거부(403)"**. 인가에서는 가용성보다 안전이 우선이다.
- **스페이스를 만들면 만든 사람이 자동 관리자**가 된다 — 이를 위해 proto에 `CreateGrant` rpc를 추가했다(v0.2.0). 위키가 org에 "이 사람에게 이 스페이스 ADMIN 줘"라고 요청하는 내부 전용 창구.

### 이벤트: 우편함에 넣고 끝 (Redis Streams)

페이지를 저장하면 "PageUpdated 사건이 있었음"이라는 쪽지를 **Redis Streams**(`platform:events:v1`)에 넣는다. 나중에 검색 서비스(Wave C)와 알림 서비스(Wave E)가 각자 속도로 꺼내 간다. [[15 ALM·Wiki 백엔드 요구사항 + 서비스 분할 설계 (2026-07-19)|15번]]에서 Kafka 대신 확정한 결정 — 이미 스택에 있는 Redis를 재사용해 설치판 컨테이너 +0.

세부 설계 네 가지:

1. **커밋 후 발행**: DB 트랜잭션이 실제로 커밋된 **뒤에만** 쪽지를 넣는다. 저장이 롤백됐는데 "저장됐어요" 쪽지가 나가는 사고 방지. (스프링 TransactionSynchronization의 afterCommit 훅 — 롤백/비트랜잭션 경로까지 TDD 3케이스)
2. **발행 실패는 경고 로그만**: Redis가 잠깐 죽어도 페이지 저장은 성공한다. 정합의 최종 근거는 어디까지나 DB이고, 색인이 어긋나면 Wave C의 전체 재색인으로 보정한다는 설계.
3. **쪽지에 본문을 안 싣는다**: "페이지 12가 바뀌었다"만 알리고, 내용이 필요한 소비자는 API로 가져간다. 이벤트 비대화 방지.
4. **`EventPublisher` 인터페이스 뒤에 숨겼다**: 나중에 정말 Kafka가 필요해지면 구현체만 갈아끼운다 — 위키 코드는 한 줄도 안 바뀐다.

---

## 2부. 과정 — 어떻게 진행됐나 (세세하게)

### 진행 방식: 태스크마다 "만드는 AI"와 "검사하는 AI"를 분리

Wave A와 같은 SDD(서브에이전트 구동 개발): 플랜의 14개 태스크마다 ① 구현 에이전트가 브리프(플랜에서 추출한 요구+전체 코드)를 받아 TDD로 구현·커밋하고, ② 별도의 리뷰 에이전트가 브리프와 diff를 대조해 "스펙 일치 + 품질" 두 평결을 내린다. 리뷰가 Needs fixes면 구현자에게 돌려보내 고치고 재리뷰. 전부 끝나면 ③ 최상위 모델이 **전체 브랜치를 통째로** 다시 본다 — 태스크 단위 리뷰가 못 보는 서비스 간 정합·보안 구멍·태스크 경계의 사각지대 담당.

### 타임라인 (2026-07-20 ~ 07-21)

| 단계 | 내용 | 결과 |
|---|---|---|
| T1~T3 | **proto v0.2.0 선행** — events.v1 스키마 + CreateGrant rpc를 platform-backend에 추가, org-service CreateGrant 구현(TDD 3케이스), 태그 발행 | 32/32, 발행 green |
| T4 | wiki-backend repo 골격 — **발행된 proto를 GitHub Packages에서 소비하는 첫 외부 소비자** | contextLoads green |
| T5 | 스키마·엔티티·리포지토리 | 사건 ① 발생(아래) |
| T6~T7 | PermissionClient(gRPC+캐시+fail-closed, in-process 가짜 org 서버로 검증) · EventPublisher+Redis Streams | 10/10 |
| T8~T11 | 스페이스 → 페이지 트리 → 리비전·복원 → 첨부 (도메인 REST 4연속) | 사건 ②, 28/28 |
| T12 | compose 통합(wikidb·redis AOF·라우트) + **Wave A 이월 2건 동승**(iss/aud 명시 주입, 부트스트랩 경고) | 13컨테이너 healthy, 401 실측 |
| T13 | GitHub push + CI | 사건 ④ 우회 후 첫 시도 green |
| T14 | 스모크·README(사실관계 소스 대조 리뷰 통과) | Approved |
| 최종 | fable 전체 리뷰 → fix W-1·W-2 → **재검증에서 W-1 회귀 재발견(사건 ③)** → 1단어 fix → Ready | 29/29 |

### 사건 일지 — 무엇이 터졌고 무엇을 배웠나

**사건 ① `key`라는 이름의 함정 (T5)** — 스페이스의 URL용 식별자 컬럼을 `key`라고 지었는데 테스트가 전부 실패했다. 원인은 두 겹: (1) `KEY`는 H2 데이터베이스의 예약어다(운영 Postgres에선 문제없음). URL에 `NON_KEYWORDS=KEY` 옵션을 넣어 대비해 뒀는데, (2) **`@DataJpaTest`는 기본으로 설정된 DB를 무시하고 자기 임베디드 H2로 갈아탄다** — 그래서 옵션이 무시돼 테이블 생성이 조용히 실패했다. `spring.test.database.replace: none` 한 줄로 "내가 설정한 URL을 그대로 써라"고 지시해 해결. *교훈: 테스트 슬라이스의 "친절한 기본값"이 내 설정을 덮어쓸 수 있다. 덤으로 구현자가 red 상태로 커밋하는 계약 위반을 해서, 이후 모든 브리프에 "green 전 커밋 금지"가 명시됐다.*

**사건 ② H2 테스트 세계에는 FK가 아예 없다 (T9→T11→최종까지 연쇄)** — 페이지 삭제 시 하위 페이지·리비전은 운영에서 Postgres의 `ON DELETE CASCADE`(외래키 연쇄 삭제)가 지워준다. 그런데 우리 엔티티는 JPA 연관관계 대신 평범한 `Long` 컬럼을 쓰므로, Hibernate가 테스트 스키마를 만들 때 **FK 제약을 아예 생성하지 않는다** — 테스트와 운영의 동작이 갈라진다(테스트에선 고아 행이 남음). 구현자는 하위 페이지에 대해선 스스로 발견해 명시 재귀 삭제로 고쳤지만 리비전은 빠뜨렸고 리뷰가 잡았다. 이 교훈은 T11(첨부 정리 추가 요구)로, 다시 최종 리뷰의 W-1(스페이스 삭제 경로)로 연쇄됐다. *교훈: "DB가 알아서 해준다"는 가정은 테스트 환경에서도 성립하는지 확인할 것. cascade는 코드로 명시해야 두 환경 모두에서 참이 된다.*

**사건 ③ 고친 코드가 새 폭탄이 된 사례 (최종 fix W-1 → 재검증)** — 최종 리뷰가 "스페이스 삭제는 첨부 디스크 파일을 정리 안 한다(영구 누수)"를 잡아서(스페이스는 T3, 첨부는 T11 담당 — **태스크 경계의 사각지대**, 전체 리뷰의 존재 이유), 전체 페이지 순회 정리로 고쳤다. 그런데 재검증에서 그 수정 안의 `pages.deleteAll(목록)`이 새 회귀로 판정됐다: 개별 DELETE를 순서대로 날리는데, 운영 Postgres에선 부모 페이지 DELETE가 자식을 cascade로 **먼저** 지워버려서 이어지는 자식 DELETE가 "0행 삭제됨" — Hibernate가 이를 이상 상태(StaleStateException)로 보고 500을 낸다. H2엔 FK가 없어 29/29 green이 이걸 절대 못 본다. 처방은 한 단어: `deleteAllInBatch`(단일 bulk DELETE — 행 수 기대가 없어 cascade와 안 충돌). *교훈: 수정도 리뷰 대상이다. 그리고 "테스트 green"은 테스트 환경이 운영과 같을 때만 의미가 있다.*

**사건 ④ GitHub 패키지 권한의 벽 (T13)** — wiki-backend CI가 platform-backend의 private 패키지(common-proto)를 읽으려면 권한이 필요한데, 정석인 "패키지 설정 → Manage Actions access" 섹션이 **UI에 아예 없었다**(패키지가 repo 상속 권한 모드면 안 뜬다). 사용자가 "Danger Zone만 보인다"고 알려와서, PAT를 repo secret(`GH_PACKAGES_TOKEN`)으로 넣고 workflow가 그걸 쓰는 방식으로 우회 — 첫 CI가 바로 green. *운영 메모: gh 재로그인으로 토큰이 바뀌면 secret도 갱신(README·CI 주석 이중 문서화).*

**작은 것들**: 자바 테스트 메서드명에 가운뎃점(·)은 식별자 불가(언더스코어로) · jsonPath는 작은 수를 Integer로 줘서 Long 비교가 실패한다((int) 캐스팅) · `TransactionTemplate`은 자동구성 빈이 아니다(직접 생성) · Windows 커밋 gradlew는 실행 비트가 없다(Wave A 교훈 — 이번엔 첫 커밋부터 적용) · 옵시디언 문서 파일명에 콜론 금지(Windows).

### 리뷰 게이트 통계

- 태스크 리뷰 14회 중 **Needs fixes 3회**(T4 우산 gitignore 누락 / T5 사건 ① / T9 사건 ② 리비전), 나머지 Approved
- 최종 전체 리뷰: Critical 0 · Important 4(W-1 파일 누수 / W-2 redis 볼륨 / W-3 실 발행 스모크→체크리스트 / W-4 이벤트 계약 갭→Wave C) · 누적 minor 14건 판정(fix 2·defer 6·dismiss 6) · **재검증에서 수정 코드의 회귀 1건 추가 발견**(사건 ③)
- 리뷰가 잡은 실질 결함 4건 — 리비전 미삭제, 스페이스 삭제 파일 누수, redis 볼륨 누락, deleteAll cascade 충돌 — **전부 테스트 green 상태에서 존재하던 버그**다. green은 필요조건이지 충분조건이 아니다.

---

## 3부. 지금 상태와 남은 것

### 검증 현황 (스펙 성공 기준 6개)

| # | 기준 | 상태 |
|---|---|---|
| 1 | proto v0.2.0 발행 + 외부 소비 | ✅ 발행 green, wiki resolve 실증 |
| 2 | CRUD·트리·버전·복원·첨부 테스트 | ✅ 29/29 |
| 3 | 권한 3레벨 + 자동 ADMIN | ✅ 테스트(페이크) / 실연동은 체크리스트 ④ |
| 4 | compose 통합 + 401 | ✅ 13컨테이너, 401 실측 |
| 5 | Redis 스트림 이벤트 축적 | ⏳ 체크리스트 ①~③ (XLEN 0 = 아직 아무도 안 씀, 정상) |
| 6 | dev·docker 양 모드 | docker ✅ / dev(IDE+Eureka)는 체크리스트 ⑤ |

### 사용자 수동 체크리스트 (브라우저 토큰 필요 — 5분)

1. 브라우저 로그인 → AT 복사 → `POST /api/wiki/spaces`(201) → `POST /api/wiki/pages`(201)
2. `docker exec platform-redis redis-cli XLEN platform:events:v1` → **2 이상** (실 발행 증명 — 최종 리뷰 W-3)
3. `docker exec platform-redis redis-cli XRANGE platform:events:v1 - + COUNT 3` → payload 확인
4. 일반 계정 AT로 남의 스페이스 PUT → **403** (권한 실연동)
5. IntelliJ `wiki-backend bootRun`(GITHUB_TOKEN env 필요) → Eureka 등록 → 게이트웨이 경유 200

### Wave C로 이월 (최종 리뷰 판정)

- **proto v0.3.0 선행**: `SpaceUpdated`·`AttachmentDeleted` 이벤트(현재 미발행 — 스테일 색인 원인, W-4) + `RevokeGrant` rpc(스페이스 삭제 시 org 고아 grant 회수 — T8부터 알려진 위생 문제)
- **순환 가드 보강**: 동시 이동 레이스로 사이클이 생기면 삭제 재귀가 무한루프 — visited-set 2줄(Wave C 최우선)
- Testcontainers-PG로 Flyway↔엔티티 정합 테스트(사건 ②·③류를 테스트가 잡게) · PermissionClient 실패분기 테스트 · listAccessible DB 필터 전환(규모 커지면)
- 운영 메모: 기존 볼륨 환경은 wikidb 수동 생성 필요(init 스크립트는 새 볼륨 전용) · GH_PACKAGES_TOKEN 갱신 절차

### 커밋 지도

wiki-backend(빈트리→HEAD, 15커밋): 골격 b559050 → 스키마 5fe3a53+3c86124(fix①) → 보안/권한 0c94762 → 이벤트 501022b → 스페이스 3c9b9af → 페이지 908da5a+08db3c3(fix②) → 리비전 70955aa → 첨부 80cd74d → Docker 0b8a4f3 → CI 0a03e97 → README 25c0d6d → W-1 88e03f8 → deleteAllInBatch fix(사건 ③) — 전부 push
platform-backend: 4ec98e6(proto)+fd7f43b(CreateGrant), v0.2.0 태그 · gateway: 6e01414(라우트, push) · 우산(미push 3커밋): f123b7f(compose)+26965cf(README)+1997f7a(redis 볼륨) — 병행 세션(관측 스택) 종료 후 함께 push 권장
