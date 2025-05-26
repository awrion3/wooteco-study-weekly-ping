# Transaction

## 트랜잭션(Transaction)

- DB의 상태를 변경시키는 작업의 단위
- 트랜잭션은 DB 작업을 일관성 있게 처리하기 위한 ACID 속성을 갖음
    - Atomicity(원자성), Consistency(일관성), Isolation(고립성), Durability(지속성)

---

## ACID 속성

- **Atomicity (원자성)**
    - 모두 성공하거나, 모두 실패 (부분 성공 없음)
    - @Transactional 메서드 안에서 오류가 발생하면 전체 작업을 롤백
- **Consistency (일관성)**
    - 트랜잭션 전후에 데이터 무결성 보장
    - @Transactional은 실패 시 롤백해서 불일치 상태로 남지 않도록 보장
- **Isolation (고립성)**
    - 여러 트랜잭션이 동시에 실행될 때, 서로의 영향을 받지 않게 DB가 격리 수준을 보장
    - @Transactional은 DB의 기본 격리 수준(READ_COMMITTED)을 사용하지만, 필요하면 직접 지정 가능
- **Durability (지속성)**
    - @Transactional이 커밋하면, JDBC가 DB에 COMMIT 명령을 전송
    - DB는 해당 변경을 디스크에 기록해 전원 꺼져도 복구 가능하게 유지

---

## 트랜잭션 사용

- Spring의 @Transactional은 내부적으로 DB의 트랜잭션 기능을 사용해서 ACID 속성 보장
- Spring은 단순히 시작/커밋/롤백을 자동화해주는 도구일 뿐, 실제 ACID는 DBMS와 JDBC, JPA가 함께 책임짐
    - @Transactional은 트랜잭션 시작/커밋/롤백 명령 스위치 역할
    - 실제 트랜잭션은 Spring → JPA → JDBC → DBMS가 실행하고 DB가 ACID 보장

### 트랜잭션에서 각 계층별 역할

- **Spring (@Transactional)**
    - 트랜잭션 관리 명령 전달 (시작/커밋/롤백/종료)
    - 개발자 입장에서 선언형으로 @Transactional만 붙이면 됨
    - 내부에서는 AOP 프록시를 통해 트랜잭션을 시작하거나 종료함
    - 실제 DB 작업은 하지 않음
    - Spring은 TransactionManager를 통해 JDBC 트랜잭션에 명령을 내림
- **JPA (Hibernate)**
    - SQL을 생성하고 JDBC를 통해 실행되도록 함
    - 엔티티 상태를 추적하고, 변경사항을 감지해서 SQL을 생성 (flush 시점)
    - 트랜잭션이 commit될 때 엔티티 상태 변경을 SQL로 만들어 JDBC를 통해 전송
- **JDBC (Java Database Connectivity)**
    - DB에게 실제 트랜잭션 명령 전달 (commit, rollback)
    - Java와 DB를 연결하는 저수준 API
    - 트랜잭션 명령 (setAutoCommit(false), commit(), rollback())을 DB에 직접 전달
    - Connection 객체가 트랜잭션을 관리함
- **DBMS (MySQL)**
    - 트랜잭션 격리, 정합성, ACID 속성을 실제로 구현하고 보장하는 주체
    - JDBC로부터 전달받은 SQL 명령을 실행함
    - 트랜잭션 격리 수준, 락, ACID 보장 등 모든 진짜 트랜잭션 제어는 DBMS에서 수행
    - 시스템 장애나 동시성 상황에서도 정합성을 유지하는 최후 보루

### 트랜잭션 계층별 역할 구분

- @Transactional을 붙였다고 해서 ACID가 보장되는 건 아님
- 실제 ACID는 DB 설정(격리 수준 등)에 따라 다르게 보장됨
- @Transactional이 무시되거나 프록시 적용이 안 되면 트랜잭션 자체가 적용되지 않을 수도 있음
- 성능 최적화, 동시성 처리, 롤백 정책을 이해하려면 Spring과 DB의 역할을 구분 필요

---

# Isolation

## 고립 수준 (Isolation Level)

- 동시에 여러 트랜잭션이 실행될 때, 서로의 작업이 보이는지 말지를 제어하는 수준
- Spring에서는 @Transactional을 @Transactional(isolation = Isolation.READ_COMMITTED)과 같이 설정
- 트랜잭션 격리 수준에 따라 3가지 현상 발생
    - Dirty Read, Non-repeatable Read, Phantom Read
- 고립 수준은 ANSI SQL 표준에 따라 동시성 문제를 처리하는 4가지 방식 존재
    - READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE

---

## Isolation Level에 따른 현상

