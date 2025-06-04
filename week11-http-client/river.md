# 외부 API 연동하기

## RestClient

### RestClient 생성하기

- 동기식 HTTP Client로 유연하게 자바 객체를 HTTP Request로, HTTP Response에서 자바 객체로 변환
- 정적 create() 메서드를 사용하거나, 정적 builder() 메서드로 세부 옵션 설정 가능
    - 어떤 HTTP 라이브러리를 사용할 것 인지(Client Request Factories) 설정 가능
- 한 번 RestClient가 생성되면 멀티 스레드 환경에서 안전하게 사용 가능

### Basic 인증

- 가장 기본적인 HTTP 인증 방식
- Base64로 인코딩한 “사용자ID:비밀번호”를 Basic과 함께 헤더에 입력
    
    ```java
    Authorization: Basic base64({USERNAME}:{PASSWORD})
    ```
    
- Base64는 쉽게 복호화할 수 있으므로 HTTP가 아닌 HTTPS, SSL/TLS로 통신
- 토스 같은 경우 상점 정보가 필요한 API는 Basic 인증 사용
- 토스페이먼츠 같은 경우 시크릿키를 사용자ID로 사용하고, 비밀번호 사용X
    
    ```java
    Authorization: Basic base64({SECRET_KEY}:)
    ```
    

### 타임아웃

- **네트워크 타임아웃(Timeout)**
    - 서버로 요청을 보냈지만 일정 시간 동안 답변을 받지 못하면 발생
    - 리소스를 절약하기 위해 사용
- **Connection Timeout**
    - 클라이언트가 설정한 시간까지 서버에 연결되지 않으면 발생
    - 3-Way HandShake가 정상적으로 수행되어 Connection이 맺어지기 위해 소요된 시간
- **Read Timeout**
    - 클라이언트가 설정한 시간까지 서버에 응답이 오지 않으면 발생
    - 서버에 연결은 됐으나 서버가 클라이언트 요청을 정상적으로 처리하지 못한 경우 발생
    - Socket Timeout
        - Socket Timeout도 Read Timeout에 해당
        - 서버가 데이터를 클라이언트에 전송할 때 보내는 각 패킷 간의 시간 차이(Gap)의 제한(임계치)
    - 타임아웃 이후 중요한 API(e.g. 결제 승인) 중복 호출 문제는 멱등키로 안전하게 요청
        - 멱등성: 연산을 여러 번 하더라도 결과가 달라지지 않는 성질

### 멱등성

- **멱등하다는 것**
    - 첫 번째 수행을 한 뒤 여러 차례 적용해도 결과를 변경시키지 않는 작업 또는 기능의 속성
- **HTTP 메서드의 멱등성**
    - 멱등한 메서드: GET, PUT, DELETE
        - 리소스를 조회하거나 대체하는 메서드
    - 멱등하지 않은 메서드: POST, PATCH
        - 출할 때마다 응답이 달라지기 때문에 멱등하지 않은 메서드에 멱등성을 제공하려면 서버에서 멱등성을 구현
    - HTTP 메서드의 안전성과 멱등성
        - 안전성이 보장된 메서드는 리소스 변경X: GET
        - 안전성 보장 → 멱등성 보장
        - 멱등성 보장 → 반드시 안전성 보장X
            - PUT, DELETE == 멱등한 메서드 → 리소스에 변화X → 안전한 메서드X

## TossPaymentClientConfig 구성하기

```java
@Configuration
public class TossPaymentClientConfig {

    private static final String AUTH_TYPE_BASIC = "Basic";
    private static final String AUTH_DELIMITER = ":";
    private static final String URL_DELIMITER = "/";

    @Value("${payment.toss.base-url}")
    private String baseUrl;

    @Value("${payment.toss.secret-key}")
    private String secretKey;

    @Value("${payment.connect-timeout-length}")
    private Duration connectTimeout;

    @Value("${payment.read-timeout-length}")
    private Duration readTimeout;

    @Bean
    public TossPaymentClient tossPaymentClient() {
        return new TossPaymentClient(createRestClient());
    }

    private RestClient createRestClient() {
        return RestClient.builder()
                .requestFactory(createRequestFactory())
                .baseUrl(String.join(URL_DELIMITER, baseUrl, "v1", "payments"))
                .defaultHeader(HttpHeaders.AUTHORIZATION, createAuthorizationHeader())
                .build();
    }

    private ClientHttpRequestFactory createRequestFactory() {
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        factory.setConnectTimeout((int) connectTimeout.toMillis());
        factory.setReadTimeout((int) readTimeout.toMillis());
        return factory;
    }

    private String createAuthorizationHeader() {
        return AUTH_TYPE_BASIC + " " + base64Encode(secretKey + AUTH_DELIMITER);
    }

    private String base64Encode(String input) {
        return Base64.getEncoder()
                .encodeToString(input.getBytes(StandardCharsets.UTF_8));
    }
}

```

---

## 외부 API 연동 테스트하기

### RestClientTest

- Spring Boot 3.1+ 에서 등장한 테스트 슬라이스, RestClient를 테스트하기 위한 전용 환경 구성
    - RestClient 관련 Bean만 로드
        - 테스트할 대상만 Bean으로 등록
    - Jackson, RestClient, MockRestServiceServer 등을 자동 설정
    - 외부 API 호출 코드를 단위 테스트하는 데 적합
