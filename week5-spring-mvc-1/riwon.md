## static vs template vs json 방식의 차이는 무엇일까?

### 1. static

> HTML, CSS, JS 파일을 그대로 웹 클라이언트에게 전달합니다.

* 장점
    - 빠름: 서버는 정적 파일만 전달하므로 처리 속도가 빠릅니다.
    - 설정 쉬움: 스프링에서는 /static 폴더에 두면 자동 제공됩니다.

* 단점
    - 동적 처리 불가능: 사용자 이름, 시간 등 동적 데이터 반영이 불가능합니다.
    - 재배포 필요: HTML 내용을 바꾸려면 다시 빌드하고 배포해야 합니다.

* 예시
    - 로그인 화면, 로고, 설명 페이지, 에러 페이지 등 단순한 페이지

```html
    <!DOCTYPE html>
    <html>
    <head>
        <title>정적 페이지</title>
    </head>
    <body>
        <h1>환영합니다!</h1>
        <img src="logo.png" alt="로고">
    </body>
    </html>
```

### 2. template

> 서버에서 데이터를 받아 HTML에 직접 넣어주게 됩니다.

* 장점
    - 동적 HTML 생성 가능: 사용자 정보, DB 내용 등을 반영한 페이지를 생성 가능합니다.
    - 간단한 CRUD에 유리: 서버-클라이언트 분리가 필요 없는 경우에 적절합니다.

* 단점
    - 서버 부하: HTML을 서버에서 매번 렌더링하므로 트래픽이 많을 땐 부담이 됩니다.

* 예시
    - 사내 관리자 페이지, 블로그 등

```java
    @Controller
    public class TemplateController {
        @GetMapping("/users")
        public String getUsers(Model model) {
            List<User> users = userService.findAll();
            model.addAttribute("users", users);
            return "users";
        }
    }
```

> 참고: static, template 중 어느 것이 먼저 읽히는가?

1. resources/static, resources/public 등 정적 리소스(static) 경로에 파일이 있는지 먼저 확인합니다.
2. 없으면 @Controller 등의 매핑된 핸들러 메서드(template)를 탐색합니다.
3. 그래도 없으면 404 에러를 반환합니다.

### 3. json

> 서버는 데이터를 JSON 형태로만 주고, 클라이언트(JS)에서 HTML을 구성하게 됩니다.

* 장점
    - 프론트엔드와 백엔드 분리: 개발 협업 및 유지보수에 유리합니다.
    - 빠른 사용자 경험: 클라이언트가 필요한 부분만 업데이트할 수 있습니다.
    - 재사용성: 모바일 앱, 웹 등 다양한 클라이언트가 API 공유가 가능합니다.

* 단점
    - SEO 어려움: 클라이언트측 렌더링이므로 검색엔진 노출에 한계가 있습니다.

* 예시
    - SPA(Single Page Application), 모바일 앱 백엔드 등

```java
    @RestController
    @RequestMapping("/api")
    public class ApiController {
        @GetMapping("/users")
        public ResponseEntity<List<User>> getUsers() {
            return ResponseEntity.ok(userService.findAll());
        }
    }
```

---
