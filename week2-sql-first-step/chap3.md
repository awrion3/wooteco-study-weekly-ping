# 3장 정렬과 연산

## ORDER BY로 행 정렬

---

```sql
SELECT column_name FROM table_name ORDER BY sort_by;
```

- 오름차순 : `ORDER BY sort_by ASC`
- 내림차순 : `ORDER BY sort_by DESC`

- `ORDER BY`를 지정하지 않을 시, DB 내부에 저장된 순서대로 반환 함
- DB는 서버 당시 상황에 따라 어떤 순서로 행을 반환할지 경정됨
    - 따라서 항상 같은 순서를 보장하고 싶다면 `ORDER BY` 를 써줘야 함

- 정렬 기준
    - 숫자, 날짜는 숫자대로 (작을수록 이전, 클수록 이후)
    - 문자열은 사전순

## 복수의 열을 지정해 정렬하기

---

```sql
SELECT column_name FROM table_name ORDER BY sort_by1 ASC, sort_by2 DESC, ...;
```

- sort_by1 의 값이 같으면 그 다음 지정한 열로 비교해서 정렬함

- 정렬 기준의 열이 NULL일 경우
    - 데이터베이스 시스템에 따라 다른데, 가장 작은 값으로 취급하거나 가장 큰 값으로 처리함
    - ex ) MySQL은 가장 작은 값으로 취급함

## 결과 행 제한하기 - LIMIT

---

```sql
SELECT column_name FROM table_name LIMIT column_count [OFFSET start_column];
```

- `start_column`부터 시작해서 `column_count`만큼 데이터 조회하기
- `LIMIT`은 MySQL, PostgreSQL에서만 사용 가능

## 수치 연산

---

