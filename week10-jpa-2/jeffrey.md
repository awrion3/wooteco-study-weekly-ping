# @Transactional(readOnly=true) 무조건 사용해야할까?

Status: Done

# Transaction

**A transaction is an atomic operation that consists of one or more statements.**

**Another critical thing to understand is that all statements in the InnoDB engine become a transaction, if not explicitly, then implicitly.**

**트랜잭션은 하나 이상의 명령문으로 구성된 원자적 연산입니다.**

원자적? 연산 내에 모든 명령문이 성공(커밋) 하거나, 실패(롤백) 하는것.

모든것을 하거나, 하지 않는것을 의미

**또 다른 중요한 점은 InnoDB 엔진의 모든 명령문이 명시적으로는 아니더라도 암묵적으로 트랜잭션이 된다는 것입니다.**

InnoDB의 모든 명령문은 트랜잭션이므로 커밋되거나 롤백될 수 있음.

## Why and Where to Use?

InnoDB 엔진에서의 모든 명령문은 트랜잭션이므로, 락이나 스냅샷과 같은 것들이 포함될 수 있다.

단순 조회 쿼리에서는, 트랜잭션의 Id를 부여하는 작업 등은 필요하지 않다.

또, 자체 DB 내부에서 읽기 전용  트랜잭션 최적화 를 적용하기도 한다.

## Application vs Database

![image.png](/images/week10-jpa-2/image.png)

