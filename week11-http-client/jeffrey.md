# 서킷브레이커 패턴

Date: 2025년 6월 2일
Status: Done

## 서킷브레이커?

서킷 브레이커 패턴은 전기 회로의 차단기(누전 차단기) 개념을 소프트웨어 시스템, 특히 마이크로서비스 아키텍처(MSA)나 분산 시스템에 적용한 디자인 패턴이다. 이 패턴의 목적은 외부 시스템이나 서비스에 장애가 발생했을 때, 장애가 전체 시스템으로 확산되는 것을 막고, 빠르게 실패를 감지하여 자원을 보호하며, 시스템의 안정성과 복원력을 높이는 데 있다.

### 왜 필요한가?

(서비스 간의 통신에서) 장애의 전파를 막기 위해.

분리된 서비스 간 통신에서, 다른 서비스 장애가 시스템 전체로 확산되는 것을 막기 위해.

### 서킷 브레이커의 상태

1. Closed : 모든 요청이 정상적으로 처리되는 상태. 장애가 감지되지 않은 상태
2. Open : 장애가 임계치 이상 발생해, 해당 서비스로의 요청을 차단한 상태
    1. Count-based, Time-based
3. Half-Open : Open상태에서 일정 시간이 지난 후, 일부 요청만 허용하여 서비스가 복구됐는지 확인

### 흐름

장애 발생 시, 실패가 연속적으로 발생하면 해당 서비스에 대한 호출을 차단한다. (Open)

일정 시간이 지나거나, 조건이 만족되면 제한적으로 호출을 허용한다. (Half-Open)

장애가 복구되면 정상적으로 호출을 재개한다. (Closed)

![image.png](/images/week11-http-client/jeffrey/image.png)

### 동작 예시

1. 서비스 A가 서비스 B를 호출: A → B
2. 서비스 B에서 장애가 발생하여 연속적으로 실패(예: 3번 연속 실패)
3. 서킷 브레이커가 Open 상태로 전환, 이후 요청은 즉시 실패 응답 또는 Fallback 처리
    1. fallback: 서비스 호출이 실패하거나 차단될 때 대체 동작을 제공하는 방식
4. 일정 시간이 지나면 Half-Open 상태로 전환, 일부 요청만 서비스 B로 전달
5. 요청이 성공하면 Closed로 복귀, 실패하면 다시 Open 상태로 전환

### 장점

1. 장애 전파 방지
2. 빠른 실패 → 장애 발생시, Open 상태면 불필요한 대기가 발생하지 않고 바로 실패 응답 가능
    1. 빠른 실패로 인한 리소스를 절약할 수도 있음
3. Fallback 처리가능 → 미리 정의한 대체 로직(캐시 데이터 반환,  대체 서비스 호출 등)을 적용

## Resilience4j - https://resilience4j.readme.io/docs/getting-started

### 의존성 추가

```java
dependencies {
  implementation("io.github.resilience4j:resilience4j-all:${resilience4jVersion}")
}
```

기본적으로 모든 예외(Exception) 를 실패로 집게하고,

특정 예외만 포함 `recordExceptions` 및 제외 `ignoreExceptions` 하는 등 커스터마이징 가능하다.

보통 `4xx` 에러 (클라이언트 오류) 는 서버 장애가 아니므로, `ignoreExceptions` 로 제외하는 것이 일반적이다.

Fallback 메서드 호출은 설정과 무관하게 모든 예외에서 발생한다.

### 1. 설정 파일 (application.yml) 과 `@CircuitBreaker` 사용

```yaml
resilience4j:
  circuitbreaker:
    instances:
      hello-world:
	      # (1) 서킷브레이커 상태를 /actuator/health에 노출하여 모니터링 가능하게 함
        registerHealthIndicator: true
        # (2) Closed 상태에서 최근 7번의 호출 결과(성공/실패)만 저장해서 장애 판단에 사용        
        ringBufferSizeInClosedState: 7
        # (3) Half-Open 상태에서 최대 5번의 호출만 허용하여 정상 복구 여부 판단       
        ringBufferSizeInHalfOpenState: 5     
        # (4) Open 상태(모든 요청 차단)로 전환된 후 30초간 유지, 이후 Half-Open 상태로 자동 전환
        waitDurationInOpenState: 30s         
        # (5) 실패율이 60% 이상이면 서킷을 Open 상태로 전환(예: 7번 중 5번 실패 시 Open)
        failureRateThreshold: 60             

```

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class HelloWorldGateway {

    private final RestTemplate restTemplate;

    @CircuitBreaker(name = "hello-world", fallbackMethod = "fallbackForHelloWorld")
    public String getHelloWorld() {
        log.info("calling getHelloWorld()");
        return restTemplate.getForObject("/world", String.class);
    }

    // 장애 발생 시 호출되는 Fallback 메서드
    public String fallbackForHelloWorld(Throwable t) {
        log.error("Inside fallbackForHelloWorld, cause - {}", t.toString());
        return "Sorry ... Service not available!!!";
    }
}

```

```java
@RestController
@RequiredArgsConstructor
public class HelloWorldController {
    private final HelloWorldService helloWorldService;

    @GetMapping("/hello")
    public String hello() {
        return helloWorldService.getHelloWorld();
    }
}
```

### 2. CircuitBreakerConfig 사용후 `@Bean` 등록

```java
@Configuration
public class Resilience4jConfig {

    @Bean
    public CircuitBreakerConfig helloWorldCircuitBreakerConfig() {
        return CircuitBreakerConfig.custom()
            // (2) Closed 상태에서 7번의 호출 결과 저장 (COUNT 기반 슬라이딩 윈도우)
            .slidingWindowType(SlidingWindowType.COUNT_BASED)
            .slidingWindowSize(7)
            
            // (5) 실패율 60% 초과 시 Open 상태 전환
            .failureRateThreshold(60)
            
            // (4) Open 상태 30초 유지
            .waitDurationInOpenState(Duration.ofSeconds(30))
            
            // (3) Half-Open 상태에서 5번 호출 허용
            .permittedNumberOfCallsInHalfOpenState(5)
            
            // (1) 헬스 체크 활성화 (Spring Actuator와 자동 연동)
            .enableAutomaticTransitionFromOpenToHalfOpen()
            .build();
    }

    @Bean
    public CircuitBreakerRegistry helloWorldCircuitBreakerRegistry() {
        return CircuitBreakerRegistry.of(
            helloWorldCircuitBreakerConfig()
        );
    }

    @Bean
    public CircuitBreaker helloWorldCircuitBreaker() {
        // "hello-world" 인스턴스 생성 및 등록
        return helloWorldCircuitBreakerRegistry()
            .circuitBreaker("hello-world");
    }
}
```

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class HelloWorldGateway {

    private final RestTemplate restTemplate;
    private final CircuitBreaker helloWorldCircuitBreaker; // 주입

    public String getHelloWorld() {
        return helloWorldCircuitBreaker.executeSupplier(() -> {
            log.info("calling getHelloWorld()");
            return restTemplate.getForObject("/world", String.class);
        });
    }

    // 장애 발생 시 호출되는 Fallback 메서드
    public String fallbackForHelloWorld(Throwable t) {
        log.error("Inside fallbackForHelloWorld, cause - {}", t.toString());
        return "Sorry ... Service not available!!!";
    }
}
```

### 추가 고려사항

상태 변경 예외 발생 기준은 몇 회가 적당한지에 대한 판단기준?

Bulkhead 패턴이란 무엇일까?
