# @RequestParam

Date: 2025년 4월 21일
Status: Done

## @RequestParam vs @PathVariable

https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-methods/requestparam.html

https://www.baeldung.com/spring-request-param

`@RequestParam` : bind Servlet request parameters (that is, query parameters or form data) to a method argument in a controller.

```java
@GetMapping("/hello")
public String world(@RequestParam(name = "name", required = false, defaultValue = "World") String name,
                    Model model) {
    model.addAttribute("name", name);
    return "hello";
}
```

`/hello?name=Brie` → name에 해당하는 Brie가 바인딩되어 model에 전달됨

---

https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/bind/annotation/PathVariable.html

https://www.baeldung.com/spring-pathvariable

`@PathVariable` : indicates that a method parameter should be bound to a URI template variable

```java
@DeleteMapping("/members/{id}")
public ResponseEntity<Void> delete(@PathVariable long id) {
    Member member = members.stream()
        .filter(it -> Objects.equals(it.getId(), id))
        .findFirst()
        .orElseThrow(RuntimeException::new);

    members.remove(member);
    return ResponseEntity.noContent().build();
}
```

`/members/1` → id에 1이 전달됨

### 어노테이션 내부

```java
@Target({ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RequestParam {
    @AliasFor("name")
    String value() default "";

    @AliasFor("value")
    String name() default "";

    boolean required() default true;

    String defaultValue() default "\n\t\t\n\t\t\n\ue000\ue001\ue002\n\t\t\t\t\n";
}
```

`value`,  `name` : 요청 매개변수 이름 지정

`required` : 필수인지, 선택인지 결정할 수 있도록

`defaultValue`  : 기본값 지정

1. @Target, @Retention , @Documented 어노테이션?
    - @Target

      https://www.geeksforgeeks.org/java-target-annotations/

      어노테이션은 클래스, 메서드 , 인스턴스 변수등과 같은  프로그램 요소에 메타데이터를 첨부하는데 사용됨

      자바에는 세 유형의 어노테이션이 있는데, `Marker` - without any methods, `Single-Valued` - with a single method, `Multi-Valued` - wit more than one method

      이중 `Target`은 `Single-Valued` 어노테이션.

      `Target` 은 또한 메타 어노테이션이라고도 하는데,  메타 어노테이션은 다른 어노테이션에 어노테이션을 추가하는데만 사용되며, 다른 어노테이션에 붙일 수만 있음.

      역할은 해당 어노테이션을 적용할 수 있는 요소의 유형(클래스, 인터페이스, 생성자 등)을 지정.

        ```java
        @Documented
        @Retention(RetentionPolicy.RUNTIME)
        @Target(ElementType.ANNOTATION_TYPE)
        public @interface Target {
        
            ElementType[] value();
        }
        ```

      `ElementType` enum만을 argument로 가짐

      | **Element Type** | **Element to be Annotated** |
              | --- | --- |
      | Type | Class, interface or enumeration |
      | Field | Field |
      | Method | Method |
      | Constructor | Constructor |
      | Local_Variable | Local variable |
      | Annotation_Type | Annotation Type |
      | Package | PACKAGE |
      | Type_Parameter | Type Parameter |
      | Parameter | Formal Parameter |

        ```java
        @Target(ElementType.TYPE)
        @interface CustomAnnotation {}
        ```

      이런식으로 작성했다면, CustomAnnotation은 Class, interface or enumeration에만 붙일 수 있음

      해당 타입 이외의 요소에 붙였다면 컴파일 에러가 난다.

    - @Retention

      https://www.geeksforgeeks.org/java-retention-annotations/

      `@Target` 과 마찬가지로 메타 어노테이션이며,

      `Retention Policy` 를 결정하는 역할, `Retention Policy`, `보존 규정` 이란 애노테이션이 삭제되는 시점을 결정하는 역할

      세가지 종류가 있으며, https://github.com/mtumilowicz/java-annotations-retention-policy

      https://docs.oracle.com/javase/8/docs/api/java/lang/annotation/RetentionPolicy.html

      | **정책** | **공식 설명** | 예시 |
              | --- | --- | --- |
      | **RetentionPolicy.SOURCE** | "Annotations are to be discarded by the compiler." 즉, 어노테이션은 컴파일러에 의해 버려지며, .class 파일에도 남지 않습니다. 주로 개발자 참고용(@Override, @SuppressWarnings 등)으로 사용됩니다. | @Override, @SuppressWarnings |
      | **RetentionPolicy.CLASS** | "Annotations are to be recorded in the class file by the compiler but need not be retained by the VM at run time." 즉, 어노테이션 정보는 .class 파일에 기록되지만, JVM이 클래스를 로드할 때는 유지되지 않습니다. 런타임 리플렉션으로 읽을 수 없습니다. 별도의 정책을 지정하지 않으면 기본값입니다. | lombok @NonNull |
      | **RetentionPolicy.RUNTIME** | "Annotations are to be recorded in the class file by the compiler and retained by the VM at run time, so they may be read reflectively." 즉, .class 파일에도 남고, JVM이 실행할 때도 유지되어 리플렉션을 통해 사용할 수 있습니다. 프레임워크에서 주로 사용됩니다. | @Deprecated, @Target, @FunctionalInterface |
    - @Documented

      https://www.geeksforgeeks.org/java-documented-annotations/

      `@Target` , `@Retention` 과 마찬가지로 메타 어노테이션.

      해당 어노테이션이 Javadoc 등의 API 문서에 포함되도록 지정하는 어노테이션.

      Javadoc이란, 자바 소스코드에 작성한 주석을 기반으로 HTML형식의 API문서를 자동으로 생성해주는 도구.

        ```java
        @Target(ElementType.TYPE)
        @Documented
        public @interface CustomInterface {
            String value();
        }
        
        @CustomInterface(value = "Documented")
        public class DocumentedAnnotationDemo {
        
            public static void main(String[] args) {
                System.out.println("DocumentedAnnotationDemo.main");
            }
        }
        ```


