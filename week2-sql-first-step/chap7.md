# 7장. 복수의 테이블 다루기

---

## 집합 연산

- 다음과 같은 두 테이블이 있다고 가정하자.

> recipe_1

| ingredient      |
|-----------------|
| Flour           |
| Sugar           |
| Eggs            |
| Milk            |
| Vanilla Extract |
| Salt            |

> recipe_2

| ingredient |
|------------|
| Flour      |
| Sugar      |
| Eggs       |
| Cinnamon   |
| Nutmeg     |
| Salt       |

---

1. 합집합

- UNION: table1과 table2의 합집합을 반환한다.
- UNION ALL: table1과 table2의 합집합을 중복을 허용하여 반환한다.

[예제] 두 레시피의 재료에 대해 UNION ALL을 사용해보자.

```sql
    SELECT ingredient
    FROM recipe_1
    UNION ALL
    SELECT ingredient
    FROM recipe_2;
```

> 결과

| ingredient |
|------------|
| Flour |
| Sugar |
| Eggs |
| Milk |
| Vanilla Extract |
| Salt |
| Flour |
| Sugar |
| Eggs |
| Cinnamon |
| Nutmeg |
| Salt |

---

2. 교집합

- INTERSECT: table1과 table2의 교집합을 반환한다.

[예제] 두 레시피의 재료에 대해 INTERSECT를 사용해보자.

```sql
    SELECT ingredient
    FROM recipe_1
    INTERSECT
    SELECT ingredient
    FROM recipe_2;
```

> 결과

| ingredient |
|------------|
| Flour |
| Sugar |
| Eggs |
| Salt |

---

3. 차집합

- EXCEPT: table1에서 table2의 차집합을 반환한다.

[예제] 두 레시피의 재료에 대해 EXCEPT를 사용해보자.

```sql
    SELECT ingredient
    FROM recipe_1
    EXCEPT
    SELECT ingredient
    FROM recipe_2;
```

> 결과

| ingredient |
|------------|
| Milk |
| Vanilla Extract |

---

## 테이블 결합

1. 교차 결합

- CROSS JOIN: 두 테이블의 모든 행을 결합한다. 한쪽 테이블의 모든 행과 다른 쪽 테이블의 모든 행을 조인하는 기능이다.
    - 상호 조인 결과의 전체 행 개수는 두 테이블의 각 행의 개수를 곱한 수만큼 된다.
    - 카티션 곱(CARTESIAN PRODUCT)라고도 한다.

[예제] 두 레시피의 재료에 대해 CROSS JOIN을 사용해보자.

```sql
    SELECT *
    FROM recipe_1
    CROSS JOIN recipe_2;
```

> 결과

| ingredient      | ingredient |
|-----------------|------------|
| Flour           | Flour      |
| Flour           | Sugar      |
| Flour           | Eggs       |
| Flour           | Cinnamon   |
| Flour           | Nutmeg     |
| Flour           | Salt       |
| Sugar           | Flour      |
| Sugar           | Sugar      |
| Sugar           | Eggs       |
| Sugar           | Cinnamon   |
| Sugar           | Nutmeg     |
| Sugar           | Salt       |
| Eggs            | Flour      |
|...|...|

---

2. 내부 결합

- (INNER) JOIN: 두 테이블의 교집합을 반환한다. 두 테이블을 조인할 때, 두 테이블에 모두 지정한 열의 데이터가 있어야 한다.

[예제] 두 레시피의 재료에 대해 INNER JOIN(혹은 그냥 JOIN)을 사용해보자.

```sql
    SELECT *
    FROM recipe_1
    INNER JOIN recipe_2
    ON recipe_1.ingredient = recipe_2.ingredient;
```

> 결과

| ingredient      | ingredient |
|-----------------|------------|
| Flour           | Flour      |
| Sugar           | Sugar      |
| Eggs            | Eggs       |
| Salt            | Salt       |

---

3. 외부 결합

    * OUTER JOIN: 두 테이블을 조인할 때, 1개의 테이블에만 데이터가 있어도 결과가 나온다. 
        * 즉, 내부 조인은 두 테이블에 모두 데이터가 있어야만 결과가 나오지만, 외부 조인은 한쪽에만 데이터가 있어도 결과가 나온다.


- LEFT (OUTER) JOIN: 왼쪽 테이블의 모든 값이 출력되는 조인이다.

[예제] 두 레시피의 재료에 대해 LEFT OUTER JOIN을 사용해보자.

```sql
    SELECT *
    FROM recipe_1
    LEFT OUTER JOIN recipe_2
    ON recipe_1.ingredient = recipe_2.ingredient;
```

> 결과

| ingredient      | ingredient |
|-----------------|------------|
| Flour           | Flour      |
| Sugar           | Sugar      |
| Eggs            | Eggs       |
| Milk            | NULL       |
| Vanilla Extract | NULL       |
| Salt            | Salt       |


- RIGHT (OUTER) JOIN: 오른쪽 테이블의 모든 값이 출력되는 조인이다.

[예제] 두 레시피의 재료에 대해 RIGHT OUTER JOIN을 사용해보자.

```sql
    SELECT *
    FROM recipe_1
    RIGHT OUTER JOIN recipe_2
    ON recipe_1.ingredient = recipe_2.ingredient;
```

> 결과

