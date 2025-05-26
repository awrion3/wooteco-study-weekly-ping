## @DynamicUpdate

**Reservation entity**

```java
@Entity
public class Reservation{
	private ReservationStatus status;
	
	...
	
	public void updateStatus(ReservationStatus status){
		this.status = status;
	}
}
```

**Dirty checking으로 예약 상태를 업데이트**

```java
@Transcational
public void updateStatus(Long reservationId){
	Reservation reservation = reservationRepository.findById(reservationId)
			.orElseThrow(() -> new InvalidReservationException());
			
	reservation.updateStatus(ReservationStatus.CONFIRMED);
}
```

**발생하는 쿼리**

```sql
Hibernate: 
    update
        reservation 
    set
        created_at=?,
        date=?,
        member_id=?,
        status=?,
        theme_id=?,
        time_id=? 
    where
        id=?
```

데이터베이스에 이렇게 불필요하게 모든 데이터 컬럼을 포함해서 전달하면 데이터 전송량이 증가한다는 단점이 존재한다.

**Q. 왜 모든 컬럼을 업데이트할까?**

JPA가 이런 방식을 사용하는 이유는 성능 최적화 때문이다. JDBC의 `PreparedStatement`는 SQL 구문을 캐시하고 파라미터 부분만 변경하면서 재사용하는데, 항상 동일한 SQL 구조를 사용하면 SQL 캐시 히트율이 높아진다. 또한 엔티티 객체의 변경 여부만 추적하면 되므로 필드 수준의 복잡한 추적이 불필요하다.

`@DynamicUpdate`는 Hibernate에서 제공하는 클래스 수준의 어노테이션으로, 엔티티의 변경된 컬럼만 포함하는 동적 업데이트 SQL을 생성하도록 지시한다. JPA 표준 스펙이 아닌 Hibernate만의 확장 기능이다.

```java
@Entity
@DynamicUpdate
public class Reservation{
   ...
}
```

**`@DynamicUpdate`를 사용한 후**

```sql
Hibernate: 
    update
        reservation 
    set
        status=? 
    where
        id=?
```

**Q. @DynamicUpdate는 쿼리를 캐싱해둘 수 없나?**

`@DynamicUpdate` 자체가 동적 SQL을 생성한다. 따라서 엔티티의 변경된 컬럼만 포함하는 SQL문이 매번 새롭게 생성되기 때문에 캐싱 메커니즘을 사용할 수 없다.

### 그럼 언제 @DynamicUpdate를 쓰는게 좋을까?

**@DynamicUpdate를 사용하는 것이 좋은 경우**

1. 컬럼 수가 많은 대형 테이블

   엔티티가 15~20개 이상의 컬럼을 가지고 있고, 그 중 일부만 자주 업데이트되는 경우에 효과적이다. 특히 텍스트나 바이너리 데이터 같은 대용량 컬럼이 있는 경우 더욱 유용하다.
2. 빈번한 부분 업데이트가 발생 하는 경우

   좋아요 수, 조회수, 상태 변경 등 특정 필드만 자주 업데이트되는 시나리오에서 성능 향상이 뚜렷하다.

**사용 하지 않는 것이 좋은 경우**

1. 컬럼 수가 적은 간단한 테이블

   5개 미만의 컬럼을 가진 테이블에서는 오히려 성능 오버헤드만 발생할 수 있다.
2. 대부분의 컬럼이 함께 업데이트되는 경우

   엔티티의 대부분 필드가 동시에 변경되는 경우에는 기본 동작이 더 효율적이다.

결론적으로 @DynamicUpdate 사용 여부는 컬럼 개수, 업데이트 패턴, 사양 등을 종합적으로 고려해야 한다. 15개 이상 컬럼과 빈번한 부분 업데이트가 있는 시스템에서는 35% 이상의 성능 향상을 기대할 수 있으나, 소규모 테이블에서는 오히려 20% 성능 저하가 발생할 수 있다.
최신 데이터베이스의 컬럼 단위 잠금 기능과 부분 업데이트 최적화를 함께 적용할 때 효과적이다.

**`실무에서도 @DynamicUpdate를 자주 사용하나요?`에 대한 킹영한의 답변**

