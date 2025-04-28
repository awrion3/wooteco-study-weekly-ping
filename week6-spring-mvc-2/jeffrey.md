# NamedParameterJdbcTemplate

Date: 2025년 4월 28일
Status: Done

https://docs.spring.io/spring-framework/reference/data-access/jdbc/core.html#jdbc-JdbcTemplate

# JdbcTemplate을 왜 쓰는가?

JdbcTemplate은 템플릿 콜백 패턴을 사용해서, JDBC를 직접 사용할 때 발생하는 대부분의 반복 작업을 대신 처리해준다.

- 템플릿 콜백 패턴?

  https://inpa.tistory.com/entry/GOF-%F0%9F%92%A0-Template-Callback-%EB%B3%80%ED%98%95-%ED%8C%A8%ED%84%B4-%EC%95%8C%EC%95%84%EB%B3%B4%EA%B8%B0


## 장점

- 기존 코드 (순수 JDBC)

    ```java
    public Connection getConnection() {
        String url = "jdbc:sqlite:database.db";
        try {
            return DriverManager.getConnection(url);
        } catch (final SQLException e) {
            System.err.println("DB 연결 오류:" + e.getMessage());
            e.printStackTrace();
            throw new RuntimeException("DB 연결 중 오류 발생", e);
        }
    }
    
    public void saveBoard(JanggiBoard board) {
        final String boardQuery = "INSERT INTO board (piece_id, x, y, piece, side) VALUES (?, ?, ?, ?, ?) " +
                "ON CONFLICT(piece_id) DO UPDATE SET x = excluded.x, y = excluded.y, piece = excluded.piece, side = excluded.side";
        final String statusQuery = "UPDATE board_status SET status = ?";
    
        try (final Connection connection = getConnection();
             final PreparedStatement boardStatement = connection.prepareStatement(boardQuery);
             final PreparedStatement statusStatement = connection.prepareStatement(statusQuery)) {
    
            for (Map.Entry<Position, Piece> entry : board.getBoard().entrySet()) {
                boardStatement.setInt(1, entry.getKey().hashCode());
                boardStatement.setInt(2, entry.getKey().getX());
                boardStatement.setInt(3, entry.getKey().getY());
                boardStatement.setString(4, entry.getValue().getSymbol().toString());
                boardStatement.setString(5, entry.getValue().getSide().toString());
                boardStatement.executeUpdate();
            }
    
            statusStatement.setString(1, board.getStatus().toString());
            statusStatement.executeUpdate();
        } catch (SQLException e) {
            throw new RuntimeException("장기판 저장 중 오류 발생", e);
        }
    ```


### 1. 설정이 편하다 → `spring-jdbc` 라이브러리만 추가하면 됨.

```
// JdbcTemplate 추가
implementation 'org.springframework.boot:spring-boot-starter-jdbc'
// H2 데이터베이스 추가
runtimeOnly 'com.h2database:h2'
```

데이터베이스 접근 설정
src/main/resources/application.properties

```
spring.profiles.active=local
spring.datasource.url=jdbc:h2:tcp://localhost/~/test
spring.datasource.username=sa
```

스프링 부트가 해당 설정을 사용해서 커넥션 풀과 DataSource, 트랜잭션 매니저를 스프링 빈으로 자동 등록한다.

### 2. 우리가 생각할 수 있는 대부분의 반복 작업을 대신 처리해준다.

개발자는 SQL을 작성하고, 전달할 파리미터를 정의하고, 응답 값을 매핑하기만 하면 된다.

- 커넥션 획득
- `statement` 를 준비하고 실행
- 결과를 반복하도록 루프를 실행
- 커넥션 종료, `statement`, `resultset`종료
- 트랜잭션 다루기 위한 커넥션 동기화
- 예외 발생시 스프링 예외 변환기 실행

## 단점

### 동적 SQL을 해결하기 어렵다

동적 쿼리란? 사용자가 검색하는 값에 따라서 실행하는 SQL 이 동적으로 달라지는 쿼리

→ `MyBatis` 등을 사용해 SQL 직접 사용시 동적 쿼리 작성

- 동적 쿼리 예시

    ```java
    @Override
    public List<Item> findAll(final ItemSearchCond cond) {
        String itemName = cond.getItemName();
        Integer maxPrice = cond.getMaxPrice();
    
        String sql = "select id, item_name, price, quantity from item";
        //동적 쿼리
        if (StringUtils.hasText(itemName) || maxPrice != null) {
            sql += " where";
        }
        boolean andFlag = false;
        List<Object> param = new ArrayList<>();
        if (StringUtils.hasText(itemName)) {
            sql += " item_name like concat('%',?,'%')";
            param.add(itemName);
            andFlag = true;
        }
        if (maxPrice != null) {
            if (andFlag) {
                sql += " and";
            }
            sql += " price <= ?";
            param.add(maxPrice);
        }
        log.info("sql={}", sql);
        return template.query(sql, itemRowMapper(), param.toArray());
    }
    ```


## 주로 사용되는 메서드

### 1. `update()`

데이터 변경(INSERT, UPDATE, DELETE) 시 사용, 반환값은 영향받은 로우 수 (int)

