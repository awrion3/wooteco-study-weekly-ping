# JPA 설계

# 도메인 분석 설계

- 기능 목록
    - 회원 기능
        - 회원 등록
        - 회원 조회
    - 상품 기능
        - 상품 등록
        - 상품 수정
        - 상품 조회
    - 주문 기능
        - 상품 주문
        - 주문 내역 조회
        - 주문 취소
    - 기타 요구사항
        - 상품은 재고 관리가 필요하다.
        - 상품의 종류는 도서, 음반, 영화가 있다.
        - 상품을 카테고리로 구분할 수 있다.
        - 상품 주문시 배송 정보를 입력할 수 있다.

## 엔티티 클래스 개발

- 도메인 모델과 테이블 설계

  ![Untitled](/images/week9-jpa-1/Untitled.png)

    - 회원, 주문, 상품의 관계: 회원은 여러 상품을 주문할 수 있다. 그리고 한 번 주문할 때 여러 상품을 선택할 수 있으므로주문과 상품은 다대다 관계다. 하지만 이런 다대다 관계는 관계형 데이터베이스는 물론이고 엔티티에서도 거의 사용하지 않는다. 따라서 그림처럼 주문상품이라는 엔티티를 추가해서 다대다 관계를 일대다, 다대일 관계로 풀어냈다.
    - 상품 분류: 상품은 도서, 음반, 영화로 구분되는데 상품이라는 공통 속성을 사용하므로 상속 구조로 표현했다
- 엔티티 분석

  ![Untitled](/images/week9-jpa-1/Untitled%201.png)

    - 회원(Member): 이름과 임베디드 타입인 주소( Address ), 그리고 주문( orders ) 리스트를 가진다.
    - 주문(Order): 한 번 주문시 여러 상품을 주문할 수 있으므로 주문과 주문상품( OrderItem )은 일대다 관계다. 주문은상품을 주문한 회원과 배송 정보, 주문 날짜, 주문 상태( status )를 가지고 있다. 주문 상태는 열거형을 사용했는데 주문( ORDER ), 취소( CANCEL )을 표현할 수 있다.
    - 주문상품(OrderItem): 주문한 상품 정보와 주문 금액( orderPrice ), 주문 수량( count ) 정보를 가지고 있다. (보통 OrderLine , LineItem 으로 많이 표현한다.)
    - 상품(Item): 이름, 가격, 재고수량( stockQuantity )을 가지고 있다. 상품을 주문하면 재고수량이 줄어든다. 상품의종류로는 도서, 음반, 영화가 있는데 각각은 사용하는 속성이 조금씩 다르다.
    - 배송(Delivery): 주문시 하나의 배송 정보를 생성한다. 주문과 배송은 일대일 관계다.
    - 카테고리(Category): 상품과 다대다 관계를 맺는다. parent , child 로 부모, 자식 카테고리를 연결한다.
    - 주소(Address): 값 타입(임베디드 타입)이다. 회원과 배송(Delivery)에서 사용한다.
- 테이블 분석

  ![Untitled](/images/week9-jpa-1/Untitled%202.png)

    - MEMBER: 회원 엔티티의 Address 임베디드 타입 정보가 회원 테이블에 그대로 들어갔다. 이것은 DELIVERY 테이블도 마찬가지다.
    - ITEM: 앨범, 도서, 영화 타입을 통합해서 하나의 테이블로 만들었다. DTYPE 컬럼으로 타입을 구분한다.
- 연관관계 매핑 분석
    - 회원과 주문: 일대다 , 다대일의 양방향 관계다. 따라서 연관관계의 주인을 정해야 하는데, 외래 키가 있는 주문을 연관관계의 주인으로 정하는 것이 좋다. 그러므로 Order.member 를 ORDERS.MEMBER_ID 외래 키와 매핑한다.
    - 주문상품과 주문: 다대일 양방향 관계다. 외래 키가 주문상품에 있으므로 주문상품이 연관관계의 주인이다. 그러므로OrderItem.order 를 ORDER_ITEM.ORDER_ID 외래 키와 매핑한다.
    - 주문상품과 상품: 다대일 단방향 관계다. OrderItem.item 을 ORDER_ITEM.ITEM_ID 외래 키와 매핑한다.
    - 주문과 배송: 일대일 양방향 관계다. Order.delivery 를 ORDERS.DELIVERY_ID 외래 키와 매핑한다.
    - 카테고리와 상품: @ManyToMany 를 사용해서 매핑한다.(실무에서 @ManyToMany는 사용하지 말자. 여기서는 다대다 관계를 예제로 보여주기 위해 추가했을 뿐이다)