> 흥미롭게도 JPA는 애플리케이션 로딩 시점에 PrepareStatement 스타일로 해당 엔티티의 UPDATE 쿼리를 만들어 둡니다. 그래서 성능만 생각하면 컬럼에 값을 다 넘기는 것이나 하나만 동적으로 만들어서 넘기는 것이나 크게 차이가 나지 않습니다. 상황에 따라 다르겠지만, 오히려 전체 컬럼을 하나의 PrepareStatement 스타일의 SQL을 반복해서 사용하는게, 더 속도가 빠를 수도 있습니다. 물론 컬럼이 너무 많거나, 길이가 너무 길거나, 데이터가 너무 크다면 상황이 달라질 수 있습니다. 하지만 그만큼 크다면 이미 테이블을 분할했겠지요.    
> 여기서 핵심은 이런 단순하게 엔티티 하나를 조회해서 사용하는 핵심 비즈니스 로직들은 전체 애플리케이션 성능에 크게 영향을 주지 않습니다. 대부분 단순 쿼리들이니까요^^ 성능 때문에 우리를 고통스럽게 하는 것은, 잘 인덱스 처리를 해야 하는 복잡한 쿼리들이지요.

## findById() vs getReferenceById()

### **findById()**
```java
Optional<Reservation> reservation = reservationRepository.findById(id);
```
구현 내용
```java
public Optional<T> findById(ID id) {
    return Optional.ofNullable(entityManager.find(entityClass, id));
}
```
EntityManager의 `find()` 메서드를 래핑한 `findById()`는 즉시 로딩(Eager Loading)을 구현한다. 메서드 호출 시점에 즉시 데이터베이스에 SELECT 쿼리를 전송하며, 이 과정에서 영속성 컨텍스트의 1차 캐시 조회 → 데이터베이스 조회 → 엔티티 생성 순서를 따른다. 반환 타입인 `Optional<T>`는 null 안전성을 보장하며, 엔티티 존재 여부를 호출자에게 명시적으로 알린다.

### **getReferenceById()**
```java
Reservation reservation = reservationRepository.getReferenceById(id);
```
구현 내용
```java
public T getReferenceById(ID id) {
    return entityManager.getReference(entityClass, id);
}
```
`EntityManager.getReference()`를 기반으로 하는 `getReferenceById()`는 지연 로딩(Lazy Loading)을 구현한다. 이 메서드는 실제 엔티티 인스턴스 대신 Hibernate의 라이브러리가 생성한 프록시 객체를 즉시 반환한다. 프록시는 실제 데이터 접근이 발생하는 시점(보통 getter 메서드 호출 시)에 초기화를 트리거한다.
해당 id에 대한 프록시 객체가 존재하지 않으면 `EntityNotFoundException`를 발생시킨다. 

```java
// findById() 사용 시
for (Long id : ids) {
    Entity e = repository.findById(id).get(); // 1000회 SELECT 즉시 실행
    process(e);
}

// getReferenceById() 사용 시
List<Entity> proxies = ids.stream()
                         .map(repository::getReferenceById)
                         .toList(); // 0회 SELECT
proxies.forEach(proxy -> {
    proxy.getField(); // 사용 시점에 1000회 SELECT
});
```

### 언제 무엇을 써야할까?
엔티티를 외래키로 등록할 때 해당 키만 필요할 경우 (필드가 필요하지 않은 경우)

ex) 특정 Article에 대한 댓글을 생성할 때

`findById()`를 사용하면 Article의 모든 필드를 가져와버린다.
```java
// 비효율적 방식
Article article = articleRepository.findById(articleId).get(); // Article의 모든 필드를 가져옴
Comment comment = new Comment(content, article);
```

```java
// 최적화 방식
Article articleProxy = articleRepository.getReferenceById(articleId); // 프록시 객체만으로도 Comment를 등록할 수 있음
Comment comment = new Comment(content, articleProxy);
```
하지만 `getReferenceById()`는 해당 id 값이 실제 유효한 값인지 확인하고서 객체를 만드는게 아니다. 정말 껍데기만 만들어 두는 것이다. 따라서 유효하지 않은 id 값으로 `getReferenceById()`를 호출해도 실패하지 않는다. 
```java
public void testGetReferenceById(Long id){
    Reservation reservation = reservationRepository.getReferenceById(id); // 예외 발생 X
    System.out.println(reservation.getStatus()); // EntityNotFoundException 예외 발생
}

jakarta.persistence.EntityNotFoundException: Unable to find roomescape.reservation.domain.Reservation with id 400
```

따라서 대부분의 경우는 `findById()`를 사용해, 해당 객체가 존재하는 경우와 아닌 경우를 유동적으로 코드 레벨에서 관리하는 것이 좋다.


1. CRUD 상세 조회: `findById()`
2. 외래키 설정: `getReferenceById()`
3. 존재 여부 불확실: `findById()`
