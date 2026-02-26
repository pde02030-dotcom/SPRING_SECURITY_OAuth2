# `OAuth2LoginAuthenticationProvider` 내부에서 `DefaultAuthorizationCodeTokenResponseClient`가 어떻게 쓰이나? 🔥🧠

(= “Authorization Code → Token 교환이 **정확히 어디에서**, **어떤 분기로** 호출되는가”)

Spring Security OAuth2 로그인에서 “진짜 인증 로직의 본체”는 `OAuth2LoginAuthenticationProvider`입니다.
이 Provider가 하는 일은 딱 3단계로 압축됩니다. 🎯

1. **Authorization Code를 Token으로 교환** 🎫 (여기서 `DefaultAuthorizationCodeTokenResponseClient` 호출)
2. (OIDC면) **ID Token 처리/검증 + OidcUserRequest 구성** 🪪
3. **UserInfo 호출해서 Principal(OAuth2User/OidcUser) 생성** 👤

---

## 1) 먼저 전체 호출 위치: Provider 내부에서 딱 여기서 호출됩니다 📌

`OAuth2LoginAuthenticationFilter`가 만든 `OAuth2LoginAuthenticationToken(unauthenticated)`을 받아서,

✅ `OAuth2LoginAuthenticationProvider.authenticate()` 내부에서
✅ **토큰 교환을 먼저 하고**
✅ 그 결과(Access Token, Refresh Token, id_token)를 가지고 다음 단계를 진행합니다.

---

## 2) 핵심 의사코드: “분기 포함” 내부 흐름을 그대로 보기 🔍

아래는 Spring Security 내부 로직을 “코드 구조 그대로” 이해하기 좋은 형태로 재구성한 의사코드입니다.

```java
Authentication authenticate(Authentication authentication) {

  OAuth2LoginAuthenticationToken loginAuth =
      (OAuth2LoginAuthenticationToken) authentication;

  // ✅ 0) 요청/응답(authorization code, state, redirect_uri 등) 묶음
  OAuth2AuthorizationExchange exchange = loginAuth.getAuthorizationExchange();
  OAuth2AuthorizationResponse authorizationResponse = exchange.getAuthorizationResponse();

  // ✅ 0-1) state 검증 등 사전 검증 (CSRF 성격)
  // state mismatch -> OAuth2AuthenticationException

  // ✅ 1) "Authorization Code Grant Request" 구성
  OAuth2AuthorizationCodeGrantRequest codeGrantRequest =
      new OAuth2AuthorizationCodeGrantRequest(
          loginAuth.getClientRegistration(),
          exchange
      );

  // 🔥 2) 여기서 "토큰 교환" 호출됨 (DefaultAuthorizationCodeTokenResponseClient)
  OAuth2AccessTokenResponse tokenResponse =
      this.accessTokenResponseClient.getTokenResponse(codeGrantRequest);

  // tokenResponse 안에:
  // - access_token (필수)
  // - refresh_token (옵션)
  // - scope
  // - additionalParameters (OIDC면 id_token이 여기 들어오는 경우 많음)

  // ✅ 3) OIDC 분기: id_token이 있는/없는지 + openid scope/provider 설정 기반
  if (isOidcLogin(loginAuth.getClientRegistration())) {

      // (A) id_token 추출
      String idTokenValue = (String) tokenResponse.getAdditionalParameters().get("id_token");
      if (idTokenValue == null) {
          throw new OAuth2AuthenticationException("missing_id_token");
      }

      // (B) ID Token 검증/파싱 (JwtDecoder 기반)
      OidcIdToken idToken = parseAndValidateIdToken(idTokenValue, loginAuth.getClientRegistration());

      // (C) OidcUserRequest 만들기 (AccessToken + IdToken)
      OidcUserRequest oidcUserRequest =
          new OidcUserRequest(
              loginAuth.getClientRegistration(),
              tokenResponse.getAccessToken(),
              idToken,
              tokenResponse.getAdditionalParameters()
          );

      // (D) OidcUserService 호출 -> UserInfo 필요하면 호출 -> DefaultOidcUser 생성
      OidcUser oidcUser = this.oidcUserService.loadUser(oidcUserRequest);

      // (E) 최종 AuthenticationToken 생성(Authenticated=true)
      return new OAuth2LoginAuthenticationToken(
          loginAuth.getClientRegistration(),
          loginAuth.getAuthorizationExchange(),
          oidcUser,
          oidcUser.getAuthorities(),
          tokenResponse.getAccessToken(),
          tokenResponse.getRefreshToken()
      );
  }

  // ✅ 4) OAuth2(비-OIDC) 분기: DefaultOAuth2UserService로 UserInfo 호출
  OAuth2UserRequest oauth2UserRequest =
      new OAuth2UserRequest(
          loginAuth.getClientRegistration(),
          tokenResponse.getAccessToken(),
          tokenResponse.getAdditionalParameters()
      );

  OAuth2User oauth2User = this.userService.loadUser(oauth2UserRequest);

  return new OAuth2LoginAuthenticationToken(
      loginAuth.getClientRegistration(),
      loginAuth.getAuthorizationExchange(),
      oauth2User,
      oauth2User.getAuthorities(),
      tokenResponse.getAccessToken(),
      tokenResponse.getRefreshToken()
  );
}
```

