# Spark 내부 동작 정리 — RDD, DAG, Shuffle, Catalyst

> W5M1 "How Spark Works Internally: RDD and DAG" 학습 + 후속 질문(narrow 파이프라이닝, broadcast join, constant folding)을 정리한 복습용 노트.

---

## 1. 왜 RDD가 필요했는가

MapReduce가 실제로 가진 두 가지 구체적인 문제:

1. **반복 연산의 중복 (Redundancy Problem)**
   k-means 같은 ML 알고리즘은 같은 데이터를 10~100번 반복해서 훑는다. MapReduce에서는 iteration마다 별도 Job이라, 매 iteration마다 HDFS에서 **다시 읽고 → 계산 → 다시 씀**. 100번 반복 = HDFS I/O 100번.

2. **데이터 재사용 불가 (No Data Reuse)**
   전처리한 결과를 빈도 분석 + 감성 분석 두 곳에 쓰고 싶어도, MapReduce는 각 Job이 독립적이라 전처리를 두 번 하거나 수동으로 디스크에 저장/재로드해야 함.

RDD가 답한 것: **인메모리 캐싱**(반복 재읽기 제거) + **여러 다운스트림 작업의 재사용**(같은 RDD를 여러 곳에서 참조 가능).

---

## 2. RDD란 무엇인가

- **정의**: Resilient Distributed Dataset. 클러스터 노드 전체에 분산된, 병렬 연산이 가능한 데이터 컬렉션.
- **핵심 속성 — 불변성(immutable)**: RDD는 절대 직접 수정되지 않음. `map`/`filter` 등을 걸면 항상 **새 RDD**가 생성됨.
- **불변성 → lineage로 이어지는 이유**:
  새 RDD는 "나는 어떤 부모 RDD에 어떤 연산을 적용해서 만들어졌다"는 포인터를 기록한다. 이 포인터 체인이 **lineage**. lineage가 있기 때문에 캐시 안 된 RDD도 필요하면 처음부터 재계산 가능 → 이것이 Spark의 fault tolerance 메커니즘(복제가 아니라 재계산).
- **생성 방법 3가지**:
  1. Parallelized collections (코드 내 리스트를 병렬화)
  2. External datasets (HDFS/S3 등 외부 파일)
  3. Existing RDDs (변환으로 파생)

---

## 3. Transformation vs Action — Lazy Evaluation

| | Transformation | Action |
|---|---|---|
| 예시 | `map`, `filter`, `flatMap`, `groupBy` | `count`, `collect`, `saveAsTextFile`, `reduce` |
| 실행 시점 | 즉시 실행 안 됨 (lineage에 기록만) | 호출되는 즉시 lineage 전체 실행 트리거 |
| 결과 | 새로운 RDD 생성 | RDD가 아닌 결과값 반환 or 외부 저장 |

```
RDD(textFile) --filter--> RDD(filter결과) --map--> RDD(map결과) --count()--> [실행!]
   (지연, 미실행)         (지연, 미실행)          (지연, 미실행)    (액션 호출 순간 전체 lineage 실행)
```

지연 평가의 이점: Spark가 실행 직전에 **전체 계획을 미리 보고** 최적화할 수 있음 (narrow 변환 파이프라이닝, 불필요한 shuffle 제거 등). MapReduce는 이런 전체 시야가 없어서 이게 원천적으로 불가능했음.

### ⚠️ 캐싱하지 않으면 액션마다 재계산된다 (실전 함정)

```python
rdd = sc.textFile("path_to_your_data.csv")
rdd_filtered = rdd.filter(lambda x: x[2] > 0)
rdd = rdd.map(parse_row).filter(lambda x: x is not None)
total_trips = rdd_filtered.count()          # 액션 1
all_records_count = rdd.count()             # 액션 2
```

`rdd_filtered`와 (재할당된) `rdd`는 둘 다 같은 조상(`sc.textFile(...)`)에서 뻗어나간 **서로 다른 lineage**. 조상 RDD에 `.cache()`가 안 걸려 있으면:

