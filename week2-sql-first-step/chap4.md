# 4장 - 데이터의 추가, 삭제, 갱신

### 목차
- INSERT
- DELETE
- UPDATE
- 논리삭제, 물리삭제 


## 16강. 행 추가하기 - INSERT

---

```sql
INSERT INTO 테이블명 VALUES(값1, 값2, ...)
```
INSERT 명령을 사용해 테이블의 행 단위로 데이터 추가

INSERT INTO 뒤에 테이블 명, 
VALUES 뒤에 값을 지정.

이때, 해당 열의 데이터 형식에 맞춰서 지정해야 한다.

```sql
INSERT INTO 테이블명 (열1, 열2, ...) VALUES (값1, 값2, ...)
```
지정한 열에 값을 넣어 행을 추가하기 

NOT NULL로 설정이 되어있는 열에 대해서는, VALUES에 NULL을 넣어 INSERT를 실행시킬 수 없다.

```sql
INSERT INTO 테이블명 (열1, 열2, ...) VALUES (값, DEFAULT, ...)
```
DEFAULT 값이 설정되어있는 열에 대해서는, VALUES에 DEFAULT 키워드를 추가해 기본 값으로 추가할 수 있다.

만약, 값을 넣지 않더라도 지정된 DEFAULT값이 들어간다.

## 17강. 삭제하기 - DELETE 

---

```sql
DELETE FROM 테이블명 WHERE 조건식
```
조건식에 걸맞는 테이블의 행을 삭제한다. 

```sql
DELETE FROM 테이블명
```
테이블의 모든 데이터가 삭제된다. 
WHERE 를 생략한 경우, 모든 행을 대상으로 동작한다. 

```sql
DELETE FROM 테이블명 WHERE no = 3;
```
no 열의 값이 3인 행을 삭제한다.

DELETE에서는 SELECT에서 사용한 조건식을 지정할 수 있지만, ORDER BY는 사용할 수 없다.

(삭제의 순서는 중요하지 않아서)

## 18강. 데이터 갱신하기 - UPDATE

---

```sql
UPDATE 테이블명 SET 열1 = 값1, 열2 = 값2, ... WHERE 조건식
```
테이블의 셀 값을 갱신
UPDATE에서는, SET구를 사용하여 갱신할 열과 값을 지정한다. 

```sql
UPDATE 테이블명 SET no = no + 1
``` 
갱신 대상이 되는 열이어도 갱신할 값을 열이 포함된 식으로 표현할 수 있다.

```sql
UPDATE sample SET no = no + 1, a = no;
UPDATE sample SET a = no, no = no + 1;
```
콤마로 구분된 갱신 식의 순서가 서로 다를 경우, 명령을 실행한 결과가 같을까?

데이터베이스 제품마다 결과가 다르다. Oracle 은 같고, MySQL은 다르다.

**MySQL**에서는 순서대로 대입된다.

현재 no = 3, a = ABC 라면
위 구문은 no = 3 + 1 인 4가 대입된 후, 해당 4가 a에도 대입되어 no = 4, a = 4가 된다.

아래 구문은 a = 3 이 먼저 대입된 후, no = no + 1 이 실행되어 a = 3, no = 4 가 된다.

마치 SET no = no + 1 , SET a = no 두 구문을 순서대로 실행한 것과 같은 결과를 갖는다. 

**오라클** 에서는 SET 구에 기술한 식의 순서가 처리에 영향을 주지 않는다. 

SET에 위치한 no는 모두 갱신 이전의 값을 반환하기 때문에, 위아래 구문 모두 no = 4, a = 3가 된다. 

```sql
UPDATE 테이블명 SET 열명1 = 값1, 열명2 = 값2, ...WHERE 조건식
```
갱신할 열을 여러 개 지정할 수 있다. 

```sql
UPDATE 테이블명 SET a = NULL;
```
NOT NULL 제약이 걸려있지 않다면, NULL값으로 갱신을 할 수 있다.


## 19강. 물리삭제와 논리삭제 

---

SQL 명령에 관한 내용이 아닌, 시스템을 구축할 때 사용하는 용어

물리삭제 : DELETE로 직접 데이터를 삭제

논리삭제 : 삭제 플래그 열을 준비해두고, 해당 열을 UPDATE하는 방식 

## Quiz

---


```sql
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    price DECIMAL(10,2),
    discount_price DECIMAL(10,2) DEFAULT NULL
);
```
1. 가격이 50,000원 이상인 제품에 대해 10% 할인을 적용하고,
2. 그 결과를 discount_price 컬럼에 반영하는 SQL을 작성하라.


