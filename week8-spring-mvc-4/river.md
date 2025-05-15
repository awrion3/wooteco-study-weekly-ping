# DispatcherServlet과 스프링 컨테이너 생성 과정

- Spring MVC에서 HTTP 요청을 처리하는 핵심은 DispatcherServlet
- DispatcherServlet은 서블릿 컨텍스트에 등록되면서 스프링 컨테이너(WebApplicationContext)를 생성하며, 그 안에서 MVC 구성 요소들을 빈으로 등록하여 전체 요청-응답 사이클을 처리
- 브라우저 요청 → WAS(톰캣) → DispatcherServlet → 컨트롤러 호출 → 응답 반환
    
    ![image.png](/images/spring-mvc4/river-image1.png)
    

---

## WAS(Tomcat) 실행

- WAS는 배포된 웹 애플리케이션을 실행할 때 초기 설정 파일을 탐색하고, 그에 따라 서블릿을 등록
- WAS 설정 방식
    - 전통적인 방식: web.xml에 DispatcherServlet 등록
        - DispatcherServlet 클래스가 서블릿으로 등록됨
        - 설정 파일(XML 위치)을 통해 **스프링 설정 정보**를 전달
        
        ```xml
        <servlet>
            <servlet-name>dispatcher</servlet-name>
            <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
            <init-param>
                <param-name>contextConfigLocation</param-name>
                <param-value>/WEB-INF/spring/appServlet/servlet-context.xml</param-value>
            </init-param>
        </servlet>
        ```
        
    - 현대적인 방식: WebApplicationInitializer 구현 클래스에서 자바 코드로 등록
        - XML 없이 자바 코드로 서블릿 등록
        - @Configuration 기반 클래스 사용 → 애노테이션 기반 설정
        
        ```java
        public class MyWebAppInitializer implements WebApplicationInitializer {
            @Override
            public void onStartup(ServletContext servletContext) throws ServletException {
                AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
                context.register(WebConfig.class); // @Configuration 클래스 등록
        
                ServletRegistration.Dynamic dispatcher =
                        servletContext.addServlet("dispatcher", new DispatcherServlet(context));
                dispatcher.setLoadOnStartup(1);
                dispatcher.addMapping("/");
            }
        }
        ```
        

## 설정 파일 스캔

- 설정 파일을 스캔하면서 설정 파일(XML 또는 @Configuration)을 기반으로 WebApplicationContext를 생성
- 이 컨텍스트는 일반적인 ApplicationContext의 하위 타입이며, Spring MVC에 특화된 설정을 포함

## DispatcherServlet 초기화

- DispatcherServlet은 단순히 요청을 처리하는 역할뿐 아니라, WebApplicationContext를 초기화하고 필요한 빈들을 구성하는 책임도 수행
    - 생성자에서 WebApplicationContext를 주입하거나, 내부적으로 생성
    - 생성 직후, 내부적으로 refresh()를 호출하여 스프링 컨테이너 초기화

## DispatcherServlet 생성 시점

- 생성자에서 ApplicationContext(WebApplicationContext)를 주입하거나 내부에서 생성
- 내부적으로 refresh()를 호출해 컨테이너 초기화 → 빈 등록

## 스프링 컨테이너 초기화

- DispatcherServlet은 설정 파일을 기반으로 다음과 같은 작업을 수행
    - @Component, @Controller, @Service, @Repository 등 애노테이션 기반 빈 등록
    - HandlerMapping, HandlerAdapter, ViewResolver, 메시지 컨버터 등 MVC 처리에 필요한 기본 빈 구성

## MVC 구성 설정

### 저수준 설정

```java
@Configuration
public class WebConfig {
    @Bean
    public HandlerAdapter handlerAdapter() {
        return new RequestMappingHandlerAdapter();
    }
}
```

- 모든 구성 요소를 직접 @Bean으로 등록
- DispatcherServlet은 이런 Bean들을 getBean(...)으로 주입받아 사용

### 고수준 설정 (@EnableWebMvc 사용)

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    // 필요한 부분만 override하여 설정
}
```

- 내부적으로 DelegatingWebMvcConfiguration을 통해 기본 설정 자동 등록
- WebMvcConfigurer를 구현하면 일부 설정만 쉽게 커스터마이징 가능

---

## DispatcherServlet 내부 코드 예시

- DispatcherServlet은 내부 구성 요소를 모두 스프링 컨테이너의 빈으로부터 주입받아 초기화
    
    ```java
    private void initHandlerAdapters(ApplicationContext context) {
        this.handlerAdapters = null;
    
        // 직접 빈으로 등록된 HandlerAdapter를 꺼내오는 방식
        HandlerAdapter ha = context.getBean("handlerAdapter", HandlerAdapter.class);
    }
    ```
