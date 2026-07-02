---
tags: [msa, template, board-service, sso, jwt, jwks, tdd]
작성일: 2026-06-30
상태: 구현 완료 + 라이브 SSO 증명 완료
---

# 3단계 — board-service (샘플 백엔드 + SSO 증명)

상위: [[00 개요 — 전체 구조]]

## 정의

MSA의 핵심 증명용 **샘플 백엔드(리소스 서버)**. auth-server가 발급한 JWT를 **공유 비밀 없이 JWKS만으로** 검증해, 발급자 1개(:9000)를 여러 서비스가 신뢰하는 SSO 모델을 입증한다. 별도 repo `backend-server.git`.

## 스택 / 구조

- Spring Boot 4.0.6, JDK24, group `com.platform`, 패키지 `com.platform.boardservice`.
- feature 폴더: `post` / `comment` / `security`(AccessGuard) / `config`(SecurityConfig) / `common`.
- DB: postgres `boarddb`(:5433), Flyway V1(post)·V2(comment). 글 삭제 시 댓글 ON DELETE CASCADE.
- **TDD 9 태스크**(스캐폴드→Post→AccessGuard→DTO/서비스→수정삭제→Controller→Comment…), 테스트 30개 green.

## JWT 검증 설정 (연결 지점)

```yaml
spring.security.oauth2.resourceserver.jwt.jwk-set-uri:
  http://localhost:9000/.well-known/jwks.json   # auth-server JWKS
```
- `roles` claim → `ROLE_*` (auth-server와 동일 컨버터).
- 인가: `GET /api/posts/**` public, 그 외 인증 필요. 수정/삭제는 **본인 또는 ADMIN**(`AccessGuard`).

## API

- `GET /api/posts?page=&size=` → Spring Data `Page`(createdAt DESC)
- `GET /api/posts/{id}` / `POST /api/posts`(201) / `PUT /api/posts/{id}` / `DELETE /api/posts/{id}`(204)
- DTO: `PostResponse{id, title, content, authorId, authorName, createdAt, updatedAt}` — **작성자는 폼 입력이 아니라 토큰(sub/name)에서 서버가 박음**.
- Comment CRUD도 구현돼 있음(프론트 UI는 아직 미연결).

## 라이브 SSO 증명 (8/8)

curl로 alice/admin 토큰을 받아(`get_token.sh`, kcContext의 loginAction 추출 → OIDC 코드플로우 → `/api/auth/refresh`) board-service에 호출:

| 시나리오 | 기대 | 결과 |
|---|---|---|
| GET /posts (토큰X) | 200 public | ✅ |
| POST (토큰X) | 401 | ✅ |
| POST (위조 토큰) | 401 서명거부 | ✅ |
| POST (alice 토큰) | 201, authorId=토큰 sub | ✅ |
| GET /posts/{id} | 200 영속 | ✅ |
| 본인 글 삭제 | 204 | ✅ |
| 타인 글 삭제 | 403 | ✅ |
| ADMIN이 타인 글 삭제 | 204 (roles 클레임 교차서비스) | ✅ |

→ board-service는 auth-server 소스에 **코드 의존 0**, 실행 시점에 JWKS로만 신뢰. 이게 SSO/페더레이션 증명.

## 함정

- curl `-d`로 한글 인라인하면 셸이 깨뜨려 400 → `--data-binary @파일`로 보낼 것.

다음: [[04 프론트 연동 (myFront + board-service)]]
