# nginx 통합 프론트 배포 + 3앱 SSO Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** nginx 하나(`http://localhost`)에서 myFront·wiki-front·alm-front를 경로 기반으로 서빙하고, 기존 Keycloak+auth-server BFF에 물려 한 번 로그인이면 3앱 전부 인증되는 SSO를 완성한다.

**Architecture:** nginx(docker)가 맨 앞에서 정적 3앱을 직접 서빙하고 `/api·/oauth2·/login·/.well-known`만 gateway(:8000)로 프록시한다. RT 쿠키가 host-only+`path=/api/auth`라 단일 오리진이면 자동 공유 — CookieFactory 무수정. 로그인 후 복귀는 `post_login_redirect` 쿠키(returnTo 패턴)를 auth-server가 검증 후 소비한다. wiki/alm은 myFront의 auth 모듈을 복사해 로그인 게이트를 단다.

**Tech Stack:** nginx:alpine(docker compose), Spring Boot 4(auth-server, JDK24), React 19 + vite(myFront=npm/MUI, wiki·alm=pnpm/@chanho 디자인시스템, react-router 7)

**Spec:** `C:\myBrain\내 로컬\msa\MSA_TEMPLATE 설계문서\specs\2026-07-14-nginx-sso-unified-frontend-design.md`

## Global Constraints

- repo 5개 교차: 각 태스크는 **자기 repo 루트에서** 커밋한다 — auth-server(`C:\MSA_TEMPLATE\auth-server`, main) / myFront(`C:\MSA_TEMPLATE\myFront`) / wiki-front(`C:\MSA_TEMPLATE\wiki-front`, main) / alm-front(`C:\MSA_TEMPLATE\alm-front`, main) / 우산 MSA_TEMPLATE(`C:\MSA_TEMPLATE`, master — infra만)
- 커밋은 로컬만. **push는 사용자 요청 시에만**
- wiki/alm에 MUI·기타 UI 라이브러리 추가 금지 — @chanho/react만 (기존 계약)
- wiki/alm 패키지 매니저는 **pnpm**, myFront는 **npm**
- auth-server 빌드/테스트: `./gradlew test` (JDK24, 기존 14+ green 유지)
- wiki-front 기존 77 tests·alm-front 61 tests green 유지 (게이트: `pnpm typecheck && pnpm test && pnpm build`)
- 작업기록/문서는 Obsidian에만 — 우산 repo `docs/`에 새 문서 금지
- 쿠키 이름 계약(전 repo 공통): **`post_login_redirect`** / 검증 규칙: `/`로 시작 AND `//`로 시작 금지 / 폴백 `/app`
- realm 재import(`docker compose down -v`)는 DB 볼륨 파괴 — **자동 실행 금지**, 수동 단계로만 문서화

---

### Task 1: auth-server — LoginSuccessHandler returnTo 쿠키 (TDD)

**Files:**
- Test: `C:\MSA_TEMPLATE\auth-server\src\test\java\com\platform\authserver\auth\LoginSuccessHandlerTest.java` (신규)
- Modify: `C:\MSA_TEMPLATE\auth-server\src\main\java\com\platform\authserver\auth\LoginSuccessHandler.java`

**Interfaces:**
- Consumes: `RefreshTokenService.Issued(String rawToken, String kcIdToken)` record, `CookieFactory(long ttlSeconds, boolean secure)` 생성자, `UserService.provision(sub,email,name,roles,provider)`, `OidcClaims.roles/provider`
- Produces: 로그인 성공 시 `post_login_redirect` 쿠키(있으면)를 소비·삭제하고 `frontendUrl + <검증된 경로 | "/app">`로 302. 프론트 3앱(Task 2~4)이 이 쿠키 이름·의미에 의존한다.

- [ ] **Step 1: 실패하는 테스트 작성**

`LoginSuccessHandlerTest.java` 전체 내용:

```java
package com.platform.authserver.auth;

import com.platform.authserver.token.CookieFactory;
import com.platform.authserver.token.RefreshTokenService;
import com.platform.authserver.user.User;
import com.platform.authserver.user.UserService;
import jakarta.servlet.http.Cookie;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;
import org.springframework.security.oauth2.client.OAuth2AuthorizedClientService;
import org.springframework.security.oauth2.client.authentication.OAuth2AuthenticationToken;
import org.springframework.security.oauth2.core.oidc.OidcIdToken;
import org.springframework.security.oauth2.core.oidc.user.DefaultOidcUser;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

/**
 * returnTo(post_login_redirect) 쿠키 계약:
 * - 유효(상대경로)면 frontendUrl+경로로 복귀 + 쿠키 삭제(일회용)
 * - 절대 URL / "//host" / 부재 → /app 폴백 (오픈 리다이렉트 방어)
 */
class LoginSuccessHandlerTest {

    UserService userService = mock(UserService.class);
    RefreshTokenService refreshTokenService = mock(RefreshTokenService.class);
    OAuth2AuthorizedClientService authorizedClientService = mock(OAuth2AuthorizedClientService.class);
    LoginSuccessHandler handler;

    @BeforeEach
    void setUp() {
        when(userService.provision(any(), any(), any(), any(), any())).thenReturn(new User("kc-1"));
        when(refreshTokenService.issue(any(), any(), any()))
                .thenReturn(new RefreshTokenService.Issued("raw-rt", "kc-id-token"));
        handler = new LoginSuccessHandler(userService, refreshTokenService,
                new CookieFactory(60, false), authorizedClientService, "http://localhost");
    }

    private OAuth2AuthenticationToken authentication() {
        OidcIdToken idToken = new OidcIdToken(
                "tok", Instant.now(), Instant.now().plusSeconds(60),
                Map.of("sub", "kc-1", "email", "a@b.com", "name", "Alice"));
        return new OAuth2AuthenticationToken(new DefaultOidcUser(List.of(), idToken), List.of(), "keycloak");
    }

    private MockHttpServletResponse loginWith(Cookie... cookies) throws Exception {
        MockHttpServletRequest request = new MockHttpServletRequest();
        if (cookies.length > 0) request.setCookies(cookies);
        MockHttpServletResponse response = new MockHttpServletResponse();
        handler.onAuthenticationSuccess(request, response, authentication());
        return response;
    }

    @Test
    void redirectsToReturnToPathAndDeletesCookie() throws Exception {
        MockHttpServletResponse response = loginWith(new Cookie("post_login_redirect", "/wiki/pages/3"));

        assertThat(response.getRedirectedUrl()).isEqualTo("http://localhost/wiki/pages/3");
        assertThat(response.getHeaders("Set-Cookie"))
                .anyMatch(h -> h.startsWith("post_login_redirect=;") && h.contains("Max-Age=0"));
    }

    @Test
    void fallsBackToAppWhenCookieAbsent() throws Exception {
        MockHttpServletResponse response = loginWith();

        assertThat(response.getRedirectedUrl()).isEqualTo("http://localhost/app");
    }

    @Test
    void rejectsAbsoluteUrl() throws Exception {
        MockHttpServletResponse response = loginWith(new Cookie("post_login_redirect", "https://evil.com/phish"));

        assertThat(response.getRedirectedUrl()).isEqualTo("http://localhost/app");
        // 무효 값이라도 쿠키는 소비(삭제)한다
        assertThat(response.getHeaders("Set-Cookie"))
                .anyMatch(h -> h.startsWith("post_login_redirect=;") && h.contains("Max-Age=0"));
    }

    @Test
    void rejectsSchemeRelativeUrl() throws Exception {
        MockHttpServletResponse response = loginWith(new Cookie("post_login_redirect", "//evil.com/phish"));

        assertThat(response.getRedirectedUrl()).isEqualTo("http://localhost/app");
    }
}
```

