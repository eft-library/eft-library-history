# 📂 목록

- [폐지된 커뮤니티 기능](./community.md)
- [디자인 리뉴얼 이슈 및 요청](./design.md)
- [로드맵 - 최고의 컨텐츠](./roadmap.md)
- [다국어 지원을 위하여](./i18n.md)
- [3D Map 도입 및 성능 개선 과정](./3dmap.md)
- [Analytics, Search Console, AdSense 도입기 및 경험 공유](./google.md)
- [NextAuth 도입기 – 프론트 중심 인증 경험](./auth.md)
- [프론트엔드 개발 비하인드 – 3번의 마이그레이션 여정](./migration.md)
- [사이트 통계 대시보드 개발기](./dashboard.md)

---

# NextAuth 도입기 – 프론트 중심 인증 경험

## 기존 경험과의 차이

- 이전에는 `Express.js` 기반 서버에서 `passport.js`를 활용하여 인증을 구현한 경험이 있었습니다.
- 하지만 `Next.js` 환경에서 `app` 디렉토리 기반으로 서버 기능을 함께 사용하는 방식은 **이번이 처음**이었고,  
  인증 흐름 자체가 **클라이언트-서버 간 역할이 섞여 있는 구조**여서 개념적으로 이해하는 데 다소 시간이 걸렸습니다.

---

## 개념 이해의 어려움

- 기존 백엔드 서버 방식과 달리 `NextAuth.js`는 **Next.js 내부의 API 라우트(또는 app router의 route handlers)**에 인증 로직을 포함하는 구조입니다.
- 특히, 인증 요청이 클라이언트에서 발생하고 그 응답이 API 라우트에서 처리되는 과정이  
  익숙한 MVC 구조와 다르다 보니, 초반에 **흐름을 따라가기 어려웠습니다**.

---

## 초기 구성의 복잡성

- 첫 도입부터 **Google과 Naver OAuth를 동시에 연동**하려다 보니 설정 파일 (, `.env`, provider 설정 , 조건 분기 설정 등)이 많아지고 복잡도가 증가하였습니다.
- 특히 리다이렉트 URI나 설정 등에서 오류가 발생하여 여러 시행착오를 겪음.
- 이후 글로벌 서비스를 염두에 두고, **Google 인증만을 남기기로 결정**하였습니다.
  - 다양한 국가의 사용자들이 사용하기에 Google OAuth는 범용적이고 접근성이 좋습니다.
  - 실제로 인증 흐름이 단순해지고 관리가 쉬워졌으며, 결과적으로 **좋은 선택**이었다고 판단했습니다.
- Google의 경우 Token을 받아야 하는데, 첫 로그인시만 Token을 건네주고 Refresh Token을 발급할 때는 Token을 주지 않았습니다.
  - `grant_type: "refresh_token"` 옵션을 주어서 해결했습니다.
- 커뮤니티 기능이 개발 되며 사용자 정보를 로그인 후 받아 오는 것으로 수정되었습니다.

---

## 해결 과정과 학습