2. value와 name의 차이  + @AliasFor 어노테이션?
    - @AliasFor

      https://docs.spring.io/spring-framework/docs/4.2.0.RC2_to_4.2.0.RC3/Spring%20Framework%204.2.0.RC3/org/springframework/core/annotation/AliasFor.html

        ```java
        @Retention(RetentionPolicy.RUNTIME)
        @Target({ElementType.METHOD})
        @Documented
        public @interface AliasFor {
            @AliasFor("attribute")
            String value() default "";
        
            @AliasFor("value")
            String attribute() default "";
        
            Class<? extends Annotation> annotation() default Annotation.class;
        }
        ```

      서로 대체할 수 있는, 별칭(Alias) 관계임을 알려줌

      name이나 value중 하나만 사용해도 동일한 동작을 함.

      둘 다 지정한다면 같은 값을 지정해야 하고, 다른 값을 지정하면 에러가 남.

        ```java
        @Target(ElementType.TYPE)
        public @interface CustomInterface {
        
            @AliasFor(attribute = "aa", value = "bb")
            String value();
        }
        
        @CustomInterface(value = "Documented")
        public class DocumentedAnnotationDemo {
        
            public static void main(String[] args) {
                System.out.println("DocumentedAnnotationDemo.main");
            }
        }
        ```

      이 코드에서는 제대로 동작하는데, 그 이유는 Spring 프레임워크에서만 동작하는 어노테이션이기 때문.

      자바 JVM이나 컴파일러는 `@AliasFor` 를 인식하거나 강제하지 않고, 위에서는 단순 어노테이션 정보로만 존재하며 아무런 동작을 하지 않음.

      실제로 동작하려면,
      Spring의 **`AnnotationUtils`** 또는 **`AnnotatedElementUtils`**와 같은 유틸리티가 있어야 동작함.

      공식문서에는

        ```
        Like with any annotation in Java, the mere presence of @AliasFor on its own will not enforce alias semantics. For alias semantics to be enforced, annotations must be loaded via the utility methods in AnnotationUtils. Behind the scenes, Spring will synthesize an annotation by wrapping it in a dynamic proxy that transparently enforces attribute alias semantics for annotation attributes that are annotated with @AliasFor. Similarly, AnnotatedElementUtils supports explicit meta-annotation attribute overrides when @AliasFor is used within an annotation hierarchy. Typically you will not need to manually synthesize annotations on your own since Spring will do that for you transparently when looking up annotations on Spring-managed components.
        ```

      해당 해석본 AI 요약

      ## **해석**

        - **@AliasFor** 어노테이션이 자바 코드에 붙어 있다고 해서, 자바 자체가 별칭(aliased) 기능을 자동으로 적용해주는 것은 아니다.
        - 실제로 별칭 기능이 동작하려면, Spring이 제공하는 **`AnnotationUtils`** 같은 유틸리티 메서드로 어노테이션을 읽어야 한다[2](https://docs.spring.io/spring-framework/docs/4.2.0.RC2_to_4.2.0.RC3/Spring%20Framework%204.2.0.RC3/org/springframework/core/annotation/AliasFor.html)[4](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/annotation/AliasFor.html).
        - Spring은 내부적으로 어노테이션을 읽을 때, 동적 프록시(dynamic proxy)로 감싸서, @AliasFor가 붙은 속성들에 대해 별칭 규칙을 자동으로 적용해준다.
        - 또한, **`AnnotatedElementUtils`**를 사용하면 어노테이션 계층(메타 어노테이션)에서의 속성 오버라이드도 지원한다.
        - 일반적으로 개발자가 직접 어노테이션을 합성(synthesize)할 필요는 없다. Spring이 스프링 빈(컴포넌트)에서 어노테이션을 찾을 때 알아서 이 과정을 처리해준다.

      ## **요약**

        - **@AliasFor**의 별칭 기능은 자바 표준이 아니라 Spring이 제공하는 기능이다.
        - Spring이 어노테이션을 읽을 때만 별칭이 적용된다.
        - 개발자가 직접 신경 쓸 필요 없이, Spring이 알아서 동작시켜 준다.

3. "\n\t\t\n\t\t\n\ue000\ue001\ue002\n\t\t\t\t\n";
    - 센티널 값(sentinel value) , 감시값

      https://en.wikipedia.org/wiki/Sentinel_value

      센티널 값이란, 일반적인 값과 절대 겹치지 않도록, 줄바꿈 문자(\n, \t)와 유니코드 특수문자(\ue000, \ue001, \ue002 등)를 조합해 만드는 값.

      모든 유효한 값, 사용자가 입력할 수 있을만한 값과 구별되도록 선택해야함.

      Spring 프레임워크에서, 사용자가 defalutValue를 지정하지 않았음을 내부적으로 구분하기 위한 값.

      실제 기본값이 아니며, 해당 값이 반환되면 "defaultValue가 지정되지 않았다"는 신호로만 사용됨
