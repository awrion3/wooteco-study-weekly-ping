# 5장. 집계와 서브쿼리

---

### sample database

| category    | product          | user_id | spend  | transaction_date    |
|-------------|------------------|---------|--------|---------------------|
| appliance   | washing machine  | 123     | 219.80 | 03/02/2022 11:00:00 |
| electronics | vacuum           | 178     | 152.00 | 04/05/2022 10:00:00 |
| electronics | wireless headset | 156     | 249.90 | 07/08/2022 10:00:00 |
| electronics | vacuum           | 145     | 189.00 | 07/15/2022 10:00:00 |
| electronics | computer mouse   | 195     | 45.00  | 07/01/2022 11:00:00 |
| appliance   | refrigerator     | 165     | 246.00 | 12/26/2021 12:00:00 |
| appliance   | refrigerator     | 123     | 299.99 | 03/02/2022 11:00:00 |

---

## 20강. 행 개수 구하기: COUNT

- 집계 함수: 집합을 특정 방식으로 계산하고 결과를 반환하는 함수

- **COUNT**: 행의 개수를 반환하는 집계 함수

```sql
    SELECT COUNT(user_id)
    FROM product_spend;
```

> **결과**: 7

- **DISTINCT**: 중복을 제거한 행의 개수를 반환하는 집계 함수

```sql
    SELECT COUNT(DISTINCT category)
    FROM product_spend;
```

> **결과**: 2

- distinct category: appliance, electronics이므로.

## 21강. COUNT 이외의 집계 함수

- **SUM**: 합계를 반환하는 집계 함수

```sql
    SELECT SUM(spend)
    FROM product_spend;
```

> **결과**: 1401.69 (전체 금액 합계)

- **AVG**: 평균을 반환하는 집계 함수

```sql
    SELECT AVG(spend)
    FROM product_spend;
```

> **결과**: 200.24 (전체 평균 금액)

- **MAX**: 최대값을 반환하는 집계 함수

```sql
    SELECT MAX(spend)
    FROM product_spend;
```

> **결과**: 299.99 (최대 금액)

- **MIN**: 최소값을 반환하는 집계 함수

```sql
    SELECT MIN(spend)
    FROM product_spend;
```

> **결과**: 45.00 (최소 금액)

## 22강. 그룹화 하기: GROUP BY

- **GROUP BY**: 특정 열을 기준으로 그룹화하여 집계 함수를 사용할 수 있게 하는 구문

```sql
    SELECT category,
           SUM(spend)
    FROM product_spend
    GROUP BY category;
```

> **결과**:
>
> | category    | SUM(spend) |
> |-------------|------------|
> | appliance   | 765.79     |
> | electronics | 635.90     |

- **HAVING**: 그룹화된 결과에 조건을 걸어주는 구문

```sql
    SELECT category,
           SUM(spend)
    FROM product_spend
    GROUP BY category
    HAVING SUM(spend) > 700;
```

> **결과**:
>
> | category  | SUM(spend) |
> |-----------|------------|
> | appliance | 765.79     |

---

## 23강. 서브쿼리

- 서브쿼리: 쿼리 안에 또 다른 쿼리를 넣어서 사용하는 것

```sql
    SELECT category,
           AVG(spend) AS avg_spend,
           MAX(spend) AS max_spend
    FROM product_spend
    GROUP BY category
    ORDER BY category;
```

> **결과**:
>
> | category    | avg_spend | max_spend |
> |-------------|-----------|-----------|
> | appliance   | 255.26    | 299.99    |
> | electronics | 158.98    | 249.90    |

## 24강. 상관 서브쿼리

- 상관 서브쿼리: 외부 쿼리의 결과에 따라 내부 쿼리의 결과가 달라지는 서브쿼리

- **EXISTS**: 서브쿼리의 결과가 존재하는지 확인하는 연산자

```sql
    SELECT category,
           product
    FROM product_spend p1
    WHERE EXISTS (SELECT 1
                  FROM product_spend p2
                  WHERE p1.category = p2.category
                  GROUP BY category
                  HAVING COUNT(DISTINCT product) > 1);
```

> **결과**:
>
> | category    | product          |
> |-------------|------------------|
> | appliance   | washing machine  |
> | electronics | vacuum           |
> | electronics | wireless headset |
> | electronics | vacuum           |
> | electronics | computer mouse   |
> | appliance   | refrigerator     |
> | appliance   | refrigerator     |

- **IN** : 서브쿼리의 결과가 포함되어 있는지 확인하는 연산자

```sql
    SELECT category, user_id
    FROM product_spend p1
    WHERE user_id IN (SELECT user_id
                      FROM product_spend p2
                      WHERE p1.category = p2.category
                        AND p2.product = 'vacuum');
```

> **결과**:
>
> | category   | user_id |
> |------------|---------|
> | electronics| 178     |
> | electronics| 145     |

---

### mini quiz

```sql
    SELECT category, user_id, spend
    FROM product_spend p1
    WHERE spend > (SELECT AVG(spend)
                   FROM product_spend p2
                   WHERE p1.category = p2.category);
```

> 주어진 [데이터베이스](#sample-database)를 기반으로 위의 결과를 계산해보세요 :)

<details>
  <summary>정답과 해설</summary>

- 이 쿼리는 상관 서브쿼리(Correlated Subquery)를 사용하여,
  각 카테고리별로 평균 지출액보다 많은 금액을 지출한 사용자를 찾는 문제이며, 결과는 다음과 같습니다.

| category     | user_id | spend  |
|--------------|---------|--------|
| appliance	   | 123     | 299.99 |
| electronics	 | 156     | 249.90 |
| electronics	 | 145     | 189.00 |

- 즉, 각 사용자가 자신이 속한 카테고리의 평균 지출액보다 많은 금액을 지출한 경우만을 보여줍니다.

> 외부 쿼리 (Outer Query):
> SELECT category, user_id, spend는 product_spend 테이블에서 category, user_id, spend를 선택합니다.
> 이 쿼리는 각 사용자의 spend가 자신이 속한 category의 평균 지출액보다 더 많은지를 확인합니다.

> 내부 쿼리 (Inner Query):
> SELECT AVG(spend)는 spend의 평균 값을 계산합니다.
> FROM product_spend p2에서 p2는 내부 쿼리에서 사용되는 별칭입니다.
> WHERE p1.category = p2.category는 상관 관계를 나타냅니다.

> 상관 서브쿼리 (Correlated Subquery):
> 내부 쿼리에서 사용하는 p1.category는 외부 쿼리에서 선택된 행의 category 값을 참조합니다.
> 즉, 외부 쿼리의 각 행에 대하여 내부 쿼리가 실행되어야 합니다.
> 내부 쿼리는 각 category에 대한 평균 값을 계산합니다.

</details>

---
