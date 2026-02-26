## 1) Consent란 무엇인가요? 🤝

OAuth에서 **Consent(동의)** 는 한마디로,

> **사용자(Resource Owner)가 “이 앱(Client)이 내 리소스(Resource)에 이 범위(scope)만큼 접근해도 된다”고 승인(Approval)하는 UI/절차** 입니다.

OAuth 2.0은 “제3자 앱이 사용자 대신 리소스에 제한적으로 접근”하도록 설계되어 있고, 그 핵심 중 하나가 바로 **“승인(approval) 상호작용”** 입니다. ([IETF Datatracker][1])

즉, Consent는 OAuth 흐름에서 **“권한 부여(authorization) 결정을 사람이 확인하는 단계”** 에 해당합니다. ✅

---

## 2) Consent는 인증(Authentication)과 다릅니다 🔐≠🪪

OAuth에서 자주 섞이는 개념이 있어요.

* **인증(Authentication)**: “너 누구야?” (로그인)
* **인가/권한 부여(Authorization)**: “너(앱)가 뭐까지 해도 돼?” (권한 승인)

Consent는 **인가(Authorization)** 의 핵심 UX입니다.
사용자가 로그인(인증)을 했더라도, **앱이 요청한 권한(scope)을 허용할지** 는 별도의 결정이니까요. 🙋‍♂️✅

---

## 3) Consent 화면에는 무엇이 나오나요? 🧾

대부분의 프로바이더(구글/마이크로소프트 등)에서 Consent 화면은 보통 다음을 요약해 보여줍니다.

* 앱 이름/로고 (누가 요청하는지) 🏷️
* 요청하는 권한(Scopes) 목록 (뭘 하려는지) 🔎
* 데이터 사용 목적/정책 링크(개인정보처리방침 등) 📜
* 허용 / 거부 버튼 ✅❌

특히 scope는 OAuth에서 “접근 범위를 제한”하기 위한 메커니즘이고, 이 scope 정보가 **동의 화면에 표시** 된다는 점이 핵심입니다. ([oauth.net][2])

---

## 4) Consent가 “항상” 뜨지 않는 이유 😮

많은 분들이 여기서 헷갈립니다.

### ✅ 이미 한 번 동의했다면?

사용자가 과거에 같은 앱/같은(또는 포함 관계) scope에 동의했다면, 프로바이더는 보통 **동의 화면을 생략**하고 바로 코드를 발급하거나 토큰을 발급할 수 있습니다(SSO처럼 “조용히” 진행).

즉, Consent는 “매번 강제”가 아니라 **“필요할 때만”** 나타나는 것이 일반적입니다.

---

## 5) OIDC(로그인)에서 Consent는 더 미묘해요 🧠

OIDC는 OAuth 2.0 위에 “로그인(Identity Layer)”을 얹은 프로토콜입니다. ([openid.net][3])

* OIDC에서 `scope=openid` 는 “로그인 요청”의 신호예요.
* 그런데 로그인만 한다고 끝이 아니라, 앱이 `email`, `profile` 같은 추가 scope를 요청하면 “프로필/이메일 제공 동의”가 결합될 수 있습니다.

그래서 OIDC 환경에서는 UI상으로는 **“로그인 + 정보제공 동의”** 가 한 화면에 함께 보이는 경우가 많습니다. 🙌

---

## 6) Consent를 강제로 띄우는 방법: `prompt=consent` 💥

OIDC에는 `prompt` 파라미터가 있는데, 여기서 `prompt=consent` 를 주면 의미가 명확합니다.

* **“이전 동의 여부와 무관하게 동의 화면을 다시 보여줘”** ✅

예를 들어 구글 문서에서도 `prompt=consent`를 넣으면 매번 동의 화면이 뜨며, 필요할 때만 쓰라고 안내합니다. ([Google for Developers][4])
(마이크로소프트도 유사하게 `prompt=consent`로 동의 UI를 트리거한다고 설명합니다.) ([Microsoft Learn][5])

> ⚠️ 실무 팁: “무조건 consent”는 UX를 해치므로, **권한이 추가로 필요해졌을 때** 같은 확실한 이유가 있을 때만 쓰는 게 일반적입니다. 😅

---

## 7) “권한을 나중에 추가로 요청”하는 패턴 (Incremental Authorization) ➕

처음 로그인 때는 최소 권한만 요청하고,
나중에 특정 기능(예: 캘린더 연동)을 켤 때 추가 scope를 요청하는 전략이 흔합니다.

