# Specifications

- Specifications는 Spring Data JPA에서 제공하는 동적 쿼리 생성 도구
- JPA 2에서 도입된 Criteria API를 기반으로, Java 코드로 WHERE 조건을 프로그래밍 방식으로 조립 가능
- 메서드 이름이나 JPQL 대신 자바 코드로 유연하게 조건을 구성 가능
    - 메서드 이름
        - 조건이 많아지면 메서드 이름 길어짐
        - 일부 조건을 선택적으로 표현해야 하는 추가 검증 필요
            
            ```java
            // Method Name
            List<Customer> findByNameAndAgeGreaterThan(String name, int age);
            ```
            
    - JPQL
        - 조건이 많아지면 복잡해지고, 동적으로 조합하기 어려움
            
            ```java
            // JPQL
            @Query("SELECT c FROM Customer c WHERE c.name = :name AND c.age > :age")
            List<Customer> findCustom(@Param("name") String name, @Param("age") int age);
            ```
            
    - 자바 코드
        - 자바 코드만으로 WHERE 조건 직접 생성 가능
        - null 체크, 조건 조합, 재사용이 쉬움
            
            ```java
            // Java
            Specification<Customer> spec = (root, query, builder) -> 
                builder.equal(root.get("name"), "river");
            ```
            

---

## 사용 목적

- 조건에 따라 동적 필터링
    - 파라미터가 null일 수 있는 상황에서 조건을 유연하게 조립
- 비즈니스 로직 추상화
    - “특정 멤버 ID”, “특정 기간” 등 의미 있는 조건 표현
- 조건 재사용 및 조합
    - Specification 객체를 and(), or() 등으로 결합
- 동적 검색 API 구성
    - 다양한 검색 조건을 조합해 동적으로 쿼리 생성 가능

---

## 기본 구조

```java
public interface Specification<T> {
    Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder builder);
}
```

```java
public static Specification<User> complexSearchSpec() {
    return (root, query, builder) -> {
        // 조건 1: 이름이 'river'로 시작
        Predicate nameStartsWith = builder.like(root.get("name"), "river%");

        // 조건 2: 나이가 18세 이상
        Predicate adultOnly = builder.greaterThanOrEqualTo(root.get("age"), 18);

        // 중복 제거
        query.distinct(true);

        // 정렬: createdAt 오름차순
        query.orderBy(builder.asc(root.get("createdAt")));

        // 두 조건을 AND로 결합
        return builder.and(nameStartsWith, adultOnly);
    };
}
```

- `Root<T> root`
    - 쿼리의 루트 (FROM 절에 해당)
    - 엔터티 클래스의 필드에 접근하기 위한 **시작점**
        
        ```java
        root.get("email") // "email" 필드에 접근
        root.get("member").get("id") // member 연관관계의 id 필드
        ```
        
- `CriteriaQuery<?> query`
    - 현재 작성 중인 쿼리 자체
    - SELECT, ORDER BY, GROUP BY 등을 지정할 때 사용
        
        ```java
        query.distinct(true); // 중복 제거
        query.orderBy(builder.asc(root.get("name"))); // 정렬
        ```
        
- `CriteriaBuilder builder`
    - Predicate(조건) 생성 (WHERE 절에 해당)
        
        ```java
        builder.equal(root.get("age"), 20) // age = 20
        builder.greaterThan(root.get("sales"), 100000) // sales > 100000
        builder.between(root.get("createdAt"), from, to) // createdAt BETWEEN from AND to
        ```
        

---

## **설정 예시**

### **Repository 설정**

```java
public interface CustomerRepository extends
        JpaRepository<Customer, Long>,
        JpaSpecificationExecutor<Customer> {
}
```

### **Specification 정의**

```java
public class CustomerSpecs {

    public static Specification<Customer> isLongTermCustomer() {
        return (root, query, builder) ->
            builder.lessThan(root.get("createdAt"), LocalDate.now().minusYears(3));
    }

    public static Specification<Customer> hasSalesOfMoreThan(BigDecimal amount) {
        return (root, query, builder) ->
            builder.greaterThan(root.get("totalSales"), amount);
    }
}
```

---

## **사용 예시**

### **단일 조건**

```java
List<Customer> customers = customerRepository.findAll(
    CustomerSpecs.isLongTermCustomer()
);
```

### **조건 조합**

```java
Specification<Customer> spec = CustomerSpecs.isLongTermCustomer()
    .or(CustomerSpecs.hasSalesOfMoreThan(BigDecimal.valueOf(100000)));

List<Customer> customers = customerRepository.findAll(spec);
```

### **복잡한 조건 조립**

```java
Specification<Customer> spec = Specification
    .where(CustomerSpecs.isLongTermCustomer())
    .and(CustomerSpecs.hasSalesOfMoreThan(BigDecimal.valueOf(100000)))
    .and((root, query, builder) ->
         builder.equal(root.get("region"), "Seoul")
    );
```

### **삭제**

```java
Specification<User> underageSpec = (root, query, builder) ->
    builder.lessThan(root.get("age"), 18);

userRepository.delete(underageSpec);
```

- Specification은 삭제에도 사용 가능 (JPA 2.1 이상)
- JpaSpecificationExecutor는 delete(Specification) 메서드를 지원
- 내부적으로 JPA CriteriaDelete를 사용해 조건에 맞는 레코드를 삭제

---
