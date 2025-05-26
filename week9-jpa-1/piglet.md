## N+1 문제

---

### N+1 문제란?

특정 객체를 대상으로 수행한 쿼리가 해당 객체의 연관관계까지 조회하면서 추가적인 쿼리가 N번 더 발생하는 현상을 말한다. 여기서 N은 처음 조회한 엔티티의 개수이다.

기본적으로 JPA에서 N+1 문제는 다음과 같은 상황에서 발생한다.

- 연관 관계가 설정된 엔티티를 조회할 때
- 조회된 데이터 개수(N)만큼 연관관계의 조회 쿼리가 추가로 발생
- 결과적으로 1(첫 쿼리) + N(추가 쿼리)개의 쿼리가 실행됨

예를 들어, 회원(Member)과 팀(Team)이 다대일(N:1) 관계를 맺고 있을 때, 5명의 회원을 조회하면 처음 회원을 조회하는 쿼리 1번과 각 회원이 속한 팀을 조회하는 쿼리 5번, 총 6번의 쿼리가 실행되는 현상이다.

JPA에서 연관관계가 있는 엔티티를 조회할 때, 즉시 로딩(EAGER)과 지연 로딩(LAZY) 방식이 있다.

fetch type에 따라 발생 안할 수도 있나? → X : 두 방식 모두 N+1 문제가 발생할 수 있다.

**EAGER에서의 N+1**

엔티티를 조회할 때 연관된 엔티티도 함께 조회하는 전략이다.

```java
@Entity
public class User {
    @Id
    @GeneratedValue
    private long id;
    
    private String name;
    
    @ManyToOne(fetch = FetchType.EAGER) // 즉시 로딩
    @JoinColumn(name = "team_id")
    private Team team;
}
```

- `userRepository.findAll()` 호출 시, 먼저 모든 사용자를 조회하는 쿼리를 실행한 후 각 사용자가 속한 팀을 조회하기 위해 N번의 추가 쿼리를 실행한다.

`userRepository.findAll()` 실행 시 N+1 발생한 쿼리

```sql
-- User 전체 조회
SELECT u.id, u.name, u.team_id FROM user u;

-- Team 개별 조회 (team_id 1, 2, 3)
SELECT t.id, t.name FROM team t WHERE t.id = 1;
SELECT t.id, t.name FROM team t WHERE t.id = 2;
SELECT t.id, t.name FROM team t WHERE t.id = 3;
```

**LAZY에서의 N+1**

연관된 엔티티를 실제로 사용할 때까지 조회를 미룬다.

```java
@Entity
public class User {
    @Id
    @GeneratedValue
    private long id;
    
    private String name;
    
    @ManyToOne(fetch = FetchType.LAZY) // 지연 로딩
    @JoinColumn(name = "team_id")
    private Team team;
}
```

- LAZY 전략에서 N+1 문제 발생하는 법

```java
public List<String> getUserTeamNames() {
    List<User> users = userRepository.findAll(); // 1번 쿼리

    return users.stream()
        .map(user -> user.getTeam().getName()) // N번 쿼리 발생 (team에 접근)
        .collect(Collectors.toList());
}
```

- 사용자마다 팀을 조회하는 쿼리가 추가로 발생해서 N+1 문제가 발생한다.

```sql
-- 사용자 전체 조회
SELECT u.id, u.name, u.team_id FROM user u;

SELECT t.id, t.name FROM team t WHERE t.id = 1;
SELECT t.id, t.name FROM team t WHERE t.id = 2;
SELECT t.id, t.name FROM team t WHERE t.id = 3;
```

- 결론 : fetch type은 쿼리 실행 시점에만 차이가 있을 뿐, N+1 문제에는 의미가 없다.

### N+1 문제 예시

```java
@Entity
public class Team {
    @Id @GeneratedValue
    private Long id;
    private String name;
}

@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @ManyToOne(fetch = FetchType.LAZY) // 단방향 관계
    @JoinColumn(name = "team_id")
    private Team team;
}
```

- 모든 Team을 조회한 후, 각 팀 별 Member 목록을 조회할 때

```java
public List<String> findAllMemberWithTeamNames() {
    List<Member> members = memberRepository.findAll(); // 쿼리 1번: 모든 Member 조회

    return members.stream()
        .map(member -> {
            // 여기가 N번: 각 Member의 team 접근 시 팀 조회
            return member.getTeam().getName();
        })
        .collect(Collectors.toList());
}
```

