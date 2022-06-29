# 6장: DML 튜닝

## 6.2 Direct Path I/O 활용
- 대량 데이터를 처리할 때 버퍼캐시를 경유하는 I/O 메커니즘은 오히려 성능을 떨어뜨릴 수 있다.
- 버퍼캐시를 경유하지 않고 곧바로 데이터 블록을 읽고 쓸 수 있는 기능을 배운다.

### 6.2.1 Direct Path I/O
일반적인 블록 I/O는 자주 읽는 블록에 대한 반복적인 I/O를 줄임으로써 성능을 높이기 위해 버퍼캐시를 이용하지만,
대량 데이터를 읽고 쓸 때 버퍼캐시를 탐색한다면 성능이 좋지 않다.

- 데이터를 변경할 때도 먼저 블록을 버퍼캐시에서 찾는다. 변경된 블록들을 주기적으로 찾아 데이터 파일에 반영해 준다.

- 버퍼캐시에서 블록을 찾을 가능성이 없다.
  - Full Scan으로 수행되어 읽어 들인 대용량 데이터는 재사용성이 낮다.
    - 이런 데이터 블록들이 버퍼 캐시를 점유하면 다른 프로그램 성능에도 나쁜 영향이 생긴다.
- 건건이 디스크로부터 버퍼캐시에 적재하고서 읽어야 하는 부담이 있다.
  
#### **Direct Path I/O 기능이 동작하는 경우**
1. **`병렬 쿼리`로 Full Scan을 수행할 때**
2. **병렬 DML을 수행할 때**
3. **Direct Path Insert를 수행할 때**
4. Temp 세그먼트 블록들을 읽고 쓸 때
5. direct 옵션을 지정하고 export를 수행 할 때
6. nocache 옵션을 지정한 LOB 컬럼을 읽을 때

> **병렬 쿼리**
> 
> ```sql
> select /*+ full(t) parallel(t 4) */ * from big_table t;
> select /* index_ffs(t big_table_x1) parallel_index(t big_table_x1 4) */ count(*) from big_table t;
> ```
> 위 쿼리처럼 parallel 또는 parallel_index 힌트를 사용하면 지정한 병렬도(4) 만큼 병렬 프로세스가 떠서 동시에 작업을 진행한다.
> 병렬처리 뿐만 아니라 디스크로부터 버퍼캐시에 적재하는 부담도 없어 수십 배 빨라진다.

### 6.2.2 Direct Path Insert
- 일반적인 INSERT 방식
  1. 데이터를 입력할 수 있는 블록을 Freelist에서 찾는다.
  2. Freelist에서 할당받은 블록을 버퍼캐시에서 찾는다.
  3. 버퍼캐시에 없으면 데이터파일에서 읽어 버퍼캐시에 적재한다.
  4. INSERT 내용을 Undo 세그먼트에 기록
  5. INSERT 내용을 REDO 로그에 기록
   > **Freelist**: HWM(High-Water Mark) 아래쪽에 있는 블록 중 데이터 입력이 가능한 블록을 목록으로 관리
   
- Direct Path Insert 방식 (일반적인 INSERT 방식 보다 빠르다)
  1. INSERT ... SELECT 문에 append 힌트 사용
  2. parallel 힌트를 이용해 병렬 모드로 INSERT
  3. direct 옵션을 지정하고 SQL*Loader(sqlldr)로 데이터 적재
  4. CTAS(create table ... as select)문 수행

- Direct Path Insert 방식이 빠른 이유
  1. Freelist를 참조하지 않고 `HWM` 바깥 영역에 데이터를 순차적으로 입력
  2. 블록을 버퍼캐시에서 탐색하지 않는다.
  3. 버퍼캐시에 적재하지 않고, 데이터 파일에 직접 기록한다.
  4. Undo 로깅을 안 한다.
  5. Redo 로깅을 안 하게 할 수 있다. (테이블에 nologging 모드로 전환한 상태에서) 
  ```SQL
  alter table t NOLOGGING;
  ```

#### **Direct Path Insert 사용 시 주의할 점**
1. 성능은 빨라지지만 Exclusive 모드 TM Lock이 걸려 커밋하기 전까지 다른 트랜잭션은 해당 테이블에 DML을 수행하지 못한다. 트랜잭션이 빈번한 주간에 이 옵션을 사용하는 것은 절대 금물.
2. HWM 바깥 영역에 입력하므로 테이블에 여유 공간이 있어도 재활용하지 않는다. 여유 공간이 생겨도 이 방식으로 INSERT를 수행하기 때문에 테이블은 사이즈가 줄지 않고 계속 늘어간다.
   - Range 파티션 테이블이면 파티션 DROP 방식으로 데이터를 지워줘야한다.
   - 비파티션 테이블이면 주기적으로 Reorg 작업을 수행해야한다.

### 6.2.3 병렬 DML
INSERT는 append 힌트로 Direct Path Write 방식으로 유도가 가능하지만, UPDATE와 DELETE는 불가능하기 때문에 병렬 DML로 처리하면된다.

- 병렬 DML 활성화
    ```SQL
    alter session enable parallel dml;
    ```
> **두 단계 전략**
> 
> DML 문에 두 단계 전략을 사용한다.
>
> `Consistent` 모드로 대상 레코드를 찾고
> `Current` 모드로 추가/변경/삭제한다.