구글 OAuth 문서에는 이런 흐름에서 사용할 수 있는 옵션으로 `include_granted_scopes` 같은 파라미터를 언급합니다. ([Google for Developers][6])
(또한 구글은 “granular permissions(더 세분화된 권한)” 관련 가이드도 별도로 제공합니다.) ([Google for Developers][7])

이때 사용자는 “추가 권한”에 대해서만 다시 Consent를 보게 되는 흐름이 자연스럽습니다. 🧩

---

## 8) 구글 프로바이더에서 Consent가 특히 중요한 이유 🇬🇧🟦

구글은 OAuth 동의 화면을 **Cloud Console에서 ‘브랜딩/정책/스코프’ 관점으로 등록**하게 하고, 특정 scope는 **검증(verification)** 을 요구합니다.

* “OAuth consent screen 설정” 자체가 앱 등록/게시(publish)와 연결됩니다. ([Google for Developers][8])
* 구글은 “어떤 스코프는 민감(sensitive)해서 심사/리뷰가 필요”하다고 안내합니다. ([Google for Developers][9])
* 그리고 “브랜딩/데이터 접근 검증 상태”를 구분해서 관리한다는 공식 도움말도 있습니다. ([구글 지원 센터][10])

즉, 구글에서 Consent는 단순 UI가 아니라,
**앱 신뢰(브랜딩) + 데이터 접근 통제(스코프) + 검증 프로세스** 까지 포함한 큰 체계예요. 🏛️

---

## 9) Consent를 시스템 관점으로 보면 (아키텍처 그림처럼) 🧱

Consent는 사실 “토큰 발급” 앞단의 **Grant(승인 기록)** 를 만들기 위한 의사결정 단계입니다.

1. Client가 scope를 포함해 Authorization Request 전송 📤
2. Authorization Server가 사용자 로그인(필요 시) 처리 🔐
3. 사용자에게 “이 scope 허용?” 동의 UI 표시 🧾
4. 사용자가 승인하면 서버는 그 승인(Grant)을 기록 📝
5. 이후 코드/토큰 발급 🎟️

OAuth 2.0 자체가 이런 “승인 상호작용(orchestration)”을 핵심으로 삼습니다. ([IETF Datatracker][1])
(요즘은 Grant를 더 잘 관리하기 위한 스펙도 논의되고 있습니다.) ([openid.net][11])

---

## 10) 정리: Consent를 한 문장으로 🎯

✅ **Consent는 “사용자가 앱의 요청 권한(scope)을 보고 승인/거부하는 인가 UX”이며, OAuth의 ‘승인(Grant)’을 생성하는 핵심 단계** 입니다.
OIDC에서는 로그인과 함께 자연스럽게 결합되며, 필요 시 `prompt=consent`로 강제 재동의도 가능합니다. 🔁

---


[1]: https://datatracker.ietf.org/doc/html/rfc6749?utm_source=chatgpt.com "RFC 6749 - The OAuth 2.0 Authorization Framework"
[2]: https://oauth.net/2/scope/?utm_source=chatgpt.com "OAuth 2.0 Scopes"
[3]: https://openid.net/specs/openid-connect-core-1_0.html?utm_source=chatgpt.com "OpenID Connect Core 1.0 incorporating errata set 2"
[4]: https://developers.google.com/identity/openid-connect/openid-connect?utm_source=chatgpt.com "OpenID Connect | Sign in with Google"
[5]: https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols-oidc?utm_source=chatgpt.com "OpenID Connect (OIDC) on the Microsoft identity platform"
[6]: https://developers.google.com/identity/protocols/oauth2/web-server?utm_source=chatgpt.com "Using OAuth 2.0 for Web Server Applications | Authorization"
[7]: https://developers.google.com/identity/protocols/oauth2/resources/granular-permissions?utm_source=chatgpt.com "How to handle granular permissions"
[8]: https://developers.google.com/workspace/guides/configure-oauth-consent?utm_source=chatgpt.com "Configure the OAuth consent screen and choose scopes"
[9]: https://developers.google.com/identity/protocols/oauth2/scopes?utm_source=chatgpt.com "OAuth 2.0 Scopes for Google APIs"
[10]: https://support.google.com/cloud/answer/15549049?hl=en&utm_source=chatgpt.com "Manage OAuth App Branding - Google Cloud Platform ..."
[11]: https://openid.net/specs/oauth-v2-grant-management-ID1.html?utm_source=chatgpt.com "Grant Management for OAuth 2.0 (Draft)"
