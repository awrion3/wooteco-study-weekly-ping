### Http 비상태성

* HTTP는 기본적으로 클라이언트의 이전 요청 정보를 서버가 기억하지 않는 비상태(stateless) 프로토콜입니다.
* 특징:
    - 서버는 클라이언트의 이전 요청 상태를 저장하지 않습니다.
    - 매 요청은 독립적이며, 요청 간에 관계가 없습니다.
* 장단점:
    - 확장성 측면에서 유리합니다.
    - 로그인 상태 유지, 장바구니 저장 등 상태 정보가 필요한 기능 구현이 어렵습니다.

따라서 HTTP만으로는 클라이언트가 요청 시, 이미 이전에 인증 과정을 거쳤는지 알기 어렵습니다.
이를 해결하기 위해 쿠키(Cookie), 세션(Session), 토큰(Token) 등을 이용하여 상태를 간접적으로 유지할 수 있습니다.

### 쿠키

HTTP 쿠키는 웹 쿠키, 브라우저 쿠키라고도 부르며, `서버가 사용자의 웹 브라우저에 전송하는 작은 데이터 조각`이라고 합니다.
브라우저는 이러한 데이터 조각들을 저장해 놓았다가, 이후 동일한 서버에 다시 요청할 시 저장된 데이터를 함께 전송하게 됩니다.

따라서 쿠키는 어떤 두 요청이 들어왔을 때, 이들이 같은 브라우저에서 들어왔는지 여부를 판단할 때 주로 사용합니다. 
비상태성을 지니는 HTTP 프로토콜이지만, 이를 이용하면 사용자의 로그인 상태를 유지할 수 있게 되는 것입니다.

```http request
Set-Cookie: <cookie-name>=<cookie-value>
```
웹 서버가 클라이언트로 보내는 응답에 있어, `Set-Cookie` 헤더에 키하고 값을 함께 보내게 됩니다.
그럼 해당 응답을 받은 브라우저는 쿠키를 저장하고, 그 다음 요청부터 해당 쿠키를 자동으로 헤더에 넣어 보내게 됩니다.

```http request
Cookie: <cookie-list>
```
이후 `Cookie` HTTP 요청 헤더는`Set-Cookie` 헤더와 함께 서버로부터 이전에 전송되어 저장된 HTTP 쿠키들을 포함하게 됩니다.
이때 여기서 쿠키 리스트는 `<cookie-name>=<cookie-value>`의 이름-값 쌍의 목록 형태를 가지게 됩니다.

덧붙여 세션의 경우, 클라이언트를 식별하기 위해 서버는 클라이언트에게 JSESSIONID라는 고유 식별값을 쿠키 형태로 전달합니다.
이후 클라이언트가 요청을 보낼 때마다 이 세션 ID가 쿠키에 포함되어 함께 전송되며, 
서버는 이 ID를 바탕으로 내부에 저장된 세션 정보를 찾아 해당 사용자와 연결된 데이터를 조회할 수 있습니다.

```http request
GET /members/me/session HTTP/1.1
cookie: JSESSIONID=E7263AC9557EF658C888F02EEF840A19
accept: application/json
```

세션은 데이터가 서버 측에 저장되므로 쿠키 방식에 비해 보안성이 높고, 브라우저를 닫거나 일정 시간이 지나면 자동으로 만료되게 됩니다.

---

### HttpServletRequest

- ServletRequest: 클라이언트에서 서버로 전달되는 정보를 캡슐화하며, 요청의 콘텐츠 타입, 매개 변수, 헤더 정보, 속성 등의 데이터를 서블릿에 제공하게 됩니다.
서블릿은 이 객체를 통해 클라이언트가 보낸 요청 데이터를 읽고 처리할 수 있습니다.
- ServletResponse: 서블릿이 처리 결과를 클라이언트에게 응답할 때 사용하는 객체로, 응답 헤더를 설정하거나 출력 내용을 전송하는 데 사용됩니다.

클라이언트가 URL을 입력하면 HTTP Request는 서블릿 컨테이너로 전송되며, HttpServletRequest 및 HttpServletResponse 객체가 생성됩니다.
그리고 이때 HttpServletRequest가 제공하는 메서드를 사용해 클라이언트에서 보낸 사용자 입력 및 데이터들을 추출할 수 있습니다.

- 사용 예시:

```java
public AuthInfo extract(HttpServletRequest request) {
    String header = request.getHeader(AUTHORIZATION);
    //...
}
```

- 대표 메서드:

| 메서드                       | 설명                                           |
| ------------------------- | -------------------------------------------- |
| `getParameter(String name)` | 클라이언트가 보낸 요청 파라미터 값을 가져옴 (ex. form 입력값)      |
| `getMethod()`             | 요청 방식 확인 (GET, POST 등)                       |
| `getRequestURI()`         | 요청한 URI 경로 확인                                |
| `getHeader(String name)`  | 요청 헤더 값 조회 (ex. User-Agent, Authorization 등) |
| `getCookies()`            | 클라이언트가 보낸 쿠키 배열 조회                           |
| `getSession()`            | 현재 요청과 연결된 세션 객체를 가져옴 (없으면 새로 생성)            |

덧붙여서 HttpRequest가 요청되고 HttpResponse가 반환되기까지 서블릿 컨테이너에서는 다음과 같은 작업을 거치게 됩니다.
1. 우선 전송된 HttpRequest에 대해 서블릿 컨테이너에서 HttpServletRequest와 HttpServletResponse 객체가 생성됩니다.
2. 매핑할 서블릿을 확인한 뒤, 해당 서블릿 인스턴스가 존재하지 않는 경우에는 init() 메서드를 호출하여 생성하게 됩니다.
3. 그 다음 해당 서블릿에서 service() 메서드를 실행하여 클라이언트의 GET, POST 여부에 따라 doGet(), doPost() 등을 호출합니다.
4. 이후 호출된 메서드들로부터 동적 페이지를 생성하고, HttpServletResponse 객체에 응답을 보내게 됩니다.
5. 응답을 처리한 후에는 destroy() 메서드를 통해 HttpServletRequest 및 HttpServletResponse 객체를 소멸시킵니다.

---
