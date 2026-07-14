# nginx 통합배포 + 3앱 SSO — 수동 E2E 체크리스트 (2026-07-14 웨이브)

사전: 프론트 3개 빌드 + compose up(nginx 포함) + 백엔드 4개 기동(auth-server는 FRONTEND_URL=http://localhost)
+ realm 재import(localhost 오리진 반영, ⚠ down -v)

- [ ] http://localhost → myFront 랜딩 렌더 (콘솔 에러 0)
- [ ] http://localhost/wiki → (미로그인) Keycloak 로그인 화면으로 리다이렉트
- [ ] alice/alice 로그인 → **/wiki로 복귀** (returnTo 검증 — /app이 아니라)
- [ ] wiki 헤더에 "Alice Kim" + 로그아웃 버튼 표시
- [ ] http://localhost/alm → **무프롬프트 즉시 진입** (SSO — RT 쿠키 공유)
- [ ] http://localhost/app → myFront도 로그인 상태
- [ ] wiki 딥링크 새로고침(예: /wiki/spaces/.../pages/...) → 404 없이 렌더 (SPA 폴백)
- [ ] /wiki에서 로그아웃 → 포털(/)로 이동, 이후 /wiki 재진입 시 Keycloak **로그인 폼**(백채널로 KC 세션도 끊김)
- [ ] (선택) 구글 로그인 왕복 — myFront /login의 Google 버튼
- [ ] (검증) 로그인 콜백의 redirect_uri가 http://localhost/login/oauth2/code/keycloak 인지 브라우저 네트워크 탭에서 확인 — nginx→gateway 이중 홉의 X-Forwarded 병합이 올바른지 (gateway가 자체 X-Forwarded-Port를 append하면 포트가 틀어질 수 있음)
- [ ] (주의) 위 redirect_uri가 :8000으로 뜨더라도 로그인은 성공한다(게이트웨이가 X-Forwarded-Port를 append하는 경우) — realm의 http://localhost:8000 redirectUris 항목은 이 폴백을 위해 유지 필수. "realm 정리" 시 삭제 금지.
- [ ] (알려진 리스크 재현 확인용, 선택) wiki·alm 두 탭 동시 새로고침 → 간헐 전체 로그아웃 가능(RT 재사용탐지 오인) — 발생 빈도 기록만
