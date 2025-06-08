## HTTP Client 종류 및 다양한 사용 예시


### RestTemplate

- Spring에서 제공하는 동기식 HTTP Client
- Spring MVC 기반 애플리케이션에서 주로 사용한다.
- 응답을 받을 때까지 스레드가 차단되는 동기 방식이다.
- Spring 최신 버전부터는 유지 관리 모드 (신규 기능 없음) ⇒ 대체 기술을 사용하길 권장한다.

사용 예시

- 간단하고 단순하게 외부 API를 호출할 때 사용하면 좋다.

```java
RestTemplate restTemplate = new RestTemplate();
String content = restTemplate.getForObject("https://api.board.com/"+id, String.class);
System.out.println(content);
```

### RestClient

- Spring 6.1부터 도입된 최신 동기식 HTTP Client
- Fluent API를 제공한다.
- RestTemplate의 대체 목적으로 별도 의존성 없이 사용 가능하다.
- builder 방식을 지원해 더욱 가독성 좋다.

사용 예시

- 외부 API 호출 결과에 따라 반드시 즉시 다음 로직을 수행해야 하는 경우에 사용한다.

```java
@Service
public class OrderService {
    private final RestClient restClient;

    public OrderService() {
        this.restClient = RestClient.create();
    }

    public void processOrder(Order order) {
        // 1. 결제 API 동기 호출
        PaymentResult result = restClient.post()
            .uri("https://api.payment.com/pay")
            .body(order)
            .retrieve()
            .body(PaymentResult.class);

        // 2. 결제 결과에 따라 주문 상태 저장
        if (result.isSuccess()) {
            order.setStatus("PAID");
            orderRepository.save(order);
        } else {
            order.setStatus("FAILED");
            orderRepository.save(order);
        }
    }
}
```

### WebClient

- Spring 5.0부터 도입된 비동기/논블로킹 방식의 HTTP Client
- 리액티브 스트림(Reactive Stream)을 지원한다.
    - Reactive Stream?
      - 데이터는 비동기적으로 발생하며, 소비자는 이러한 데이터를 처리하는 동안 블로킹되지 않는다.
- 대량 요청을 효율적으로 처리한다.
- 동기식도 지원은 하지만 비동기/논블로킹이 강점이다.
- WebFlux 모듈이 필요하다.

사용 예시

- 비동기/논블로킹 방식으로 여러 외부 API를 동시에 호출하거나 응답이 오면 이벤트 기반으로 후처리(ex: 알림 발송)가 필요할 때 사용한다.

```java
@Service
public class NotificationService {
    private final WebClient webClient;

    public NotificationService(WebClient.Builder webClientBuilder) {
        this.webClient = webClientBuilder.baseUrl("https://api.notification.com").build();
    }

    public void sendNotifications(List<String> userIds) {
        userIds.forEach(userId -> {
            webClient.post()
                .uri("/notify")
                .bodyValue(new NotificationRequest(userId, "New message"))
                .retrieve()
                .bodyToMono(Void.class)
                .doOnSuccess(v -> log.info("Notification sent to user {}", userId))
                .doOnError(e -> log.error("Failed to notify user {}", userId, e))
                .subscribe(); // 실제로 요청이 비동기로 처리됨
        });
    }
}
```

- 여러 사용자에게 알림을 비동기로 발송하고, 응답 결과에 따라 로그만 남기고 비즈니스 흐름을 멈추지 않는다.
- 논블로킹으로 대량 처리에 적합하다.

### OkHttp

- Square사가 개발한 HTTP Client
- 성능이 좋고 사용이 간편하다.
- Spring 기본 지원은 아니라서 라이브러리 의존성을 추가해야 한다.

사용 예시

- 동기/비동기는 모두 지원하므로, 고성능/확장성 요구(ex: 파일 업로드/다운로드)나 인터셉터(ex: 로깅, 인증)가 필요한 비즈니스에 적합하다.

