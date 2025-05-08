# 예외 처리

## Exceptions

- `@Controller`와 `@ControllerAdvice`는 컨트롤러의 메서드에서 발생한 예외를 처리하는 `@ExceptionHandler` 사용 가능
    
    ```java
    @Controller
    public class SimpleController {
    
        @ExceptionHandler(IOException.class)
        public ResponseEntity<String> handle() {
            return ResponseEntity.internalServerError().body("예외 메시지");
        }
    
    }
    ```
    

---

## Exception Mapping

### Spring MVC에서 예외 처리

- 예외가 발생한 경우, 위로 전파(propagate)되는 예외(top-level exception)로 매칭될 수 있다.
- 중첩된 예외인 경우, 가장 바깥쪽의 예외(`RuntimeException`) 뿐만 감싸진 예외(`RuntimeException`가 래핑한 `IOException`)도 매칭할 수 있다.
    
    ```java
    throw new RuntimeException(new IOException("입출력 예외 발생!"));
    ```
    
- Spring 5.2 이하
    - 던져진 예외(top-level exception)만 매칭 가능: `RunTimeException`
    - 직접적인 원인(cause)을 하나만 체크: `IOException`
    - 이후의 nested cause 무시
    - `@ExceptionHandler(IOException.class)` 호출 ❌
- Spring 5.3 이상
    - 던져진 예외뿐 아니라, 내부에 감싸진 모든 nested cause 체인 전체 탐색
    - `@ExceptionHandler(IOException.class)` 호출 ⭕️
- `@ExceptionHandler`가 매칭하는 예외 타입은, 되도록이면 타겟(target) 예외를 메서드의 파리미터로 선언 추천
    - 여러 개의 예외 메서드가 매치될 때는 직접 던져진 예외을 우선으로 선택
    - `ExceptionDepthComparator`가 던져인 예외 타입의 깊이를 기준으로 예외를 정렬
    - 아래 같은 경우 `IOException`이 던져지면 `handleIo` 메서드 선택
        
        ```java
        @ExceptionHandler(IOException.class)
        public ResponseEntity<String> handleIo(IOException ex) { ... }
        
        @ExceptionHandler(Exception.class)
        public ResponseEntity<String> handleAll(Exception ex) { ... }
        ```
        

---

### ExceptionHandler 매칭 우선순위

- 발생한 예외의 상위 타입 예외에 대한 `@ExceptionHandler` 매칭
    
    ```java
    throw new FileSystemException("파일 시스템 예외 발생!");  // 예외 발생
    
    @ExceptionHandler(IOException.class)       // ✅ 1순위: 상위 타입 (depth=0)
    @ExceptionHandler(RuntimeException.class)  // ❌
    @ExceptionHandler(Exception.class)         // ✅ 2순위: 상위 타입 (depth=1)
    ```
    
- 발생한 예외의 상위 타입/cause 체인 예외에 대한 `@ExceptionHandler` 매칭
    
    ```java
    throw new RuntimeException(new IOException("입출력 예외 발생!"));  // 예외 발생
    @ExceptionHandler(IOException.class)            // ✅ 2순위: cause 체인 매칭
    @ExceptionHandler(Exception.class)              // ✅ 1순위: 상위 타입 매칭
    ```
    
    - 직접 던져진 `RuntimeException`에 대한 `@ExceptionHandler` 등록 ❌
    - cause 체인을 따라가며 매칭
        - `RuntimeException.getCause()` → `IOException` 매칭 ⭕️
    - `IOException`이 먼저 매칭되고, `Exception.class`이 그보다 상위 타입 매칭으로 우선순위 결정
- Root Exception vs. Cause Exception
    - Root Exception: 가장 깊숙한 원인 예외, getCause()를 마지막까지 따라감
    - Cause Exception: 바로 한 단계 안쪽 예외, getCause()를 한 번 따라감
    
    ```java
    throw new ExceptionA(new ExceptionB(new ExceptionC("예외 발생!")));
    ```
    
    - ExceptionA: top-level exception)
    - ExceptionB: ExceptionA의 cause exception
    - ExceptionC: root exception, ExceptionB의 cause exception

---

### 예외 처리 주의사항

- 가능한 구체적인 메서드 시그니처 사용하기
    - 정확한 예외 타입을 메서드 인자에 지정하면 예외 매칭을 할 때 root exception과 cause exception 혼동 방지
        
        ```java
        @ExceptionHandler(IOException.class)
        ```
        
- 여러 타입을 처리하는 메서드는 나눠서 핸들링하기
    
    ```java
    // 한 번에 여러 타입 처리 ❌
    @ExceptionHandler({IOException.class, SQLException.class})
    public ResponseEntity<?> handle(Exception ex) { ... }
    ```
    
    ```java
    // 분리해서 타입 처리 ⭕️
    @ExceptionHandler(IOException.class)
    public ResponseEntity<?> handleIOException(IOException ex) { ... }
    
    @ExceptionHandler(SQLException.class)
    public ResponseEntity<?> handleSQLException(SQLException ex) { ... }
    ```
    