- 공식 문서(https://next-auth.js.org/)를 꼼꼼히 참고하며 설정을 하나씩 진행.
- 각각의 provider에 필요한 설정값 (`clientId`, `clientSecret`, `profile`, `authorization`) 등을 구성하면서 점점 이해도가 높아졌습니다.
- 결국에는 **두 provider 모두 정상 동작**하도록 설정을 마무리했고, 이후부터는 사용자 세션을 활용하는 데 큰 어려움이 없었습니다.

---

## route.ts 코드

현재 적용중인 코드 이며 Refresh Token까지 적용한 상태입니다.

<details>
<summary>🔍 <strong>Route Code</strong></summary>

```js
/* eslint-disable @typescript-eslint/no-explicit-any */
import NextAuth from "next-auth";
import { JWT } from "next-auth/jwt";
import Google from "next-auth/providers/google";
import { USER_API_ENDPOINTS } from "@/lib/config/endpoint";

// Google Access Token 갱신 함수
async function refreshAccessToken(token: JWT) {
  try {
    const url = "https://oauth2.googleapis.com/token";
    const body = new URLSearchParams({
      client_id: process.env.NEXT_PUBLIC_GOOGLE_ID ?? "",
      client_secret: process.env.NEXT_PUBLIC_GOOGLE_SECRET ?? "",
      grant_type: "refresh_token",
      refresh_token: token.refreshToken ?? "",
    });

    const response = await fetch(url, {
      method: "POST",
      headers: { "Content-Type": "application/x-www-form-urlencoded" },
      body: body.toString(),
    });

    const refreshedTokens = await response.json();

    if (!response.ok) {
      console.error("Error refreshing access token:", refreshedTokens);
      throw refreshedTokens;
    }

    return {
      ...token,
      accessToken: refreshedTokens.access_token,
      accessTokenExpires: Date.now() + refreshedTokens.expires_in * 1000,
      refreshToken: refreshedTokens.refresh_token ?? token.refreshToken,
    };
  } catch (error) {
    console.error("Failed to refresh access token:", error);
    return { ...token, error: "RefreshAccessTokenError" };
  }
}

const handler = NextAuth({
  providers: [
    Google({
      clientId: process.env.NEXT_PUBLIC_GOOGLE_ID || "",
      clientSecret: process.env.NEXT_PUBLIC_GOOGLE_SECRET || "",
      authorization: process.env.NEXT_PUBLIC_GOOGLE_AUTHORIZATION || "",
    }),
  ],
  session: {
    strategy: "jwt",
    maxAge: 2 * 60 * 60, // 2시간
    updateAge: 2 * 60 * 60,
  },
  jwt: {
    maxAge: 2 * 60 * 60, // 2시간
  },
  callbacks: {
    // 로그인 시 사용자 추가
    async signIn({ user }) {
      if (!user) return false;

      try {
        const res = await fetch(USER_API_ENDPOINTS.ADD_USER, {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({
            id: user.id,
            name: user.name,
            email: user.email,
            image: user.image ?? "",
          }),
        });
        if (!res.ok) throw new Error("Failed to add user");
        return true;
      } catch (err) {
        console.error(err);
        return false;
      }
    },

    // JWT 콜백
    async jwt({ token, account, user, trigger, session }) {
      // 1️⃣ 최초 로그인 시 항상 새 토큰 세팅
      if (account && user) {
        const accessTokenExpires = account.expires_at
          ? account.expires_at * 1000
          : Date.now() + 3600 * 1000;

        let nickname: string | null = null;
        try {
          const res = await fetch(USER_API_ENDPOINTS.GET_USER, {
            headers: { Authorization: `Bearer ${account.access_token}` },
          });
          if (res.ok) {
            const data = await res.json();
            nickname = data.data?.nickname ?? null;
          }
        } catch {
          nickname = null;
        }

        return {
          ...token,
          accessToken: account.access_token ?? "",
          refreshToken: account.refresh_token ?? "",
          accessTokenExpires,
          nickname,
        };
      }

      // 2️⃣ 세션 업데이트 시 nickname 반영
      if (trigger === "update" && session?.userInfo?.nickname) {
        token.nickname = session.userInfo.nickname;
      }

      // 3️⃣ 토큰 만료 체크
      if (
        token.accessTokenExpires &&
        Date.now() < (token.accessTokenExpires as number)
      ) {
        return token;
      }

      // 4️⃣ 만료 시 refresh
      return refreshAccessToken(token);
    },

    // Session 콜백
    async session({ token }) {
      const safeToken = token ?? {};
      const sessionUser = { ...safeToken } as any;

      delete sessionUser.refreshToken;

      try {
        const res = await fetch(USER_API_ENDPOINTS.GET_USER, {
          method: "GET",
          headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${token.accessToken}`,
          },
        });

        if (res.ok) {
          const data = await res.json();
          sessionUser.userInfo = data.data ?? null;
        } else {
          sessionUser.userInfo = null;
        }
      } catch {
        sessionUser.userInfo = null;
      }

      return sessionUser;
    },
  },
});

export { handler as GET, handler as POST };

```

</details>

## 회고 및 요약

| 항목        | 내용                                                            |
| ----------- | --------------------------------------------------------------- |
| 초기 경험   | `passport.js` 기반 백엔드 인증 경험 O, Next.js 기반 인증은 처음 |
| 어려웠던 점 | 클라이언트 중심 인증 흐름 이해 / Google + Naver 동시 연동       |
| 해결 방법   | 공식 문서 기반 설정, provider 설정 하나씩 테스트                |
| 성과        | Google & Naver 연동 성공, 인증 흐름 및 session 처리 이해        |
| 현재 선택   | **글로벌화를 고려해 Google 인증만 유지**                        |

---