- [ ] **Step 2: 실패 확인**

Run (PowerShell, cwd=`C:\MSA_TEMPLATE\auth-server`):
```powershell
.\gradlew test --tests "com.platform.authserver.auth.LoginSuccessHandlerTest"
```
Expected: FAIL — `redirectsToReturnToPathAndDeletesCookie`가 `http://localhost/app` (현재 하드코딩) 리턴이라 assertion 실패. (`fallsBackToAppWhenCookieAbsent`는 이미 통과할 수 있음 — 정상)

- [ ] **Step 3: 구현**

`LoginSuccessHandler.java` 수정. import에 `jakarta.servlet.http.Cookie`, `org.springframework.http.ResponseCookie` 추가. 클래스 상단에 상수 추가:

```java
    /** 프론트가 로그인 직전에 심는 복귀 경로 쿠키. 일회용 — 여기서 소비·삭제한다. */
    static final String RETURN_TO_COOKIE = "post_login_redirect";
```

`onAuthenticationSuccess`의 마지막 줄(`response.sendRedirect(frontendUrl + "/app");`)을 다음으로 교체:

```java
        String rawReturnTo = readCookie(request, RETURN_TO_COOKIE);
        if (rawReturnTo != null) {
            response.addHeader(HttpHeaders.SET_COOKIE, deleteReturnToCookie().toString());
        }
        String target = isSafeRelativePath(rawReturnTo) ? rawReturnTo : "/app";
        response.sendRedirect(frontendUrl + target);
```

클래스 하단(extractKcRefreshToken 아래)에 private 헬퍼 3개 추가:

```java
    private static String readCookie(HttpServletRequest request, String name) {
        if (request.getCookies() == null) {
            return null;
        }
        for (Cookie cookie : request.getCookies()) {
            if (name.equals(cookie.getName())) {
                return cookie.getValue();
            }
        }
        return null;
    }

    /** 오픈 리다이렉트 방어: 우리 오리진 안의 상대 경로("/...")만 허용, "//host" 형태 금지. */
    private static boolean isSafeRelativePath(String path) {
        return path != null && path.startsWith("/") && !path.startsWith("//");
    }

    private static ResponseCookie deleteReturnToCookie() {
        return ResponseCookie.from(RETURN_TO_COOKIE, "").path("/").maxAge(0).build();
    }
```

클래스 주석(1)~4) 목록)의 `4) 프론트 /app 로 리다이렉트` 를 `4) returnTo 쿠키(검증 통과 시) 또는 /app 로 리다이렉트` 로 갱신.

- [ ] **Step 4: 통과 확인 + 전체 회귀**

```powershell
.\gradlew test
```
Expected: 신규 4개 포함 전체 green (기존 14+ 유지)

- [ ] **Step 5: 커밋**

```powershell
git add src/test/java/com/platform/authserver/auth/LoginSuccessHandlerTest.java src/main/java/com/platform/authserver/auth/LoginSuccessHandler.java
git commit -m "feat(auth): 로그인 후 복귀 경로 returnTo 쿠키(post_login_redirect) 지원 — 상대경로만 허용, 폴백 /app"
```

---

### Task 2: myFront — returnTo 쿠키 심기 + same-origin 프로덕션 빌드

**Files:**
- Create: `C:\MSA_TEMPLATE\myFront\src\auth\returnTo.ts`
- Create: `C:\MSA_TEMPLATE\myFront\.env.production`
- Modify: `C:\MSA_TEMPLATE\myFront\src\auth\index.ts`
- Modify: `C:\MSA_TEMPLATE\myFront\src\app\pages\LoginPage.tsx`

**Interfaces:**
- Consumes: Task 1의 쿠키 계약(`post_login_redirect`, path=/), 기존 `loginUrl()/googleLoginUrl()` 배럴 함수, `ProtectedRoute`가 넘기는 `location.state.from`
- Produces: `rememberReturnTo(path: string): void` — Task 3·4의 wiki/alm 복사본과 동일 시그니처

**참고:** myFront에는 테스트 러너가 없다(스크립트: dev/build/preview뿐). 게이트는 `npm run build`(tsc 포함) green — 기존 웨이브와 동일 기준.

- [ ] **Step 1: returnTo.ts 작성**

```ts
// 로그인 리다이렉트 왕복(SPA → auth-server → Keycloak → auth-server) 동안
// "돌아올 경로"를 나르는 일회용 쿠키. auth-server LoginSuccessHandler 가 읽고
// 검증(상대경로만) 후 삭제한다. SAML RelayState / NextAuth callbackUrl 과 같은 패턴.
// 값은 인코딩하지 않는다 — 서버가 raw 값을 그대로 검증하며, 라우트 경로에는
// 쿠키 금지 문자(세미콜론·쉼표·공백)가 없다.
export const RETURN_TO_COOKIE = 'post_login_redirect';

export function rememberReturnTo(path: string): void {
  document.cookie = `${RETURN_TO_COOKIE}=${path}; path=/; max-age=300; SameSite=Lax`;
}
```

- [ ] **Step 2: 배럴에 노출**

`src/auth/index.ts` 끝에 추가:

```ts
export { rememberReturnTo, RETURN_TO_COOKIE } from './returnTo';
```

- [ ] **Step 3: LoginPage에서 리다이렉트 직전에 쿠키 심기**

