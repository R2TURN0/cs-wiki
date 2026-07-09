# Github
- Github Wiki 만드는 법
  - 리포에서 settings에서 wiki 체크
- Github Wiki 목차 만드는 법
  - 우측에 _Sidebar.md 파일을 생성하여 관리
  - 마크다운 글머리 기호(들여쓰기 공백 2~4칸)를 활용해 계층 구조 생성
- .gitignore 설정
  - 프로젝트 루트에 .gitignore 파일 생성하여 깃 추적 제외 대상 지정
  - 제외 대상: 가상환경 폴더(.venv/), 파이썬 캐시 폴더(**/__pycache__/), 환경변수 파일(.env) 등

# Mac Book
- homebrew 설치
  - homebrew를 통한 파이썬 설치: brew install python
- 터미널 기본 파이썬 버전 고정 및 별칭 설정
  - ~/.zshrc 파일에 Homebrew 경로 추가: `echo 'export PATH="/opt/homebrew/bin:$PATH"' >> ~/.zshrc`
  - python 명령어 입력 시 python3 실행되도록 설정: `echo 'alias python="python3"' >> ~/.zshrc`
  - 설정 반영: `source ~/.zshrc`
- 가상환경 실행 및 종료
  - 실행: `source .venv/bin/activate`
  - 종료: `deactivate`

# Python
## pandas
- csv파일 로드
  - `df = pd.read_csv('data.csv')`
  - 인덱스 열 지정 로드: `df = pd.read_csv('data.csv')`, index_col=0 하면 이름 없는 컬럼이 Unnamed: 0 같이 바뀌는 것 방지 가능
- 출력 줄바꿈 방지 설정
  - `pd.set_option('display.expand_frame_repr', False)` (너비에 따른 열 줄바꿈 방지)
  - `pd.set_option('display.max_columns', None)` (열 생략 방지)
- head
  - `df.head(n)`
    - 위에서부터 n개 행 출력
    - 기본값 5
- tail
  - `df.tail(n)`
    - 아래에서부터 n개 행 출력
    - 기본값 5
- shape
  - `df.shape`
    - 데이터프레임의 크기 확인
    - (행, 열) 형태의 튜플로 반환
- columns
  - `df.columns`
    - 모든 컬럼의 이름 목록 반환
- dtypes
  - `df.dtypes`
    - 각 열의 데이터 타입만 요약 출력
- info
  - `df.info()`
    - 데이터의 종합 명세서 출력
    - 전체 행/열 개수, 데이터 타입, 결측치 개수, 메모리 사용량 확인
    - 주의: 자체 출력 기능이 있어 print(df.info())로 쓰면 맨 아래에 None이 출력되므로 단독 호출해야 함
- describe
  - `df.describe()`
    - 수치형 데이터의 기술 통계량 요약 출력
    - 개수, 평균, 표준편차, 최솟값, 사분위수(25%, 50%, 75%), 최댓값 자동 계산
- 열 이름 바꾸기
  - `df.rename(columns={'기존이름': '새이름'}, inplace=True)`
  - inplace=True 속성을 주어야 원본 데이터프레임에 즉시 반영됨
- 고유값 개수 및 종류 확인
  - `df['열이름'].nunique()`: 중복을 제외한 고유값의 총 개수 반환
  - `df[['열1', '열2']].nunique()`: 대괄호 쌍을 이용해 여러 열의 고유값 개수 동시 확인
  - `df['열이름'].unique()`: 고유값의 종류 자체를 배열로 반환
- 데이터 개수 세기
  - `size`: 결측치 포함 여부와 관계없이 그룹 내 전체 행 개수를 단일 숫자로 반환
  - `count()`: 결측치를 제외한 유효 데이터 개수를 모든 열별로 각각 반환
- 조합별 빈도수 구하기 (경우의 수)
  - 방법 1: `pd.crosstab(df['열1'], df['열2'])` (가로·세로 격자 형태의 교차표 출력)
  - 방법 2: `df.groupby(['열1', '열2']).size()` (세로로 길게 나열되는 리스트 형태 출력)
## SQLite3
- with
  - 에러 없으면 자동으로 커밋, 에러 있으면 자동 롤백
  - ```
    with sqlite3.connect('w1m2.db') as conn:
      cursor = conn.cursor()
      cursor.execute("""쿼리내용""")
    ```
- TOP 50 PERCENT
  - 퍼센트 기능 없기에 다음과 같은 방법 사용
  - ```
    SELECT * FROM Customers. 
    LIMIT (SELECT COUNT(*) FROM Customers) / 2;
    ```
  - 기본적으로 소숫점 아래는 버려짐
  - 파이썬에서 계산하려면 총 데이터 개수 가져오고 계산하고 다시 쿼리 보내야 하기에 그냥 쿼리에서 계산시키는게 더 효율적임
- 컬럼명 같이 출력
  - ```
    with sqlite3.connect('w1m2.db') as conn:
      conn.row_factory = sqlite3.Row
      cursor = conn.cursor()
      cursor.execute("""
      SELECT MIN(Price) AS SmallestPrice
      FROM Products;
                   """)
      results = [dict(row) for row in cursor.fetchall()]
    
      print(results)
    ```
- GROUP BY
  - GROUP BY 에 적은 컬럼과 집계함수 외에는 SELECT하면 안됨
  - ```
    SELECT MIN(Price) AS SmallestPrice, CategoryID
    FROM Products
    GROUP BY CategoryID;
    ```
  - HAVING 으로 조건 걸 수 있음, WHERE는 그룹 묶기 전, HAVING은 그룹 묶은 후 필터
