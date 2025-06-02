### Timeout

> Timeout 소개

Timeout은 서버로 요청을 보낸 뒤, 일정 시간 동안 응답을 받지 못한 경우 발생하며, 요청을 보냈는데 응답이 없으면 해당 요청을 실패로 처리하고 다시 요청을 보내는 방식으로 구성됩니다.
<br>그리고 아래와 같이 두 가지 유형이 존재합니다.
- Connection Timeout : 해당 Timeout 유형은 클라이언트에서 설정한 시간까지 서버에 연결되지 않으면 발생하게 됩니다.
- Read Timeout : 해당 Timeout 유형은 클라이언트에서 설정한 시간까지 서버에서 응답이 돌아오지 않으면 발생하게 됩니다.

이러한 Timeout을 통해 한 서버가 다수의 클라이언트와 연결을 생성했을 때, 응답 시간이 지연되는 요청에 대해서 리소스를 절약할 수 있게 됩니다.

### Circuit Breaker

> Circuit Breaker 소개

서킷 브레이커는 다량의 오류를 감지하게 되면 서킷을 열어 새 호출을 받지 않습니다.
그리고 서킷이 열려 있을 때 빠른 실패 로직을 수행하는데, 이어지는 호출에서 시간 초과나 예외 발생 등의 오류가 발생하지 않게끔 폴백 메서드를 호출합니다.
시간이 지나면 서킷 브레이커는 반열림 상태로 전환돼 새로운 호출을 허용하고, 문제를 일으킨 원인이 사라졌는지 확인합니다.
이후 새로운 오류를 감지하게 되면 서킷을 다시 열어 빠른 실패 로직을 다시 수행하며, 오류가 사라졌으면 서킷을 닫고 정상 작동하는 상태로 돌아갑니다.

> Circuit Breaker 장점

여러 번의 호출 실패 후에는 해당 서비스가 사용 불가능하거나 과부하 상태라고 간주할 수 있으며, 이후의 모든 요청을 즉시 거부할 수 있습니다.
이렇게 하면 실패 가능성이 높은 호출에 시스템 자원을 낭비하지 않고 절약할 수 있습니다.

> Circuit Breaker 상태

- CLOSED – 모든 것이 정상이며, 서킷 브레이커가 작동하지 않습니다.
- OPEN – 원격 서버가 다운된 상태로, 모든 요청이 즉시 차단(short-circuit)됩니다.
- HALF_OPEN – OPEN 상태에 진입한 후 일정 시간이 지나면, 서킷 브레이커가 일부 요청을 허용하여 원격 서비스가 복구되었는지 확인합니다.

> Circuit Breaker 예시

- Resilience4j 라이브러리: 원격 통신의 장애 허용(fault tolerance)을 관리하여 탄력적인 시스템을 구현하는 데 도움을 줍니다.

```java
CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
```

- 아래의 경우 실패한 호출에 대한 임계값을 20%로, 그리고 최소 호출 시도 횟수를 5회로 설정했습니다.

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
        .failureRateThreshold(20)
        .withSlidingWindow(5)
        .build();

