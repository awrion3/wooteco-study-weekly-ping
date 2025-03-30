# Real MySQL 트랜잭션과 잠금

## 트랜잭션(Transaction)

- 데이터의 정합성을 보장하기 위한 기능
- 작업의 완전성을 보장 (전체 적용(COMMIT) 또는 원상태 복구(ROLLBACK))
- MyISAM, MEMORY 스토리지 엔진: 트랜잭션 미지원
- InnoDB 스토리지 엔진: 트랜잭션 지원
- 트랜잭션이 활성화된 범위를 최소화해야 함 (네트워크 작업 포함 X)

## MySQL 엔진의 잠금

### MySQL 엔진 레벨 잠금

- **글로벌 락** (`FLUSH TABLES WITH READ LOCK`)
    - MySQL 서버 전체 잠금
    - `mysqldump`로 일관된 백업을 위해 사용
    - 웹 서비스 환경에서는 사용 지양
    - MySQL 8.0부터 백업 락 도입 (DML 허용, DDL 제한)
- **테이블 락** (`LOCK TABLES ... [READ | WRITE]`, `UNLOCK TABLES`)
    - 개별 테이블 잠금 (특별한 경우가 아니면 사용 X)
    - 묵시적 테이블 락: DML 시 자동 잠금 및 해제
- **네임드 락** (`GET_LOCK()`, `RELEASE_LOCK()`)
    - 특정 문자열에 대해 잠금 설정
- **메타데이터 락**
    - 테이블/뷰 변경 시 자동 적용 (명시적 해제 불가)

### InnoDB 스토리지 엔진 잠금

- **레코드 락**: 특정 레코드만 잠금
- **갭 락**: 레코드 사이 간격 잠금 (INSERT 방지)
- **넥스트 키 락**: 레코드 락 + 갭 락 (레플리카 일관성 유지 목적)
    - 갭 락으로 인해 데드락 발생 가능 → ROW 포맷 사용 권장
- **자동 증가 락**
    - `AUTO_INCREMENT` 값 증가 시 테이블 단위의 짧은 잠금
    - 증가한 값은 줄어들지 않음
- **인덱스와 잠금**
    - 인덱스가 없으면 테이블 풀 스캔 및 전체 잠금 발생
    - 동시성 향상을 위해 적절한 인덱스 설계 필요

## MySQL의 격리 수준

| 격리 수준 | DIRTY READ | NON-REPEATABLE READ | PHANTOM READ |
|-----------|-----------|-------------------|--------------|
| READ UNCOMMITTED | O | O | O |
| READ COMMITTED | X | O | O |
| REPEATABLE READ (기본) | X | X | O (InnoDB: X) |
| SERIALIZABLE | X | X | X |

### READ UNCOMMITTED
- 트랜잭션 격리 수준으로 적절하지 않음
- **DIRTY READ**: 완료되지 않은 트랜잭션의 데이터를 읽을 수 있음

### READ COMMITTED
- **NON-REPEATABLE READ**: 동일 트랜잭션 내에서 같은 SELECT 결과가 달라질 수 있음

### REPEATABLE READ (기본값)
- **MVCC (Multi Version Concurrency Control)**
    - 변경된 레코드의 백업(Undo 로그)을 유지하여 트랜잭션 격리 보장
- 트랜잭션이 오래 지속되면 Undo 로그가 커질 수 있음 → 성능 저하 가능

### SERIALIZABLE
- 가장 엄격한 격리 수준 (성능 저하 가능)
- 읽기/쓰기 작업 중인 레코드는 다른 트랜잭션에서 접근 불가

## 논의
- MySQL이 기본적으로 REPEATABLE READ를 사용하지만, 많은 기업이 READ COMMITTED를 선택하고 있다고 한다. 그 이유는 무엇일까?

<details>
<summary>답변</summary>

> 이유 1. 동시성(Concurrency) 향상 & 데드락(Deadlock) 감소
- REPEATABLE READ는 동일 트랜잭션에서 같은 SELECT 결과를 보장하기 위해 갭 락(Gap Lock)을 사용할 수 있음.
갭 락이 활성화되면, 특정 범위 내에서 새로운 데이터 삽입이 차단될 수 있어 데드락(Deadlock)이 발생할 가능성이 높아짐.
- READ COMMITTED는 개별 SELECT 실행 시점마다 최신 데이터를 읽으므로, 갭 락을 최소화할 수 있고, 동시성(concurrency)이 향상됨.

> 이유 2. 최신 데이터 제공 (READ COMMITTED는 최신 데이터를 읽음)
- READ COMMITTED는 트랜잭션 중간에 다른 트랜잭션이 커밋한 데이터를 바로 반영함.
- REPEATABLE READ는 트랜잭션 시작 시점을 기준으로 데이터를 유지하므로, 트랜잭션 도중 최신 데이터가 반영되지 않음.

</details>

## 퀴즈
1. MySQL에서 제공하는 잠금 방식 중, 스토리지 엔진 내부에서 레코드 기반의 잠금 방식을 탑재하고 있는 것은?
    1. MyISAM
    2. MEMORY
    3. InnoDB
    4. CSV

<details>
<summary>정답</summary>
> 정답: 3
> (InnoDB 스토리지 엔진은 MySQL에서 제공하는 잠금과는 별개로 스토리지 엔진 내부에서 레코드 기반의 잠금 방식을 탑재하고 있다.)
</details>

2. MySQL에서 제공하는 잠금 방식 중, 특정 문자열에 대해 잠금 설정을 제공하는 것은?
    1. 글로벌 락
    2. 테이블 락
    3. 네임드 락
    4. 메타데이터 락

<details>
<summary>정답</summary>
> 정답: 3
> (네임드 락은 MySQL에서 제공하는 잠금 방식 중, 특정 문자열에 대해 잠금 설정을 제공하는 것이다.)
</details>

3. MySQL에서 제공하는 잠금 방식 중, 테이블/뷰 변경 시 자동 적용되는 것은?
    1. 글로벌 락
    2. 테이블 락
    3. 네임드 락
    4. 메타데이터 락

<details>
<summary>정답</summary>
> 정답: 4
> (메타데이터 락은 MySQL에서 제공하는 잠금 방식 중, 테이블/뷰 변경 시 자동 적용되는 것이다.)
</details>

4. MySQL에서 제공하는 트랜잭션 격리 수준 중, DIRTY READ가 발생할 수 있는 것은?
    1. READ UNCOMMITTED
    2. READ COMMITTED
    3. REPEATABLE READ
    4. SERIALIZABLE

<details>
<summary>정답</summary>
> 정답: 1
> (DIRTY READ는 READ UNCOMMITTED에서 발생할 수 있다.)
</details>

5. MySQL에서 제공하는 트랜잭션 격리 수준 중, NON-REPEATABLE READ가 발생할 수 있는 것은?
    1. READ UNCOMMITTED
    2. READ COMMITTED
    3. REPEATABLE READ
    4. SERIALIZABLE

<details>
<summary>정답</summary>
> 정답: 2
> (NON-REPEATABLE READ는 READ COMMITTED에서 발생할 수 있다.)
</details>
