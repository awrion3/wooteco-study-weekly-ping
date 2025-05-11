## @ExceptionHandler

`@ExceptionHandler`는 `@Controller`가 붙은 곳에서만 사용 가능하다.

```java
// 예외 감지 가능
@RestController
public class MemberController {
    @GetMapping("/members/{id}")
    public ResponseEntity<Void> getMember(@PathVariable Long id) {
        if (true) {
            throw new NotFoundException("Member not found: id=" + id);
        }

        return ResponseEntity.ok().build();
    }

    @ExceptionHandler(NotFoundException.class)
    public ResponseEntity<String> handle(){
        return ResponseEntity.notFound().build();
    }
}
```

- **@Service, @Repository**는 **사용 불가**

`@ExceptionHandler` 는 Spring MVC에서 요청 처리 흐름에서만 작동한다.

`클라이언트 요청` → `DispatcherServlet` → `Controller` → `Response` 흐름 내에서 예외가 발생하면 `@ExceptionHandler`가 감지 가능하다.

`@Controller`로 등록된 클래스의 경우, DispatcherServlet이 요청을 처리할 때 이 bean들을 Http 요청에 대한 handler로 인식한다. 즉, Spring은 해당 클래스가 HTTP 요청에 대해 응답을 반환할 수 있는 대상임을 알고 있다.

`@Service`, `@Repository`는 웹 요청 처리 bean이 아닌 일반 bean (component)로 인식되기에, 해당 클래스들 내에서 `@ExceptionHandler`로 예외 처리를 추가해도 감지할 수 없다.

---

원래 `@ExceptionHandler`는 해당 컨트롤러 클래스 내에서 발생하는 예외만 처리된다.

  ⇒ 모든 컨트롤러마다 중복이 발생할 수 밖에 없다.

  ⇒ 그래서 같이 사용하는게 **`@RestControllerAdvice`**


## @ControllerAdvice

전역 처리를 위해 클래스에 붙이는 애노테이션이다.

만약 여러 개의 `ControllerAdvice`로 예외를 핸들링하게 된다면 `@Order`로 처리 순서를 지정해주어야 한다.
```java
@RestControllerAdvice
@Order(value = 1)
public class Handler {
    @ExceptionHandler(NotFoundException.class)
    public ResponseEntity<String> handle(){
        return ResponseEntity.badRequest().body("첫 번째 핸들러");
    }
}
```

- 정해주지 않으면 Spring이 알아서 임의 순서로 처리해버리므로 예외 처리의 일관된 응답이 안될 수 있다.
- 개발자가 Order를 관리 하기가 복잡해질 수 있으므로, 사실상 한 프로젝트 당 하나의 `ControllerAdvice`만 사용하는 것이 좋다.
- 만약 여러 개가 필요 하게 되면 `basePackages`나 `assignableTypes`를 사용해 관리 범위를 지정 해줘야 한다.

1. `basePackages` 설정하기
```java
@ControllerAdvice(basePackages = {"roomescape.auth.presentation.controller"})
public class Handler {
    @ExceptionHandler(NotFoundException.class)
    public ResponseEntity<String> handle(){
        return ResponseEntity.badRequest().body("첫 번째 핸들러");
    }
}
```
2. `assginableTypes` 설정하기
```java
@ControllerAdvice(assignableTypes = {AdminReservationController.class, MemberReservationController.class})
public class Handler {
    @ExceptionHandler(NotFoundException.class)
    public ResponseEntity<String> handle(){
        return ResponseEntity.badRequest().body("첫 번째 핸들러");
    }
}
```

### 예외 처리 외 `@ControllerAdvice` 사용법

**1. 글로벌한 데이터 바인딩**

요청을 전달하기 전에 전역적으로 데이터를 바인딩할 수 있다.
```java
@ControllerAdvice
public class GlobalBindingInitializer {
    
    @InitBinder // 해당 Controller로 들어오는 요청에 대해 추가적인 설정을 하고 싶을 때 사용
    public void initBinder(WebDataBinder binder) {
        // 문자열 값의 앞뒤 공백 제거
        binder.registerCustomEditor(String.class, new StringTrimmerEditor(true));
        
        // 날짜 형식 지정
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
        dateFormat.setLenient(false);
        binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, false));
    }
    
    @InitBinder("User")
    public void initUserBinder(WebDataBinder binder) {
        // User 객체에 대해서만 검증을 추가
        binder.addValidators(new UserValidator());
    }
}
```

**2. 응답 형태 공통화하기**

`ResponseBodyAdvice` 인터페이스를 구현하면 모든 요청에 대한 body에 대해 전역적으로 수정할 수 있다.
```java
@ControllerAdvice
public class GlobalAdvice implements ResponseBodyAdvice<Object> {

    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return true; // 모든 응답에 적용
    }

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType,
                                  Class<? extends HttpMessageConverter<?>> selectedConverterType,
                                  ServerHttpRequest request, ServerHttpResponse response) {

        // 이미 예외 response 형태면 그대로 반환
        if (body instanceof CustomExceptionResponse) {
            return body;
        }
        
        CustomResponse<Object> customResponse = new CustomResponse<>();
        customResponse.setSuccess(true);
        customResponse.setData(body);
        customResponse.setTimestamp(System.currentTimeMillis());

        return customResponse;
    }
}

class CustomResponse<T> {
    private boolean success;
    private T data;
    private long timestamp;
    // setter, getter 등
}
```
