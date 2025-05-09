# 아키텍처

두뇌 : MySQL 엔진
손발 : 스토리지 엔진 (핸들러 API 구현체) - InnoDB, MyISAM

## MySQL의 전체 구조

### MySQL 엔진
커넥션 핸들러 : 클라이언트의 접속 및 쿼리 요청
SQL 파서,
전처리기, 
옵티마이저 : 쿼리의 최적화 

### 스토리지 엔진
디스크 스토리지로부터 데이터를 읽고 저장하는 역할
플러그인 형태로, 여러개를 동시에 사용가능하다.
MySQL엔진에서 스토리지에 읽기/쓰기 요청하는것을 핸들러 요청이라고 하고, 여기서 사용되는 API가 핸들러 API

**데이터를 읽기/쓰기 이외의 모든 작업을 MySQL 엔진에서 처리한다고 이해**

## MySQL 스레딩 구조

- 포그라운드 스레드: 
- 클라이언트 스레드, 사용자 스레드 라고도 불리며, 최대 스레드 개수를 설정할 수 있다.
- 주로 서버에 접속한 클라이언트 수 만큼 생성되며, 요청 쿼리를 처리한다. 

- 백그라운드 스레드:
- 로그를 디스크로 기록하는 스레드, 버퍼의 데이터를 디스크로 내려쓰는 작업을 처리하는 쓰기 쓰레드 등 다양함
- 읽기 스레드는 사용자 스레드에서 주로 처리되지만, 쓰기 스레드는 백그라운드에서 아주 많은 작업을 처리하기 때문에 충분한 양을 설정하는것이 중요

## 메모리 할당 및 사용구조 

- 글로벌 / 로컬 : MySQL 서버 시작 시 운영체제로부터 할당된다. 
- 많은 스레드가 공유하는지의 여부로 구분한다.

- 글로벌
  - 클라이언트의 스레드 수와 무관하며, 모든 스레드에 의해 공유되는 메모리 영역
  - 테이블 캐시, 버퍼 풀 등이 해당된다.
- 로컬
  - 세션 메모리 영역, 클라이언트 메모리 영역 등으로 구분한다.
  - 클라이언트 스레드가 사용하며, 절대 공유되지 않는다.
  - 쿼리 별 필요할때만 공간이 할당되고, 필요하지 않은경우 할당이 안되기도 함
  - 정렬 버퍼, 조인 버퍼 등이 해당된다. 

## 플러그인 스토리지 엔진 모델
스토리지 엔진 뿐만 아니라, 검색어 파서, 사용자 인증 등도 플러그인으로 구현되어 제공이 가능하다.
데이터의 읽기/쓰기는 단순해보여도 처리방식이 정말 다양하기 때문에, 핸들러 API를 구현한 스토리지 엔진도 다양하다.
문제점
1. MySQL 서버와만 통신, 플러그인끼리는 통신할 수 없음
2. 서버의 변수나 함수 직접호출 (캡슐화X)
3. 상호 의존관계 설정이 어려워 초기화가 어려움

## 컴포넌트
8.0 버전부터, 기존의 플러그인을 대체하기 위해 지원된다.

## 쿼리 실행 구조
MySQL 엔진
쿼리 파서 -> 전처리기 -> 옵티마이저 -> 실행 엔진 -> 스토리지 엔진
1. 쿼리 파서 : 쿼리 문장을 토큰(MySQL이 인식할 수 있는 최소 단위의 어휘나 기호) 으로 분리후 트리구조로 배치
2. 전처리기 : 파서 트리(퀴리 파서에서 만들어진 트리) 를 기반으로, 쿼리 문장에 구조적 문제가 있는지 확인
3. 옵티마이저 : 쿼리 실행 최적화. 쿼리 변환, 비용 최적화, 실행 계획 수립 등의 두뇌 역할을 담당한다.
4. 실행 엔진 : 중간 관리자 역할로, 핸들러사이의 연결을 담당한다.
5. 핸들러(스토리지 엔진) : 실무자 역할로, 데이터를 저장 및 읽어온다. 