- [**DAO](https://www.baeldung.com/java-dao-pattern):** Acts as a bridge between business logic and persistence nuances
- Transactional abstraction: Takes care of the application level complexity of transactions (Begin, Commit, Rollback)
- [**JPA Abstraction**](https://www.baeldung.com/the-persistence-layer-with-spring-data-jpa): Java specification that offers a standard API between vendors
- ORM Framework: The actual implementation behind JPA (for example, Hibernate)
- [**JDBC**](https://www.baeldung.com/java-jdbc): Responsible for actually communicating with the database

### Transactional Interface

```java
	/**
	 * A boolean flag that can be set to {@code true} if the transaction is
	 * effectively read-only, allowing for corresponding optimizations at runtime.
	 * <p>Defaults to {@code false}.
	 * <p>This just serves as a hint for the actual transaction subsystem;
	 * it will <i>not necessarily</i> cause failure of write access attempts.
	 * A transaction manager which cannot interpret the read-only hint will
	 * <i>not</i> throw an exception when asked for a read-only transaction
	 * but rather silently ignore the hint.
	 * @see org.springframework.transaction.interceptor.TransactionAttribute#isReadOnly()
	 * @see org.springframework.transaction.support.TransactionSynchronizationManager#isCurrentTransactionReadOnly()
	 */
	boolean readOnly() default false;
```

트랜잭션이 읽기 작업만 수행된다는 힌트를 주는 boolean값

엄격한 제한을 걸어주는것이 아니라, 힌트이기 때문에 readOnly=true 로 했다고 해서 쓰기작업 시 무조건 에러가 나는 것은 아니다. 해당 힌트를 해석할 수 없는 트랜잭션매니저는 읽기전용 트랜잭션을 요청받았을 때 예외를 던지는 것이 아니라 무시한다.

## [**데이터 접근에 대한 원자성 보장을 위한 JPA Transaction 역할**](https://tech.kakaopay.com/post/jpa-transactional-bri/#%EB%8D%B0%EC%9D%B4%ED%84%B0-%EC%A0%91%EA%B7%BC%EC%97%90-%EB%8C%80%ED%95%9C-%EC%9B%90%EC%9E%90%EC%84%B1-%EB%B3%B4%EC%9E%A5%EC%9D%84-%EC%9C%84%ED%95%9C-jpa-transaction-%EC%97%AD%ED%95%A0)

그렇다면 언제 이 원자성이 보장되어야 할까요?

바로 여러 데이터에 대한 update가 필요할 때입니다.

이 외의 경우에는 필요가 없는데요 다음 3가지가 필요 없는 경우에 해당됩니다.

1. 조회만 필요한 경우, Transactional이 필요하지 않습니다.
    - 이미 영구적인 데이터를 조회만 하고 있기 때문에, Transaction이 보장해 주는 ACID의 기능 중 Durability, 영구 적용성이 필요가 없습니다.
    - DB에서 제공해 주는 일관된 읽기를 위한다면 application 입장에서의 단일 스레드 안에서 같은 data 요청을 여러 번 하는 것이 유지보수성이 높아지는지, 성능적 이점이 있는지 고민해 볼 필요가 있습니다.
2. 하나의 row만 update 할 경우, Transactional이 필요하지 않습니다.
    - 이미 하나의 데이터에 관한 update는 DB에서 원자성을 보장해주기 때문에 Transactional이 필요하지 않습니다.
3. 동시성 제어만 필요한 경우, 다른 방법을 고려할 수 있습니다.
    - 단일 select update와 같은 경우에 동시성 제어가 필요한 경우에는 Transactional 만으로는 완벽한 제어가 불가능한 경우가 있습니다. (mysql의 경우 [phantom read](https://dev.mysql.com/doc/refman/8.4/en/innodb-next-key-locking.html))
        - Transactional isolation을 serializable 모드로 강하게 한다던지 (이 경우 성능 영향)
        - optimistic lock을 따로 적용을 한다던지 추가적인 작업이 필요합니다.
        - 그렇게 까지 진행할 작업이 아니라면 비즈니스-도메인 구조에 대해서 다시 생각해 볼 수 있습니다.
            - 단일 key에 대한 update가 동시적으로 들어올 수 있는 비즈니스 구조 자체가 문제가 있는 건 아닌지 생각해 볼 수 있습니다.
    - redis 등을 활용한 서비스 api 요청 단위에서의 중복 요청 제어 등을 고려해 볼 수 있습니다.

위 3가지 경우를 제외하고 나면 JPA 입장에서 여러 데이터를 save 하게 되는 경우로 귀결됩니다.


## [**`@Transactional(readOnly=true)` 동작에서 DB로의 쿼리 전파**](https://tech.kakaopay.com/post/jpa-transactional-bri/#transactionalreadonlytrue-%EB%8F%99%EC%9E%91%EC%97%90%EC%84%9C-db%EB%A1%9C%EC%9D%98-%EC%BF%BC%EB%A6%AC-%EC%A0%84%ED%8C%8C)

`@Transactional(readOnly=true)` 사용 시 spring 진영에서는 애플리케이션 입장에서 dirtyChecking을 진행하지 않기 때문에 성능 개선의 측면이 있다고 이야기합니다. 또한 몇몇 개발자들의 입장에서는 명시적인 효과도 있다고 이야기하고 있습니다.

다만 여기서 의도치 않은 동작이 추가로 진행되는데요.

아래와 같이 `@Transactional(readOnly=true)`를 사용하는 메서드가 있다고 가정해 봅시다.

```kotlin
@Transactional(readOnly = true)
fun getDetail(
    @PathVariable id: Long,
): String = testRepository.findByIdOrNull(id)?.transactionId ?: "nohit"
```

실제 위 메서드를 수행한 뒤 Mysql general log를 살펴보면 다음과 같습니다.

![image.png](/images/week10-jpa-2/image%201.png)

우리가 실제 필요한 쿼리는 5번 row의 `select ~....`인데요. `@Transaction(readOnly = true)` 사용 시 실제 DB 트랜잭션 모드를 readOnly로 설정해 DB 트랜잭션을 사용하게끔 동작합니다. 이를 위해 autoCommit setting, commit 요청, set session transaction 세팅 등으로 6개의 쿼리가 더 날아가고 있네요.

사실 위와 같은 상황이 생기는 게 맞는 것이 `@Transactional(readOnly=true)` 경우에는 propagation이 설정되어 있지 않습니다. default propagation setting은 `REQUIRED`로써 상위 메서드의 transaction이 없을 시에는 만들고, 있을 시에는 참여하여 DB로 실제 Transaction 사용을 요청하는 동작을 수행합니다.

## [`@Transactional(readOnly=true)`, 성능에 좋은 영향을 미치나?](https://tech.kakaopay.com/post/jpa-transactional-bri/#transactionalreadonlytrue-%EC%84%B1%EB%8A%A5%EC%97%90-%EC%A2%8B%EC%9D%80-%EC%98%81%ED%96%A5%EC%9D%84-%EB%AF%B8%EC%B9%98%EB%82%98)

spring 진영에서는 `@Transactional(readOnly=true)`가 dirtyChecking 모드를 Manual모드로 바꿔 줄 수 있기 때문에 application 성능 개선에 도움이 될 수 있다고 설명합니다.

다만 대부분의 DB 입장에서는 read-only 세팅으로 트랜잭션을 열게 되면 성능적 이점이 아주 약간 있거나 없을 것이라 설명하고 있습니다.

이 기능의 주요 목적은 ACID 중 읽기 작업만 수행했을 때의 Isolation 제공을 위한 일관된 읽기 수행 제공에 있기 때문입니다.

단순히 이를 위해 `@Transactional(readOnly=true)`를 사용하는 것은 지불해야 할 비용이 너무 큽니다. DB 입장 에서의 Transaction read-only 세팅 비용이 크지 않다 하더라도, 이 단순한 추가 쿼리로 인해 DB까지의 네트워크 요청 건수 또한 최대 6배까지 늘어나게 됩니다.

실제로 이러한 의문을 가지고 위의 [실제로 set_option과 commit이 성능에 영향을 미칠까?](https://tech.kakaopay.com/post/jpa-transactional-bri/#%EC%8B%A4%EC%A0%9C%EB%A1%9C-set_option%EA%B3%BC-commit%EC%9D%B4-%EC%84%B1%EB%8A%A5%EC%97%90-%EC%98%81%ED%96%A5%EC%9D%84-%EB%AF%B8%EC%B9%A0%EA%B9%8C)에서 성능 테스트를 수행한 건데요. 위에 설명은 안 했지만 코드를 잘 보시면 `@Transactioanl(readOnly=true)` 유무로 성능 비교를 진행했습니다. 그리고 아시다시피 결과는 없는 게 낫다는 결론으로 나왔습니다.

거기에 대부분의 성능 허들은 application이 아닌 DB에 있습니다. scale-out이 쉽지 않기 때문이죠.

이를 위해 보통의 경우 `@Transactional(readOnly=true)`를 활용한 CQRS 패턴 까지도 적용하여 성능의 이점을 꾀하게 됩니다.

다만 저희 서비스의 경우 DB 복제 지연의 영향 또한 막기 위해 실시간 서비스에 한해서 slave DataSource 또한 master DB를 보게 해 DB connection pool만 분리해 둔 상황입니다.

### [**set_option 및 auto commit 옵션**](https://tech.kakaopay.com/post/jpa-transactional-bri/#set_option-%EB%B0%8F-auto-commit-%EC%98%B5%EC%85%98)

추가로 transactional 사용 중 최초 db config 옵션에 auto commit을 빼면 최소한 트랜잭션 수행 시 set autoCommit=1,0에 대한 요청은 줄일 수 있습니다. 다만 이렇게 사용할 경우 [위](https://tech.kakaopay.com/post/jpa-transactional-bri/#1-%EB%8B%A8%EA%B1%B4-%EC%9A%94%EC%B2%AD%EC%97%90-%EB%8C%80%ED%95%B4%EC%84%9C%EB%8A%94-transaction-%EC%A0%9C%EA%B1%B0%ED%95%98%EA%B8%B0) 경우에도 무조건 트랜잭션을 붙여줘야 하기에 해당 방법은 빼기로 했습니다.

게다가 `@Transcational(readOnly=true)`를 사용하게 될 경우 auto commit을 0으로 둔다 하더라도 무조건 commit 쿼리가 따라서 요청되고 set session 변환에 관한 쿼리는 없어지지 않습니다. 따라서 해당 옵션 변경에 관한 노력보다는 Transactional 자체를 줄이는데 집중하기로 했습니다.
