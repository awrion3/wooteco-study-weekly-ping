### Spring Bean과 Application Context

스프링 프레임워크의 주된 특징 중 하나는 IoC (Inversion of Control) 컨테이너로, 이는 애플리케이션의 객체를 관리하는 데 쓰입니다.<br>
이러한 스프링 컨테이너가 인스턴스화하고 관리하는 객체를 Spring Bean이라고 합니다.<br>
그리고 ApplicationContext는 IoC 컨테이너를 구성하는데 사용되는 인터페이스이며, 다음과 같은 일들을 도맡고 있습니다.

- `@Configuration`이 붙은 클래스들을 설정 정보로 등록해 둡니다. 
- `@Bean`이 붙은 메서드들로 빈 목록을 생성해 둡니다.
- 클라이언트가 어떤 Bean을 요청하면, 빈 목록에서 요청한 빈을 찾습니다.
- 설정 클래스로부터 빈 생성을 요청하고, 생성된 빈을 반환하게 됩니다.

스프링은 StaticApplicationContext도 지원하고 있으며, 애플리케이션 실행보다는 테스트 샘플로 사용하는데 더 적절합니다.<br>
아래와 같이 ApplicationContext와 다르게 수동으로 빈을 등록하는 과정이 별도로 필요합니다.

```java
public class StaticAppContextExample {
    public static void main(String[] args) {
        StaticApplicationContext context = new StaticApplicationContext();
        
        // 수동으로 빈 등록
        context.registerSingleton("myService", MyService.class);
        
        MyService service = context.getBean("myService", MyService.class);
        service.doSomething();
    }
}

class MyService {
    public void doSomething() {
        System.out.println("Hello from MyService!");
    }
}
```

ApplicationContext로 등록된 빈은 기본적으로 싱글톤으로 관리되며, 따라서 여러 번 해당 빈을 요청하더라도 매번 동일한 객체를 돌려줍니다.<br>
하지만 매번 클라이언트에서 요청이 올 때마다 관련된 빈을 새로 만들어서 사용하게 되는데 괜찮을까요?<br>
그래서 빈은 싱글톤 스코프로 관리되고 있습니다. 즉, 1개의 요청이 왔을 때 다수의 스레드가 관련 빈을 공유하도록 처리하고 있습니다.

다시 정리하면 ApplicationContext는 애플리케이션을 실행하기 위한 환경이라고 볼 수 있습니다.<br>
그리고 DI 컨테이너라고 불리기도 하는데, 이는 ApplicationContext가 빈들을 생성하는 BeanFactory 인터페이스를 부모로 상속받고 있기 때문입니다.<br>
BeanFactory는 다음과 같은 메서드들을 보유하고 있으며, 이를 바탕으로 특정 빈을 찾을 수 있게 됩니다.

<details>
<summary>BeamFactory 들여다보기</summary>

```java
public interface BeanFactory {

    String FACTORY_BEAN_PREFIX = "&";

    Object getBean(String name) throws BeansException;

    <T> T getBean(String name, Class<T> requiredType) throws BeansException;

    Object getBean(String name, Object... args) throws BeansException;

    <T> T getBean(Class<T> requiredType) throws BeansException;

    <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

    <T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);

    <T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);

    boolean containsBean(String name);

    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

    boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;

    boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

    @Nullable
    Class<?> getType(String name) throws NoSuchBeanDefinitionException;

    @Nullable
    Class<?> getType(String name, boolean allowFactoryBeanInit) throws NoSuchBeanDefinitionException;

    String[] getAliases(String name);
}
```

</details>

그리고 스프링 빈들은 ApplicationContext 자체에서 관리되는 것은 아닙니다.<br>
ApplicationContext는 빈들을 관리하는 BeanFactory의 구현체인 DefaultListableBeanFactory를 내부에 가지고 있습니다. <br>
이를 사용해서 ApplicationContext는 자신에게 온 빈 등록 혹은 탐색 요청을 위임하여 처리하게 되는 것입니다.

#### + Spring Bean과 Bean Definition

스프링 빈 `@Bean`마다 하나의 메타 정보가 생성됩니다.
해당 정보는 BeanDefinition에 추상화되어 담기게 되며 주요 속성은 다음과 같습니다.

- BeanClassName: 생성할 빈의 클래스명
- factoryBeanName: 팩토리 역할을 하는 빈 이름
- factoryMethodName: 빈을 생성할 팩토리 메서드명
- Scope: 빈의 범위 (기본값은 싱글톤입니다)
- lazyInit: 빈을 지연 초기화할지의 여부
- InitMethodName: 빈의 초기화 메서드
- DestroyMethodName: 빈의 소멸 메서드
- Constructor Arguments, Properties: 의존관계 주입 시에 사용

---