## 쿼리 캐시
기존 쿼리 캐시는 동일 SQL 쿼리 실행 결과를 저장해 빠른 실행 결과를 반환했지만,
테이블 데이터 변경 시 삭제할 양이 너무 많아져 8.0 버전부터 삭제 

## 스레드 풀
엔터프라이즈 : MySQL 서버가 직접 제공
커뮤니티 에디션 : Percona Server에서 스레드풀 플러그인 설치

스레드 풀 사이즈만큼 생성해놓고 재활용, 
기본적으로 CPU의 코어 수 만큼이지만 별도 설정이 가능하다. 

모든 스레드가 일을 한다면, 
기다리거나, 설정된 시간보다 밀리면 새로 생성한다. 

## 트랜잭션 지원 메타데이터

메타데이터 : 테이블의 구조 정보, 스토어드 프로그램 등의 정보 등의 정보들 (다른 데이터를 정의하고 기술하는 데이터)

초기에는 이를 파일로 저장하다가, 현재는 InnoDB 테이블에 시스템 테이블로 저장함으로써
기존 파일로 저장시 지원하지 못한 트랜잭션을 지원해 더욱 안정성 있게 사용가능
스토리지 엔진의 메타 정보는 SDI 파일을 사용하고, InnoDB 테이블 구조와 호환이 가능하다. 


## MyISAM 스토리지 엔진 아키텍처
### 키 캐시 
키 버퍼라고도 하며, 인덱스만을 대상으로 쓰기 작업에 대해서만 버퍼링
히트율 99% 이상 권장, 해당 수치로 키 캐시 설정
추가로 할당한다면, 어떤 인덱스를 캐시할지도 설정하기

### 운영체제의 캐시 및 버퍼
인덱스는 키 캐시로 빠르게 검색할 수 있지만, 테이블데이터는?
운영체제의 디스크 읽기 또는 쓰기 작업을 사용 => 운영체제가 사용할 캐시공간 비워두기 

### 데이터 파일과 프라이머리 키 (인덱스) 구조 
MyISAM : Heap 공간 사용 -> Insert 순으로 데이터 파일 저장, 
MyISAM 테이블에 저장되는 레코드는 모두 ROWID 라는 물리적인 주솟값을 가짐, 가변길이와 고정 길이 두 방법으로 저장가능
- 가변 길이 : 최대로 가지는 레코드가 한정된 테이블, ROWID로 4byte 사용 
- 고정 길이 : 2~7byte 가변, 첫번쨰는 ROWID 길이저장하고 나머지에 값 저장 


# 문제!
```
1. MySQL 서버에서 사용자의 쿼리를 분석하고 실행 계획을 수립하는 부분은 어디인가요?

(a) 스토리지 엔진

(b) 쿼리 캐시

(c) SQL 파서 및 옵티마이저

(d) 데이터 버퍼

2. MySQL의 스토리지 엔진이 담당하는 역할은 무엇인가요?

(a) 데이터 저장 및 읽기/쓰기

(b) 사용자 인증

(c) SQL 문법 검사

(d) 쿼리 캐싱

3. SQL 옵티마이저(Optimizer)의 주요 기능은 무엇인가요?

(a) SQL 쿼리를 가장 효율적으로 실행할 방법을 결정

(b) 사용자 로그인 인증

(c) 데이터 백업 수행

(d) 데이터 복구

4. MySQL에서 SQL 문장을 실행할 때 먼저 처리되는 순서는?

(a) SQL 파서 → 옵티마이저 → 실행 엔진

(b) 실행 엔진 → SQL 파서 → 옵티마이저

(c) 스토리지 엔진 → SQL 파서 → 실행 엔진

(d) 옵티마이저 → SQL 파서 → 실행 엔진

```