- 실행 결과

```sql
-- 쿼리 1: Member 전체 조회
SELECT m.id, m.name, m.team_id FROM member m;

-- 쿼리 2~N: 각 Member의 team 조회 (LAZY 이므로 접근 시마다 실행)
SELECT t.id, t.name FROM team t WHERE t.id = 1;
SELECT t.id, t.name FROM team t WHERE t.id = 2;
SELECT t.id, t.name FROM team t WHERE t.id = 3;
...
```

## N+1 문제 해결법

---

### **1. Fetch Join**

JPQL에서 성능 최적화를 위해 제공하는 join 방법이다. 연관된 엔티티를 함께 조회하도록 지정하는 방법이다.

```java
public interface TeamRepository extends JpaRepository<Team, Long> {
    @Query("SELECT t FROM Team t JOIN FETCH t.members")
    List<Team> findAllWithMembers();
}
```

```sql
SELECT t.*, m.* 
FROM team t 
INNER JOIN member m ON t.id = m.team_id;
```

쿼리를 한 번 날려 N+1 문제를 해결할 수 있다.

- 단일 쿼리로 해결 하고자 할 때 사용한다.

장점

- 단일 쿼리로 연관 엔티티 전체를 조회하므로 가장 직관적이다.
- JPQL을 사용해 명시적으로 조인 대상을 지정하면 SQL INNER JOIN으로 변환되어 데이터 정합성을 보장하면서도 최소한의 네트워크 왕복으로 데이터를 가져온다.
- 특히 `@OneToMany` 관계에서 컬렉션 조회 시 발생하는 지연 로딩 문제를 근본적으로 해결할 수 있으며, 영속성 컨텍스트에 모든 엔티티를 한 번에 로드하므로 트랜잭션 내에서의 추가 쿼리가 발생할 일이 없다.

단점

- 페이징 처리가 불가능하다.
    - 컬렉션 fetch join 시 DB에서 모든 레코드를 메모리로 로드한 후 애플리케이션 레벨에서 페이징을 수행하므로, 대량 데이터 처리 시 메모리 오버플로우 위험이 있다.

    ```sql
    ex) team A와 team B의 
    @Query("""
        SELECT t FROM Team t 
        JOIN FETCH t.members 
        ORDER BY t.id DESC
        """)
    Page<Team> findTeamsWithMembers(Pageable pageable);
    
    SELECT t.*, m.* 
    FROM team t 
    INNER JOIN member m ON t.id = m.team_id 
    ORDER BY t.id DESC 
    LIMIT 5; // 다섯 팀에 대한 멤버들도 가져오려던 것이 목적이다.
    
    이렇게 하면 한 팀에 대해 모든 멤버가 안나올 수도 있음
    ex) A팀 3명, B팀 5명 존재할 경우,
    A팀 3명, B팀 2명만 조회 결과로 나옴 (B팀 데이터가 불완전)
    ```

- 동시에 두 개 이상의 컬렉션을 join 할 수 없는 제약이 존재한다.
- 쿼리 결과에 중복 데이터가 발생하므로 `DISTINCT` 키워드가 필수 사용 되어야 하며, 이는 추가 CPU 연산 부하를 유발한다.

### **2. @EntityGraph**

Spring Data JPA에서 제공하는 애노테이션으로, fetch join을 수행하는 애노테이션이다.

```java
public interface TeamRepository extends JpaRepository<Team, Long> {
    @EntityGraph(attributePaths = {"members"})
    List<Team> findAll();
}
```

```sql
SELECT t.*, m.* 
FROM team t 
LEFT JOIN member m ON t.id = m.team_id;
```

직전의 fetch join과 동일한 결과를 얻을 수 있다.

- Spring Data JPA에서 간편하게 fetch join을 적용할 때 사용한다.

장점

- Spring Data JPA의 선언적 인터페이스와 자연스럽게 통합되는 설정 중심의 접근 방식이다.
- JPQL 작성을 필요로 하지 않고 메서드 이름 쿼리와 결합해 사용할 수 있어 개발 생산성이 뛰어나다.
- LEFT OUTER JOIN을 기본 전략으로 사용하므로 주 엔티티의 존재 여부와 무관하게 결과를 보장하며, 동적 프로젝션과 결합해 필요한 속성만 선택적으로 로딩할 수 있는 유연성을 제공한다.