- 액션 1 실행 시 → CSV 파일을 **한 번** 디스크에서 읽음
- 액션 2 실행 시 → 캐시가 없으니 CSV 파일을 **또 한 번** 처음부터 읽음

→ 액션이 2개면 파일 읽기도 사실상 2번. `sc.textFile(...).cache()`로 시작했다면 두 번째 읽기는 메모리에서 바로 시작 가능했음.

---

## 4. Partition / Narrow vs Wide Transformation / Shuffle

- **Partition**: RDD를 구성하는 최소 물리 단위. **파티션 1개 = task 1개 = executor 위의 실행 단위 1개.**
- **Narrow transformation** (`map`, `filter`, `flatMap`): 입력 파티션 1개 → 출력 파티션 1개. 노드 간 데이터 이동 불필요.
- **Wide transformation** (`groupBy`, `reduceByKey`, `join`): 입력 파티션 1개 → 여러 출력 파티션에 기여 가능. 데이터를 노드 간 재배치해야 함 = **shuffle**.
- **Shuffle이 비싼 이유**: 네트워크로 실제 데이터 이동, 메모리 부족 시 디스크까지 거침. **shuffle 발생 지점 = Stage 경계.**

---

## 5. Spark 내부 실행 구조: Job → Stage → Task

```
사용자 코드(transformation들)
  → SparkContext에 lineage로 축적
  → 액션 호출
  → DAGScheduler: lineage를 shuffle 경계 기준으로 Stage로 분할
  → TaskScheduler: 각 Stage를 파티션 수만큼 Task로 분할, Executor에 배치
```

```
Job (액션 호출로 생성)
 ├─ Stage 1 (narrow 변환들, 셔플 없음 — 파이프라인)
 │   ├─ Task 1
 │   ├─ Task 2
 │   └─ Task 3
 │
 │   ── Shuffle 경계 ──
 │
 └─ Stage 2 (wide 변환 이후, 셔플 발생)
     ├─ Task 4
     ├─ Task 5
     └─ Task 6
```

- **DAGScheduler**: 논리적 lineage를 받아 shuffle 경계 기준으로 Stage로 쪼갬.
- **TaskScheduler**: Stage 내부의 각 파티션을 Task로 변환하고 실제 Executor 배치를 결정.

### Narrow 변환이 Task 안에서 파이프라인으로 합쳐지는 원리

RDD는 내부적으로 `compute()` 메서드를 가짐 — "부모 RDD의 iterator에서 레코드를 하나씩 꺼내 함수를 적용하고 다시 iterator로 돌려줌". `map`/`filter`로 만든 RDD(`MapPartitionsRDD`)는 데이터를 들고 있는 게 아니라 **"부모 RDD + 적용할 함수"** 정보만 가짐.

```
[입력 파티션] → map(f1) → filter(f2) → [출력 파티션]
   (Task 1 하나 안에서, record 단위로 즉시 처리, 중간 저장 없음)
```

`rdd.map(f1).filter(f2)`는 실제로 `f2가_통과된_것만( f1(레코드) )`처럼 **레코드 한 개씩** 함수 체인을 통과하며 즉시 계산됨. 왜 가능한가 — narrow 변환은 다른 파티션/executor의 데이터가 전혀 필요 없기 때문에, 다른 Task를 기다릴 필요도, 중간 결과를 디스크에 쓸 필요도 없음. 반대로 wide 변환은 다른 파티션의 데이터가 모여야 계산 가능 → 모든 Task가 shuffle write를 끝낼 때까지 기다리는 **barrier**가 강제로 생김 → 이 barrier가 Stage 경계.

**실전 함의**: Stage 개수를 줄이는 것 = shuffle 횟수를 줄이는 것. 코드에서 narrow 변환을 최대한 오래 연결하고 wide 변환(shuffle)을 최소화하면 성능이 좋아짐.

---

## 6. 왜 DAG가 필요했는가