- **특징**
    - Spring Context 필요 (테스트 슬라이스)
    - Spring Bean (@Component, @Bean) 대상
    - 외부 API 컴포넌트 슬라이스 테스트 목적
    - @Value와 같은 설정 주입 가능
    - 슬라이스 테스트지만 Spring을 띄워 약간 느림
    - 요청/응답 조작이 MockRestServiceServer에 비해 상대적으로 덜 유연함

### MockRestServiceServer

- RestTemplate이나 RestClient 요청을 가로채서 Mock 응답을 반환해주는 도구
    - 네트워크 없이 외부 API 동작을 흉내냄
        - 실제 Bean이 요청하는 HTTP 요청에 대한 가짜 응답 반환
    - 요청 URI, HTTP 메서드, 헤더 등을 검증 가능
    - 응답도 임의로 구성 가능
- **특징**
    - Spring Context 필요 없음
    - 직접 생성한 RestClient 또는 RestTemplate
    - HTTP 호출만 단위 테스트 목적
    - @Value와 같은 설정 주입 불가능
    - Spring 없이 테스트하므로 매우 빠름
    - 요청/응답을 정밀하게 제어 가능
- Spring Context 없이 직접 생성한 RestClient를 테스트할 땐 MockRestServiceServer만 사용

### 예시

```java
// 프로덕션 코드
@Component
public class TossPaymentClient {

    private final RestClient restClient;

    public TossPaymentClient(RestClient restClient) {
        this.restClient = restClient;
    }

    public PaymentResponse requestPayment(PaymentRequest request) {
        return restClient.post()
            .uri("/confirm")
            .body(request)
            .retrieve()
            .body(PaymentResponse.class);
    }
}
```

```java
// RestClientTest
@RestClientTest(TossPaymentClient.class)
@TestPropertySource(properties = {
    "payment.toss.base-url=https://api.toss.com",
    "payment.toss.secret-key=test-secret"
})
class TossPaymentClientTest {

    @Autowired
    private TossPaymentClient tossPaymentClient;

    @Autowired
    private MockRestServiceServer server;

    @Test
    void requestPayment_shouldSucceed() {
        server.expect(requestTo("https://api.toss.com/confirm"))
              .andRespond(withSuccess("{\"status\":\"done\"}", MediaType.APPLICATION_JSON));

        PaymentRequest request = new PaymentRequest("paymentKey", "orderId", 1000);
        PaymentResponse response = tossPaymentClient.requestPayment(request);

        assertEquals("done", response.getStatus());
    }
}
```

```java
// MockRestServiceServer
class TossPaymentClientTest {

    private static final String BASE_URL = "https://api.toss.com/v1/payments";

    private final RestClient restClient = RestClient.builder()
        .baseUrl(BASE_URL)
        .build();

    private final TossPaymentClient tossPaymentClient = new TossPaymentClient(restClient);
    private final MockRestServiceServer server = MockRestServiceServer.bindTo(restClient).build();

    @BeforeEach
    void setUp() {
        server.reset();
    }

    @Test
    void requestPayment_shouldSucceed() {
        server.expect(requestTo(BASE_URL + "/confirm"))
              .andRespond(withSuccess("{\"status\":\"done\"}", MediaType.APPLICATION_JSON));

        PaymentRequest request = new PaymentRequest("paymentKey", "orderId", 1000);
        PaymentResponse response = tossPaymentClient.requestPayment(request);

        assertEquals("done", response.getStatus());
    }
}
```

---

## 고민

### **트랜잭션 내부에서 외부 API 호출**

- **트랜잭션 내부에서 외부 API를 호출하는게 맞는가?**
    - API 호출 실패 시 DB만 커밋되어 **데이터 불일치(무결성 오류)** 발생 가능
    - 외부 API 지연이 트랜잭션 유지 시간 지연 → **리소스 낭비 및 데드락 가능성 증가**
- **문제 개선 설계**
    - AOP 기반의 @Transactional이 분리된 Bean에서만 동작하므로, 같은 클래스에서 분리하면 트랜잭션 적용X
        
        ```
        ReservationService (Application Service Layer)
        ├── ReservationDomainService (트랜잭션: DB 작업)
        └── PaymentService (비트랜잭션: 외부 API)
        ```
        
        - ReservationService: 전체 예약 프로세스 조율, 트랜잭션 범위 분리
        - ReservationDomainService: 순수 도메인 로직 + 트랜잭션 처리
        - PaymentService: 외부 결제 API 연동 책임 전담
- **개선 효과**
    - 트랜잭션과 외부 통신 분리
    - 외부 호출 실패 시 재시도 처리 용이
    - 단일 책임 원칙(SRP) 보완
- **추가 문제점**
    - **결제 실패 시 이미 저장된 예약은 어떻게 할까?**
        - 트랜잭션을 분리해도 외부 API 호출 실패 시 DB에 저장된 예약이 그대로 남음 → 데이터 무결성 문제 여전히 존재
    - **해결 방안**
        - 보상 트랜잭션 적용: PaymentService 실패 시 예약을 취소하거나 상태를 FAILED로 바꿈
            
            ```java
            // 결제 실패 시 예약 취소
            public class ReservationService {
            
                public ReservationResponse createReservation(...) {
                    Reservation reservation = reservationDomainService.save(...);
            
                    try {
                        paymentService.confirmPayment(...);
                    } catch (Exception e) {
                        reservationDomainService.markAsPaymentFailed(reservation);
                        throw new PaymentFailedException("결제 실패", e);
                    }
            
                    return ReservationResponse.from(reservation);
                }
            }
            ```
