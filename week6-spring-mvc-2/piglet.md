## 1. DTO 변환의 책임은 어느 계층?

---

콘솔 view와 REST API에서 바라는 응답의 형태가 다르면 service 계층에서 DTO를 반환하는게 맞는가?

원래  생각

controller는 오로지 요청/응답을 전달 하는 역할을 해야 한다고 생각 했다. 그래서 엔티티의 값을 알면 안된다고 생각해서 Service 계층의 메서드 리턴 값을 DTO로 반환 했었다.

controller에서 변환해도 되나?

콘솔, 웹을 모두 지원하기 위해 Service 계층을 재사용하게 되었다. 만약 Service 계층에서 DTO를 변환 하고 넘기게 되면, Console과 Web의 응답 형태를 통일 해야한다. 만약 둘의 응답 형태가 다르게 하고싶다면 Service에서 entity 객체를 넘겨주고 controller에서 각자 원하는 형태의 DTO로 변환해서 사용할 수 있지 않을까 싶은 생각이 들기도 한다.

**“Service에서 엔티티를 반환하고, 각 Controller에서 dto를 변환 하되, 변환 로직 자체는 Mapper 같은 클래스를 만들어 분리 하면 절충안이 될 것 같아요.”**

⇒ ‘DTO의 역할 중 하나가 어느정도 데이터를 캡슐화하는 것 아닌가?’ : 데이터 테이블의 모든 내용을 바깥에서 알지 못하게 함

or ‘object Mapper에 변환 책임을 넘긴 것이니까 이 것도 데이터 캡슐화가 깨진건 아니라고 볼 수 있는건가?’

- Controller ↔ Service를 위한 DTO : **`Command 객체`**
    - 단점 : 똑같은 데이터에 대해 객체를 또 만들어야 한다

- DTO ← `Controller` ↔ Command 객체 ↔ `Service`

## 2. DATE, TIME vs VARCHAR

---

- reservation_time

```sql
CREATE TABLE reservation_time
(
    id   BIGINT       NOT NULL AUTO_INCREMENT,
    start_at VARCHAR(255) NOT NULL, // TIME
    PRIMARY KEY (id)
);
```

- reservation

```sql
CREATE TABLE reservation
(
    id   BIGINT       NOT NULL AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    date VARCHAR(255) NOT NULL, // DATE
    time_id BIGINT,                           
    PRIMARY KEY (id),
    FOREIGN KEY (time_id) REFERENCES reservation_time (id)
);
```

왜 VARCHAR(255)를 줬을까? → 답을 찾지 못했다

### DATE, TIME 쓸 때의 장점

1. DATE, TIME 타입으로 넣어두면, 나중에 꺼내 쓸 때 다양한 시간 형태로 가공해 쓰기 편하다.
2. 올바른 날짜/시간 형태로 삽입 됐는지 DB insert 차원에서 2차 검증이 가능하다.
3. DBMS에서 제공하는 날짜/시간 관련 연산을 수행 하기 편리하다.

### 단점

1. ~~데이터를 삽입할 때 올바른 시간 형태가 맞을지 유효성 검증 책임을 온전히 애플리케이션에서 갖고 있어야 한다. ?~~

### VARCHAR(255)로 쓸 때의 장점

1. 저장 형태가 유연해진다.

### 단점

1. 추후 저장하려는 날짜/시간 데이터 형태가 변경 되면 데이터 정합성 문제가 발생한다.

   ( ex : 누구는 HH:mm:ss, 누구는 HH:mm, yyyy.MM.dd와 yyyy-MM-dd 등 )


## 3. 의존 데이터의 객체 갖기 vs id만 갖기

---

뭔 차이이고 왜 객체를 갖는 식으로 만들어야 했을까?

jpa를 사용 하지 않는 지금 시점에선 id만 갖는게 더 나을 수 있지 않나?

id만 갖고 있으면 추가 조회 쿼리를 날려야 하니까 문제인가?

### 1. 의존 entity의 객체 갖기

```java
public class Reservation {
    private Long id;
    private final String name;
    private final LocalDate date;
    private final ReservationTime time;
    ...
}
```

**장점**

- 객체지향적인 코드가 된다.
- 한 번에 time 정보를 조회 해올 수 있다.

**단점**

- 불필요하게 객체를 가져오게 될 수도 있다.

### 2. 의존 entity의 id 갖기

```java
public class Reservation {
    private Long id;
    private final String name;
    private final LocalDate date;
    private final Long timeId;
    ...
}
```

**장점**

- 모르겠다.

단점

- time과 관련된 필드 정보를 조회하고 싶으면 Time 조회 쿼리를 한 번 더 써야한다.
    - 2번의 쿼리가 날라간다.

## 4. Spring Bean은 Console Application에 필요할까?

---

10단계 미션 진행하면서 Console Application을 구현했는데, Spring Bean으로 관리했다. Console Application도 Spring Context를 사용해서 bean 관리를 해야할까?

Spring bean을 왜 써야하나?

- Spring의 핵심 가치를 IoC/DI라고 하면 필요할 만 하다.

  ⇒ 변경의 유연함에 대처하기 위함

- Spring이 제공하는 `@Configuration` 같은 걸 사용하면 빈의 싱글톤 관리 등 객체 관리가 쉬워진다.
- 콘솔/웹을 동시에 띄우고 같은 데이터베이스를 공유한다면 DI가 훨씬 쉽다.