MapReduce는 **Map → Reduce 딱 2단계 고정 파이프라인**. 그 이상 복잡한 로직을 하려면 Job을 체인으로 엮어야 하는데, 체인으로 엮인 Job들은:
- 전체를 아우르는 최적화 불가능 (Job1이 끝나야 Job2 시작, 서로의 사정을 모름)
- Job 사이 불필요한 대기 + 디스크 I/O
- 전체 데이터 흐름에 대한 가시성 없음

Spark는 이걸 **하나의 논리적 DAG**로 통째로 계획한 뒤 실행 → narrow 변환 파이프라이닝, 불필요한 shuffle 최소화 같은 **전역 최적화**가 가능해짐.

DAG(Directed Acyclic Graph) 이름의 의미:
- **Directed**: 엣지에 방향 있음 = 연산 순서 있음
- **Acyclic**: 순환 없음 = 한번 변환되면 되돌아갈 수 없음 (RDD 불변성과 직결되는 성질)
- **Graph**: 정점(RDD/연산)과 엣지(변환)의 조합

| 항목 | MapReduce의 한계 | Spark의 해법 | 결과 |
|---|---|---|---|
| Data Reuse | 재사용 불가, 재계산 필요 | RDD + 캐싱 | 반복 워크로드 가속 |
| Fault Tolerance | 복제 기반, 복구 느림 | RDD lineage | 경량, 효율적 복구 |
| Execution Model | 고정된 Map→Reduce | DAG 기반 실행 | 유연하고 최적화된 워크플로우 |
| Multi-Stage Jobs | 수동 체이닝, 비효율 | 전역 DAG 계획 | I/O 감소, 성능 향상 |

---

## 7. RDD vs DataFrame

RDD는 저수준 API — 최적화가 전부 개발자 책임. DataFrame은 그 위에 얹힌 고수준 API로, **Catalyst 옵티마이저**의 도움을 받음.

### Catalyst의 4단계
1. **Analysis**: 컬럼명·타입 실존 여부 검증
2. **Logical Optimization**: predicate pushdown(필터를 앞으로 당김), constant folding(상수 미리 계산) 등 규칙 기반 최적화
3. **Physical Planning**: 여러 실행 후보 중 비용 최저 선택 (예: join 전략으로 BroadcastHashJoin 선택)
4. **Code Generation**: 실제 JVM 바이트코드로 컴파일 (Tungsten 엔진이 메모리 관리·whole-stage codegen 담당)

### AQE (Adaptive Query Execution)
Catalyst가 실행 *전에* 한 번만 계획을 세우는 것과 달리, AQE는 Stage가 하나씩 끝날 때마다 **실제 데이터 통계**(파티션 크기 등)를 보고 다음 Stage 계획을 실시간 조정. 예: join할 테이블이 예상보다 작다는 게 shuffle 이후 드러나면, 계획된 SortMergeJoin을 실행 중에 BroadcastHashJoin으로 전환 가능. 정적 Catalyst 최적화는 실행 전 데이터 크기를 모르므로 이건 못 함.

### RDD는 왜 못 받는가
RDD의 `lambda`는 Spark 입장에서 **불투명한 함수**라 내부를 들여다볼 수 없음. DataFrame의 `col("price") > 1 + 2`는 **구조화된 expression tree**(`>`, `+`, `1`, `2`, `col`)로 표현되어 Spark가 의미를 이해하고 재배열·최적화 가능.

### RDD가 아직 유용한 경우
- 원본 텍스트 스트림, 바이너리 파일, 커스텀 포맷처럼 구조화되지 않은 데이터
- 파티셔닝·메모리 사용을 아주 세밀하게 직접 제어하고 싶을 때
- 커스텀 연산 정의, 복잡한 처리 로직 디버깅

---

## 8. Shuffle Join vs Broadcast Join

### 왜 데이터 이동이 필요한가
Join을 하려면 같은 키를 가진 두 행이 **물리적으로 같은 executor**에 있어야 함. 원래 두 테이블은 그렇게 배치되어 있지 않으므로 누군가는 이동해야 함.