단점

- 암묵적 LEFT JOIN 사용으로 불필요한 데이터 조회가 발생할 수 있다.
- fetch join처럼 컬렉션 페이징의 한계가 있다.
- 복잡한 join 조건이나 커스텀 필터링을 적용하기 어려운 구조이다.
- 여러 컬렉션을 동시 로딩할 때 제한 사항이 동일하고, 1:N:M 관계의 다중 조인 시 예상치 못한 데이터 증식 문제가 발생할 수 있다.

### **3. @BatchSize**

Hibernate에서 제공하는 기능으로, 지정된 크기만큼 데이터를 일괄 조회 해 N+1 문제를 완화한다.

```java
@Entity
public class Team {
    @Id @GeneratedValue
    private Long id;
    private String name;
    
    @BatchSize(size = 100)
    @OneToMany(mappedBy = "team", fetch = FetchType.LAZY)
    private List<Member> members = new ArrayList<>();
}
```

이렇게 하면 Team에 접근한 후 회원을 조회할 때 아래같은 IN 쿼리가 발생한다.

```sql
SELECT * FROM member WHERE team_id IN (?, ?, ?, ...);
```

N개의 쿼리 대신 (전체 데이터 수 / 배치 사이즈) 만큼의 쿼리가 발생한다.

- 여러 컬렉션을 효율적으로 로딩하거나 페이징과 함께 사용할 때 효과적이다.

장점

- 점진적 로딩 전략을 통해 시스템 자원 사용을 최적화한다.
- 페이징 처리와 완벽 호환되고, 여러 컬렉션을 동시 처리할 수 있는 방법이다.

    ```java
    @Query("SELECT t FROM Team t")
    Page<Team> findAllTeams(Pageable pageable);
    
    SELECT * FROM team LIMIT 5;
    SELECT * FROM member WHERE team_id IN (team_id_1, team_id_2, ..., team_id_5);
    
    ex) 5팀에 대해 페이징 적용
    5팀에 대한 모든 멤버들을 조회해올 수 있다. 
    ```

- Hibernate의 글로벌 설정으로 전체 애플리케이션에 일관된 전략 이용이 가능하다.

    ```yaml
    # application.yml
    spring:
      jpa:
        properties:
          hibernate:
            default_batch_fetch_size: 100
    ```


단점

- 여러 번의 쿼리 실행으로 소규모 데이터셋에서는 단일 fetch join보다 성능이 저하될 수 있다.
- 적절한 batch 크기 설정이 필수이고, 이를 위해 실제 데이터 분포 분석과 성능 테스트가 선행 되어야 한다.
- 지연 로딩 프록시 객체가 활성화될 때마다 배치 쿼리가 실행되므로 트랜잭션 경계 관리에 주의가 필요하다.

종합 비교

|  | fetch join | @EntityGraph | @BatchSize |
| --- | --- | --- | --- |
| 쿼리 실행 횟수 | 1 | 1 | N / 배치 크기 |
| join 유형 | INNER JOIN | LEFT OUTER JOIN | IN절 |
| 페이징 지원 | 제한 | 제한 | 지원 |
| 다중 컬렉션 처리 | X | X | O |
| 메모리 사용량 | 높음 | 높음 | 낮음 |
| 네트워크 부하 | 낮음 | 낮음 | 중간 |
| 복잡도 | 높음 | 중간 | 낮음 |

---

### 상황 별 N+1 해결법

1. 소규모 데이터 전체 조회
    - fetch join, @EntityGraph
2. 대규모 데이터 페이징
    - @BatchSize 적용으로 메모리 사용량 제어하며 점진 로딩
3. 다중 컬렉션 관계
    - @BatchSize와 fetch join 조합해 핵심 컬렉션은 join, 나머지는 batch 로딩
4. 동적 프로젝션이 필요할 경우
    - @EntityGraph를 활용해 필요한 속성만 선택적으로 로딩
5. 데이터 정합성 강조
    - fetch join의 INNER JOIN으로 NULL 데이터 방지

---

보통은 한 가지 방법만 쓰지 않는다. 조회 시 시스템 데이터 특성과 비즈니스 요구사항에 맞는 하이브리드 접근법으로 개발해야 한다.
