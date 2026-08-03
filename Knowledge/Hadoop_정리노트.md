# Apache Hadoop 정리노트

> W3 Introduction to Apache Hadoop 자료 + 심화 Q&A 정리

---

## 목차

1. [왜 Hadoop이 필요했나](#1-왜-hadoop이-필요했나)
2. [4개 모듈과 3계층 구조](#2-4개-모듈과-3계층-구조)
3. [HDFS 설계 가정 6가지](#3-hdfs-설계-가정-6가지)
4. [HDFS 아키텍처 상세](#4-hdfs-아키텍처-상세)
5. [YARN](#5-yarn)
6. [MapReduce](#6-mapreduce)
7. [Erasure Coding과 Reed-Solomon](#7-erasure-coding과-reed-solomon)
8. [Erasure vs Error, 그리고 체크섬](#8-erasure-vs-error-그리고-체크섬)
9. [데몬과 물리 서버의 관계](#9-데몬과-물리-서버의-관계)
10. [RS-k-m-cellsize 표기법](#10-rs-k-m-cellsize-표기법)
11. [미션 실무 힌트](#11-미션-실무-힌트)

---

## 1. 왜 Hadoop이 필요했나

### 물리적 제약: 디스크 용량 vs 전송 속도

디스크 용량은 기하급수적으로 늘었지만 **전송 속도는 그만큼 늘지 않았다.**

- 디스크 100MB/s 기준, 1TB 순차 읽기 → 약 3시간
- 10TB로 늘어도 속도는 그대로 → 30시간

### 선택지 두 가지

| | Scale-up | Scale-out |
|---|---|---|
| 방식 | 더 비싼 단일 서버 | 값싼 서버 여러 대에 분산 |
| 비용 | 용량당 초선형 증가 | 선형 증가 |
| 한계 | 결국 한 대의 I/O 한계 | 노드 추가로 확장 |

Hadoop은 **Scale-out**을 택했고, 그 순간 따라오는 대가:

> 서버 1,000대 규모에서는 **고장이 예외가 아니라 상수**다. 연간 고장률 4%인 장비 1,000대면 매주 한 대씩 죽는다.

→ 하드웨어 신뢰성 대신 **소프트웨어가 고장을 정상 상태로 취급**하도록 설계 (HDFS 복제, 태스크 재실행의 뿌리).

### Google 논문과의 대응

| Google 논문 | Hadoop 구현 |
|---|---|
| GFS (2003) | HDFS |
| MapReduce (2004) | Hadoop MapReduce |
| BigTable (2006) | HBase |

- Doug Cutting이 웹 크롤러 **Nutch**를 만들다가 "크롤링 데이터를 저장·처리할 수단이 없다"는 문제로 GFS/MapReduce 논문을 구현 → Hadoop
- 이름은 아들의 노란 코끼리 인형에서 유래 (로고가 코끼리인 이유)
- 2006년 Yahoo가 Cutting을 채용하며 대규모 검증

---

## 2. 4개 모듈과 3계층 구조

### 4개 모듈

| 모듈 | 역할 |
|---|---|
| Hadoop Common | 다른 모듈이 공유하는 유틸리티·RPC·설정 |
| HDFS | 분산 파일 저장 |
| YARN | 자원 관리 + job 스케줄링 |
| Hadoop MapReduce | YARN 기반 병렬 처리 엔진 |

### 3계층 (Common 제외)

```
Application Layer      →  Hadoop MapReduce
Resource Mgmt Layer    →  Hadoop YARN
Storage Layer          →  HDFS
```

### 왜 이 분리가 오늘까지 살아남았나

저장(HDFS) / 자원 관리(YARN) / 연산(MapReduce)이 분리되어 있어서, **맨 위 연산 엔진 자리를 다른 것으로 교체할 수 있다.**

```
[MapReduce] [Spark] [Storm] [Hive]   ← 교체 가능
        └──────┬──────┘
             YARN                    ← 자원 관리
              │
             HDFS                    ← 저장
```

### 현재 상황

- **MapReduce(연산 엔진)는 사실상 죽음** — 신규 프로젝트에서 안 씀
- **HDFS·YARN은 온프레미스에서 아직 살아있음** — 클라우드에서는 S3(HDFS 대체), Kubernetes(YARN 대체)로 이동 중
- **MapReduce의 "사고 모델"은 여전히 살아있음** — Spark의 `groupByKey`, Flink의 `keyBy`도 결국 shuffle이고, 이게 왜 비싼지 이해하려면 MR 단계를 알아야 함

---

## 3. HDFS 설계 가정 6가지

각 가정은 **"무엇을 얻기 위해 무엇을 버렸다"**는 트레이드오프 선언이다.

### (1) Hardware Failure — 고장을 정상 상태로

- 블록 기본 3중 복제
- DataNode heartbeat 3초 간격
- 응답 없으면 `2 × recheck-interval(5분) + 10 × heartbeat(3초)` ≈ **10분 30초** 후 dead 판정 → 다른 노드로 재복제

**왜 3개?** 1개는 즉시 손실, 2개는 복제 중 재고장 위험, 3개면 실용적 안전 + 저장 오버헤드 200%에서 멈춤.

→ Hadoop 3의 **Erasure Coding**은 이 200% 오버헤드를 50%까지 낮춘 대안 ([7장](#7-erasure-coding과-reed-solomon) 참조).

### (2) Streaming Data Access — 지연시간을 버리고 처리량을 얻음

HDFS는 **대용량 순차 읽기**에 최적화, 저지연 랜덤 접근은 명시적으로 포기.

**블록 크기 128MB의 근거**: seek 오버헤드를 전송 시간의 1% 이하로 억누르려면, seek(10ms) 이후 최소 1초 이상 읽어야 하고 그게 100MB/s 기준 100MB → 128MB로 결정.

- 1MB 파일 저장 시 실제 디스크는 1MB만 소모 (블록 낭비 없음)
- 단, NameNode 메타데이터는 128MB 파일과 **동일하게** 소모

### (3) Large Data Sets — 그리고 Small Files Problem

NameNode는 모든 메타데이터를 **메모리에** 유지 (파일/블록당 약 150바이트).

| 구성 | 블록 수 | 메타데이터 |
|---|---|---|
| 1GB 파일 1개 | 8 | 약 1.3KB |
| 1MB 파일 1,000개 (합 1GB) | 1,000 | 약 300KB (200배↑) |

**실무 함정**: Kafka Connect / Flink가 HDFS에 스트리밍으로 쓸 때 롤링 정책(`flush.size`, `rotate.interval.ms`)을 잘못 잡으면 작은 파일이 폭발함. 해결책: 롤링 간격 늘리기, 또는 주기적 compaction 배치.

### (4) Simple Coherency Model — WORM

**Write-Once-Read-Many.** 한 번 쓰고 닫은 파일은 수정하지 않는다는 가정.

- 동시 쓰기 허용 시 필요한 락/버전관리/충돌해결이 통째로 사라짐
- **append**: Hadoop 2.x부터 지원됨
- **random write (중간 수정)**: 지금도 불가능

**파급 효과**: HDFS/S3에서 `UPDATE`/`DELETE`가 안 되니, GDPR 삭제나 CDC 반영 시 전체 파티션을 다시 써야 함 → 이 문제를 풀기 위해 **Apache Hudi, Iceberg, Delta Lake**가 등장. 실제 파일은 불변으로 두고 메타데이터 레이어에서 "현재 유효한 파일"을 관리해 update/delete를 흉내냄.

### (5) "Moving Computation is Cheaper than Moving Data"

1TB 처리 시:
- 데이터를 옮기면: 1Gbps 네트워크로 약 2시간 15분, 1,000대가 동시에 하면 스위치 마비
- 코드를 옮기면: JAR 파일 수 MB 배포, 전송 비용 사실상 0

→ **계산 노드와 저장 노드를 같은 기계로 합침** (data locality).

**클라우드에서는 이 전제가 깨짐.** S3+EC2는 저장·연산이 분리돼 locality가 없지만, 데이터센터 내부 네트워크가 10~100Gbps로 빨라져 병목이 아니게 됨. locality를 포기한 대신 **저장·연산을 독립적으로 스케일**할 수 있는 더 큰 이득을 얻음 → 요즘 스택은 대부분 분리형.

### (6) Portability

Java로 작성 → 어떤 OS/하드웨어에서든 실행 가능 (M1/M2 맥 Docker에서도 구동 가능한 이유).

---

## 4. HDFS 아키텍처 상세

### NameNode의 실제 저장 구조

| 구성요소 | 내용 | 저장 위치 |
|---|---|---|
| fsimage | 특정 시점의 전체 스냅샷 | 디스크 |
| edits log | 이후 모든 변경사항 순차 기록 | 디스크 |
| 메모리 상태 | fsimage + edits 재생 결과 | 메모리 |

append-only 로그 패턴 → **PostgreSQL WAL, Kafka 로그와 동일한 아이디어.**

문제: edits가 계속 자라면 재부팅 시 재생 시간이 길어짐 → Secondary NameNode가 주기적으로 병합(checkpoint).

### ⚠️ Secondary NameNode ≠ 백업/Failover

**가장 헷갈리는 지점.**

| | Secondary NameNode | HA (Active/Standby) |
|---|---|---|
| 목적 | checkpoint 병합 (fsimage+edits → 새 fsimage) | 자동 failover |
| 구성 | SNN 1개 | Active NN + Standby NN + JournalNode 3개 + ZKFC + ZooKeeper |
| NN 장애 시 도움 | ❌ 전혀 없음 | ✅ 수십 초 내 자동 승격 |
| HA 구성 시 | 존재하지 않음 (Standby가 겸함) | — |

HA에서는 Active NN이 edits를 JournalNode 과반수에 쓰고, Standby가 재생하며 동기화. ZKFC가 Active 생존을 감시하고 죽으면 승격.

### 쓰기 파이프라인

1. 클라이언트가 NameNode에 "블록 쓸 위치" 요청
2. NameNode가 DataNode 3개 목록 반환
3. 클라이언트는 **DN1에만** 전송
4. DN1→DN2→DN3 **파이프라인**으로 전달 (역방향 ack)

→ 클라이언트 업링크를 1배만 쓰고 나머지 복제는 클러스터 내부망으로 처리.

### 복제 배치 정책 (Rack Awareness)

기본 3중 복제 배치:

1. 첫 번째: 클라이언트 로컬 노드
2. 두 번째: **다른 랙**
3. 세 번째: 두 번째와 **같은 랙**의 다른 노드

**이유**: 로컬 배치로 쓰기 속도↑, 다른 랙 배치로 랙 전체 장애 대비, 세 번째를 같은 랙에 둬서 랙 간 트래픽 최소화 (가용성 vs 네트워크 비용의 절충).

### 읽기 경로 — NameNode는 데이터 경로에 없음

클라이언트는 NameNode에 **위치만** 물어보고, 데이터는 DataNode에서 **직접** 받음 → NameNode가 단일 노드여도 병목이 안 되는 이유.

블록 위치는 거리순 정렬: 같은 노드 → 같은 랙 → 다른 랙 (Data Locality 3단계와 동일).

**참고**: NameNode는 블록 위치를 디스크에 저장하지 않고, 부팅 시 모든 DataNode의 block report로 매번 재구성 (휘발성 정보라 영구 저장 불필요).

---

## 5. YARN

### Hadoop 1의 문제: JobTracker

- 클러스터 자원 관리 + job 스케줄링 + 태스크 추적 + 실패 재실행을 **혼자 다 함**
- 확장성 한계: 태스크 수만 개면 터짐 (실용 한계 약 4,000노드)
- MapReduce에 종속: 고정 slot 방식으로 map/reduce 단계별 자원 낭비

### YARN의 해법: 관심사 분리

| | 역할 | 개수 |
|---|---|---|
| **ResourceManager** | 자원 배분만 (개별 태스크는 신경 안 씀) | 클러스터당 1개 |
| **ApplicationMaster** | 자기 job의 태스크 추적/재실행 | 애플리케이션마다 1개 |

- RM 부하는 애플리케이션 개수(수백~수천)에 비례, 태스크 개수(수십만)에 비례하지 않음 → 확장성 확보
- AM이 프레임워크별 코드이므로 YARN은 그게 뭔지 몰라도 됨 → **Spark on YARN**이 가능한 이유
- slot 대신 **Container**(메모리+CPU 묶음)로 유연하게 자원 표현

### ⚠️ ApplicationsManager vs ApplicationMaster

이름이 거의 같아서 혼동 주의:

| | **Applications**Manager | **Application**Master |
|---|---|---|
| 위치 | ResourceManager **내부** 모듈 | slave 노드의 **컨테이너**에서 실행 |
| 개수 | 클러스터에 1개 | 애플리케이션마다 1개 |
| 역할 | job 제출 접수, AM용 첫 컨테이너 확보, AM 죽으면 재시작 | 태스크용 컨테이너 협상, 추적, 실패 재실행 |

> 기억법: ApplicationsManager는 복수형 — 모든 애플리케이션의 문지기. ApplicationMaster는 단수형 — 한 애플리케이션의 사령관.

### Scheduler in ResourceManager

- **pure scheduler**: 자원 배분만 결정, 모니터링·재시작 보장 안 함 (JobTracker로 회귀하지 않기 위한 의도된 설계)
- 정책 종류:

| 정책 | 특징 |
|---|---|
| FIFO | 선착순, 큰 job이 작은 job을 굶김 (실무 부적합) |
| Capacity Scheduler | 큐별 % 할당, 팀별 자원 보장 (**Hadoop 3 기본값**) |
| Fair Scheduler | 실행 중 job에 균등 분배, 작은 job 응답성 좋음 |

### 9단계 실행 흐름

```
1. 클라이언트 → RM에 job 제출
2. RM의 ApplicationsManager가 유효성 검사 → Scheduler에 자원 요청
3. Scheduler가 임의 노드에 컨테이너 1개 확보 → AM 실행
4. AM → RM에 "태스크용 컨테이너 필요 (locality, CPU, 메모리 조건)" 요청
5. RM이 locality 고려해 컨테이너 할당, 노드 정보 전달
6. AM → 해당 노드 NodeManager에 "이 컨테이너 실행" 요청
7. AM이 태스크 모니터링, 완료 시 RM에 보고 후 종료
8. NM들이 주기적으로 가용 자원을 RM에 heartbeat
9. 노드 장애 시 RM이 새 컨테이너 할당 → AM이 job 완주
```

**주목**: AM은 RM에게 컨테이너를 "받고", 실행은 NM에게 **직접** 요청 (RM은 실행에 관여 안 함).

### 이름 정정

슬라이드 제목 "Yet Another Resource **Manager**"는 오타. 공식 확장은 **Yet Another Resource Negotiator**.

---

## 6. MapReduce

### 왜 map/reduce 두 개뿐인가

대부분의 대규모 처리는 "각 레코드에 독립적으로 처리(map)" + "같은 키끼리 모아 집계(reduce)"로 표현 가능 → map 태스크들이 **서로 통신 필요 없음(shared nothing)** → 하나 죽어도 그것만 재실행. **표현력을 제한한 대가로 자동 병렬화 + 자동 내결함성을 공짜로 얻음.**

### 전체 파이프라인

```
HDFS 파일
  ↓ InputFormat이 논리적 split으로 자름
Input Split
  ↓ RecordReader가 (key, value) 레코드로 변환
map(k1, v1) → list(k2, v2)          [사용자 코드]
  ↓ 메모리 버퍼(기본 100MB) → 80% 차면 spill
  ↓ spill 시 partition별로 정렬
  ↓ [Combiner: 있으면 여기서 지역 집계]
  ↓ Partitioner: hash(k2) % numReducers
로컬 디스크의 정렬된 파티션 파일들
  ↓ ===== SHUFFLE: 네트워크 전송 =====
  ↓ reducer가 자기 파티션을 모든 mapper에서 pull
  ↓ merge sort
reduce(k2, list(v2)) → list(k3, v3)  [사용자 코드]
  ↓ OutputFormat / RecordWriter
HDFS 출력 (part-r-00000, part-r-00001, ...)
```

### Input Split ≠ HDFS Block

| | HDFS Block | Input Split |
|---|---|---|
| 성격 | 물리적 저장 단위 | 논리적 처리 단위 |
| 자르는 기준 | 128MB에서 **바이트 단위로 무조건** | **레코드 경계 존중** |

- 경계에 레코드가 걸치면 RecordReader가 다음 블록의 첫 줄까지 네트워크로 읽어와 완성
- **map 태스크 개수 = input split 개수** (직접 지정 불가, 데이터 크기가 결정)
- **reduce 태스크 개수는 직접 지정** → output 파일 개수(`part 0`, `part 1`...)와 일치

### map 출력은 왜 로컬 디스크인가

| 이유 | 설명 |
|---|---|
| 일회용 데이터 | reduce가 읽어가면 버려짐, job 끝나면 삭제 |
| 복제 불필요 | 유실되면 그 map 태스크만 재실행하면 됨 |
| 오버헤드 회피 | HDFS에 쓰면 네트워크 I/O + 복제 비용 추가 |

반면 **최종 reduce 출력은 HDFS에 저장** (장기 보존 대상).

### Combiner — 조건과 함정

Combiner는 map 쪽에서 지역 집계로 shuffle 전송량을 줄이는 최적화.

**조건**: reduce 함수가 **교환법칙 + 결합법칙**을 만족해야 안전.

| 연산 | Combiner 안전성 |
|---|---|
| sum, max, min, count | ✅ 안전 |
| average | ❌ `avg(avg(1,2),3)=2.25 ≠ avg(1,2,3)=2` |

평균 구하려면 Combiner가 `(합계, 개수)` 쌍을 내보내고 reducer가 마지막에 나눠야 함.

**함정**: Combiner 실행은 **보장되지 않음** (0번, 1번, 여러 번 호출 가능) → "Combiner 없이도 결과가 같아야" 올바른 구현.

### Partitioner와 데이터 스큐

기본: `hash(key) % numReducers`. 같은 키는 반드시 같은 reducer로.

**함정**: 키 분포가 치우치면 특정 reducer만 과부하. (예: 감정분류 positive/negative/neutral 3종 → reducer 10개 늘려도 3개만 일함, neutral 압도적으로 많으면 그 reducer가 전체 시간 결정)

**완화**: salting (키에 랜덤 접미사 붙여 분산 후 2단계 job으로 재집계).

### Shuffle이 가장 비싼 이유

모든 mapper가 모든 reducer에게 데이터를 보내는 **all-to-all** 통신 (M×R 전송 경로) + 정렬 비용 + 디스크 spill.

> **분산 처리 성능 최적화는 거의 항상 "shuffle을 어떻게 줄이느냐"로 귀결된다.** (Spark `reduceByKey` vs `groupByKey`, broadcast join 등 전부 같은 원리)

### 왜 Spark가 MapReduce를 대체했나

MapReduce의 치명적 약점: **job 사이마다 반드시 HDFS 경유** (3중 복제 포함 I/O 반복).

Spark의 개선:
1. 중간 결과를 **메모리에 유지** (필요시 spill)
2. 연산을 **DAG**으로 표현해 여러 단계를 하나의 실행 계획으로 최적화

→ 반복 워크로드에서 10~100배 차이. 단, **원리는 동일** — Spark의 stage 경계가 곧 shuffle 경계.

---

## 7. Erasure Coding과 Reed-Solomon

### 출발점: 복제는 안전하지만 무식하다

3중 복제 = 1PB 저장에 3PB 필요 (오버헤드 200%), 그중 2PB는 정보량 0인 순수 중복.

→ **"복사보다 효율적인 안전장치가 있을까?"** = Erasure Coding

**Erasure**의 의미: 어느 조각이 사라졌는지는 알지만 내용은 모르는 상황 (HDFS는 NameNode가 "몇 번 블록이 사라졌다"를 항상 알고 있는 이 상황에 해당).

### 가장 쉬운 EC: XOR 패리티

```
D1 = 1011
D2 = 0110
D3 = 1101
────────────── XOR
P  = 0000     ← 패리티
```

D2 소실 시: `D1 ⊕ D3 ⊕ P = D2` 복원 가능. 오버헤드 33%.

**한계**: 2개 동시 소실 시 방정식(1개)보다 미지수(2개)가 많아 복원 불가능. → 일반화 필요.

### Reed-Solomon — 다항식 보간

데이터 k조각을 **다항식의 계수**로 해석.

예: `d₀=3, d₁=5` → `p(x) = 3 + 5x`

| x | 1 | 2 | 3 | 4 |
|---|---|---|---|---|
| p(x) | 8 | 13 | 18 | 23 |

**직선은 점 두 개로 결정된다** → 4개 중 어느 2개가 남아도 직선(=원본) 복원 가능.

### 일반화 규칙

> 데이터 k조각을 n = k+m개로 부호화하면, n개 중 어느 k개만 살아남아도 원본이 완전히 복원된다. 즉 **m개 고장까지 견딘다.**

- k=2 → 직선 (점 2개로 결정)
- k=3 → 포물선 (점 3개)
- k=6 → 5차 곡선 (점 6개)

### 왜 최적인가 (MDS 코드)

k조각의 정보를 복원하려면 이론상 최소 k조각을 읽어야 함. Reed-Solomon은 이 하한을 정확히 달성 → **MDS(Maximum Distance Separable)**: "패리티 m개로 정확히 m개 고장을 견딘다, 1개도 더 덜도 아니고" = 저장 효율의 이론적 최적.

### 유한체(Galois Field) 연산

실수로 계산하면 오버플로/소수점 오차 발생 → **GF(2⁸)** (원소 256개)에서 계산.

| | 일반 산술 | GF(2⁸) |
|---|---|---|
| 덧셈 | 3+5=8 | **XOR** |
| 뺄셈 | 별개 연산 | 덧셈과 동일 (XOR) |
| 곱셈 | 오버플로 가능 | 기약다항식 modulo → 항상 1바이트 |
| 나눗셈 | 소수점 발생 | 항상 정확히 나눠짐 |

**흥미로운 관찰**: GF에서 덧셈=XOR이므로 **앞의 XOR 패리티는 Reed-Solomon의 m=1 특수 케이스.**

구현은 로그/역로그 테이블로 곱셈 처리, Intel **ISA-L** 라이브러리가 SIMD로 가속.

### 행렬 관점 (구현자 시각)

$$G_{n\times k} \times \mathbf{d}_{k\times1} = \mathbf{c}_{n\times1}$$

- 핵심 조건: **G의 어떤 k개 행을 골라도 부분행렬이 가역**
- 복원: 살아남은 k개 행으로 부분행렬 G′ 구성 → $\mathbf{d} = G'^{-1}\mathbf{c}'$
- **systematic form**: G의 위쪽 k행을 단위행렬로 → 고장 없을 때 복호화 없이 그냥 읽기 가능 (HDFS 방식)

### HDFS 구현

**표기법**: `RS-{k}-{m}-{cell size}` (10장에서 상세)

| 방식 | 저장 배수 | 오버헤드 | 견디는 고장 |
|---|---|---|---|
| 3중 복제 | 3.0× | 200% | 2개 |
| RS-3-2 | 1.67× | 67% | 2개 |
| **RS-6-3 (기본)** | **1.5×** | **50%** | **3개** |
| RS-10-4 | 1.4× | 40% | 4개 |

**Striped layout**: 파일을 cell(기본 1MB) 단위로 잘라 라운드로빈으로 여러 노드에 분산.

```
원본 파일: [c0][c1][c2][c3][c4][c5][c6]...

    DN1  DN2  DN3  DN4  DN5  DN6  │  DN7   DN8   DN9
스트라이프0: c0   c1   c2   c3   c4   c5  │  p0    p1    p2
             └──── 데이터 6 ────┘   └── 패리티 3 ──┘
             이 9개 = 블록 그룹(Block Group)
```

→ RS-6-3에는 **최소 DataNode 9대** 필요.

**쓰기 경로 차이**:

| | 복제 | EC |
|---|---|---|
| 전송 구조 | 파이프라인 (순차) | **병렬** (클라이언트→9개 DN 동시) |
| 패리티 계산 | 없음 | **클라이언트가** RS 인코딩 |
| 클라이언트 부담 | 1배 업링크 | 1.5배 업링크 + CPU |

### EC의 대가

| 대가 | 설명 |
|---|---|
| **복구 비용** | 블록 1개 유실 시, 복제는 1블록만 복사, RS-6-3은 6블록 읽어서 재계산 (**6배 트래픽**) |
| **Degraded read** | 노드 죽어있으면 읽는 도중 실시간 복원 → 지연 증가 |
| **Data Locality 상실** | 파일 구간이 이미 6개 노드에 분산 → 태스크가 어디서 돌든 5개 노드에서 네트워크로 데이터를 끌어와야 함 |
| **Small file에 재앙** | 1MB 파일도 패리티 3개 필요 → 오버헤드 300% (3중 복제보다 나쁨). 이득은 파일 크기 ≥ k×cell size일 때만 |
| **기능 제약** | append, truncate, hflush/hsync 미지원/제한 |

### 완화 변종

- **LRC (Local Reconstruction Codes, Azure)**: 지역 패리티 추가로 흔한 1중 고장을 적은 노드만 읽고 복구 (MDS 최적성 일부 포기하고 복구 비용 절약)
- **Clay codes**: 비슷한 방향

### 언제 쓰나 — Hot/Cold 계층화

| | 3중 복제 | Erasure Coding |
|---|---|---|
| 적합 데이터 | 자주 읽고 쓰는 최근 데이터 | 거의 안 읽는 아카이브 |
| 파일 크기 | 무관 | 큰 파일만 (≥ k×cell) |
| 읽기 지연 | 낮음 | 높음 |
| Locality | 있음 | 없음 |

```
/data/telemetry/dt=최근7일/   → 3중 복제 (핫, 쿼리 대상)
/data/telemetry/dt=30일이전/  → RS-6-3 (콜드, 규정상 보관)
```

**명령어**:
```bash
hdfs ec -listPolicies
hdfs ec -enablePolicy -policy RS-6-3-1024k
hdfs ec -setPolicy -path /data/archive -policy RS-6-3-1024k
hdfs ec -getPolicy -path /data/archive
```

⚠️ `setPolicy`는 **이후 새로 쓰이는 파일**에만 적용. 기존 파일은 `distcp`로 다시 써야 전환됨.

### RS의 보편성

QR코드, CD/DVD/Blu-ray, 보이저 우주탐사선 통신, DVB/DSL/위성방송, **S3/Azure Storage/GCS 내부 저장계층**, Ceph, MinIO, Backblaze 등 물리 매체 오류정정과 분산 스토리지가 동일한 수학을 사용.

---

## 8. Erasure vs Error, 그리고 체크섬

### 핵심 구분: 위치를 아는가

| | 위치 | 값 |
|---|---|---|
| **Erasure (소실)** | ✅ 안다 (NameNode 메타데이터, heartbeat 끊김) | 없음 (비어있음) |
| **Error (오류)** | ❌ 모른다 | 있는데 틀림 (silent corruption) |

**위치를 아는 게 왜 필수인가**: RS는 다항식 보간에서 x값(위치)이 좌표 역할. 위치를 모르면 어떤 y값이 어느 x에 대응하는지 몰라 보간 자체가 불가능.

### 정량적 차이

패리티 m개로:
- **소실(erasure)은 m개까지** 복구 (RS-6-3 → 3개)
- **오류(error)는 ⌊m/2⌋개까지만** 정정 (RS-6-3 → 1개)

→ 오류 정정은 "어디가 틀렸는지 찾는 데" 패리티 절반, "고치는 데" 나머지 절반을 씀. 위치를 아는 소실은 같은 패리티로 2배 효과.

### 위치를 진짜 모르는 경우 — 체크섬

디스크의 **silent data corruption**(조용한 비트 반전)은 erasure가 아니라 error. HDFS는 512바이트마다 CRC32C를 `.meta` 파일에 저장, 읽을 때 검증. `DataBlockScanner`가 백그라운드로 주기적 전수 검사.

> **체크섬이 하는 일: 위치를 모르는 error를, 위치를 아는 erasure로 강등시킨다.**

```
체크섬  →  손상을 탐지하고 위치를 확정 (진단)
RS      →  확정된 위치의 내용을 재계산 (치료)
```

체크섬 없이 RS만 있으면 조용한 손상 앞에서 무력, RS 없이 체크섬만 있으면 손상을 알아도 못 고침. **3중 복제도 체크섬에 의존** (손상된 복제본을 감지해야 성한 복제본에서 재복사 가능).

---

## 9. 데몬과 물리 서버의 관계

### 핵심: "기계"와 "데몬"은 다른 개념

- **물리 서버 한 대**: 저장도 하고 연산도 함 ✅
- **NameNode/DataNode 데몬**: **저장만** 담당, 연산은 절대 안 함
- **ResourceManager/NodeManager 데몬**: **연산만** 담당, 저장은 절대 안 함

같은 기계 위에 별개 프로세스 두 개가 나란히 떠 있는 구조.

| 계층 | 마스터 데몬 | 워커 데몬 | 담당 |
|---|---|---|---|
| HDFS (저장) | NameNode | DataNode | 블록 저장, 읽기/쓰기 서비스 |
| YARN (연산) | ResourceManager | NodeManager | 컨테이너 할당, 코드 실행 |

- DataNode는 블록을 디스크에 넣고 꺼내주기만 함. 사용자 코드 실행 능력 없음
- NodeManager는 컨테이너(JVM)를 띄워 태스크를 실행. HDFS 블록은 전혀 갖고 있지 않음

### NameNode는 데이터를 1바이트도 저장하지 않음

NameNode가 갖는 건 fsimage+edits log(위치 목록)뿐. 실제 파일 내용은 지나가지 않음. 읽기 시 클라이언트는 위치만 물어보고 데이터는 DataNode에서 **직접** 받음 → NameNode가 병목이 안 되는 이유.

ResourceManager도 마찬가지로 사용자 코드를 실행하지 않음 — "누구에게 컨테이너를 줄까"만 결정, 실행은 NodeManager가 함.

→ **마스터 서버는 데이터도 없고 연산도 안 하는 순수 관리 노드.** NameNode는 메타데이터를 전부 메모리에 올려야 해서 RAM이 많이 필요하고, 죽으면 클러스터 전체가 멈추므로 디스크도 RAID 구성. DataNode는 진짜 싼 장비여도 되지만 NameNode는 아님.

### 왜 같은 기계에 붙여놓나

"Moving Computation is Cheaper than Moving Data"를 실현하는 유일한 방법. 블록이 워커 3번에 있으면 NodeManager도 워커 3번에 있으니 그 기계에서 컨테이너를 띄우면 태스크가 로컬 디스크를 읽음 → 네트워크 전송 0.

**참고**: 로컬이어도 기본적으로는 loopback TCP로 DataNode 프로세스를 거쳐 받음. 프로세스 경유마저 없애려면 **short-circuit local read**:

```xml
<property><name>dfs.client.read.shortcircuit</name><value>true</value></property>
<property><name>dfs.domain.socket.path</name><value>/var/lib/hadoop-hdfs/dn_socket</value></property>
```

DataNode가 Unix 도메인 소켓으로 파일 디스크립터를 넘겨주고 클라이언트가 직접 읽음 (기본 꺼짐, HBase 등 지연 민감 워크로드에서 켬).

### `jps`로 직접 확인

```bash
# 마스터 컨테이너
$ jps
1234 NameNode
1567 ResourceManager
1890 SecondaryNameNode

# 워커 컨테이너
$ jps
2345 DataNode
2678 NodeManager

# 잡 실행 중 워커
$ jps
2345 DataNode
2678 NodeManager
3012 MRAppMaster    ← 이 잡의 사령관 (잡 끝나면 사라짐)
3456 YarnChild      ← 실제 map/reduce 태스크가 도는 JVM
```

### 클라우드에서는 이 공존이 사라짐

| | 온프레미스 Hadoop | S3 + EMR/EKS |
|---|---|---|
| 저장 | DataNode (연산 노드와 같은 기계) | S3 (완전히 다른 시스템) |
| 연산 | NodeManager (같은 기계) | EC2 / 파드 |
| Locality | 있음 | 없음 |
| 스케일링 | 저장·연산이 묶여서 같이 늘어남 | 독립적 |

온프레미스는 저장이 부족하면 연산 능력까지 같이 사야 했음(공존의 대가). 클라우드는 locality를 포기하고 그 결합을 끊어 독립적 스케일링을 얻음.

---

## 10. RS-k-m-cellsize 표기법

`RS-6-3-1024k` 분해:

| 변수 | 예시 값 | 의미 |
|---|---|---|
| **k** | 6 | 원본 데이터를 몇 조각으로 쪼갤지 |
| **m** | 3 | 만들어낼 패리티 조각 개수 = 견디는 고장 수 |
| **cell size** | 1024k (1MB) | 조각(cell) 하나의 크기 |

### k, m — 내구성과 오버헤드

```
3중 복제:  원본 1 + 복제 2  = 총 3개  →  2개 고장까지 버팀
RS-6-3:   원본 6 + 패리티 3 = 총 9개  →  3개 고장까지 버팀
```

오버헤드 공식: $\text{오버헤드} = m/k$

- RS-6-3 → 3/6 = 50%
- RS-3-2 → 2/3 = 67%
- RS-10-4 → 4/10 = 40%

**k+m = 필요한 최소 DataNode 수** (RS-6-3 → 9대 필요).

### Cell size — 완전히 다른 축

k, m은 "몇 조각으로 나누냐"이고, cell size는 **조각 하나의 크기**.

```
원본 파일: [cell0][cell1][cell2][cell3][cell4][cell5][cell6]...
             ↓      ↓      ↓      ↓      ↓      ↓
            DN1    DN2    DN3    DN4    DN5    DN6  (+패리티 DN7,8,9)
```

파일을 cell size만큼씩 잘라 라운드로빈 분산. **k개 cell을 한 바퀴 돌면 스트라이프**, 그 스트라이프에서 m개 패리티 계산.

| | k, m | cell size |
|---|---|---|
| 결정하는 것 | 내구성 / 오버헤드 비율 | 얼마나 잘게 쪼개 뿌릴지 |
| 단위 | 조각의 개수 | 조각 하나의 바이트 크기 |
| 영향 | 필요 노드 수, 복구 시 읽을 양 | I/O 병렬도, 작은 파일 낭비 |

**트레이드오프**:
- 작을수록 병렬도↑ (더 많은 노드에 분산되어 처리량 좋아짐), 단 너무 작으면 요청 오버헤드↑
- 파일이 cell size보다 작으면 데이터 cell 1개도 못 채우는데 패리티는 여전히 m개 필요 → **EC 효율은 파일 크기 ≥ k×cell size일 때만** (RS-6-3-1024k면 최소 6MB)

> **한 문장**: k, m은 "몇 조각으로 나눠서 몇 개까지 잃어도 되나"를 정하고, cell size는 "그 조각 하나가 몇 바이트인가"를 정한다.

---

## 11. 미션 실무 힌트

### W3M2b — Hadoop 설정 파일 지도

| 파일 | 담당 | 핵심 키 |
|---|---|---|
| `core-site.xml` | 클러스터 공통 | `fs.defaultFS` (예: `hdfs://namenode:9000`) |
| `hdfs-site.xml` | HDFS | `dfs.replication`, `dfs.namenode.name.dir`, `dfs.datanode.data.dir`, `dfs.blocksize` |
| `yarn-site.xml` | YARN | `yarn.resourcemanager.hostname`, `yarn.nodemanager.resource.memory-mb`, `yarn.nodemanager.aux-services` |
| `mapred-site.xml` | MapReduce | `mapreduce.framework.name=yarn`, map/reduce 메모리 |
| `workers` | DataNode/NM 목록 | 호스트명 한 줄씩 (Hadoop 2는 `slaves`) |
| `hadoop-env.sh` | 환경 변수 | `JAVA_HOME`, 힙 크기 |

### Docker 멀티노드 구성 시 흔한 함정

- **호스트명 해석**: 컨테이너 이름을 그대로 호스트명으로 써야 함. `localhost` 설정 시 컨테이너 간 통신 깨짐 (Compose의 사용자 정의 네트워크에 붙이면 DNS 자동 해결)
- **`namenode -format`은 최초 1회만**: 재시작마다 포맷하면 clusterID 불일치 → `Incompatible clusterIDs` 오류
- **볼륨 영속화**: `dfs.namenode.name.dir`을 named volume에 안 두면 컨테이너 재생성 시 메타데이터 소실
- **arm64**: 공식 이미지 지원 확인 필요, 없으면 `eclipse-temurin:11-jre` 베이스에 Hadoop 바이너리 직접 얹어 빌드 (Hadoop 3.3+ arm64 네이티브 라이브러리 포함)
- **주요 UI 포트**: NameNode 9870, ResourceManager 8088, DataNode 9864, JobHistory 19888

### YARN 메모리 설정 함정

```xml
<property>
  <name>yarn.nodemanager.resource.memory-mb</name>
  <value>6144</value>
</property>
```

이 값은 "YARN 컨테이너에 내줄 수 있는" 메모리. 8GB 기계면 6GB 정도만 주고, 남은 2GB는 DataNode JVM + NodeManager 자신 + OS 페이지 캐시 몫으로 남겨야 함. 기본값(8192)을 그대로 두면 물리 메모리 전체를 컨테이너에 배분하려다 OOM killer가 DataNode를 죽이고, 노드가 dead로 판정되어 블록 재복제가 시작되는 연쇄 반응이 발생.

### W3M4 — "predefined keywords" 대안

**1) 가중치 사전 (AFINN, VADER)**
단순 키워드 매칭은 `not good`을 positive로 잡는 문제 있음. AFINN은 단어에 -5~+5 점수, VADER는 부정어·강조어·대문자·느낌표·이모티콘까지 규칙으로 반영. 학습 불필요, 순수 함수라 mapper에 적합.

**2) 학습된 분류기 배포 (Naive Bayes / Logistic Regression)**

```bash
hadoop jar ...hadoop-streaming.jar \
  -files mapper.py,reducer.py,model.pkl \
  -mapper mapper.py -reducer reducer.py \
  -input /tweets -output /out
```

`-files`가 **distributed cache**로 각 노드에 파일 복사. 중요 패턴: **모델 로드는 루프 밖에서 1회만**.

```python
model = pickle.load(open('model.pkl','rb'))   # 태스크당 1회
for line in sys.stdin:                          # 레코드마다
    print(f"{model.predict([line])[0]}\t1")
```

**3) 왜 트랜스포머 모델은 안 맞는가**
BERT 계열은 모델이 수백 MB, 태스크마다 로드해야 하고 GPU 없이 느림. **MapReduce는 무거운 상태를 가진 연산에 구조적으로 불리함.** Spark(`mapPartitions`로 파티션당 1회 로드) 또는 별도 추론 서비스로 처리하는 게 일반적. "왜 이 도구를 안 쓰는지"를 근거 있게 설명하는 게 평가에서 좋은 포인트.

**추가 논의**: 3개 클래스 강제 분류 대신 연속 점수 후 임계값 적용 → 나중에 조정 가능 + 키 3개 스큐 문제 완화.

---

## 최종 요약

> **Hadoop = "고장을 전제한 값싼 서버 무리에, 데이터를 잘라 흩뿌려 저장하고(HDFS), 데이터가 있는 곳으로 코드를 보내(Data Locality), 자원 관리와 태스크 추적을 분리해(YARN), 병렬로 처리한다(MapReduce)."**
>
> **NameNode/ResourceManager는 관리만, DataNode/NodeManager는 각각 저장·연산만 한다. 같은 물리 서버에 이 둘을 공존시킨 이유가 곧 data locality다.**
>
> **Reed-Solomon = 데이터 k조각을 다항식 계수로 보고 n개 점에서 평가해 저장. 차수 k−1 다항식은 점 k개로 결정되므로 n개 중 어느 k개만 남아도 복원된다.** HDFS EC는 이 원리로 3중 복제의 200% 오버헤드를 50%로 낮추지만, 복구 비용 6배·locality 상실·small file 취약·append 불가라는 대가로 콜드 데이터 전용.

### 면접 대비 우선순위

1. HDFS 블록/복제/NameNode-DataNode 역할
2. Secondary NN vs Standby NN 구분
3. YARN이 나온 이유 (JobTracker 병목)
4. Shuffle이 왜 비싼가
5. Small files problem
6. Spark가 MapReduce를 대체한 이유
7. Erasure Coding의 트레이드오프 (복구 비용, locality 상실)