> ✅ 결론: `DefaultAuthorizationCodeTokenResponseClient`는
> Provider의 `authenticate()` 안에서 **가장 먼저** 호출되는 “토큰 교환 1번 단계”입니다. 🔥

---

## 3) “분기”가 갈리는 정확한 지점 2곳 🧭

### ① OIDC 여부 판별 분기 🪪 vs 🎫

Provider는 대략 이런 기준으로 OIDC 분기로 갑니다.

* `ClientRegistration`의 scopes에 `openid`가 포함되어 있거나
* provider 메타데이터가 OIDC로 구성되어 있거나(issuer-uri 기반 구성 등)

✅ OIDC 분기로 가면 **반드시 `id_token` 처리가 뒤따릅니다.**

---

### ② UserInfo 호출 주체 분기 👤

* OIDC 분기 → `OidcUserService.loadUser(OidcUserRequest)`
* OAuth2 분기 → `DefaultOAuth2UserService.loadUser(OAuth2UserRequest)`

즉:

```text
(토큰 교환) → [OIDC?]
   ├─ YES: id_token 검증 → OidcUserService(UserInfo 호출 가능)
   └─ NO : DefaultOAuth2UserService(UserInfo 호출)
```

---

## 4) “토큰 교환 결과”가 이후 단계에 어떻게 전달되나? 🧪

`tokenResponse`에서 핵심 값들이 뽑혀 다음 단계로 전달됩니다.

### ✅ 공통

* `tokenResponse.getAccessToken()`
  → UserInfo 호출에 사용되고, 최종 AuthenticationToken에도 저장됨
* `tokenResponse.getRefreshToken()`
  → 있으면 최종 AuthenticationToken에 저장됨

### ✅ OIDC만

* `tokenResponse.getAdditionalParameters()["id_token"]`
  → `OidcIdToken`으로 파싱/검증 후 `OidcUserRequest`로 전달

---

## 5) 최종적으로 SecurityContext에 저장되는 값 📦🗄️

Provider가 `OAuth2LoginAuthenticationToken(Authenticated=true)`를 리턴하면, 이후 흐름에서 SecurityContext에 저장됩니다.

내부에 들어가는 주요 구성:

* principal: `DefaultOidcUser` 또는 `DefaultOAuth2User`
* accessToken, refreshToken
* authorities
* clientRegistrationId

---

## 6) 디버깅 포인트: 로그를 어디서 찍으면 “한 방에” 보이나? 🧯

실무에서 가장 잘 깨지는 지점은 거의 아래 3곳입니다.

1. **토큰 교환 단계** (여기서 `DefaultAuthorizationCodeTokenResponseClient` 예외)

* redirect_uri 불일치
* code 만료/재사용
* client 인증 실패

2. **OIDC id_token 처리 단계**

* id_token 누락
* signature/issuer/audience 검증 실패
* clock skew 문제(exp/iat)

3. **UserInfo 호출 단계**

* userInfoUri 누락/오류
* scope 부족
* sub 불일치(OIDC에서 중요)

---

## 7) 한 줄 요약 🧾✨

`OAuth2LoginAuthenticationProvider`는

✅ `DefaultAuthorizationCodeTokenResponseClient`로 **토큰 교환을 먼저 수행**하고 🎫
그 결과로

* OIDC면 `id_token` 검증 → `OidcUserService`로 principal 생성 🪪👤
* 아니면 `DefaultOAuth2UserService`로 principal 생성 👤

한 뒤, 최종 `OAuth2LoginAuthenticationToken`을 만들어 SecurityContext에 저장합니다. 🔐
