# 🔐 OAuth2 / OIDC에서 `nonce`란 무엇인가?


OAuth2 로그인 흐름을 깊게 들어가면 `state`는 익숙해지지만,
OIDC(OpenID Connect)까지 확장되면 반드시 등장하는 값이 바로 **`nonce`** 입니다. 🧩

---

# 1️⃣ nonce 한 줄 정의 🎯

> `nonce`는 **ID Token의 재사용(Replay) 공격을 방지하기 위한 일회성 난수 값**입니다.

✔ state = CSRF 방지 <br>
✔ nonce = ID Token 재사용 방지 <br>

이 둘은 목적이 다릅니다. 👇

---

# 2️⃣ 왜 nonce가 필요한가? (문제 상황부터 이해하기) 🧠⚠️

OAuth2는 기본적으로 “Access Token 기반 API 접근” 프로토콜입니다.
하지만 OIDC는 여기에 **ID Token(JWT)** 을 추가하여 “사용자 인증 정보”를 전달합니다.

즉:

* OAuth2 → Authorization
* OIDC → Authentication + Authorization

OIDC에서는 Provider가 로그인 성공 후 **ID Token**을 발급합니다.

### 그런데 문제는…

ID Token은 JWT입니다.
JWT는 서명되어 있으니 위변조는 어렵습니다.

하지만 ❗

> 누군가 과거에 발급된 유효한 ID Token을 탈취해서
> 나중에 다시 사용하면 어떻게 될까요?

이게 바로 **Replay Attack(재사용 공격)** 입니다. 💣

---

# 3️⃣ nonce의 역할 🔐

nonce는 이 공격을 막기 위해 존재합니다.

### 작동 원리:

1️⃣ 로그인 요청 시 클라이언트가 랜덤 `nonce` 생성 <br>
2️⃣ Authorization Request에 포함 <br>
3️⃣ Authorization Server가 ID Token 안에 그 nonce를 포함 <br>
4️⃣ 클라이언트는: <br>

* “내가 보낸 nonce와”
* “ID Token에 들어있는 nonce”

를 비교합니다.

같지 않으면 ❌ 인증 거부

---

# 4️⃣ state vs nonce 차이 완벽 정리 🆚

| 구분           | state                  | nonce           |
| ------------ | ---------------------- | --------------- |
| 목적           | CSRF 방지                | ID Token 재사용 방지 |
| 포함 위치        | Authorization Response | ID Token Claim  |
| 검증 위치        | Callback 처리 시          | ID Token 검증 시   |
| OAuth2 필요 여부 | 필수                     | OIDC에서 필요       |
| 공격 방어        | 요청 위조                  | 토큰 재사용          |

👉 OAuth2 로그인만 쓰면 state만 필요 <br>
👉 OIDC 로그인(openid scope) 쓰면 nonce까지 필요 🪪 <br>

---

# 5️⃣ 실제 OIDC 로그인 흐름에서 nonce 🔁

OIDC Authorization Code Flow 기준:

```
GET /oauth2/authorization/google
```

Spring Security 내부에서:

* state 생성
* nonce 생성 (OIDC인 경우)
* 둘 다 세션에 저장

Provider로 리다이렉트 시:

```
https://accounts.google.com/o/oauth2/v2/auth?
  response_type=code
  &client_id=...
  &scope=openid profile email
  &state=abc123
  &nonce=xyz789
```

---

# 6️⃣ ID Token 내부에 nonce 포함 🪙

로그인 성공 후 ID Token 예시:

```json
{
  "iss": "https://accounts.google.com",
  "sub": "123456789",
  "aud": "client-id",
  "exp": 1720000000,
  "iat": 1719990000,
  "nonce": "xyz789"
}
```

Spring Security는 여기서:

* 세션에 저장된 nonce
* ID Token에 있는 nonce

를 비교합니다.

같지 않으면:

```
OAuth2AuthenticationException: Invalid nonce
```

발생 🔥

---

# 7️⃣ Spring Security 내부에서 nonce 처리 위치 🧠

OIDC일 경우 사용되는 클래스:

```
OidcAuthorizationCodeAuthenticationProvider
```

이 Provider는:

* ID Token 서명 검증
* issuer / audience 검증
* exp / iat 검증
* nonce 검증

을 수행합니다.

즉, nonce는 **ID Token 검증 단계에서 검사**됩니다.

---

# 8️⃣ Spring Security가 nonce를 어떻게 저장하나? 🗃️

nonce는 보통:

```
HttpSessionOAuth2AuthorizationRequestRepository
```

에 저장됩니다.

즉:

* AuthorizationRequest 객체에 nonce 포함
* 세션에 저장
* callback에서 복원
* ID Token과 비교

---

# 9️⃣ nonce가 없는 경우는? 🤔

### OAuth2 Only 로그인 (openid scope 없음)

→ ID Token이 없음
→ nonce 필요 없음

### OIDC Hybrid / Implicit Flow

→ nonce 필수

---

# 🔟 실무에서 nonce 관련 문제 TOP 6 💥

### ❌ 1) 세션이 유지되지 않는 경우

→ 저장된 nonce를 못 찾음
→ 인증 실패

특히:

* Stateless 구성
* Redis 세션 설정 오류
* 로드밸런서 sticky session 문제

---

### ❌ 2) 여러 탭에서 동시에 로그인 시도

→ nonce mismatch 가능

---

### ❌ 3) OIDC인데 openid scope 빠짐

→ ID Token 없음
→ OIDC 기대 코드에서 오류

---

### ❌ 4) Custom AuthorizationRequestRepository 구현 실수

→ nonce 저장/복원 누락

---

### ❌ 5) clock skew 문제

→ exp 검증 실패와 nonce 오류를 혼동

---

### ❌ 6) 프록시 환경에서 세션 유실

→ nonce 비교 실패

---

# 1️⃣1️⃣ nonce와 PKCE의 차이 🆚

이 둘도 많이 헷갈립니다.

| 구분    | nonce           | PKCE                           |
| ----- | --------------- | ------------------------------ |
| 목적    | ID Token 재사용 방지 | Authorization Code 탈취 방지       |
| 보호 대상 | ID Token        | Authorization Code             |
| 적용 영역 | OIDC            | OAuth2 (특히 public client)      |
| 위치    | ID Token claim  | token exchange 시 code_verifier |

즉:

* nonce = "이 ID Token이 지금 로그인에서 온 게 맞냐?"
* PKCE = "이 Authorization Code가 내가 보낸 요청에서 온 게 맞냐?"

---

# 1️⃣2️⃣ 보안 관점 정리 🔐

nonce는 다음을 보장합니다:

✔ ID Token이 현재 세션의 로그인 요청에서 발급된 것임
✔ 탈취된 ID Token의 재사용 방지
✔ 인증 단계에서의 Replay 공격 차단

state가 “요청 위조”를 막는다면,
nonce는 “토큰 재사용”을 막습니다.

---

# 🎯 최종 정리

> nonce는 OIDC 로그인에서 ID Token이 해당 인증 요청에 대해 발급된 것임을 증명하는 일회성 난수이며, 재사용 공격을 방지하기 위한 보안 장치다.