- Member

    ```java
    @Entity
    @Getter @Setter
    public class Member {
    
    		// 엔티티의 식별자는 id 사용, PK 컬럼명은 member_id 사용
    		// 테이블은 타입이 없으므로 구분이 어렵고, 관례상 테이블명 + id를 많이 사용 
    		
    		@Id @GeneratedValue
    		@Column(name = "member_id")
    		private Long id;
    		
    		private String name;
    		
    		@Embedded
    		private Address address;
    		
    		@OneToMany(mappedBy = "member")
    		private List<Order> orders = new ArrayList<>();
    }
    ```

- **Order**

    ```java
    @Entity
    @Table(name = "orders")
    @Getter @Setter
    public class Order {
    
    		@Id @GeneratedValue
    		@Column(name = "order_id")
    		private Long id;
    		
    		@ManyToOne(fetch = FetchType.LAZY)
    		@JoinColumn(name = "member_id")
    		private Member member; //주문 회원
    		
    		@OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    		private List<OrderItem> orderItems = new ArrayList<>();
    		
    		@OneToOne(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    		@JoinColumn(name = "delivery_id")
    		private Delivery delivery; //배송정보
    		
    		private LocalDateTime orderDate; //주문시간
    		
    		@Enumerated(EnumType.STRING) // Type은 무조건 String 
    		private OrderStatus status; //주문상태 [ORDER, CANCEL]
    		
    		//==연관관계 메서드==//
    		// 양방향 연결 시, 두줄 코드를 하나로 줄여줌
    		// 위치는 핵심적으로 컨트롤 하는곳에 
    		public void setMember(Member member) {
    		this.member = member;
    		member.getOrders().add(this);
    		}
    		
    		public void addOrderItem(OrderItem orderItem) {
    		orderItems.add(orderItem);
    		orderItem.setOrder(this);
    		}
    		
    		public void setDelivery(Delivery delivery) {
    		this.delivery = delivery;
    		delivery.setOrder(this);
    		}
    }
    ```

- OrderStatus

    ```java
    public enum OrderStatus {
        ORDER, CANCEL
    }
    ```

- OrderItem

    ```java
    @Entity
    @Table(name = "order_item")
    @Getter @Setter
    public class OrderItem {
    
    		@Id @GeneratedValue
    		@Column(name = "order_item_id")
    		private Long id;
    		
    		@ManyToOne(fetch = FetchType.LAZY)
    		@JoinColumn(name = "item_id")
    		private Item item; //주문 상품
    		
    		@ManyToOne(fetch = FetchType.LAZY)
    		@JoinColumn(name = "order_id")
    		private Order order; //주문
    		
    		private int orderPrice; //주문 가격
    		
    		private int count; //주문 수량
    	
    }
    ```

- Delivery

    ```java
    @Entity
    @Getter @Setter
    public class Delivery {
    
    		@Id @GeneratedValue
    		@Column(name = "delivery_id")
    		private Long id;
    		
    		@OneToOne(mappedBy = "delivery", fetch = FetchType.LAZY)
    		private Order order;
    		
    		@Embedded
    		private Address address;
    		
    		// 타입은 무조건 String 
    		@Enumerated(EnumType.STRING)
    		private DeliveryStatus status; //ENUM [READY(준비), COMP(배송)]
    }
    ```

- DeliveryStatus

    ```java
    public enum DeliveryStatus {
        READY, COMP
    }
    ```


- **Item**

    ```java
    // SINGLE_Table : 한테이블에 다 
    // TABLE_PER_CLASS : 클래스마다 테이블하나씩
    // JOINED : 가장 정교화
    
    @Entity
    @Inheritance(strategy = InheritanceType.SINGLE_TABLE)
    @DiscriminatorColumn(name = "dtype")
    @Getter @Setter
    public abstract class Item {
    
    		@Id @GeneratedValue
    		@Column(name = "item_id")
    		private Long id;
    		
    		private String name;
    		
    		private int price;
    		
    		private int stockQuantity;
    		
    		@ManyToMany(mappedBy = "items")
    		private List<Category> categories = new ArrayList<Category>();
    }
    ```

