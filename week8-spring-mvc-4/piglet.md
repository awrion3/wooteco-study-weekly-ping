## Interceptor, ArgumentResolver, (Filter)

---

![1.png](../images/week8-spring-mvc-4/1.png)

### Interceptor

---

- 위치: 스프링 컨테이너 내부
- 용도 : 인증, 인가, 요청 가공
- 리턴 타입 : `boolean`
    - true를 반환하면 다음 step으로 넘어간다.
    - false를 반환하면 더 진행 되지 않고 해당 위치에서 바로 API가 반환 된다.
- 컨트롤러 진입 전, `preHandle()`로 특정 API 경로에 대해 접근을 제어할 수 있다.
- 컨트롤러 코드의 반복 코드 (ex : 인증/인가, 로깅)를 처리할 수 있다.
- 공식문서에서는 보안 처리같은 것들은 Interceptor에서 사용하지 않기를 권장한다. (Spring Security, Filter 사용하기를 권장)

```java
public class CheckAdminInterceptor implements HandlerInterceptor {
    ...
		
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        String token = authorizationExtractor.extract(request);
        Member member = authService.getMember(token);
        if (!member.getRole().equals(Role.ADMIN)) {
		        // response.setStatus(403); 등 응답 설정으로 해줄 수도 있음
            throw new NotAdminException("접근할 수 없는 페이지입니다.");
        }
        return true;
    }
}    
```

### ArgumentResolver

---

- 위치: 스프링 컨테이너 내부
- 용도 : 커스텀 파라미터 바인딩
- 리턴 타입 : `Object`
- 실행 시점 : Interceptor의 `preHandler` 메서드 이후, 컨트롤러 메서드 실행 직전에 동작한다.
- 클래스에 직접적인 파라미터 바인딩 하거나 어노테이션으로 바인딩 하는 방법도 있다.
1. **어노테이션 바인딩**
    - `@PathVariable`, `@RequestParam` 등 Controller 파라미터에서 사용하는 어노테이션들
        - 아래는 `@PathVariable`를 처리하는 ArgumentResolver 내부

        ```java
        public class PathVariableMethodArgumentResolver extends AbstractNamedValueMethodArgumentResolver implements UriComponentsContributor {
            private static final TypeDescriptor STRING_TYPE_DESCRIPTOR = TypeDescriptor.valueOf(String.class);
        
            public PathVariableMethodArgumentResolver() {
            }
        
            public boolean supportsParameter(MethodParameter parameter) {
                if (!parameter.hasParameterAnnotation(PathVariable.class)) {
                    return false;
                } else if (!Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType())) {
                    return true; // 파라미터가 단일 값이면 @PathVariable 지원
                } else {
                    // 파라미터를 Map으로 받았을 경우 
                    PathVariable pathVariable = (PathVariable)parameter.getParameterAnnotation(PathVariable.class);
                    return pathVariable != null && StringUtils.hasText(pathVariable.value());
                }
            }
            ...
        }
        ```

        - (여담으로 PathVariable을 Map으로 처리할 수도 있다 하네요)
            - 하지만 아래 케이스는 `PathVariableMapMethodArgumentResolver`에서 따로 처리합니다. (`PathVariableMethodArgumentResolver` 는 안됨)

            ```java
            @GetMapping("/users/{userId}/reservations/{reservationId}")
            public ResponseEntity<?> handle(@PathVariable Map<String, String> pathVars) {
                String userId = pathVars.get("userId");     // "123"
                String reservationId = pathVars.get("reservationId");   // "4"
                ...
            }
            ```

2. **클래스 바인딩**

    ```java
    public class LoginMemberArgumentResolver implements HandlerMethodArgumentResolver {
        ...
        @Override
        public boolean supportsParameter(MethodParameter parameter) {
            return parameter.getParameterType().equals(Member.class);
        }
        ...
        public Member resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
                                      NativeWebRequest webRequest, WebDataBinderFactory binderFactory) {
            HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
            String token = authorizationExtractor.extract(request);
            return authService.getMember(token);
        }
    }
    ```

    - 그냥 클래스에 Member 파라미터 쓰면 이제 이 과정을 거친 Member만 추출됨