- column 값에 대해서 사칙연산 (+-/*%) 모두 가능
- 걍 쓰던대로 쓰면 됨
- ex )

    ```sql
    SELECT *, price * quantity FROM Products;
    ```


- 열 별명 쓰기

    ```sql
    SELECT *, price * quantity AS Amount FROM Products;
    SELECT *, price * quantity "금액" FROM Products;
    ```

- 예약어를 테이블명으로 쓰기

    ```sql
    SELECT price * quantity AS "SELECT" FROM Products;
    ```


- MySQL은 객체명으로 숫자 시작 가능하지만 Oracle은 숫자 시작 불가

- WHERE절에서도 연산 가능

    ```sql
    SELECT * FROM Products WHERE price * quantity >= 2000;
    ```


- 아래 예시는 불가

    ```sql
    SELECT price * quantity AS amount
    FROM Products
    WHERE amount >= 2000;
    ```

    - DB 서버 내부에서 `WHERE` 구 → `SELECT` 구 순서로 처리
        - WHERE 조건에 해당하는 컬럼인지 먼저 판단한 후에 SELECT에 지정된 열을 반환하게 됨
- 하지만 `ORDER BY`는 별명 사용이 가능

    ```sql
    SELECT price * quantity AS amount
    FROM Products
    ORDER BY amount DESC;
    ```

    - WHERE → SELECT → ORDER BY 순

- `NULL` 연산
    - SQL에서 NULL은 0이 아닌 모두 NULL로 처리됨
        - NULL + 1, 2 * NULL, 1 / NULL

- `ROUND` 함수
    - 반올림 연산

    ```sql
    SELECT ROUND(amount,1) FROM Products; // 소수점 둘 째 자리 수에서 반올림
    // 50.312 -> 50
    ```

    - 소수점 포함된 데이터는 DECIMAL 형으로 저장함 : 소수점을 포함하는 수치로 저장
    - 10단위로 반올림도 가능

    ```sql
    SELECT ROUND(amount, -2) FROM Products; // 10단위 반올림
    // 551.312 -> 600
    
    SELECT ROUND(amount, -3) FROM Products; // 100단위 반올림
    // 243.312 -> 200
    ```


## 문자열 연산

---

- `CONCAT`
    - 모두 ‘ABC1234’ 반환

    ```sql
    'ABC' || '1234' // Oracle, DB2, PostgreSQL
    CONCAT('ABC','1234') // MySQL
    'ABC' + '1234' // SQL Server
    ```

    - 보통 2개의 열 데이터를 모아 1개의 열로 처리하고 싶을 때 주로 사용
    - ex ) `SELECT * FROM Products;` 결과가 아래와 같을 때,

        >| no | price | quantity | unit |
        >| --- | --- | --- | --- |
        >| 1 | 100 | 10 | 개 |
        >| 2 | 230 | 24 | 캔 |
        >| 3 | 1980 | 1 | 장 |
        - 각 상품 별 개수 결과 String으로 얻기
        
        ```sql
        SELECT CONCAT(quantity,unit) AS count FROM Products;
        ```
        
        >| count |
        >| --- |
        >| 10개 |
        >| 24캔 |
        >| 1장 |
- `SUBSTRING`

    ```sql
    SUBSTRING('20250312',5,2) => '03' // 5째자리부터 2자리 추출
    ```


- `TRIM`
    - 문자열 앞뒤로 여분 스페이스가 있을 때 이를 제거 해줌

    ```sql
    TRIM(' A BC    ') => 'A BC'
    ```

    - 문자열 중간에 존재하는건 제거 안해줌
    - 인수를 지정해서 스페이스 외의 문자를 제거해줄 수도 있음


- `CHARACTER_LENGTH`
    - 문자열 길이를 계산해 돌려주는 함수
    - 여담 ) OCTET_LENGTH 함수 사용하면 문자 수가 아니라 바이트 단위로 길이를 계산해줌
        - 근데 문자 세트에 따라 바이트 수가 다르게 계산 됨

      | 문자세트 | ASCII | 한글 |
              | --- | --- | --- |
      | EUC-KR | 1 byte | 2 bytes |
      | UTF-8 | 1 byte | 3 bytes |

## 날짜 연산

---

- 시스템 날짜 확인하기

    ```sql
    SELECT CURRENT_TIMESTAMP; // 현재 시스템 날짜 확인하기 
    ```


- Oracle은 아래처럼 TO_DATE 사용해서 날짜 서식 지정 가능

    ```sql
    TO_DATE('2024/01/25', 'YYYY/MM/DD');
    ```

- 날짜 덧뺄셈

    ```sql
    SELECT CURRENT_DATE + INTERVAL 1 DAY; // 오늘이 2025/03/12 일 경우,
    // 결과 : 2025-03-13
    ```

    - `+ INTERVAL 1 DAY` : 1일 후라는 의미의 기간형 상수
    - N일 전으로 하고싶으면 `- INTERVAL N DAY` 로 하면 됨

- 날짜형 간 뺄셈
    - MySQL에서는 두 날짜 사이 얼마나 차이나는지 계산하는 함수로 `DATEDIFF`를 사용

    ```sql
    SELECT DATEDIFF('2025-03-15', '2025-02-01');
    ```


## CASE 문으로 데이터 변환하기

---

- CASE

```sql
CASE 
    WHEN gender = 1 THEN '남'
    WHEN gender = 2 THEN '여'
    ELSE '잘못된 값'
END
```
- ex ) NULL 값은 0으로 반환하기
    ```sql
    SELECT a, 
        CASE
        	WHEN a IS NULL THEN 0
        	ELSE a
        END
        AS "a(null=0)"
    FROM Products;
    ```
    
    >| a |
    >| --- |
    >| 1 |
    >| 2 |
    >| NUL |
    > 쿼리 수행 전

    > | a | a(null=0) |
    > | --- | --- |
    > | 1 | 1 |
    > | 2 | 2 |
    > | NULL | 0 |
    > 쿼리 수행 후

  - NULL 값을 다른 값으로 대체하는 건 `COALESCE` 함수를 사용할 수도 있음

    ```sql
    SELECT a, COALESCE(a,0) FROM Products;
    ```


- **검색 CASE** vs **단순 CASE**

  - 검색 CASE
      ```sql
      CASE 
          WHEN gender = 1 THEN '남'
          WHEN gender = 2 THEN '여'
          ELSE '잘못된 값'
      END
      ```
  - 단순 CASE
      ```sql
      CASE gender
          WHEN 1 THEN '남'
          WHEN 2 THEN '여'
          ELSE '잘못된 값'
      END
      ```

    
- CASE는 SELECT, WHERE, ORDER BY 모두 사용될 수 있음
- ELSE를 생략하면 `ELSE NULL` 가 default로 적용 됨
    - 따라서 ELSE는 생략하지 말고 지정하는 것이 나음



- Q. 아래 CASE문에서 문제점이 뭘까?
    ```sql
    CASE gender
    	WHEN 1 THEN '남'
    	WHEN 2 THEN '여'
    	WHEN NULL THEN '데이터 없음'
    	ELSE '잘못된 값'
    END
    ```

    - A : 문법적으로는 문제 없는 쿼리지만, 실제 gender 데이터가 NULL일 때 ‘데이터 없음’으로 처리되지 않음
    - **단순 CASE**의 실제 동작

        ```sql
        CASE 
                WHEN gender = 1 THEN '남'
        	WHEN gender = 2 THEN '여'
        	WHEN gender = NULL THEN '데이터 없음' // NULL은 = 로 연산 불가!!
        	ELSE '잘못된 값'
        END
        ```

        - = 연산자로는 NULL 비교가 불가능 하므로, 실제 gender 값이 NULL이어도 WHEN 조건에 걸치지 못함
    - 따라서 NULL 값에 대한 판정을 하고싶으면 **단순 CASE** 말고 **검색 CASE**를 사용하자

        ```sql
        CASE 
        	WHEN gender = 1 THEN '남'
        	WHEN gender = 2 THEN '여'
        	WHEN gender IS NULL THEN '데이터 없음' // NULL은 = 로 연산 불가!!
        	ELSE '잘못된 값'
        END
        ```

## Quiz

---
>| 주문번호 (order_id) | 고객ID (customer_id) | 주문일 (order_date)  | 금액 (amount) | 배송상태 (status) |
>|---------------------|----------------------|----------------------|---------------|-------------------|
>| 1                   | 1001                 | 2025-01-01           | 12000         | 배송중            |
>| 2                   | 1002                 | 2025-02-20           | 25000         | 배송완료          |
>| 3                   | 1003                 | 2025-02-25           | 10000         | 배송중            |
>| 4                   | 1001                 | 2025-03-01           | 18000         | 배송중            |
>| 5                   | 1004                 | 2025-03-05           | 5000          | 배송완료          |

각 고객의 총 주문 금액을 계산한 뒤, 금액이 가장 높은 고객의 customer_id와 total_amount를 출력하시오. 단, 배송완료 상태인 주문만 포함하며, 금액은 ROUND 함수를 이용해 소수 첫째 자리에서 반올림하시오.


<details>
  <summary>정답</summary>

```sql
SELECT customer_id, ROUND(SUM(amount), 1) AS total_amount
FROM Orders
WHERE status = '배송완료'
GROUP BY customer_id
ORDER BY total_amount DESC
LIMIT 1;
```
</details>
