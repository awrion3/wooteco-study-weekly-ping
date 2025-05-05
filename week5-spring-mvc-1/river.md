# 방탈출 예약 관리 미션 1-3단계

## 테스트 범위 설정

- 기존에는 `/admin` URL과 `/reservation` URL 페이지를 모두 올바르게 불러오는 지에 대한 테스트 진행
- 그러나 `Controller`가 제대로 요청을 하는지에 대한 통합 테스트를 하도록 `ControllerTest` 수정
    - URL에 해당하는 HTML을 불러오는지에 확인하는 테스트X
- `Controller` 통합 테스트가 테스트할 때마다 매번 실제 서버를 띄우고 요청을 보내므로 성능 측면에서 개선하고자 함
    - `@Disabled` 처리 및 패키지 분리를 통해 필요할 때만 활성화해 테스트를 할 의도
- `@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)` 라인 삭제 → 에러
    - `@SpringBootTest`는 `MOCK`이 기본이 때문에 실제 서버를 띄워야 하는 현재 테스트와 부적합
    - 따라서 실제 서버를 띄우는 `DEFINED_PORT` 또는 `RANDOM_PORT`로 webEnvironment 옵션 설정 필요
    - 그러나 `DEFINDED_PORT`나 `RANDOM_PORT`를 사용한다면 포트 충돌이 일어나 확률이 매우 낮지만 여전히 발생할 수 있다는 점이 염려 되어 `@Disabled`를 사용

```java
@DisplayNameGeneration(DisplayNameGenerator.ReplaceUnderscores.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
@Disabled
class ControllerTest {

    @Test
    void url을_기반으로_html을_요청받을_수_있다() {
        ExtractableResponse<Response> response = RestAssured.given()
                .log().all()
                .when().get("admin")
                .then().log().all()
                .statusCode(HttpStatus.OK.value()).extract();

        assertThat(response.statusCode()).isEqualTo(HttpStatus.OK.value());
        assertThat(response.asString()).contains("<title>방탈출 어드민 메인 페이지</title>");
    }
}
```

## @Disabled 사용

- `@Disabled` 사용에 대한 리뷰어 의견
    - `@Disabled`는 일시적인 조치로는 유용하지만, 지속적으로 관리하기에는 불안정한 방식
    - 테스트 환경은 별도의 Spring Profile을 사용해 명확히 분리하는 것 추천
    - 포트 충돌은 나중에 고민해도 되며, 지금은 테스트의 구조에 집중하는 것이 더 중요
- `@Disabled` 어노테이션의 한계
    - 테스트가 많아질수록 해당 어노테이션을 일일이 관리하는 것이 어렵고 실수가 생길 여지가 존재
    - IDE에서 전체 테스트가 성공하더라도, 사실 중요한 테스트가 비활성화되어 누락된 상태일 가능성 존재
- 테스트 실행 환경 문제
    - 테스트가 실행되는 환경에 로컬 서버가 돌아가지 않을 수 있음
        - 테스트 코드가 외부 리소스(DB, 내부 API)에 의존할 경우, **해당 리소스가 없으면 테스트 자체가 실패**
    - Spring의 `@Profile`이나 `application-{profile}.yml` 같은 기능을 사용해 문제 해결
        - 테스트 전용 환경을 만들고, 테스트 실행 시에만 특정 config나 Bean을 등록하게 해서 문제 해결
- Port 충돌 가능성
    - 포트 충돌은 로컬 환경에서 여러 개발 서버나 테스트 서버가 동시에 돌아갈 경우 생기는 문제
    - 지금은 테스트의 구조를 잡는 게 우선이고, 아직 Port 충돌 문제까지 고민할 단계X

---

## **RestAssuredMockMvc 도입**

- 비즈니스 로직(예약 추가/삭제 등, 일반적으로 서비스 계층에서 실행할 로직)을 Controller에 두었기 때문에 비즈니스 로직을 테스트하기 위해서는 Controller를 통해 테스트 필요
- 주어진 예제 코드는 RestAssured를 사용해 테스트를 진행했으나 실제 http 요청을 보내는 RestAssured 대신 RestAssuredMockMvc를 사용하도록 수정
- 현재는 Controller에 비즈니스 로직이 있는 경우 웹 의존성 때문에 테스트 단위가 커지므로 추후에 컨틀롤러  계층과 서비스 계층으로 분리하고, JUnit을 사용하는 작은 단위 테스트로 분리할 예정

```java
@DisplayNameGeneration(DisplayNameGenerator.ReplaceUnderscores.class)
@WebMvcTest(controllers = {ReservationController.class, AdminController.class})
public class MissionStepTest {

    @Autowired
    private WebApplicationContext context;

    @BeforeEach
    void setup() {
        RestAssuredMockMvc.webAppContextSetup(context);
    }

    @Test
    void 일단계_메인페이지에_접근가능하다() {
        given()
                .when().get("/")
                .then()
                .statusCode(HttpStatus.OK.value());
    }
}
```