```java
@Service
public class FileService {
    private final OkHttpClient client = new OkHttpClient.Builder()
        .addInterceptor(new LoggingInterceptor())
        .build();

    // 동기 파일 업로드
    public void uploadFile(File file) throws IOException {
        RequestBody body = new MultipartBody.Builder()
            .setType(MultipartBody.FORM)
            .addFormDataPart("file", file.getName(),
                RequestBody.create(MediaType.parse("application/octet-stream"), file))
            .build();
        Request request = new Request.Builder()
            .url("https://api.files.com/upload")
            .post(body)
            .build();
        try (Response response = client.newCall(request).execute()) {
            if (!response.isSuccessful()) throw new IOException("File upload failed");
            log.info("File uploaded: {}", response.body().string());
        }
    }

    // 비동기 파일 다운로드
    public void downloadFileAsync(String fileId, Consumer<String> callback) {
        Request request = new Request.Builder()
            .url("https://api.files.com/download/" + fileId)
            .build();
        client.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                log.error("Download failed", e);
            }
            @Override
            public void onResponse(Call call, Response response) throws IOException {
                String filePath = saveToLocal(response.body().bytes());
                callback.accept(filePath);
            }
        });
    }
}
```
```java
public class LoggingInterceptor implements Interceptor {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();

        // 요청 정보 출력
        System.out.println("Request: " + request.method() + " " + request.url());

        Response response = chain.proceed(request);

        // 응답 정보 출력
        System.out.println("Response: " + response.code());

        return response;
    }
}

OkHttpClient client = new OkHttpClient.Builder()
        .addInterceptor(new LoggingInterceptor()) // 로깅 인터펩터 추가
        .addInterceptor(chain -> {
            // 인증 헤더 인터셉터 추가
            Request newRequest = chain.request().newBuilder()
                    .addHeader("Authorization", "Bearer YOUR_TOKEN_HERE")
                    .build();
            return chain.proceed(newRequest);
        })
        .build();
```

- 동기/비동기를 모두 지원해 확장성이나 커스터마이징이 필요한 비즈니스에 적합하다.

### OpenFeign

- Netflix에서 개발했고, 현재는 오픈소스로 유지되는 인터페이스 기반 선언적 HTTP Client
- 애노테이션으로 API를 정의한다.
- Spring Cloud와 통합이 쉽다.
- 기본적으로 동기, 비동기 지원은 별도 모듈이 필요하다.

사용 예시

- 마이크로서비스 환경에서 외부 서비스 호출과 결과를 가공하는 비즈니스에 적합하다.

```java
@FeignClient(
        name = "naverSearchClient",
        url = "https://openapi.naver.com/v1/search",
        configuration = NaverFeignConfig.class
)
public interface NaverSearchClient {
    @GetMapping("/blog")
    NaverSearchResponse searchBlogs(@RequestParam("query") String query);
}

@Service
public class BlogSearchService {
    private final NaverSearchClient naverSearchClient;

    public BlogSearchService(NaverSearchClient naverSearchClient) {
        this.naverSearchClient = naverSearchClient;
    }

    public List<BlogSummary> getBlogSummaries(String keyword) {
        NaverSearchResponse response = naverSearchClient.searchBlogs(keyword);

        return response.getItems().stream()
                .map(item -> new BlogSummary(item.getTitle(), item.getLink()))
                .collect(Collectors.toList());
    }
}
```

## 시크릿 키를 보관하는 방법


### 시크릿 키

외부 API를 클라이언트가 호출 할 때 인증 정보를 입력 해야 한다. 토스에서는 `Authorization` HTTP 헤더를 사용해 API를 호출 하는 클라이언트를 인증할 수 있다.

이 인증 정보로 ‘시크릿 키’를 사용하는데, 시크릿 키는 API를 호출하는 클라이언트의 신원을 보장하고 보호된 데이터나 리소스에 접근하는 요청을 확인한다.