| ingredient      | ingredient |
|-----------------|------------|
| Flour           | Flour      |
| Sugar           | Sugar      |
| Eggs            | Eggs       |
| NULL            | Cinnamon   |
| NULL            | Nutmeg     |
| Salt            | Salt       |


- FULL (OUTER) JOIN: 왼쪽 외부 조인과 오른쪽 외부 조인이 합쳐진 것이다.

[예제] 두 레시피의 재료에 대해 FULL OUTER JOIN을 사용해보자.

```sql
    SELECT *
    FROM recipe_1
    FULL OUTER JOIN recipe_2
    ON recipe_1.ingredient = recipe_2.ingredient;
```

> 결과

| ingredient      | ingredient |
|-----------------|------------|
| Flour           | Flour      |
| Sugar           | Sugar      |
| Eggs            | Eggs       |
| Milk            | NULL       |
| Vanilla Extract | NULL       |
| NULL            | Cinnamon   |
| NULL            | Nutmeg     |
| Salt            | Salt       |

---

## 퀴즈

1. 다음 중 테이블 결합에 대한 설명으로 옳지 않은 것은?

    1. 교차 결합은 두 테이블의 모든 행을 결합한다.
    2. 내부 결합은 두 테이블의 교집합을 반환한다.
    3. 외부 결합은 한쪽 테이블에만 데이터가 있어도 결과가 나온다.
    4. 외부 결합은 한쪽 테이블에만 데이터가 있어도 결과가 나오지 않는다.

<details>
<summary>정답</summary>

> 정답: 4
> (외부 결합은 한쪽 테이블에만 데이터가 있어도 결과가 나온다.)

</details>

---

2. INTERSECT를 사용하여 두 학급에서 모두 'A' 학점을 받은 학생을 찾아보자.

```
Table: Class_A

student_id	name	grade
1	        Alice	A
2	        Bob	B
3	        Charlie	C
4	        David	B
5	        Emma	A

Table: Class_B

student_id	name	grade
1	        Alice	A
2	        Bob	B
6	        Frank	B
7	        Grace	C
```

<details>
<summary>정답</summary>

> 정답

```sql
    SELECT student_id, name, grade
    FROM Class_A
    WHERE grade = 'A'
    INTERSECT
    SELECT student_id, name, grade
    FROM Class_B
    WHERE grade = 'A';
```

> 결과

| student_id	 |name	| grade |
|--------|-----|---|
| 1	     |Alice	| A |

</details>

---

3. 다음 두 개의 테이블이 있다고 가정하자.

> 고객 테이블

| 고객_ID | 이름  | 도시       |  
|--------|------|-----------|  
| 1      | 철수 | 서울      |  
| 2      | 영희 | 부산      |  
| 3      | 민수 | 대구      |  
| 4      | 지수 | 인천      |  

> 주문 테이블

| 주문_ID | 고객_ID | 상품명  | 금액  |  
|--------|--------|------|------|  
| 101    | 1      | 노트북 | 100만원 |  
| 102    | 2      | 스마트폰 | 50만원  |  
| 103    | 1      | 키보드 | 5만원   |  
| 104    | 3      | 모니터 | 30만원  |  

`고객` 및 `주문` 테이블을 사용하여 모든 고객과 그들의 주문 정보를 조회하되, 
주문이 없는 고객도 포함하도록 `LEFT JOIN`을 사용한 SQL 쿼리를 작성해본다.
(주문이 없는 경우 상품명은 `NULL`로 표시된다.)

<details>
<summary>정답</summary>

> 정답

```sql
    SELECT c.고객_ID, c.이름, c.도시, o.주문_ID, o.상품명, o.금액
    FROM 고객 AS c
    LEFT JOIN 주문 AS o
    ON c.고객_ID = o.고객_ID;
```

> 결과

| 고객_ID | 이름  | 도시       | 주문_ID | 상품명  | 금액  |
|--------|------|-----------|--------|------|------|
| 1      | 철수 | 서울      | 101    | 노트북 | 100만원 |
| 1      | 철수 | 서울      | 103    | 키보드 | 5만원   | 
| 2      | 영희 | 부산      | 102    | 스마트폰 | 50만원  |
| 3      | 민수 | 대구      | 104    | 모니터 | 30만원  |
| 4      | 지수 | 인천      | NULL   | NULL  | NULL  |

</details>

---

4. `SELF JOIN`을 사용하여 직원과 그들의 관리자의 이름을 함께 조회하는 SQL 쿼리를 작성해보자.


> 직원 테이블

| 직원_ID | 이름  | 관리자_ID |  
|--------|------|--------|  
| 1      | 철수  | NULL   |  
| 2      | 영희  | 1      |  
| 3      | 민수  | 1      |  
| 4      | 지수  | 2      |  
| 5      | 유진  | 2      |  


<details>
<summary>정답</summary>

> 정답

```sql
    SELECT e1.이름 AS 직원, e2.이름 AS 관리자
    FROM 직원 AS e1
    LEFT JOIN 직원 AS e2
    ON e1.관리자_ID = e2.직원_ID;
```

- 이처럼 `SELF JOIN`은 같은 테이블을 두 번 사용하여 조인하는 방식을 의미한다.

> 결과

| 직원 | 관리자 |
|----|------|
| 철수 | NULL |
| 영희 | 철수 |
| 민수 | 철수 |
| 지수 | 영희 |
| 유진 | 영희 |

</details>

---