`LoginPage.tsx` 수정 — import 추가:

```tsx
import { useLocation } from 'react-router-dom';
import { loginUrl, googleLoginUrl, rememberReturnTo } from '../../auth';
```
(기존 `import { loginUrl, googleLoginUrl } from '../../auth';` 줄을 교체)

`export default function LoginPage(...)` 본문 첫 줄에 추가:

```tsx
  const location = useLocation();
  // ProtectedRoute 가 넘긴 원래 목적지. 직접 /login 에 온 경우는 /app.
  const from = (location.state as { from?: string } | null)?.from ?? '/app';
```

두 버튼의 onClick 을 각각 교체:

```tsx
              onClick={() => {
                rememberReturnTo(from);
                window.location.href = loginUrl();
              }}
```

```tsx
              onClick={() => {
                rememberReturnTo(from);
                window.location.href = googleLoginUrl();
              }}
```

- [ ] **Step 4: .env.production 생성**

내용 (nginx 뒤에서는 same-origin — 상대경로 fetch):

```
VITE_API_BASE=
```

`.env`(dev, `http://localhost:8000`)는 그대로 둔다. `createAuthClient`는 `''`를 받으면 `fetch('/api/...')` 상대 호출이 되고 `loginUrl()`은 `/oauth2/authorization/keycloak`이 된다 — nginx가 프록시.

- [ ] **Step 5: 빌드 게이트 + 커밋**

```powershell
npm run build
```
Expected: tsc + vite build green, `dist/` 갱신.

```powershell
git add src/auth/returnTo.ts src/auth/index.ts src/app/pages/LoginPage.tsx .env.production
git commit -m "feat(auth): 로그인 직전 returnTo 쿠키 설정 + nginx same-origin 프로덕션 env"
```

---

### Task 3: wiki-front — /wiki base + auth 모듈 + 로그인 게이트 (TDD)

**Files:**
- Modify: `C:\MSA_TEMPLATE\wiki-front\vite.config.ts`
- Modify: `C:\MSA_TEMPLATE\wiki-front\src\app\main.tsx`
- Create: `C:\MSA_TEMPLATE\wiki-front\src\auth\types.ts` (myFront 복사)
- Create: `C:\MSA_TEMPLATE\wiki-front\src\auth\client.ts` (myFront 복사)
- Create: `C:\MSA_TEMPLATE\wiki-front\src\auth\returnTo.ts` (Task 2와 동일 내용)
- Create: `C:\MSA_TEMPLATE\wiki-front\src\auth\AuthGate.tsx`
- Test: `C:\MSA_TEMPLATE\wiki-front\src\auth\AuthGate.test.tsx`
- Modify: `C:\MSA_TEMPLATE\wiki-front\src\features\wiki\components\WikiLayout.tsx`

**Interfaces:**
- Consumes: myFront `src/auth/{types,client}.ts` (그대로 복사 — in-flight refresh dedup 포함), Task 1 쿠키 계약
- Produces: `AuthGate({ children, client?, redirect?, enabled? })` 컴포넌트, `useAuth(): { user: AppUser | null; logout: () => Promise<void> }` — **Provider 밖에서는 throw 하지 않고 `{ user: null, logout: noop }` 반환** (기존 77개 테스트가 App을 게이트 없이 렌더하기 때문. myFront의 throw 계약과 의도적으로 다름)

**게이트 동작 계약:**
- `enabled`(기본 `import.meta.env.PROD`)가 false → 즉시 children 렌더, user=null. **vite dev/vitest에서는 게이트 비활성** — 인증 E2E는 nginx 프로덕션 경로로만 검증(YAGNI: dev용 프록시 배선은 스코프 밖)
- enabled=true → 마운트 시 `tryRefresh()`: 성공 → `fetchMe()` 후 children 렌더 / 실패 → `rememberReturnTo(현재경로)` 후 `redirect(loginUrl())` (기본 redirect는 `window.location.assign`)
- 현재경로 = `window.location.pathname + window.location.search` — basename(/wiki) 포함 full path라서 auth-server 복귀값으로 그대로 사용 가능
- logout = `client.logout()` 후 `redirect('/')` (포털로 이동 — 백채널이 KC 세션까지 끊으므로 재진입 시 로그인 폼)

- [ ] **Step 1: myFront auth 파일 2개 복사**

```powershell
New-Item -ItemType Directory -Force C:\MSA_TEMPLATE\wiki-front\src\auth
Copy-Item C:\MSA_TEMPLATE\myFront\src\auth\types.ts C:\MSA_TEMPLATE\wiki-front\src\auth\types.ts
Copy-Item C:\MSA_TEMPLATE\myFront\src\auth\client.ts C:\MSA_TEMPLATE\wiki-front\src\auth\client.ts
```

`returnTo.ts`는 Task 2 Step 1과 **동일한 내용**으로 생성 (다시 쓴다):

```ts
// 로그인 리다이렉트 왕복(SPA → auth-server → Keycloak → auth-server) 동안
// "돌아올 경로"를 나르는 일회용 쿠키. auth-server LoginSuccessHandler 가 읽고
// 검증(상대경로만) 후 삭제한다. SAML RelayState / NextAuth callbackUrl 과 같은 패턴.
// 값은 인코딩하지 않는다 — 서버가 raw 값을 그대로 검증하며, 라우트 경로에는
// 쿠키 금지 문자(세미콜론·쉼표·공백)가 없다.
export const RETURN_TO_COOKIE = "post_login_redirect";

export function rememberReturnTo(path: string): void {
  document.cookie = `${RETURN_TO_COOKIE}=${path}; path=/; max-age=300; SameSite=Lax`;
}
```

- [ ] **Step 2: AuthGate 실패 테스트 작성**

`src/auth/AuthGate.test.tsx` 전체 내용:

```tsx
import { render, screen, waitFor } from "@testing-library/react";
import { describe, expect, it, vi } from "vitest";
import type { AppUser, AuthClient } from "./types";
import { AuthGate, useAuth } from "./AuthGate";

const ALICE: AppUser = { email: "alice@demo.com", name: "Alice Kim" };

/** 테스트용 최소 AuthClient — 각 케이스가 tryRefresh/fetchMe만 바꿔 끼운다 */
function stubClient(overrides: Partial<AuthClient>): AuthClient {
  return {
    getAccessToken: () => null,
    setAccessToken: () => {},
    loginUrl: () => "/oauth2/authorization/keycloak",
    googleLoginUrl: () => "/oauth2/authorization/keycloak?kc_idp_hint=google",
    apiFetch: async () => new Response(null),
    tryRefresh: async () => false,
    fetchMe: async () => ALICE,
    logout: async () => {},
    ...overrides,
  };
}

function UserProbe() {
  const { user } = useAuth();
  return <div data-testid="auth-user">{user ? user.name : "(none)"}</div>;
}

describe("AuthGate", () => {
  it("미인증이면 returnTo 쿠키를 심고 로그인 URL로 리다이렉트한다", async () => {
    const redirect = vi.fn();
    render(
      <AuthGate enabled client={stubClient({ tryRefresh: async () => false })} redirect={redirect}>
        <div>비밀 콘텐츠</div>
      </AuthGate>,
    );

    await waitFor(() =>
      expect(redirect).toHaveBeenCalledWith("/oauth2/authorization/keycloak"),
    );
    expect(document.cookie).toContain("post_login_redirect=");
    expect(screen.queryByText("비밀 콘텐츠")).not.toBeInTheDocument();
  });

  it("인증되면 children을 렌더하고 useAuth로 사용자를 제공한다", async () => {
    render(
      <AuthGate enabled client={stubClient({ tryRefresh: async () => true })}>
        <UserProbe />
      </AuthGate>,
    );

    expect(await screen.findByText("Alice Kim")).toBeInTheDocument();
  });

  it("인증 성공 후 refresh를 반복 호출하지 않는다 (effect 재실행 루프 회귀 가드)", async () => {
    const tryRefresh = vi.fn(async () => true);
    render(
      <AuthGate enabled client={stubClient({ tryRefresh })}>
        <UserProbe />
      </AuthGate>,
    );

    expect(await screen.findByText("Alice Kim")).toBeInTheDocument();
    await new Promise((resolve) => setTimeout(resolve, 0)); // 후속 effect 재실행 여지를 흘려보냄
    expect(tryRefresh).toHaveBeenCalledTimes(1);
  });

  it("enabled=false(dev/test)면 인증 없이 즉시 children을 렌더한다", () => {
    const redirect = vi.fn();
    render(
      <AuthGate enabled={false} client={stubClient({})} redirect={redirect}>
        <UserProbe />
      </AuthGate>,
    );

    expect(screen.getByTestId("auth-user")).toHaveTextContent("(none)");
    expect(redirect).not.toHaveBeenCalled();
  });

  it("Provider 밖에서 useAuth는 throw 대신 user=null을 반환한다", () => {
    render(<UserProbe />);
    expect(screen.getByTestId("auth-user")).toHaveTextContent("(none)");
  });
});
```

- [ ] **Step 3: 실패 확인**

```powershell
pnpm vitest run src/auth/AuthGate.test.tsx
```
Expected: FAIL — `AuthGate.tsx` 모듈 없음

- [ ] **Step 4: AuthGate 구현**

`src/auth/AuthGate.tsx` 전체 내용:

```tsx
import { createContext, useCallback, useContext, useEffect, useMemo, useState } from "react";
import { Spinner } from "@chanho/react";
import type { AppUser, AuthClient } from "./types";
import { createAuthClient } from "./client";
import { rememberReturnTo } from "./returnTo";

// 프로덕션(nginx same-origin) 기본 클라이언트. dev/test 에서는 게이트가 꺼져 있어 안 쓰인다.
const defaultClient = createAuthClient({ baseUrl: (import.meta.env.VITE_API_BASE as string) ?? "" });

interface AuthContextValue {
  user: AppUser | null;
  logout: () => Promise<void>;
}

// 기본값 제공: 게이트 밖(기존 테스트가 App을 직접 렌더)에서도 useAuth가 throw하지 않는다.
// myFront(Provider 필수·throw)와 의도적으로 다른 계약 — 이 앱은 게이트가 옵션이기 때문.
const AuthContext = createContext<AuthContextValue>({ user: null, logout: async () => {} });

export function useAuth(): AuthContextValue {
  return useContext(AuthContext);
}

export interface AuthGateProps {
  children: React.ReactNode;
  client?: AuthClient;
  /** 전체 페이지 이동. 테스트에서 스파이로 주입한다. */
  redirect?: (url: string) => void;
  /** dev/vitest 에서는 게이트 비활성 — 인증 검증은 nginx 프로덕션 경로로만. */
  enabled?: boolean;
}

/**
 * 로그인 게이트: 마운트 시 RT 쿠키로 silent refresh 를 시도하고,
 * 실패하면 returnTo 쿠키를 심은 뒤 Keycloak 로그인으로 보낸다(SSO 세션이
 * 살아 있으면 무프롬프트 왕복). 성공하면 /api/me 사용자와 함께 children 렌더.
 */
// 모듈 레벨 상수 — 렌더마다 새 함수가 되면 아래 effect 의존성이 계속 바뀌어
// 로그인 성공 후 refresh/me 무한 재호출 루프가 된다 (Task 3 리뷰에서 실증).
const defaultRedirect = (url: string) => window.location.assign(url);

export function AuthGate({
  children,
  client = defaultClient,
  redirect = defaultRedirect,
  enabled = import.meta.env.PROD,
}: AuthGateProps) {
  const [user, setUser] = useState<AppUser | null>(null);
  const [status, setStatus] = useState<"checking" | "authed">(enabled ? "checking" : "authed");

  useEffect(() => {
    if (!enabled) return;
    let active = true;
    void (async () => {
      if (await client.tryRefresh()) {
        const me = await client.fetchMe();
        if (!active) return;
        setUser(me);
        setStatus("authed");
      } else {
        if (!active) return;
        rememberReturnTo(window.location.pathname + window.location.search);
        redirect(client.loginUrl());
      }
    })();
    return () => {
      active = false;
    };
  }, [enabled, client, redirect]);

  const logout = useCallback(async () => {
    await client.logout();
    redirect("/"); // 포털로 — 백채널이 KC 세션까지 끊어 재진입 시 로그인 폼
  }, [client, redirect]);

  const value = useMemo<AuthContextValue>(() => ({ user, logout }), [user, logout]);

  if (status === "checking") {
    return (
      <div className="app-loading">
        <Spinner size="large" label="로그인 확인 중" />
      </div>
    );
  }
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}
```

- [ ] **Step 5: 게이트 테스트 통과 확인**

```powershell
pnpm vitest run src/auth/AuthGate.test.tsx
```
Expected: 5 PASS

- [ ] **Step 6: base/basename 배선**

`vite.config.ts` 전체를 다음으로 교체:

```ts
import react from "@vitejs/plugin-react";
import { defineConfig } from "vite";

export default defineConfig({
  // nginx 경로 기반 통합 배포: http://localhost/wiki/ 아래에서 서빙된다
  base: "/wiki/",
  plugins: [react()],
});
```

`src/app/main.tsx`에서 import에 `AuthGate` 추가하고 렌더 트리를 교체:

```tsx
import { AuthGate } from "../auth/AuthGate";
```

```tsx
createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <ToastProvider>
      <AuthGate>
        <BrowserRouter basename="/wiki">
          <App />
        </BrowserRouter>
      </AuthGate>
    </ToastProvider>
  </StrictMode>,
);
```

(AuthGate는 라우터 밖 — returnTo에 window.location full path를 쓰므로 라우터 불필요. 기존 테스트는 MemoryRouter로 App만 감싸므로 영향 없음)

- [ ] **Step 7: WikiLayout 헤더에 사용자 표시 + 로그아웃**

`WikiLayout.tsx` import에 추가:

```tsx
import { useAuth } from "../../../auth/AuthGate";
```

컴포넌트 본문 상단(기존 `const { theme, toggle } = useTheme();` 다음)에 추가:

```tsx
  const { user: authUser, logout } = useAuth();
```

TopBar `actions`를 다음으로 교체 (기존 Switch·Avatar 유지, 인증 사용자일 때만 이름+로그아웃 추가 — 게이트 꺼진 테스트에서는 authUser=null이라 기존 77개 테스트에 영향 없음):

```tsx
        actions={
          <>
            <Switch
              label="다크 모드"
              checked={theme === "dark"}
              onCheckedChange={toggle}
            />
            {authUser ? (
              <>
                <span className="wiki-auth-user">{authUser.name ?? authUser.email}</span>
                <Button size="small" variant="ghost" onClick={() => void logout()}>
                  로그아웃
                </Button>
              </>
            ) : null}
            {me ? <Avatar name={me.name} size="small" /> : null}
          </>
        }
```

- [ ] **Step 8: 전체 게이트 + 커밋**

```powershell
pnpm typecheck; if ($?) { pnpm test }; if ($?) { pnpm build }
```
Expected: typecheck green, 77+5=82 tests green, build green (dist/의 asset 경로가 `/wiki/assets/...`인지 `dist/index.html`에서 확인)

```powershell
git add vite.config.ts src/app/main.tsx src/auth src/features/wiki/components/WikiLayout.tsx
git commit -m "feat(auth): /wiki base + 로그인 게이트(AuthGate) + 헤더 사용자/로그아웃 — nginx SSO 편입"
```

---

### Task 4: alm-front — /alm base + auth 모듈 + 로그인 게이트 (TDD)

Task 3과 같은 구조지만 **alm repo에서 독립 수행** (구현자는 이 태스크만 보고 작업 가능해야 함 — 코드 전부 반복).

**Files:**
- Modify: `C:\MSA_TEMPLATE\alm-front\vite.config.ts`
- Modify: `C:\MSA_TEMPLATE\alm-front\src\app\main.tsx`
- Create: `C:\MSA_TEMPLATE\alm-front\src\auth\types.ts` (myFront 복사)
- Create: `C:\MSA_TEMPLATE\alm-front\src\auth\client.ts` (myFront 복사)
- Create: `C:\MSA_TEMPLATE\alm-front\src\auth\returnTo.ts`
- Create: `C:\MSA_TEMPLATE\alm-front\src\auth\AuthGate.tsx`
- Test: `C:\MSA_TEMPLATE\alm-front\src\auth\AuthGate.test.tsx`
- Modify: `C:\MSA_TEMPLATE\alm-front\src\features\jira\components\JiraLayout.tsx`

**Interfaces:** Task 3과 동일 계약 (`AuthGate`, `useAuth` — Provider 밖 no-throw)

- [ ] **Step 1: auth 파일 복사 + returnTo.ts 생성**

```powershell
New-Item -ItemType Directory -Force C:\MSA_TEMPLATE\alm-front\src\auth
Copy-Item C:\MSA_TEMPLATE\myFront\src\auth\types.ts C:\MSA_TEMPLATE\alm-front\src\auth\types.ts
Copy-Item C:\MSA_TEMPLATE\myFront\src\auth\client.ts C:\MSA_TEMPLATE\alm-front\src\auth\client.ts
```

`src/auth/returnTo.ts`:

```ts
// 로그인 리다이렉트 왕복(SPA → auth-server → Keycloak → auth-server) 동안
// "돌아올 경로"를 나르는 일회용 쿠키. auth-server LoginSuccessHandler 가 읽고
// 검증(상대경로만) 후 삭제한다. SAML RelayState / NextAuth callbackUrl 과 같은 패턴.
// 값은 인코딩하지 않는다 — 서버가 raw 값을 그대로 검증하며, 라우트 경로에는
// 쿠키 금지 문자(세미콜론·쉼표·공백)가 없다.
export const RETURN_TO_COOKIE = "post_login_redirect";

export function rememberReturnTo(path: string): void {
  document.cookie = `${RETURN_TO_COOKIE}=${path}; path=/; max-age=300; SameSite=Lax`;
}
```

- [ ] **Step 2: AuthGate 실패 테스트 작성**

`src/auth/AuthGate.test.tsx` — Task 3 Step 2와 **동일한 내용** (앱 독립 사본):