- **Dirty Read**
    - 다른 트랜잭션이 아직 커밋하지 않은 데이터를 읽을 수 있는 현상
    - READ_UNCOMMITTED일 때만 발생
    - 격리 수준이 READ_COMMITED 이상이면 방지 가능
- **Non-repeatable Read (Fuzzy Read)**
    - 같은 쿼리를 두 번 조회 했을 때 중간에 값이 바뀌는 현상
    - READ_UNCOMMITTED, READ_COMMITTED에서 발생
    - 격리 수준이 REPEATABLE_READ 이상이면 방지 가능
- **Phantom Read**
    - 같은 조건의 쿼리를 두 번 조회 했는데 레코드 개수가 달라지는 현상
    - READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ에서 발생
    - 격리 수준이 SERIALIZABLE일 때만 방지 가능

---

## Isolation Level 종류

- **READ_UNCOMMITTED**
    - 가장 낮은 고립 수준
    - 커밋되지 않은 데이터를 읽을 수 있음 (Dirty Read)
    - 불안정하기 때문에 자주 사용X
- **READ_COMMITTED (Spring의 기본값)**
    - 다른 트랜잭션이 커밋한 데이터만 읽을 수 있음
    - Dirty Read는 방지되지만, 같은 쿼리를 두 번 조회 했을 때 결과가 달라질 수 있음 (Non-repeatable Read)
- **REPEATABLE_READ**
    - 트랜잭션 동안 읽은 데이터는 계속 동일함
    - Dirty Read와 Non-repeatable Read는 방지되지만, 새로운 데이터 삽입(Phantom Read)은 감지하지 못함
- **SERIALIZABLE**
    - 가장 엄격한 수준
    - 모든 트랜잭션을 순차적으로 실행하는 것처럼 동작
    - Phantom Read까지 방지 가능
    - 대신 락 많이 걸리고 성능 저하 가능성 있음

---

## Isolation Level 사용

| 격리 수준 | Dirty Read | Non-repeatable Read | Phantom Read | 성능 |
| --- | --- | --- | --- | --- |
| READ_UNCOMMITTED | 발생함 | 발생함 | 발생함 | 빠름 |
| READ_COMMITTED | 방지 | 발생함 | 발생함 | 중간 |
| REPEATABLE_READ | 방지 | 방지 | 발생함 | 느림 |
| SERIALIZABLE | 방지 | 방지 | 방지 | 가장 느림 (가장 안전) |
- 실무에서는 READ_COMMITTED로 충분한 경우가 많음
- 거래 처리 시스템, 은행, 재고 등 정확성이 중요한 경우에는 REPEATABLE_READ나 SERIALIZABLE 사용 고려 필요
- SERIALIZABLE은 데드락이나 성능 이슈 발생 가능성 있음 → 꼭 필요할 때만 사용 추천

---

# 동시성 문제 해결 전략

## 비관적 락과 낙관적 락

### 비관적 락 (Pessimistic Lock)

- 충돌이 발생한다고 가정하고 미리 락을 거는 전략
- 데이터를 조회할 때부터 다른 트랜잭션이 접근 못하게 막아 DB 레벨에서 락을 걸어 데이터 변경을 방지
- 락의 종류로 공유 락(Shared Lock), 배타 락(Exclusive Lock)이 존재
    - **공유 락(Shared Lock)**
        - 특정 Row를 읽을(Read) 때 사용되는 Lock
        - 여러 트랜잭션이 동시에 한 Row에 Shared Lock == 하나의 Row를 여러 트랜잭션이 동시에 읽을 수 있음
        - Shared Lock이 설정된 Row에는 Exclusive Lock을 사용 불가능
    - **배타 락(Exclusive Lock)**
        - 특정 Row를 변경(write)할 때 사용되는 Lock
        - 특정 Row에 Exclusive Lock이 걸려있을 경우, 다른 트랜잭션은 읽기 작업을 위해 Shared Lock을 걸거나, 쓰기 작업을 위해 Exclusive Lock 설정 불가능 → 쓰기 작업을 하고 있는 Row에는 모든 접근이 불가
        - SELECT … FOR UPDATE, DELETE 등의 수정 쿼리들이 실행될 때 Row에 락이 걸림

### 낙관적 락 (Optimistic Lock)

- 충돌이 발생하지 않는다고 가정하고 충돌이 발생하면 처리하는 전략
- 대신 저장할 때 충돌이 발생하면 예외 처리하기 위해 DB 락을 사용하지 않고 어플리케이션 레벨에서 락 구현
- JPA에서 버전 관리 기능(@Version)을 통해 구현 가능
    - @Version 어노테이션이 붙은 필드를 포함하는 엔티티를 정의
    - @Version은 해당 엔티티 테이블을 읽는 각 트랜잭션은 업데이트를 수행하기 전에 버전의 속성을 확인