CircuitBreakerRegistry registry = CircuitBreakerRegistry.of(config);
```

```java
CircuitBreaker circuitBreaker = registry.circuitBreaker("CircuitBreaker");
```

> Circuit Breaker의 Config 상세

1. `failureRateThreshold`
    - 설명: 실패율 임계값(%). 실패율이 이 임계값보다 크거나 같으면, 상태를 OPEN으로 변경하고 모든 호출을 차단합니다.
    - 기본값: 50


2. `slowCallRateThreshold`
    - 설명: 느린 호출 임계값(%). 호출 시간이 slowCallDurationThreshold보다 길어지면 느린 호출로 판단합니다.<br>
      이렇게 판단된 느린 호출의 비율이 이 임계값보다 크거나 같으면, 상태를 OPEN으로 변경하고 모든 호출을 차단합니다.
    - 기본값: 100


3. `slowCallDurationThreshold`
    - 설명: 호출 시간 임계값. 해당 시간보다 호출 시간이 길어지면, 해당 호출을 느린 호출로 판단합니다.
    - 기본값: 60000(ms)


4. `permittedNumberOfCallsInHalfOpenState`
    - 설명: 상태가 HALF_OPEN일 때 허용되는 호출 수입니다.
    - 기본값: 10


5. `maxWaitDurationInHalfOpenState`
    - 설명: 서킷브레이커가 HALF_OPEN 상태에서 가장 오래 대기하는 시간으로, 해당 시간 이후 OPEN으로 변경됩니다.<br>
      값이 0이면 서킷브레이커가 허용된 모든 호출이 완료될 때까지 HALF_OPEN 상태에서 대기함을 의미합니다.
    - 기본값: 0(ms)


6. `slidingWindowType`
    - 설명: 서킷브레이커가 CLOSED인 경우 슬라이딩 윈도우에서 값을 집계할 때의 방식입니다. COUNT_BASED 혹은 TIME_BASED가 되어야 합니다.
    - 기본값: COUNT_BASED


7. `slidingWindowSize`
    - 설명: 슬라이딩 윈도우의 크기로, 해당 값은 서킷브레이커가 CLOSED인 경우 값을 집계할 때 사용합니다.
    - 기본값: 100


8. `minimumNumberOfCalls`
    - 설명: 서킷브레이커가 값을 집계하기 전, 최소 호출해야 하는 횟수입니다.
    - 기본값: 100


9. `waitDurationInOpenState`
    - 설명: 서킷브레이커가 상태를 OPEN에서 HALF_OPEN으로 변경시키기 전까지 대기하는 시간입니다.
    - 기본값: 60000(ms)


10. `automaticTransitionFromOpenToHalfOpenEnabled`
    - 설명: OPEN에서 HALF_OPEN으로 넘어갈 때 자동으로 넘어갈 지 여부입니다.<br>
      false인 경우, waitDurationInOpenState 시간이 지나더라도 어떠한 호출이 일어나지 않으면 상태 변경이 일어나지 않습니다.
    - 기본값: false


11. `recordExceptions`
    - 설명: 실패로 측정될 예외 리스트로, 특정 예외들을 지정하면, 해당 예외들을 제외한 모든 예외들은 성공으로 측정됩니다.
    - 기본값: empty


12. `ignoreExceptions`
    - 설명: 실패와 성공 둘다로 측정되지 않을 예외 리스트입니다. 해당 예외와 일치하거나 상속하는 예외는 실패 또는 성공으로 측정되지 않습니다.
    - 기본값: empty


13. `recordFailurePredicate`
    - 설명: 특정 예외가 실패로 측정되도록 하는 커스텀 Predicate입니다. Predicate은 실패로 측정되고자 하는 예외는 true를, 성공으로 측정되고자 하는 경우는 false를 리턴해야 합니다.<br>
      기본값에서는 모든 예외가 실패로 기록됩니다.
    - 기본값: throwable(true)


14. `ignoreExceptionPredicate`
    - 설명: 특정 예외가 실패 또는 성공으로 측정되지 않도록 하는 커스텀 Predicate입니다. Predicate은 무시해야 하는 예외는 true를, 실패로 측정되고자 하는 경우는 false를 리턴해야 합니다.<br>
      기본값에서는 어떤 예외도 무시되지 않습니다.
    - 기본값: throwable(false)

이러한 옵션을 적용함에 있어서 아래와 같이 application.yml 파일을 통해서도 설정할 수 있습니다.

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 10
        failureRateThreshold: 50
        permittedNumberOfCallsInHalfOpenState: 5
        registerHealthIndicator: true
```

> Retry 기능

Retry 모듈을 사용하면 원격 서비스 호출 중 예외가 발생하는 상황에서 실패한 호출을 자동으로 재시도할 수 있습니다.

```java
RetryConfig config = RetryConfig.custom()
        .maxAttempts(2)
        .build();

RetryRegistry registry = RetryRegistry.of(config);

Retry retry = registry.retry("Retry");

Function<Integer, Void> decorated = Retry.decorateFunction(retry, (Integer s) -> {
        service.process(s);
        return null;
    });
```

> Bulkhead 기능

Bulkhead 모듈을 사용하면 특정 서비스에 대한 동시 호출 수를 제한하는 것이 가능합니다.

```java
BulkheadConfig config = BulkheadConfig.custom()
        .maxConcurrentCalls(1)
        .build();

BulkheadRegistry registry = BulkheadRegistry.of(config);

Bulkhead bulkhead = registry.bulkhead("Bulkhead");

Function<Integer, Integer> decorated = Bulkhead.decorateFunction(bulkhead, service::process);
```

이처럼 동시에 최대 1개의 호출만 허용하도록 구성할 수 있습니다.

> Circuit Breaker의 Sliding Window 유형

서킷브레이커는 예외 발생률과 느린 호출 비율을 함께 계산해 실패율(failure rate)을 도출합니다.
이 실패율이 사전에 설정한 임계치 이상이 되면, 상태가 즉시 OPEN으로 전환됩니다.
그리고 시스템의 안정성 확보를 위해 실패 지표를 수집하고 분석합니다.
이러한 지표는 슬라이딩 윈도우(Sliding Window) 구조에 저장되며, 이 데이터를 기반으로 상태 변경이 이루어집니다.

- Count 기반 윈도우: 
총 n개의 고정 크기 버퍼로 구성됩니다. 각 항목은 호출이 발생할 때마다 선입선출(FIFO) 방식으로 업데이트됩니다.

- Time 기반 윈도우: 
n개의 시간 단위 버퍼로 동작합니다. 단위는 epoch second 기준입니다.
예를 들어 10으로 설정하면, 1초마다 하나씩 총 10개의 시간 구간이 생성됩니다. 
시간이 흐름에 따라 오래된 구간은 새로운 구간으로 교체됩니다.

---