```tsx
import { render, screen, waitFor } from "@testing-library/react";
import { describe, expect, it, vi } from "vitest";
import type { AppUser, AuthClient } from "./types";
import { AuthGate, useAuth } from "./AuthGate";

const ALICE: AppUser = { email: "alice@demo.com", name: "Alice Kim" };

/** 테스트용 최소 AuthClient — 각 케이스가 tryRefresh/fetchMe만 바꿔 끼운다 */
function stubClient(overrides: Partial<AuthClient>): AuthClient {
  return {
    getAccessToken: () => null,
    setAccessToken: () => {},
    loginUrl: () => "/oauth2/authorization/keycloak",
    googleLoginUrl: () => "/oauth2/authorization/keycloak?kc_idp_hint=google",
    apiFetch: async () => new Response(null),
    tryRefresh: async () => false,
    fetchMe: async () => ALICE,
    logout: async () => {},
    ...overrides,
  };
}

function UserProbe() {
  const { user } = useAuth();
  return <div data-testid="auth-user">{user ? user.name : "(none)"}</div>;
}

describe("AuthGate", () => {
  it("미인증이면 returnTo 쿠키를 심고 로그인 URL로 리다이렉트한다", async () => {
    const redirect = vi.fn();
    render(
      <AuthGate enabled client={stubClient({ tryRefresh: async () => false })} redirect={redirect}>
        <div>비밀 콘텐츠</div>
      </AuthGate>,
    );

    await waitFor(() =>
      expect(redirect).toHaveBeenCalledWith("/oauth2/authorization/keycloak"),
    );
    expect(document.cookie).toContain("post_login_redirect=");
    expect(screen.queryByText("비밀 콘텐츠")).not.toBeInTheDocument();
  });

  it("인증되면 children을 렌더하고 useAuth로 사용자를 제공한다", async () => {
    render(
      <AuthGate enabled client={stubClient({ tryRefresh: async () => true })}>
        <UserProbe />
      </AuthGate>,
    );

    expect(await screen.findByText("Alice Kim")).toBeInTheDocument();
  });

  it("인증 성공 후 refresh를 반복 호출하지 않는다 (effect 재실행 루프 회귀 가드)", async () => {
    const tryRefresh = vi.fn(async () => true);
    render(
      <AuthGate enabled client={stubClient({ tryRefresh })}>
        <UserProbe />
      </AuthGate>,
    );

    expect(await screen.findByText("Alice Kim")).toBeInTheDocument();
    await new Promise((resolve) => setTimeout(resolve, 0)); // 후속 effect 재실행 여지를 흘려보냄
    expect(tryRefresh).toHaveBeenCalledTimes(1);
  });

  it("enabled=false(dev/test)면 인증 없이 즉시 children을 렌더한다", () => {
    const redirect = vi.fn();
    render(
      <AuthGate enabled={false} client={stubClient({})} redirect={redirect}>
        <UserProbe />
      </AuthGate>,
    );

    expect(screen.getByTestId("auth-user")).toHaveTextContent("(none)");
    expect(redirect).not.toHaveBeenCalled();
  });

  it("Provider 밖에서 useAuth는 throw 대신 user=null을 반환한다", () => {
    render(<UserProbe />);
    expect(screen.getByTestId("auth-user")).toHaveTextContent("(none)");
  });
});
```

- [ ] **Step 3: 실패 확인**

```powershell
pnpm vitest run src/auth/AuthGate.test.tsx
```
Expected: FAIL — `AuthGate.tsx` 모듈 없음

- [ ] **Step 4: AuthGate 구현**

`src/auth/AuthGate.tsx` — Task 3 Step 4와 동일한 전체 내용:

```tsx
import { createContext, useCallback, useContext, useEffect, useMemo, useState } from "react";
import { Spinner } from "@chanho/react";
import type { AppUser, AuthClient } from "./types";
import { createAuthClient } from "./client";
import { rememberReturnTo } from "./returnTo";

// 프로덕션(nginx same-origin) 기본 클라이언트. dev/test 에서는 게이트가 꺼져 있어 안 쓰인다.
const defaultClient = createAuthClient({ baseUrl: (import.meta.env.VITE_API_BASE as string) ?? "" });

interface AuthContextValue {
  user: AppUser | null;
  logout: () => Promise<void>;
}

// 기본값 제공: 게이트 밖(기존 테스트가 App을 직접 렌더)에서도 useAuth가 throw하지 않는다.
// myFront(Provider 필수·throw)와 의도적으로 다른 계약 — 이 앱은 게이트가 옵션이기 때문.
const AuthContext = createContext<AuthContextValue>({ user: null, logout: async () => {} });

export function useAuth(): AuthContextValue {
  return useContext(AuthContext);
}

export interface AuthGateProps {
  children: React.ReactNode;
  client?: AuthClient;
  /** 전체 페이지 이동. 테스트에서 스파이로 주입한다. */
  redirect?: (url: string) => void;
  /** dev/vitest 에서는 게이트 비활성 — 인증 검증은 nginx 프로덕션 경로로만. */
  enabled?: boolean;
}

/**
 * 로그인 게이트: 마운트 시 RT 쿠키로 silent refresh 를 시도하고,
 * 실패하면 returnTo 쿠키를 심은 뒤 Keycloak 로그인으로 보낸다(SSO 세션이
 * 살아 있으면 무프롬프트 왕복). 성공하면 /api/me 사용자와 함께 children 렌더.
 */
// 모듈 레벨 상수 — 렌더마다 새 함수가 되면 아래 effect 의존성이 계속 바뀌어
// 로그인 성공 후 refresh/me 무한 재호출 루프가 된다 (Task 3 리뷰에서 실증).
const defaultRedirect = (url: string) => window.location.assign(url);

export function AuthGate({
  children,
  client = defaultClient,
  redirect = defaultRedirect,
  enabled = import.meta.env.PROD,
}: AuthGateProps) {
  const [user, setUser] = useState<AppUser | null>(null);
  const [status, setStatus] = useState<"checking" | "authed">(enabled ? "checking" : "authed");

  useEffect(() => {
    if (!enabled) return;
    let active = true;
    void (async () => {
      if (await client.tryRefresh()) {
        const me = await client.fetchMe();
        if (!active) return;
        setUser(me);
        setStatus("authed");
      } else {
        if (!active) return;
        rememberReturnTo(window.location.pathname + window.location.search);
        redirect(client.loginUrl());
      }
    })();
    return () => {
      active = false;
    };
  }, [enabled, client, redirect]);

  const logout = useCallback(async () => {
    await client.logout();
    redirect("/"); // 포털로 — 백채널이 KC 세션까지 끊어 재진입 시 로그인 폼
  }, [client, redirect]);

  const value = useMemo<AuthContextValue>(() => ({ user, logout }), [user, logout]);

  if (status === "checking") {
    return (
      <div className="app-loading">
        <Spinner size="large" label="로그인 확인 중" />
      </div>
    );
  }
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}
```

- [ ] **Step 5: 게이트 테스트 통과 확인**

```powershell
pnpm vitest run src/auth/AuthGate.test.tsx
```
Expected: 5 PASS

- [ ] **Step 6: base/basename 배선**

`vite.config.ts` 전체 교체:

```ts
import react from "@vitejs/plugin-react";
import { defineConfig } from "vite";

export default defineConfig({
  // nginx 경로 기반 통합 배포: http://localhost/alm/ 아래에서 서빙된다
  base: "/alm/",
  plugins: [react()],
});
```

`src/app/main.tsx`에 import 추가 후 렌더 트리 교체:

```tsx
import { AuthGate } from "../auth/AuthGate";
```