---

## JPA에서 Lock

### **@Lock**

- Data JPA에서 낙관적 락과 비관적 락을 사용하기 위해 Repository 인터페이스에 지정한 커스텀 메서드에 사용
- @Lock의 설정 값인 value에 설정하고자 하는 LockModeType을 지정
    
    ```java
    public interface AccountRepository extends JpaRepository<Account, Long> {
    
        @Lock(value = LockModeType.OPTIMISTIC)
        Optional<Account> findByName(String name);
    }
    ```
    

### LockModeType

- 비관적 락 (**Pessimistic Lock) - DB에 락을 직접 거는 방식**
    - PESSIMISTIC_READ: 공유 락 (읽기만 허용, 쓰기는 차단)
    - PESSIMISTIC_WRITE: 배타 락 (읽기, 쓰기 둘 다 차단)
    - PESSIMISTIC_FORCE_INCREMENT: 배타 락 + 버전 증가
- **낙관적 락 (Optimistic Lock) - 충돌을 허용하고 저장 시 감지**
    - NONE: 기본값, 저장 시 충돌 검증만 수행
    - OPTIMISTIC: 조회 시점부터 다른 트랜잭션 수정 금지
    - OPTIMISTIC_FORCE_INCREMENT: 읽기만 해도 버전 증가시킴, 논리적 업데이트 표현

---

# 테스트에서 @Transactional

## 테스트 코드에 @Transactional를 붙이면

- 테스트 레이어에서 생성된 트랜잭션이 프로덕션 레이어에 전파
- 테스트 레이어와 프로덕션 레이어가 같은 트랜잭션 공유 가능

## 테스트 코드에 @Transactional를 붙이는 장점

- 테스트 통과/실패 시 트랜잭션 롤백 가능
- JPA 영속성 컨텍스트의 범위를 테스트 레이어까지 확장 가능
    - 지연 로딩 전략으로 설정된 연관관계 엔티티들을 테스트 코드에서 조회 가능

## 테스트 코드에 @Transactional를 붙여생기는 문제

### 실제 코드에 트랜잭션이 없음

- LazyInitializationException 발생
    - 지연 로딩으로 인해 만들어진 프록시 객체를 트랜잭션 범위 밖에서 조회하면 발생하는 에외
- 테스트 코드에서는 테스트 레이어의 @Transactional을 사용해 프로덕션 레이어는 Transactional 프록시 적용X
    - 트랜잭션이 적용되지 않은 프로덕션 코드의 문제는 @Transactional이 적용된 테스트 코드로 사전 감지 불가능
- 프로덕션과 테스트 코드의 @Transcational 활성화 옵션을 다르게 설정하는 실수로 버그 발생 가능

### 테스트할 쿼리가 실행되지 않음

- JPA에서 지원하는 변경 감지(Dirty Checking)가 SQL로 나타나지 않음
- 예를 들어 회원을 저장하고, 회원 이름을 수정하는 코드가 있다면
    - INSERT문 다음 UPDATE문이 나가는 것을 예상했지만
    - 테스트 코드에서는 회원 저장시 INSERT문만 나감
- 이유
    - 서비스 레이어에 적용된 트랜잭션 작업은 작업이 끝날 때 트랜잭션을 커밋하는 것이 기본
        - 커밋(Commit)하는 경우 1차 캐시에 대한 SQL을 전송해 DB에 반영
            
            ```java
            @Transactional
            public void updateMember(Long id, String newName) {
                Member member = memberRepository.findById(id).orElseThrow();
                member.setName(newName); // 필드 값 변경 (Dirty Checking 대상)
                // 트랜잭션 종료 시점 → flush → commit → UPDATE SQL 발생
            }
            ```
            
    - 테스트 레이어에 적용된 트랜잭션 작업은 각 테스트가 끝날 때 트랜잭션을 롤백하는 것이 기본
        - 롤백(Rollback)히는 경우 1차 캐시에 이전의 상태로 되돌려 캐시 내용 파기
            
            ```java
            @SpringBootTest
            @Transactional
            public class MemberServiceTest {
            
                @Autowired
                EntityManager em;
            
                @Test
                void testDirtyChecking() {
                    Member member = new Member("River");
                    em.persist(member);
            
                    member.setName("UpdatedRiver");
                    // 테스트 끝나면 자동으로 rollback
                    // → flush도, commit도 일어나지 않음 → UPDATE 쿼리도 안 나감
                }
            }
            ```