---

### (번외) Filter

- 위치: 웹 컨테이너 내부 (톰캣)
    - 그래서 Spring bean에 접근이 불가능하다.
        - `@ExceptionHandler + @ControllerAdvice` 못씀
- 용도 : 모든 요청/응답에 대한 전역 처리할 때 사용
- Interceptor랑 뭐가 달라요?
    - 스프링 컨텍스트가 아닌 ‘서블릿 컨테이너’(웹 컨테이너)에 있습니다
    - filter는 애플리케이션 전역적으로 적용되는 기능을 다루는데에 적합합니다
        - 이에 반해 interceptor는 인증/인가, 세부 보안, 로깅 등 MVC에 더 밀접하게 적용하는 로직에 적합하다고 하네요

## 로그인 시 응답은 Cookie vs Body 여기 다시 찾아보기

---

- 보안 여담
- Jwt 탈취 방법
    1. XSS : (Cross Site Scripting) 공격기법
        - 보안이 취약한 웹사이트에 악의적인 스크립트를 넣어, 사용자가 강제로 해당 스크립트를 실행하게끔 유도해 token을 탈취한다.
    2. CSRF : (Cross-site request forgery) 공격
        - 인터넷 사용자가 자신의 의지와는 무관하게 해커가 의도한 행위를 특정 웹 사이트에 요청하게 만드는 기법이다.
        - ex) 피싱 사이트를 만들어 로그인하게 만드는 방법

### 1. Cookie 통한 Jwt 전달

---

서버가 생성한 jwt를 Set-Cookie로 전달한다.

```java
@PostMapping("/login")
public ResponseEntity<Void> login(@RequestBody LoginRequest loginRequest, HttpServletResponse response) {
    TokenResponse token = authService.login(loginRequest);
    Cookie cookie = new Cookie(SET_COOKIE_KEY, token.accessToken());
    cookie.setHttpOnly(true);
    cookie.setPath("/");
    response.addCookie(cookie);
    return ResponseEntity.ok().build();
}
```

브라우저는 쿠키를 자동으로 저장하고, 이후 요청에도 자동으로 포함시켜준다.

**XSS 공격 대응**

- 쿠키는 HttpOnly 속성을 사용해서, JS를 통한 쿠키 접근을 차단할 수 있다.

  ⇒ JWT 보호가 가능하다.


**CSRF 공격 대응**

- 브라우저가 요청 시 자동으로 쿠키를 포함시켜서 CSRF 공격에 취약하다.
- 이를 방지하기 위해 SameSite 속성을 설정하거나 CSRF 토큰을 사용해야 한다.

### 2. Response body 통한 Jwt 전달

---

말 그대로, response body로 전달한다.

```java
@PostMapping("/login/token")
public ResponseEntity<TokenResponse> tokenLogin(@RequestBody TokenRequest tokenRequest) {
    TokenResponse tokenResponse = authService.createToken(tokenRequest);
    return ResponseEntity.ok().body(tokenResponse);
}
```

‘Authorization’ 헤더에 설정하는 것과 마찬가지로, 클라이언트는 로컬 스토리지에 저장하고 요청할 때 헤더에 포함 시켜야 한다.

**XSS 공격 대응**

- 클라이언트에서 로컬 스토리지나 세션 스토리지에 저장할 경우, JS 통한 접근이 가능

  ⇒ XSS 공격에 취약할 수 있다.


**CSRF 공격 대응**

- Cookie 사용보다 상대적으로 덜 취약하다.
    - 해커가 사용자의 브라우저에 임의 요청을 전송해도 JWT를 헤더에 포함시킬 수 없기 때문이다.

## Basic 인증은 언제 쓸까

---

Basic 인증이란, Authorization header에 `아이디:패스워드` 형태를 Base64로 인코딩해 포함하여 요청을 보낸다.

```java
Authorization: Basic dGVzdEBlbWFpbC5jb206cGFzc3dvcmQ= // test@email.com:password
```

절대절대 HTTP로 그냥 사용하기는 안된다. Base64 디코딩은 매우 쉽게 되기 때문이다. Basic 인증을 사용할거면 꼭 HTTPS로 통신 해야 한다.

