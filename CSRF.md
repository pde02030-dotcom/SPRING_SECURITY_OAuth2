:pushpin: 전제 조건
1. 사용자는 브라우저의 한 탭에서 Google 계정에 로그인되어 있음
→ 예: https://accounts.google.com 에 로그인 완료
→ 브라우저에는 Google의 세션 쿠키가 저장되어 있음
2. 같은 브라우저의 다른 탭에서 악성 웹사이트 접속
→ 예: https://evil-site.com
3. Google은 특정 기능을 단순 HTTP 요청으로 처리한다고 가정
(예: 이메일 자동 전달 설정 변경 API)

:dart: 공격 시나리오
:one: 사용자는 이미 Google에 로그인된 상태
브라우저 쿠키 저장소:
Cookie:
SID=abcd1234;
HSID=efgh5678;
...
Domain: .google.com
이 쿠키는 SameSite=Lax 또는 설정이 느슨한 상태라고 가정합니다.

:two: 사용자가 악성 사이트 방문
사용자가 두 번째 탭에서:
https://evil-site.com
접속
해당 페이지에는 다음과 같은 JavaScript가 존재합니다:
<script>
fetch("https://accounts.google.com/api/change-forwarding", {
    method: "POST",
    credentials: "include",
    headers: {
        "Content-Type": "application/x-www-form-urlencoded"
    },
    body: "forwardTo=attacker@gmail.com"
});
</script>

:three: 브라우저 동작
브라우저는 다음과 같이 동작합니다:
요청 대상 도메인: accounts.google.com
해당 도메인에 대한 쿠키 존재
credentials: "include" 설정
브라우저는 자동으로 Google 세션 쿠키를 포함하여 요청 전송
즉 실제로 전송되는 HTTP 요청:
POST /api/change-forwarding
Host: accounts.google.com

Cookie: SID=abcd1234; HSID=efgh5678;
Content-Type: application/x-www-form-urlencoded

forwardTo=attacker@gmail.com

:four: Google 서버 입장
서버는:
유효한 세션 쿠키 확인
요청은 정상 로그인 사용자의 요청으로 보임
이메일 자동 전달 설정 변경 완료
:warning: 서버는 이 요청이 악성 사이트에서 발생했다는 사실을 모름.

:mag_right: 핵심 포인트
:exclamation: 브라우저 탭은 분리되어 있지만
쿠키 저장소는 브라우저 단위로 공유
즉:
[Tab 1] Google 로그인
[Tab 2] evil-site
           ↓
   Google로 요청 전송
           ↓
   세션 쿠키 자동 포함

:rotating_light: 이것이 CSRF인 이유
사용자는 Google에 로그인한 상태
사용자는 Google에서 해당 행동을 의도하지 않음
악성 사이트가 사용자의 인증 상태를 "악용"
이것이 Cross-Site Request Forgery 입니다.

:closed_lock_with_key: 왜 <script> 요청은 실제로는 어려울까?
현실에서는 다음 방어가 존재합니다:
SameSite=Strict 또는 Lax
CSRF 토큰 검증
Origin / Referer 체크
CORS 정책
Google은 상태 변경 API를 단순 form POST로 두지 않음
특히 fetch + Content-Type: application/json 요청은 preflight가 발생하고, Google이 CORS를 허용하지 않으면 실패합니다.

:bulb: 더 현실적인 CSRF 형태
전통적인 CSRF는 오히려 이렇게 생깁니다:
<form action="https://accounts.google.com/api/change-forwarding" method="POST">
    <input type="hidden" name="forwardTo" value="attacker@gmail.com">
</form>

<script>
document.forms[0].submit();
</script>
왜냐하면:
application/x-www-form-urlencoded
단순 POST
preflight 없음
SameSite=Lax 환경에서 top-level navigation이면 쿠키 포함 가능

:brain: 정리
브라우저 탭 구조:
브라우저
 ├── 탭1 (Google 로그인)
 ├── 탭2 (악성 사이트)
        └── Google로 요청
               └── 쿠키 자동 첨부
:point_right: CSRF는 "탭 간 문제"가 아니라
:point_right: "브라우저 단위 세션 공유"가 본질