- Book

    ```java
    @Entity
    @DiscriminatorValue("B")
    @Getter @Setter
    public class Book extends Item {
    
    		private String author;
    		private String isbn;
    		
    }
    ```

- Album

    ```java
    @Entity
    @DiscriminatorValue("A")
    @Getter @Setter
    public class Album extends Item {
    
    		private String artist;
    		private String etc;
    		
    }
    ```

- Movie

    ```java
    @Entity
    @DiscriminatorValue("M")
    @Getter @Setter
    public class Movie extends Item {
    
    		private String director;
    		private String actor;
    		
    }
    ```

- Category

    ```java
    @Entity
    @Getter @Setter
    public class Category {
    
    		@Id @GeneratedValue
    		@Column(name = "category_id")
    		private Long id;
    	
    		private String name;
    		
    		// 실무에서는 ManyToMany 사용 X
    		// 중간테이블 (Category_item) 에 컬럼을 추가할 수 없고
    		// 세밀한 쿼리 실행이 어려움
    		// 다대다 매핑 -> 일대다, 다대일 매핑으로 풀어내기 
    		@ManyToMany
    		@JoinTable(name = "category_item",
    						joinColumns = @JoinColumn(name = "category_id"),
    						inverseJoinColumns = @JoinColumn(name = "item_id"))
    		private List<Item> items = new ArrayList<>();
    		
    		@ManyToOne(fetch = FetchType.LAZY)
    		@JoinColumn(name = "parent_id")
    		private Category parent;
    		
    		@OneToMany(mappedBy = "parent")
    		private List<Category> child = new ArrayList<>();
    		
    		//==연관관계 메서드==//
    		public void addChildCategory(Category child) {
    			this.child.add(child);
    			child.setParent(this);
    		}
    }
    ```

- Address

    ```java
    // 값 타입은 불변으로 설정! (Setter X, 생성자 초기화) 
    // JPA구현 라이브러리가 객체를 생성할 떄 리플랙션 같은 기술을 사용할 수 있도록 
    
    @Embeddable
    @Getter
    public class Address {
    
    		private String city;
    		
    		private String street;
    		
    		private String zipcode;
    		
    		protected Address() {
    		}
    		
    		public Address(String city, String street, String zipcode) {
    				this.city = city;
    				this.street = street;
    				this.zipcode = zipcode;
    		}
    }
    ```


### 엔티티 설계시 주의점

1. **가급적 Setter 사용 X , Getter는 열어두기**
    1. 변경 포인트가 너무 많아서 유지보수가 어렵다.
    2. setter 대신 별도의 비즈니스 메서드를 사용하는게 좋다.
2. **모든 연관관계는 지연로딩으로 설정하기**
    1. `FetchType.EAGER` → `FetchType.LAZY`
    2. **@XToOne (OneToOne, ManyToOne) 관계는 기본이 즉시로딩이므로 직접 지연로딩으로 설정**
        1. @XToMany(OneToMany, ManyToMany) 는 Lazy로 되어있음
    3. 즉시로딩( EAGER )은 예측이 어렵고, 어떤 SQL이 실행될지 추적하기 어렵다. 특히 JPQL을 실행할 때 N+1 문제가 자주 발생
        1. https://incheol-jung.gitbook.io/docs/q-and-a/spring/n+1
    4. 연관된 엔티티를 함께 DB에서 조회해야 하면, fetch join 또는 엔티티 그래프 기능을 사용
3. **컬렉션은 필드에서 바로 초기화**

    ```java
    private List<Order> orders = new ArrayList<>(); // 이거쓰기
    
    public Member() {orders = new ArrayList<>()} // 이거 X 
    ```

    1. null 문제에서 안전함
    2. 초기화 이후 건드리지 않는다.
    3. 하이버네이트는 컬렉션을 하이버네이트가 제공하는 내장 컬렉션으로 변경하기 때문에, 임의의 메서드에서 컬렉션을 잘못 생성하면 내부 메커니즘에 문제가 발생할 수 있다.

    ```java
    Member member = new Member();
    System.out.println(member.getOrders().getClass()); 
    // class java.util.ArrayList
    
    em.persist(member);
    System.out.println(member.getOrders().getClass());
    // class org.hibernate.collection.internal.PersistentBag
    ```