```tsx
createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <ToastProvider>
      <AuthGate>
        <BrowserRouter basename="/alm">
          <App />
        </BrowserRouter>
      </AuthGate>
    </ToastProvider>
  </StrictMode>,
);
```

- [ ] **Step 7: JiraLayout 헤더에 사용자 표시 + 로그아웃**

`JiraLayout.tsx` import에 추가 (`Button`이 @chanho/react import 목록에 없으면 추가):

```tsx
import { Avatar, Button, Select, SideNav, TopBar } from "@chanho/react";
import { useAuth } from "../../../auth/AuthGate";
```

컴포넌트 본문 상단에 추가:

```tsx
  const { user: authUser, logout } = useAuth();
```

TopBar `actions` 교체:

```tsx
          actions={
            <>
              <ThemeToggle />
              {authUser ? (
                <>
                  <span className="jira-auth-user">{authUser.name ?? authUser.email}</span>
                  <Button size="small" variant="ghost" onClick={() => void logout()}>
                    로그아웃
                  </Button>
                </>
              ) : null}
              {me ? <Avatar name={me.name} size="small" /> : null}
            </>
          }
```

- [ ] **Step 8: 전체 게이트 + 커밋**

```powershell
pnpm typecheck; if ($?) { pnpm test }; if ($?) { pnpm build }
```
Expected: typecheck green, 61+5=66 tests green, build green (`dist/index.html` asset 경로 `/alm/assets/...` 확인)

```powershell
git add vite.config.ts src/app/main.tsx src/auth src/features/jira/components/JiraLayout.tsx
git commit -m "feat(auth): /alm base + 로그인 게이트(AuthGate) + 헤더 사용자/로그아웃 — nginx SSO 편입"
```

---

### Task 5: 우산 repo — nginx conf + compose 서비스 + Keycloak realm

**Files:**
- Create: `C:\MSA_TEMPLATE\infra\nginx\default.conf`
- Modify: `C:\MSA_TEMPLATE\infra\keycloak\docker-compose.yml`
- Modify: `C:\MSA_TEMPLATE\infra\keycloak\realm-export.json:35-38`

**Interfaces:**
- Consumes: 3앱 dist/ (Task 2~4 빌드 산출물), gateway :8000 라우트(/api/board, /oauth2, /login, /api/auth, /.well-known, /api/me)
- Produces: `http://localhost`(포트는 `NGINX_PORT`, 기본 80) 단일 진입점. auth-server는 `FRONTEND_URL=http://localhost` env로 기동해야 함(Task 6에서 문서화)

- [ ] **Step 1: nginx conf 작성**

`infra/nginx/default.conf` 전체 내용:

```nginx
# 3-SPA 통합 서빙 + API 프록시. 브라우저 → nginx(:80) → gateway(:8000) → services.
# 쿠키(host-only localhost)가 3앱에 자동 공유되도록 단일 오리진을 만든다 — SSO의 핵심.
server {
    listen 80;
    server_name localhost;

    # /wiki, /alm (슬래시 없음) → 정규화. location / 의 myFront 폴백에 삼켜지지 않게.
    location = /wiki { return 301 /wiki/; }
    location = /alm  { return 301 /alm/; }

    # API/인증 — gateway 로 프록시. X-Forwarded-* 는 auth-server(forward-headers-strategy:
    # framework)가 OIDC redirect_uri(http://localhost/login/oauth2/code/keycloak) 구성에 쓴다.
    # 주의: 정규식 location 이라 prefix location 보다 우선 매칭된다(의도).
    # /login (SPA 라우트, 슬래시 없음)은 매칭 안 됨 → myFront 로그인 페이지가 산다.
    location ~ ^/(api|oauth2|login|\.well-known)/ {
        proxy_pass http://host.docker.internal:8000;
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-Host $http_host;
        proxy_set_header X-Forwarded-Proto $scheme;
        # X-Forwarded-Port는 보내지 않는다 — $server_port는 컨테이너 내부 포트(80) 고정이라
        # NGINX_PORT≠80일 때 오히려 틀린 값이 된다. 포트는 $http_host(X-Forwarded-Host)에 실려 간다.
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # wiki-front (vite base=/wiki/). root 방식: /wiki/... → /srv/apps/wiki/...
    # (alias+try_files 조합의 고전 버그를 피하려고 root 사용)
    location /wiki/ {
        root /srv/apps;
        try_files $uri /wiki/index.html;
    }

    # alm-front (vite base=/alm/)
    location /alm/ {
        root /srv/apps;
        try_files $uri /alm/index.html;
    }

    # myFront (base=/) — 그 외 전부. SPA 딥링크는 index.html 폴백.
    location / {
        root /srv/portal;
        try_files $uri /index.html;
    }
}
```

- [ ] **Step 2: compose에 nginx 서비스 추가**

`infra/keycloak/docker-compose.yml`의 `services:` 아래(redis 다음)에 추가:

```yaml
  # 3-SPA 통합 서빙 + API 프록시 (단일 오리진 = SSO 쿠키 공유). conf는 ../nginx.
  # dist는 로컬 빌드 산출물을 ro 마운트 — 프론트 재빌드만 하면 재기동 없이 반영.
  # 중첩 마운트 금지: portal(/srv/portal)과 apps(/srv/apps/*)를 분리해 바인드 충돌 회피.
  nginx:
    image: nginx:1.27-alpine
    container_name: platform-nginx
    ports:
      - "${NGINX_PORT:-80}:80"
    volumes:
      - ../nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ../../myFront/dist:/srv/portal:ro
      - ../../wiki-front/dist:/srv/apps/wiki:ro
      - ../../alm-front/dist:/srv/apps/alm:ro
    extra_hosts:
      # Docker Desktop(Windows/Mac)은 내장이지만 Linux 호환을 위해 명시
      - "host.docker.internal:host-gateway"
```

- [ ] **Step 3: realm-export.json에 nginx 오리진 추가**

`realm-export.json`의 platform-bff 클라이언트에서 세 값을 수정:

```json
      "redirectUris": ["http://localhost:9000/login/oauth2/code/keycloak", "http://localhost:8000/login/oauth2/code/keycloak", "http://localhost/login/oauth2/code/keycloak"],
      "webOrigins": ["http://localhost:9000", "http://localhost:8000", "http://localhost"],
```

```json
        "post.logout.redirect.uris": "http://localhost:5173/*##http://localhost:9000/*##http://localhost/*"
```