### 2. `queryForObject()`

SELECT 결과 로우가 하나일때, `RowMapper` 필요

(`RowMapper`는 데이터베이스의 반환 결과인 `ResultSet` 을 객체로 변환한다)

- 결과가 없으면 `EmptyResultDataAccessException`
- 결과가 둘 이상이면 `IncorrectResultSizeDataAccessException`

### 3. `query()`

SELECT 결과가 하나 이상일때, `RowMapper` 필요

- 결과가 없으면 빈 컬렉션을 반환한다.

### 4. `execute()`

임의의 SQL을 실행할 때 (테이블을 생성하는 DDL 등)

```java
jdbcTemplate.execute("create table mytable (id integer, name varchar(100))");
```

# 이름 지정 파라미터 - NamedParameterJdbcTemplate

기존 단점 → 파라미터를 순서대로 바인딩함

```sql
String sql = "update item set item_name=?, price=?, quantity=? where id=?";
// 의도 
template.update(sql, itemName, price, quantity, itemId);
// 실제 
template.update(sql, itemName, quantity, price, itemId);
```

`price` 와 `quantity` 가 둘 다 int 형식이라면, 두 파라미터의 위치가 바뀐다면?

파라미터가 많아지면 일일히 체크하기 어려워짐

### 사용 예시

```java
private final NamedParameterJdbcTemplate template;

// 이래도 동작은 함 
// public RoomescapeRepositoryImpl(final JdbcTemplate template) {
//     this.template = template;
// }
    
public JdbcTemplateItemRepositoryV2(DataSource dataSource) {
		this.template = new NamedParameterJdbcTemplate(dataSource);
}

public void update(final Long itemId, final ItemUpdateDto updateParam) {
    String sql = "update item set item_name=:itemName. price=:price, quantity=:quantity where id=:id";

    SqlParameterSource param = new MapSqlParameterSource()
            .addValue("itemName", updateParam.getItemName())
            .addValue("price", updateParam.getPrice())
            .addValue("quantity", updateParam.getQuantity())
            .addValue("id", itemId); 

    template.update(sql, param);
}
```

의존관계 주입은 `dataSource` 를 받고 내부에서 `NamedParameterJdbcTemplate`을 생성해서 가지고 있다. 스프링에서는 `JdbcTemplate` 관련 기능을 사용할 때 관례상 이 방법을 많이 사용한다.

Map처럼 key, value 데이터 구조를 만들어서 전달하면 됨

## 이름 지정 바인딩에서 자주 사용하는 파라미터 종류

### 1.  Map

```java
Map<String, Object> param = Map.of("id", id);
Item item = template.queryForObject(sql, param, itemRowMapper());
```

### 2. MapSqlParameterSource

Map과 유사하지만, SQL 타입을 지정할 수 있는 등 SQL에 좀 더 특화된 기능을 제공

메서드 체인을 통해 편리한 사용법도 제공한다.

```java
@Data
public class ItemUpdateDto {
    private String itemName;
    private Integer price;
    private Integer quantity;

    public ItemUpdateDto() {
    }

    public ItemUpdateDto(String itemName, Integer price, Integer quantity) {
        this.itemName = itemName;
        this.price = price;
        this.quantity = quantity;
    }
}
```

```java
@Override
public void update(final Long itemId, final ItemUpdateDto updateParam) {
    String sql = "update item set item_name=:itemName. price=:price, quantity=:quantity where id=:id";

    SqlParameterSource param = new MapSqlParameterSource()
            .addValue("itemName", updateParam.getItemName())
            .addValue("price", updateParam.getPrice())
            .addValue("quantity", updateParam.getQuantity())
            .addValue("id", itemId); // 이부분이 별도로 필요하기 때문에 BeanPropertySqlParameterSource 못씀

    template.update(sql, param);
}
```

### 3. BeanPropertySqlParameterSource

자바빈 프로퍼티 규약을 통해 자동으로 파라미터 객체를 생성함.
(getXxx() → xxx, getItemName → itemName)

객체의 필드값 이외의 추가적인 데이터를 바인딩해야할 때 사용불가.
ex) `ItemUpdateDto` 에는 id값이 없어서, 별도 메서드 파라미터로 받아야 할 때는 `MapSqlParameterSource`만 사용가능

```java
@Data
public class Item {

    private Long id;

    private String itemName;
    private Integer price;
    private Integer quantity;

    public Item() {
    }

    public Item(String itemName, Integer price, Integer quantity) {
        this.itemName = itemName;
        this.price = price;
        this.quantity = quantity;
    }
}
```

```java
public Item save(final Item item) {
    String sql = "insert into item(item_name, price, quantity) values(:itemName, :price, :quantity)";

    SqlParameterSource param = new BeanPropertySqlParameterSource(item);

    KeyHolder keyHolder = new GeneratedKeyHolder();
    template.update(sql, param, keyHolder);

    long key = keyHolder.getKey().longValue();
    item.setId(key);
    return item;
}
```

자바빈 프로퍼티 규약?

https://www.upgrad.com/blog/javabeans-properties-benefits/