### 시크릿 키를 소스 코드 레포지토리에 보관하지 말자

Github 같은 레포지토리에 시크릿 키를 보관하면 안된다. private은 괜찮다고 생각할 수 있다. 하지만 clone 및 fork 될 수 있어, 언제 어떻게 공유 되는지 추적 하기가 어렵다. 시크릿 키와 같은 민감 정보는 소스 코드에 보관하는 경우가 많기에 해킹 타깃이 되기도 한다. 또, 시크릿 키를 실수로 레포지토리에 하드 코딩 했다면 히스토리에 계속 남아 있을 수 있다.

반드시 시크릿 키와 같은 민감 정보는 별도의 파일로 관리하고 `.gitignore` 에 해당 파일을 꼭 추가해야 한다.

### **1. 로컬 개발 환경 : 별도의 환경 변수 (혹은 설정 파일) 활용**

- `.env` 파일 또는 별도의 설정 파일 사용하기
    - `resources` 디렉토리 아래, `tosspayments.yml` , `api-key.yml` 와 같은 별도의 파일에 저장한다.

    ```yaml
    # api-key.yml
    toss:
    	secret: toss_payments_secret_key
    ```

    - 이후, 이 파일을 `.gitignore`에 추가해 git에 올리지 않는다.

    ```yaml
    # .gitignore
    api-key.yml
    ```

- 환경 변수 사용하기
    - OS 환경 변수에 secret key를 저장하고, Spring에서 `${secret-key}` 식으로 참조해 사용한다.
    - macOS에서 환경 변수 추가하기

    ```bash
    export TOSS_PAYMENTS_SECRET_KEY="payments_secret_key"
    ```

    - `application.yml` 파일에서 읽어와 사용하기

    ```yaml
    # application.yml
    toss:
    	secret: ${TOSS_PAYMENTS_SECRET_KEY}
    ```


### **2. 운영 환경 : 외부 시크릿 관리 서비스 활용하기**

- AWS Secrets Manager, HashiCorp Vault 등
    - 민감한 정보를 클라우드 기반 시크릿 관리 서비스에 저장한다.
    - Springboot에서는 AWS Secrets Manager와 연동해 설정 값을 불러올 수 있다.
    - 예시
        - `bootstrap.yml` 파일을 통해 Secrets Manager에서 값을 불러온다.

        ```groovy
        // build.gradle
        implementation 'org.springframework.cloud:spring-cloud-starter-bootstrap'
        implementation 'org.springframework.cloud:spring-cloud-starter-aws-secrets-manager-config'
        ```

        - 설정 예시

        ```yaml
        # bootstrap.yml
        aws:
        	secretsmanagaer:
        		name: project-name
        	cloud: 
        		aws:
        			region:
        				static: ap-northeast-2
        ```

        - 코드에서는 바로 `@Value`로 값을 갖다 쓰면 된다.

### **3. CI/CD 및 배포 환경 : 파이프라인에서의 시크릿 관리**

- Github Actions, Jenkins 등에서 환경 변수로 주입
    - 배포 파이프라인에서 secret key를 환경 변수로 주입 하여 사용한다.
        - 예시 ) Github의 secret 등록하고 배포 파이프라인에서 사용하기
        ```yaml
        jobs:
          build:
              ...
              - name: CREDENTIAL_NAME, CREDENTIAL_PW
                run: |
                  echo "CREDENTIAL_NAME=${{ secrets.CREDENTIAL_NAME }}" >> $GITHUB_ENV 
                  echo "CREDENTIAL_PW=${{ secrets.CREDENTIAL_PW }}" >> $GITHUB_ENV 
        ```
        - 코드 상에 직접 노출이 되지 않는다.
- 배포 서버에서만 접근 가능한 설정 파일을 활용
    - 배포 서버에만 별도의 설정 파일을 두고 애플리케이션 실행 시 해당 파일을 읽도록 구성한다.