(주의: `NGINX_PORT`를 80이 아닌 값으로 바꾸면 이 URI들에도 그 포트가 들어가야 한다 — Task 6 문서에 명시)

- [ ] **Step 4: 검증**

```powershell
# JSON 유효성
python -c "import json; json.load(open(r'C:\MSA_TEMPLATE\infra\keycloak\realm-export.json', encoding='utf-8')); print('JSON OK')"
# compose 문법
docker compose -f C:\MSA_TEMPLATE\infra\keycloak\docker-compose.yml config --quiet; if ($?) { echo 'COMPOSE OK' }
# nginx conf 문법 (dist 없어도 conf 문법만 검사)
docker run --rm -v C:\MSA_TEMPLATE\infra\nginx\default.conf:/etc/nginx/conf.d/default.conf:ro nginx:1.27-alpine nginx -t
```
Expected: `JSON OK` / `COMPOSE OK` / `nginx: configuration file /etc/nginx/nginx.conf test is successful`

- [ ] **Step 5: 커밋 (우산 repo, cwd=`C:\MSA_TEMPLATE`)**

```powershell
git add infra/nginx/default.conf infra/keycloak/docker-compose.yml infra/keycloak/realm-export.json
git commit -m "feat(infra): nginx 통합 프론트 서빙(3-SPA 단일 오리진) + gateway 프록시 + realm에 localhost 오리진 추가"
```

---

### Task 6: 문서화 + 진행 원장 + 수동 E2E 체크리스트

**Files:**
- Modify: `C:\MSA_TEMPLATE\infra\README.md` (기동 절차에 nginx 추가 — 간결하게)
- Modify: `C:\MSA_TEMPLATE\.superpowers\sdd\progress.md` (새 원장 섹션)
- Create: Obsidian `C:\myBrain\내 로컬\msa\MSA_TEMPLATE 설계문서\2026-07-14 nginx 통합배포+SSO 수동 E2E 체크리스트.md`

**Interfaces:**
- Consumes: Task 1~5의 결과 전부
- Produces: 사용자가 따라 할 기동/검증 절차

- [ ] **Step 1: infra/README.md에 nginx 섹션 추가**

기존 기동 절차 다음에 추가할 내용:

```markdown
## nginx 통합 프론트 (경로 기반 3-SPA + SSO)

`http://localhost`(포트: `NGINX_PORT`, 기본 80) 하나로 myFront(`/`)·wiki(`/wiki/`)·alm(`/alm/`)을 서빙하고
`/api·/oauth2·/login·/.well-known`은 gateway(:8000)로 프록시한다. 단일 오리진이라 RT 쿠키가
3앱에 공유된다(SSO).

기동 순서:
1. 프론트 3개 빌드: `myFront`에서 `npm run build` / `wiki-front`·`alm-front`에서 `pnpm build`
2. `cd infra/keycloak && docker compose up -d` (nginx 포함)
3. 백엔드 기동 — **auth-server는 반드시 `FRONTEND_URL=http://localhost` 로**:
   `$env:FRONTEND_URL='http://localhost'; .\gradlew bootRun` (eureka → auth → board → gateway 순)
4. 최초 1회: realm에 localhost 오리진이 없다면 재import 필요 —
   `docker compose down -v && docker compose up -d` (⚠ DB 볼륨 파괴: 사용자/게시글 초기화)

`NGINX_PORT`를 80 외 값으로 쓰면 realm-export.json의 redirectUris/webOrigins/post.logout과
`FRONTEND_URL`에도 같은 포트를 반영해야 한다.
```

- [ ] **Step 2: Obsidian 수동 E2E 체크리스트 작성**

새 노트 내용:

```markdown
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
- [ ] (알려진 리스크 재현 확인용, 선택) wiki·alm 두 탭 동시 새로고침 → 간헐 전체 로그아웃 가능(RT 재사용탐지 오인) — 발생 빈도 기록만
```

- [ ] **Step 3: 진행 원장 갱신**

`.superpowers/sdd/progress.md` 끝에 새 섹션 추가:

```markdown
---

# 진행 원장 — nginx 통합배포 + 3앱 SSO (2026-07-14)

spec: Obsidian `MSA_TEMPLATE 설계문서/specs/2026-07-14-nginx-sso-unified-frontend-design.md`
plan: Obsidian `MSA_TEMPLATE 설계문서/plans/2026-07-14-nginx-sso-unified-frontend.md`
repos: auth-server / myFront / wiki-front / alm-front / MSA_TEMPLATE(infra) — 5개 교차

## 태스크 상태
- Task 1 (auth-server returnTo 쿠키): 대기
- Task 2 (myFront returnTo+prod env): 대기
- Task 3 (wiki-front /wiki base+게이트): 대기
- Task 4 (alm-front /alm base+게이트): 대기
- Task 5 (infra nginx+compose+realm): 대기
- Task 6 (문서/체크리스트): 대기
```

(각 태스크 완료 시 구현 세션이 이 상태를 갱신한다)

- [ ] **Step 4: 커밋**

```powershell
# cwd=C:\MSA_TEMPLATE
git add infra/README.md .superpowers/sdd/progress.md
git commit -m "docs: nginx 통합배포+SSO 기동 절차 및 진행 원장 — E2E 체크리스트는 Obsidian"
```

---

## 자체 리뷰 결과 (플랜 작성 시점)

- **스펙 커버리지**: 목표 4개(단일 오리진/게이트/returnTo/백채널 로그아웃) ↔ Task 1~5 매핑 확인. 백채널 로그아웃은 무변경(쿠키 공유로 자동) — Task 6 체크리스트에서 검증 항목으로만.
- **의도적 편차 2건** (스펙에 없던 구현 세부, 리뷰어 확인 요망):
  1. wiki/alm `useAuth`는 Provider 밖에서 throw 하지 않음 — 기존 테스트가 App을 게이트 없이 렌더하기 때문 (myFront 계약과 다름, 코드 주석에 명시)
  2. wiki/alm 게이트는 `import.meta.env.PROD`에서만 활성 — vite dev 단독 기동 워크플로 보존. dev에서의 인증 검증은 nginx 경로로만 (스펙의 "dev는 VITE_API_BASE로 분기"를 이 방식으로 구체화)
- **타입 일관성**: `AuthGate` props(client/redirect/enabled), `rememberReturnTo(path)`, `RefreshTokenService.Issued` — Task 간 시그니처 대조 완료.
- **수동 단계 분리**: realm 재import(`down -v`)와 브라우저 E2E는 파괴적/대화형이라 자동 태스크에서 제외, Task 6 문서로만.
```