- 띄어쓰기가 있는 컬럼명
  - 쌍따옴표(표준)나 대괄호(표준 아님)로 감쌈
  - ```
    SELECT COUNT(*) AS [Number of records]
    FROM Products;
    ```
- LIKE
  - 대소문자 구분 안함
    - 구분하려면 아래와 같이 하면 됨
    - `PRAGMA case_sensitive_like = true;`
  - 와일드카드
    - %: 0글자 이상의 아무 문자
    - _: 1글자 고정 아무 문자
- GLOB
  - LIKE랑 비슷하지만 대소문자 구분하며 []연산 지원함
  - 와일드카드
    - *: LIKE의 %
    - ?: LIKE의 _
  - []
    - 괄호 안의 문자들에 해당하는지 확인
    - [a-fA-F]와 같이 범위 지정도 가능
- IN
  - 데이터 개수 많아지면 OR 연산보다 IN이 훨씬 빨라진다고 함
- BETWEEN
  - 문자열도 가능
  - 기본적으로 대소문자 구분하며, 대소문자 구분 안하려면 COLLATE NOCASE 붙이면 됨
- 문자열 합치기
  - +
  - CONCAT
  - ||
- JOIN
  - INNER JOIN
    - 기본값
- <>
  - 같지 않다(!=)
- UNION
  - SELECT문 중복 제거하며 결합
  - 추출 컬럼 데이터타입/순서는 일치해야함
  - 아래와 같이 컬럼명에 Alias 지정해도 전체 다 반영됨
  - ```
    SELECT 'Customer' AS Type, ContactName, City, Country
    FROM Customers
    UNION
    SELECT 'Supplier', ContactName, City, Country
    FROM Suppliers;
    ```
  - 중복 포함하려면 UNION ALL 사용
- EXISTS
  - 존재하는지 여부 확인
  - 다음과 같은 경우, 안쪽 SELECT문의 컬럼명은 사실 의미없음
  - ```
    SELECT SupplierName
    FROM Suppliers
    WHERE EXISTS (
      SELECT ProductName
      FROM Products
      WHERE Products.SupplierID = Suppliers.supplierID AND Price = 22
    );
    ```
- ANY
  - 하나라도 있는지 확인하는것
  - SQLITE에는 없으므로 IN으로 대체
- COALESCE
  - 여러 값 중 NULL이 아닌 값 처음으로 나온거 반환
- IFNULL
  - 인자 두개만 쓸 수 있음
  - IFNULL(A,B): A가 NULL이면 B써라
- Stored Procedure
  - SQLITE에는 없음
- PARTITION BY
  - 같은 값을 가진 것 끼리 구역을 나눔



## matplotlib
- 모든 변수의 히스토그램 그리기
  - `df.hist(figsize=(가로, 세로), bins=구간개수, grid=False)`, 가로 세로 단위는 인치임
  - 전체 대제목 설정: `plt.suptitle('타이틀')`
  - 배치 최적화: `plt.tight_layout()` (그래프와 글자가 겹치지 않게 조절)
- Scatter 차트 그리기
  - `plt.scatter(x축_데이터, y축_데이터, color='색상', alpha=투명도, s=점크기)`
  - 타이틀 및 레이블 설정: `plt.title()`, `plt.xlabel()`, `plt.ylabel()`
  - 격자선 표시: `plt.grid(True)`
  - 화면 출력: `plt.show()`

## multiprocessing
Pool
- 워커 수 미리 할당하는 방식
Process
- 수동으로 직접 프로세스 생성하는 방식
Queue
- get
  - pop과 같은 역할
  - 데이터 없으면 생길때까지 대기
- get_nowait
  - 데이터 없으면 queue.Empty 에러 발생 후 넘어감
  - get(block=False) 랑 같다.


# 개념
## 상관계수
- 정의: 두 변수 간에 선형적 또는 비선형적 관계가 존재하는지, 그리고 그 관계가 얼마나 밀접한지를 -1부터 1 사이의 수치로 나타낸 지표
  - 1에 가까울수록 강한 양의 상관관계 (한쪽이 커지면 다른 쪽도 커짐)
  - -1에 가까울수록 강한 음의 상관관계 (한쪽이 커지면 다른 쪽은 작아짐)
  - 0에 가까울수록 두 변수 간에 아무런 상관관계가 없음
- 상관계수 표 구하기: `df.corr(numeric_only=True)` (문자열 열을 제외하고 수치형 데이터만 계산)
- 상관계수의 3가지 종류
  - 피어슨 (Pearson) 상관계수
    - 특징: 두 변수 간의 선형적(직선형) 관계의 강도를 측정
    - 조건: 두 변수가 모두 연속형 숫자 데이터이고, 정규분포를 따를 때 정확함
    - 단점: 데이터에 뜬금없이 너무 크거나 작은 값(이상치)이 있으면 수치가 크게 왜곡됨
  - 스피어먼 (Spearman) 순위 상관계수
    - 특징: 데이터의 실제 값 대신 순위(Rank)를 매겨서 상관관계를 측정
    - 장점: 꼭 직선 형태가 아니더라도, 한쪽이 증가할 때 다른 쪽도 같이 증가하는 Monotonic 관계를 잘 잡아냄. 이상치에 영향을 거의 받지 않음
  - 켄달 (Kendall) 타우
    - 특징: 스피어먼과 마찬가지로 순위 기반이지만, 데이터 쌍들이 서로 크고 작은 방향이 일치하는지 순서 관계를 따져서 계산
    - 장점: 샘플 데이터의 개수가 적거나, 같은 순위가 많을 때 스피어먼보다 더 안정적이고 정확함
  - 활용법: method 옵션 변경 (`df.corr(method='pearson')`, `df.corr(method='spearman')`, `df.corr(method='Kendall')`)

