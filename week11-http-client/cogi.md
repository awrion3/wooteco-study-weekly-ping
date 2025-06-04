## 🤔 HttpClient 란?

Java 애플리케이션에서 HTTP 요청을 보내고 응답을 받기 위해 사용하는 클라이언트 라이브러리이다.

HttpClient 객체는 Java에서 표준으로 제공하고 있다.(java.net.http.HttpClient)

```sql
HttpClient client = HttpClient.newHttpClient();

HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/data"))
        .build();

HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
System.out.println(response.body());

```

하지만 HttpClient는 사용하기가 불편하다. 그래서 스프링은 RestTemplate을 제공한다.

## ⭐ RestTemplate

- 간단 사용법

  ## **GET 요청 하기**

  `RestTemplate` 의 `getForEntity()` 메서드를 사용해 GET 요청을 하고 응답을 원하는 POJO로 변환할 수 있습니다.

    - 🤔 POJO

      POJO는 "Plain Old Java Object"의 약자로, **특정 규약이나 기술에 종속되지 않고 순수하게 자바 언어의 기본 기능만을 사용하는 객체**를 의미


    ```sql
    public Todo getTodoById(Long id) {
       try {
           ResponseEntity<Todo> response = restTemplate.getForEntity("http://jsonplaceholder.typicode.com/todos/{id}", Todo.class, id);
            return response.getBody();
       } catch (HttpClientErrorException e) {
            if (e.getStatusCode() == HttpStatus.NOT_FOUND) {
                throw new TodoException.NotFound(id);
            }
            throw new TodoException("Failed to get todo with id: " + id);
       }
    }
    ```
    
    ### **참고자료**
    
    - [The Guide to RestTemplate](https://www.baeldung.com/rest-template)

## ⭐ RestClient

- 간단 사용법

  ## **GET 요청 하기**

  `RestClient` 의 `get()` 메서드를 사용해 GET 요청을 할 수 있습니다.

    ```sql
    public Todo getTodoById(Long id) {
        return restClient.get()
               .uri("/todos/{id}", id)
               .retrieve()
               .onStatus(status -> status.value() == 404, (req, res) -> {
                   throw new TodoException.NotFound(id);
                })
               .body(Todo.class);
    }
    ```

  ### **참고자료**

    - [Spring - REST Clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)
    - [Spring blog - New in Spring 6.1: RestClient](https://spring.io/blog/2023/07/13/new-in-spring-6-1-restclient)

## ⭐ RestClient VS WebClient

| 항목 | **RestClient** | **WebClient** |
| --- | --- | --- |
| **도입 시점** | Spring Framework 6.1 / Spring Boot 3.2 이상 | Spring 5 이상 |
| **패러다임** | **동기식** (기본) | **비동기식** (기본), 동기식도 가능 (`.block()`) |
| **사용 목적** | 간단하고 직관적인 REST 호출 (RestTemplate 대체용) | 비동기/반응형 HTTP 통신을 위한 도구 |
| **기반 라이브러리** | `RestTemplate`의 개선판 (직접 구현) | Reactor 기반 (Project Reactor 사용) |
| **반응형 프로그래밍 지원** | 미지원 | **지원** (`Mono`, `Flux` 사용) |
| **코드 복잡도** | 간단하고 선언적 | 초기 학습 곡선이 약간 있음, 유연하지만 복잡할 수 있음 |
| **예외 처리** | 다양한 예외 처리 가능 (`onStatus`, `retry`, 등) | 다양한 예외 처리 가능 (`onStatus`, `retry`, 등) |
| **Streaming 처리** | 미지원 | 지원 (`Flux`로 스트리밍 응답 처리 가능) |
| **대표 사용 예시** | 동기 방식의 간단한 API 호출 | 대용량 처리, 반응형 처리, 비동기 통신이 필요한 경우 |

# ⭐ RestClient 테스트하기!

RestClientTest 에노테이션을 활용하자! 해당 애노테이션은 RestClientBuilder, MockRestServiceServer 등 필요한 빈만 등록해준다.

```sql
@RestClientTest(value = TossApiClient.class)
class TossApiClientTest {

    @Autowired
    private TossApiClient tossApiClient;

    @Autowired
    private MockRestServiceServer mockRestServiceServer; // 목 서버
    
    @BeforeEach
    public void setUp() {
        String expectedErrorResponse = """
                {
                  "code": "INVALID_API_KEY",
                  "message": "잘못된 시크릿키 연동 정보 입니다."
                }
                
                """;
        mockRestServiceServer.expect(requestTo("https://api.tosspayments.com/v1/payments/confirm"))
                .andExpect(method(HttpMethod.POST))
                .andRespond(withBadRequest()
                        .contentType(MediaType.APPLICATION_JSON)
                        .body(expectedErrorResponse));
    }
}
```

- 애노테이션 없이 직접 만들기

```java
class TossPaymentsClientTest {

    private final TossPaymentsClient paymentsClient = new TossPaymentsClient(TEST_BUILDER.build());
    private final MockRestServiceServer SERVER = MockRestServiceServer
            .bindTo(TEST_BUILDER)
            .build();
            
    @BeforeEach
    void setUp() {
        SERVER.reset();
    }

    @DisplayName("결제 승인 요청을 처리한다.")
    @Test
    void testConfirmPayments() throws JsonProcessingException {
        // given
        PaymentsConfirmRequest request = new PaymentsConfirmRequest("aaa", "111", 1000L);
        PaymentsConfirmResponse expectedResponse = new PaymentsConfirmResponse("aaa", 1000L);
        SERVER.expect(requestTo(BASE_URL + "/confirm"))
                .andExpect(method(HttpMethod.POST))
                .andRespond(withStatus(HttpStatus.OK)
                        .body(MAPPER.writeValueAsString(expectedResponse))
                        .contentType(MediaType.APPLICATION_JSON));
        // when
        PaymentsConfirmResponse actualResponse = paymentsClient.confirmPayments(request);
        // then
        assertThat(actualResponse).isEqualTo(expectedResponse);
		}
}
```
## 🤔 Mock
MockitoBean, Mock, InjectMocks, ExtendWith 등 언제 어디서 사용해야하는가?

### `@InjectMocks` `@Mock` 는 Spring의 의존성 주입이 아니다!!

- `org.mockito.InjectMocks`
- `org.mockito.Mock`
- 내가 등록한 Mock을 바탕으로 Mock 객체를 주입함
- 만약 Mock으로 등록하지 않았다면 해당 값은 null로 들어감

    ```java
    @ExtendWith(MockitoExtension.class)
    public class MockTest {
    
        @InjectMocks
        private PaymentService paymentService;
    
        @Test
        void test(){
            Assertions.assertThat(paymentService.getPaymentRepository()).isNull();
            Assertions.assertThat(paymentService.getPaymentClient()).isNull();
        }
    }
    ```

- 스프링과는 전혀 상관 없다!(MockitoBean으로 등록하고 InjectMocks을 쓰면 제대로 주입이 안됨)

    ```java
    @ExtendWith(MockitoExtension.class)
    public class MockTest {
    
        @MockitoBean
        private PaymentRepository paymentRepository;
    
        @InjectMocks
        private PaymentService paymentService;
    
        @Test
        void test(){
            Assertions.assertThat(paymentService.getPaymentRepository()).isNull();
            Assertions.assertThat(paymentService.getPaymentClient()).isNull();
        }
    }
    ```


## 🤔 퀴즈

나는 PaymentClient는 Mock으로 PaymentRepository는 실제 빈을 등록하고 싶다.

```java
@Service
public class PaymentService {

    private final PaymentClient paymentClient;
    private final PaymentRepository paymentRepository;

    public PaymentService(PaymentClient paymentClient, PaymentRepository paymentRepository) {
        this.paymentClient = paymentClient;
        this.paymentRepository = paymentRepository;
    }
    // getter 생략
 }
 
@DataJpaTest
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {

    @MockitoBean
    private PaymentClient paymentClient;

    @InjectMocks
    private PaymentService paymentService;

    @Test
    void test() {
        assertThat(paymentService).isNotNull();
        assertThat(paymentService.getPaymentClient()).isNotNull();
        assertThat(paymentService.getPaymentRepository()).isNull();
    }

}
```

## ⭐ 해결 방안

- 정답

    ```java
    @DataJpaTest
    @Import(PaymentService.class)
    class PaymentServiceTest {
    
        @MockitoBean
        private PaymentClient paymentClient;
    
        @Autowired
        private PaymentService paymentService;
    
        @Test
        void test() {
            assertThat(paymentService).isNotNull();
            assertThat(paymentService.getPaymentClient()).isNotNull();
            assertThat(paymentService.getPaymentRepository()).isNotNull();
        }
    }
    ```


## BDDMockito, Mockito 차이

`Mockito`와 `BDDMockito`는 **기능은 같지만, 스타일이 다름**

`BDDMockito`는 `Mockito` 기능에 given,when,then 식으로 테스트하도록 도와줌, 단순히 `Mockito` 를 포장한 형태임

- Mockito

```java
import static org.mockito.Mockito.*;
import static org.mockito.ArgumentMatchers.any;

@BeforeEach
void setUp() {
    PaymentResponse mockResponse = new PaymentResponse(
            "test_payment_key",
            "test_order_id",
            "CARD",
            50000,
            "DONE",
            OffsetDateTime.of(2025, 5, 28, 20, 48, 23, 0, ZoneOffset.UTC)
    );
    Mockito.when(paymentClient.authPayment(any()))
            .thenReturn(mockResponse);
}

@Test
void test() {
	verify(paymentClient).authPayment(any());
}
```

- BDDMockito

```java
import static org.mockito.BDDMockito.*;

@BeforeEach
void setUp() {
    PaymentResponse mockResponse = new PaymentResponse(
            "test_payment_key",
            "test_order_id",
            "CARD",
            50000,
            "DONE",
            OffsetDateTime.of(2025, 5, 28, 20, 48, 23, 0, ZoneOffset.UTC)
);
        when(paymentClient.authPayment(any()))
            .thenReturn(mockResponse);
    BDDMockito.given(paymentClient.authPayment(any()))
            .willReturn(mockResponse);
}

@Test
void test() {
	BDDMockito.then(paymentClient).should().authPayment(any());
}
```