### 일반 Join (양쪽 다 셔플)
```
Executor 1 [A1, B1]  ⇄  Executor 2 [A2, B2]
        (join key 해시 기준으로 서로 교차 이동 = shuffle)
```
A(큰 테이블)와 B(작은 테이블) 둘 다 join key 해시로 재파티셔닝 → 같은 키를 가진 행들이 같은 파티션으로 모임 → 로컬에서 join 완성. **문제: 큰 테이블 A까지 통째로 shuffle write/read 해야 함.**

### Broadcast Join (B만 복사)
```
                B (원본)
              ↙         ↘
   Executor1[A1 그대로 + B 전체 사본]   Executor2[A2 그대로 + B 전체 사본]
```
B가 충분히 작으면, A를 옮기는 대신 **B를 통째로 모든 executor에 복사**(보통 해시 테이블 형태로 메모리에 로드). 그러면:
- A는 원래 위치에서 전혀 움직이지 않음
- join 자체가 "A의 각 행을 보면서 로컬 B 사본에서 키를 찾는" **순수 map 연산**이 됨 → narrow transformation처럼 처리됨
- **shuffle 자체가 사라지므로 stage가 하나 줄어듦**

### 언제 자동으로 선택되는가
Spark 옵티마이저가 통계상 작은 테이블이라고 판단하면 자동으로 BroadcastHashJoin 선택 (기본 임계값 `spark.sql.autoBroadcastJoinThreshold`, 기본 10MB). `broadcast(df)` 힌트로 명시적으로 지정 가능.

**실전 예**: 택시 운행 기록(A, 수천만 건) + 날짜별 날씨 정보(B, 수백 건) join → B가 작으므로 broadcast join 대상.

---

## 9. Constant Folding

컴파일러/쿼리 옵티마이저의 오래된 최적화 기법: **결과가 이미 정해진 계산은 실행 시점이 아니라 계획(plan) 시점에 미리 해버림.**

```python
df.filter(col("price") > 1 + 2)
```
`1 + 2`는 데이터와 무관하게 항상 `3`. 최적화 없이 실행하면 모든 행마다 `1+2`를 반복 계산하는 꼴. Catalyst의 Logical Optimization 단계의 `ConstantFolding` 규칙이 실행 계획을 짤 때 미리:

```python
df.filter(col("price") > 3)
```
로 바꿔놓음 → 실행 시점엔 단순 비교만 남고 덧셈 자체가 사라짐.

predicate pushdown, constant folding, broadcast join — 셋 다 **"실제 데이터를 만지기 전에 계획 단계에서 손해를 줄여놓는다"**는 같은 철학.

**RDD에서는 왜 안 되는가**: `filter(lambda x: x[0] > 1+2)`는 불투명한 함수라 Spark가 내부에 상수 연산이 있는지 알 수 없음 → 매 행마다 그냥 통째로 실행됨. DataFrame의 `col("price") > 1 + 2`는 구조화된 expression tree라서 Spark가 상수 부분만 찾아내 미리 계산 가능.

---

## 한 줄 요약 모음

- **RDD**: 불변 + lineage → 캐싱으로 재사용, 재계산으로 장애 복구
- **Lazy evaluation**: 변환은 기록만, 액션이 실행 트리거
- **Narrow 변환**: 파티션 로컬 계산 → Task 안에서 파이프라인으로 융합
- **Wide 변환**: 파티션 간 데이터 이동 필요 → shuffle → 새 Stage 경계
- **DAG**: 전체 lineage를 미리 보고 전역 최적화 (MapReduce는 이게 불가능)
- **DataFrame > RDD (대부분의 경우)**: 구조화된 expression tree라서 Catalyst가 predicate pushdown, constant folding, join 전략 선택 같은 최적화를 대신 해줌
- **Broadcast join**: 큰 테이블을 안 옮기려고 작은 테이블을 통째로 복사 → shuffle 제거 → stage 감소
- **AQE**: 실행 중 실제 데이터 통계를 보고 계획을 동적으로 조정 (정적 Catalyst 최적화의 한계를 보완)