**장점**

- 간단하다!
    - 토큰이나 세션처럼 상태 관리가 필요 없다.
    - 별도의 로그인 페이지가 필요없다. ( 서버 - 서버 간 통신에 적합하다 )
    - 간편함 때문에 다수의 서비스가 Basic 인증 방식을 사용하고 있기도 하다.

**단점**

- 서버가 요청을 받을 때마다 요청온 id/password 기반으로 유저를 DB에서 매번 조회 해야 한다.
    - 요청한 리소스가 많거나 사용자가 많을 경우 목록에서 권한을 확인하는 시간이 길어질 것이다.
    - 그래서 사용자가 많거나 사용자 변화가 잦은 서비스에서 사용하면 서버 부담이 커진다.
- 사용자 권한을 정교하게 제어하기가 힘들다.

보안적으로 너무 구려보이는데 왜 쓰죠?

### **토스페이먼츠에서 사용하는 Basic 인증**

---

결제, 결제 취소, 현금영수증 발급 등 상점에서 자주 사용하는 [**코어 API**](https://docs.tosspayments.com/reference)는 Basic 인증 방식을 사용한다.

HTTPS/SSL을 사용해서 보안 부분은 문제되지 않는다고 한다.

시크릿 키를 사용자 ID로 사용하고 비밀번호가 없다. `{SECRET_KEY}:`문자열을 base64 인코딩해서 API 인증 헤더에 사용한다.

```java
Authorization: Basic dGVzdF9za19PeUwwcVo0RzFWT0xvYkI2S3d2cm9XYjJNUVlnOg==
```

- 토스는 왜 Basic 인증을 썼을까?

토스페이먼츠의 코어 API는 ‘상점 서버’와 ‘토스페이먼츠 서버’ 간 서버-서버 통신이 주 목적이다.

실제 클라이언트와 서버 간 통신이 아니기에, 인증 주체인 상점은 미리 발급된 시크릿 키만 알고 있으면 된다. 인증 정보인 시크릿 키는 상점 서버에서 안전하게 보관하면 된다. 서버 - 서버 통신이라는 특성으로, 외부 사용자에 대한 인증/인가 분리가 필요 없기에 Basic 인증이 적합하다.

토큰을 사용하지 않아서 별도의 토큰 발급/관리/갱신 로직이 필요 없다. JWT은 토큰 탈취 시 권한 제어가 불가해 만료될 때까지 위험이 지속된다. 하지만 시크릿 키는 서버에서 제어하는 데이터이므로 노출되면 즉시 재발급 및 교체가 가능하다.

https://docs.tosspayments.com/blog/everything-about-basic-bearer-auth#bearer-%EC%9D%B8%EC%A6%9D%EC%9D%98-%EC%9E%A5%EC%A0%90-%EB%B0%8F-%EB%8B%A8%EC%A0%90

https://docs.tosspayments.com/reference/using-api/authorization

## **Q1. Claim에 포함할 정보란?**

---

뭘 넣어야 할지 모르겠다.

Jwt의 subject로 memberId를 설정한다. 그래서 권한 확인 할 때도, Interceptor에서 subject 추출해  memberId로 Member를 조회한다. 조회한 객체의 Role을 가지고 권한을 확인한다.

Claim은 인코딩만 될 뿐, 암호화는 안되므로 민감한 정보를 넣지 말라하는데, 그럼 뭘 넣어야 하는 것일까?

Role도 사용 안되는데 뭘 넣지?

의견 정리
1. 만약 role이 ADMIN이던 토큰을 발급한 후, 만료되기 전에 role이 USER로 강등 됐을 경우 토큰에 변경 사항이 반영될 수 없다.
   - 해당 토큰을 사용한 관리가 권한 어뷰징이 가능하다.
2. 하지만 위 내용은 role을 담았을 때의 문제점이 아닌, JWT 자체의 문제점이다.
3. 애초에 보안이 중요하면 세션을 써야한다.
4. DB에 한 번 왔다 갔다 하는 비용을 어느 수준의 비용이라 여기는지 등 기회비용의 분기점을 찾아서 로그인 전략을 채택해야 한다.