- 여러 `@ControllerAdvice`가 있을 경우 우선순위 지정하기
    - `@ControllerAdvice`가 여러 개면 충돌이 날 수 있기 때문에 `@Order` 애너테이션을 이용해 **우선순위**를 부여
    - root exception 매핑은 넢은 우선순위의 우선순위의 Advice에 두는 것 추천
    
    ```java
    @ControllerAdvice
    @Order(1)
    public class HighPriorityAdvice { ... }
    
    @ControllerAdvice
    @Order(2)
    public class LowPriorityAdvice { ... }
    ```
    
- 우선순위가 높은 Advice의 cause 매칭이 낮은 Advice의 root 매칭보다 우선순위가 높음
    - 예외 매칭은 같은 @ControllerAdvice 안에서는 root match > cause match
    - 그러나 서로 다른 @ControllerAdvice 사이에서는 우선순위 높은 쪽이 우선
    - 높은 우선순위의 Advice에서 cause가 매칭되면, 낮은 우선순위의 Advice에서 root match가 있어도 무시
- 핸들러에서 예외 처리 거절 가능
    - 핸들러 내부에서 `throw ex;` 를 던지면 Spring은 다음에 가능한 핸들러를 찾아서 연쇄적으로 처리
    - 다른 핸들러(우선순위가 낮은 Advice)에게 넘기는 방식
        - 계층화된 예외 처리 설계 및 조건부 예외 처리 가능
        
        ```java
        // 우선 순위 1
        @ControllerAdvice
        @Order(1)
        public class FirstAdvice {
            @ExceptionHandler(IOException.class)
            public ResponseEntity<?> handle(IOException ex) {
                if (ex.getMessage().contains("예외 발생!")) {
                    return ResponseEntity.status(404).body("첫번째 Advice에서 처리");
                }
                throw ex; // 여기서 예외 처리 거절 (현재 예외를 다시 던짐)
            }
        }
        
        // 우선 순위 2
        @ControllerAdvice
        @Order(2)
        public class SecondAdvice {
            @ExceptionHandler(IOException.class)
            public ResponseEntity<?> fallback(IOException ex) {
                return ResponseEntity.status(500).body("두번째 Advice에서 처리");
            }
        }
        ```
        

---

## **Media Type Mapping**

- Spring은 @ExceptionHandler에서 미디어 타입(Media Type)에 따라 다른 방식으로 예외 응답 제공 가능
    - 예를 들어 브라우저는 HTML, API 클라이언트는 JSON 요청 가능
- Media Type = 클라이언트가 원하는 응답 포맷
    - Client Accept: application/json 요청 → handleJson() 실행
    - Browser Accept: text/html 요청 → handleHtml() 실행
    
    ```java
    @ExceptionHandler(produces = "application/json")
    public ResponseEntity<ErrorMessage> handleJson(IllegalArgumentException ex) {
        return ResponseEntity.badRequest()
                             .body(new ErrorMessage(ex.getMessage(), 42));
    }
    
    @ExceptionHandler(produces = "text/html")
    public String handleHtml(IllegalArgumentException ex, Model model) {
        model.addAttribute("error", new ErrorMessage(ex.getMessage(), 42));
        return "errorView"; // templates/errorView.html 렌더링
    }
    ```
    

---

## Method Arguments

| 파라미터 | 설명 |
| --- | --- |
| Exception | 실제 발생한 예외 객체 |
| WebRequest or HttpServletRequest | 요청 정보 접근 (IP, 파라미터 등) |
| Model | HTML 렌더링할 때 전달할 데이터 |
| Principal | 로그인한 사용자 정보 |
| HttpSession | 세션 접근 |
| Locale, ZoneId | 요청자의 언어/시간대 정보 |
| OutputStream / Writer | 직접 응답 본문 출력 가능 (비표준 방식) |

---

## Return Values

| 반환값 종류 | 설명 |
| --- | --- |
| @ResponseBody | 객체를 HTTP 응답 본문으로 직접 반환 (JSON 등) |
| HttpEntity<T>, ResponseEntity<T> | HTTP 응답 전체를 직접 설정 (헤더 + 상태 코드 + 본문) |
| ErrorResponse | RFC 9457 형식의 에러 응답을 보낼 때 사용 |
| ProblemDetail | RFC 9457 기반의 오류 상세 정보 포함 응답 |
| String | 뷰 이름을 반환 (HTML 페이지를 렌더링할 때 사용) |
| View | View 객체를 직접 반환하여 렌더링 |
| Map, Model | 뷰로 전달할 데이터 모델을 반환 |
| @ModelAttribute | 뷰 모델에 추가될 객체를 반환 |
| ModelAndView | 뷰 이름과 데이터 모델을 함께 반환 |
| void | 응답을 직접 처리했거나, 별도 응답이 없을 때 |
| 기타 객체 | 단순 타입이 아니면 모델에 추가됨. 단순 타입이면 무시됨 |

---

## 출처

[Spring Exceptions](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-exceptionhandler.html)