- 병렬 힌트를 제대로 기술했는데, 실수로 병렬 DML을 활성화하지 않은 경우
  - 대상을 찾는 `Consistent` 모드에서는 병렬로 진행
  - 추가/변경/삭제하는 `Current` 모드에서는 QC가 혼자 담당하므로 병목 발생
  - QC(Query Coordinator)는 병렬로 처리할 수 없거나 병렬로 처리하도록 지정하지 않은 작업을 처리

## 6.3 파티션을 활용한 DML 튜닝
- 파티션을 이용하면 대량 추가/변경/삭제 작업을 빠르게 처리할 수 있다.
- 파티션 개념

### 6.3.1 테이블 파티션
- 파티셔닝(Partitioning)
  - 테이블 또는 인덱스 데이터를 특정 컬럼(파티션 키) 값에 따라 별도 세그먼트에 나눠서 저장하는 것
    - 예시1) 계절별로 옷을 관리하면 외출 시 필요한 옷을 쉽고 빠르게 찾는다.
    - 예시2) 월별, 분기별, 연별로 분할해서 저장해 두면 빠르게 조회할 수 있고, 관리가 쉽다.
  

- 파티션이 필요한 이유
  - 관리적 측면: 파티션 단위 백업, 추가, 삭제, 변경 → 가용성 향상
  - 성능적 측면: 파티션 단위 조회 및 DML, 경합 또는 부하 분산

- 파티션의 종류   
  - Range 
  - 해시
  - 리스트 

#### Range 파티션
```SQL
-- 파티션 생성 예제
create table 주문 (주문번호 number, 주문일자 varchar2(8),  고객ID varchar2(5), ...)
partition by range(주문일자) (
    partition P2017_Q1 values less than ('20170401'),
    partition P2017_Q2 values less than ('20170701'),
    partition P2017_Q3 values less than ('20171001'),
    partition P2017_Q4 values less than ('20180101')
);
```

- Full Scan 방식으로 조회할 때 전체가 아닌 일부 파티션 세그먼트만 읽고 멈출 수 있어 성능을 크게 향상한다. 
- `클러스터`, `IOT`와 마찬가지로 데이터가 흩어지지 않고 물리적으로 인접하도록 저장하는 클러스터링 기술에 속한다.
- 과거 데이터가 저장된 파티션만 백업하고 삭제하는 등 데이터 관리 작업 또한 효율적이고 빠르게 수행할 수 있다.

> `클러스터`: 데이터를 블록 단위로 모아 저장
> 
> `IOT`: 데이터를 정렬된 순서로 저장하는 구조

**파티션 테이블에 대한 SQL 향상 원리**
- Pruning
  - 실행 시점에 조건절을 분석해서 읽지 않아도 되는 파티션 세그먼트를 액세스 대상에서 제외하는 기능
  - 인덱스로 액세스할 수 있지만, 파티션 Pruning을 이용한 스캔보다 훨씬 느리다.
  

#### 해시 파티션
```SQL
-- 파티션 생성 예제
create table 고객 (고객ID varchar2(5), 고객명 varchar2(10), ...)
partition by hash(고객ID) partitions 4;
```
파티션 키 값을 해시 함수에 입력해서 반환받은 값이 같은 데이터를 같은 세그먼트에 저장하는 방식이다.

파티션 개수는 사용자가 결정하고 데이터를 분산하는 알고리즘은 오라클 내부 해시함수가 결정한다.

> 해시 알고리즘 특성상 등치(=) 조건 또는 IN-List 조건으로 검색할 때만 파티션 Pruning이 작동한다.

#### 리스트 파티션
```SQL
-- 파티션 생성 예제
crate table 인터넷매물 (물건코드 varchar2(5), 지역분류 varchar2(4), ...)
partition by list(지역분류) (
    partition P_지역1 values ('서울'),
    partition P_지역2 values ('경기', '인천'),
    partition P_지역3 values ('부산', '대구', '대전', '광주'),partition P_지역4 values (DEFAULT)
);
```
사용자가 정의한 그룹핑 기준에 따라 데이터를 분할 저장하는 방식이다.

Range 파티션에선 값의 순서에 따라 저장할 파티션이 결정되지만, 리스트 파티션에서는 순서와 상관없이 불연속적인 값의 목록에 의해 결정된다.

### 6.3.2 인덱스 파티션
- 테이블 파티션
  - 비파티션 테이블(Non-Partitioned Table)
  - 파티션 테이블(Partitioned Table)
- 인덱스
  - 로컬 파티션 인덱스(Local Partitioned Table)
    - 테이블 파티션과 인덱스 파티션이 1:1 대응 관계가 되도록 오라클이 자동으로 관리하는 파티션 인덱스
  - 글로벌 파티션 인덱스(Global Partitioned Table)
    - 로컬이 아닌 파티션 인덱스
    - 테이블 파티션과 독립적인 구성을 갖는다. 
  - 비파티션 인덱스(Non-Partitioned Table)

#### **로컬 파티션(Local Partitioned) 인덱스**

#### **글로벌 파티션(Global Partitioned) 인덱스**

#### **비파티션(Non-Partitioned) 인덱스**

#### **Prefixed vs. Nonprefixed**

#### **중요한 인덱스 파티션 제약**