4. Enum 타입 시 `EnumType.*STRING*`으로 변경하기
    1. @Enumerate 지정
    2. EnumType의 default값 `EnumType.*ORDINAL*` → `EnumType.*STRING*`으로 변경
    3. ORDINAL→ 순서로 들어감, 중간에 다른 상태가 생기면 번호도 바뀌게 됨→ 절대 사용 X
5. 값 타입은 불변으로 설정
    1. Setter X, 생성자 초기화
    2. JPA구현 라이브러리가 객체를 생성할 떄 리플랙션 같은 기술을 사용할 수 있도록

    ```java
    @Embeddable
    @Getter
    public class Address {
    
    		private String city;
    		private String street;
    		private String zipcode;
    		
    		public Address(String city, String street, String zipcode) {
    				this.city = city;
    				this.street = street;
    				this.zipcode = zipcode;
    		}
    }
    ```

6. 외래 키가 있는 곳을 연관관계의 주인으로 정하기
    1. 비즈니스상 우위랑은 전혀 관계없음
    2. 자동차와 바퀴가 있으면, 일대다 관계에서 항상 다쪽에 외래키가 있으므로 바퀴를 주인으로
7. 양방향 연관관계
    1. ManyToOne, OneToMany
        1. `@JoinColumn(name = “member_id”)` ⇒ 연관관계의 주인! (외래 키가 있는 곳, 다대일 관계에서 다 를 담당하는 곳)
        2. mappedBy = “order”→ 나머지 연결

        ```java
            @ManyToOne
            @JoinColumn(name = "parent_id")
            private Category parent;
        
            @OneToMany(mappedBy = "parent")
            private List<Category> child = new ArrayList<>();
        ```

    2. OneToOne 의 일대일 관계
        1. FK를 아무데나 둬도 됨
        2. Access를 더 많이 하는 곳에 넣는 것을 선호 ( Order 와 Delivery가 1대1관계 면, Order에 넣는 것을 선호 )
        3. FK가 있는 Order를 연관관계의 주인으로 설정
    3. ManyToMany의 다대다 관계
        1. @JoinColumn 대신 JoinTable이 필요함
        2. 중간테이블 (Category_item) 에 컬럼을 추가할 수 없고 (필드 추가가 불가능), 세밀한 쿼리 실행이 어려움
        3. 다대다 매핑 -> 일대다, 다대일 매핑으로 풀어내기
        4. 실무에서는 ManyToMany 사용 X

        ```java
            @ManyToMany
            @JoinTable(name = "category_item",
                    joinColumns = @JoinColumn(name = "category_id"),
                    inverseJoinColumns = @JoinColumn(name = "item_id")
            )
            private List<Item> items = new ArrayList<>();
        ```

8. cascade = CascadeType.ALL
    1. https://resilient-923.tistory.com/417
    2. 부모와 자식 엔티티를 영속화하거나 제거할때, 부모와 자식 한번에 수행해줌
    3. 자식 객체를 먼저 영속화한 뒤 부모 객체를 영속화해야하는데, 부모객체만 영속화하면 자식객체도 알아서 영속화해준다.
9. 연관관계 편의 메서드
    1. 양방향 연관관계 시, 양 측에 모두 객체를 넣어줘야한다.
    2. 위치는 핵심적으로 컨트롤 하는 곳 , 자주 쓰이는 엔티티에 메서드를 정의하는것이 일반적
    3. https://joanne.tistory.com/220
        1. 변경 전

            ```java
                public void setMember(Member member) {
                    this.member = member;
                }
            
                public static void main(String[] args) {
                    Member member = new Member();
                    Order order = new Order();
            
                    member.getOrders().add(order);
                    order.setMember(member); 
                }
            ```

        2. 변경 후

            ```java
                public void setMember(Member member) {
                    this.member = member;
                    member.getOrders().add(this); // 양방향 설정
                }
            
                public static void main(String[] args) {
                    Member member = new Member();
                    Order order = new Order();
                    
                    order.setMember(member);
                }
            
            ```

10. Inheritance
    1. `@Inheritance(strategy = InheritanceType.*SINGLE_TABLE*)`
        1. SINGLE_Table : 한테이블에 다
        2. TABLE_PER_CLASS : 클래스마다 테이블하나씩
        3. JOINED : 가장 정교화
    2. https://www.baeldung.com/hibernate-inheritance#joined-table
