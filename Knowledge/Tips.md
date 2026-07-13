# Tips

개발 환경, 도구, 라이브러리를 사용하며 알게 된 잡다한 팁을 정리하는 문서

---

## Github

- **Github Wiki 만드는 법**: 리포에서 Settings에서 Wiki 체크
- **Github Wiki 목차 만드는 법**: 우측에 `_Sidebar.md` 파일을 생성하여 관리. 마크다운 글머리 기호(들여쓰기 공백 2~4칸)를 활용해 계층 구조를 만든다.
- **.gitignore 설정**: 프로젝트 루트에 `.gitignore` 파일을 생성하여 깃 추적 제외 대상을 지정한다. 제외 대상 예: 가상환경 폴더(`.venv/`), 파이썬 캐시 폴더(`**/__pycache__/`), 환경변수 파일(`.env`) 등.

## Mac Book

- **Homebrew 설치**: Homebrew를 통한 파이썬 설치는 `brew install python`
- **터미널 기본 파이썬 버전 고정 및 별칭 설정**
  - `~/.zshrc`에 Homebrew 경로 추가: `echo 'export PATH="/opt/homebrew/bin:$PATH"' >> ~/.zshrc`
  - `python` 명령어 입력 시 `python3`이 실행되도록 설정: `echo 'alias python="python3"' >> ~/.zshrc`
  - 설정 반영: `source ~/.zshrc`
- **가상환경 실행 및 종료**
  - 실행: `source .venv/bin/activate`
  - 종료: `deactivate`

## Python

### pandas

- **CSV 파일 로드**
  - `df = pd.read_csv('data.csv')`
  - 인덱스 열 지정 로드: `pd.read_csv('data.csv', index_col=0)`로 하면 이름 없는 컬럼이 `Unnamed: 0`처럼 바뀌는 것을 방지할 수 있다.
- **출력 줄바꿈 방지 설정**
  - `pd.set_option('display.expand_frame_repr', False)`: 너비에 따른 열 줄바꿈 방지
  - `pd.set_option('display.max_columns', None)`: 열 생략 방지
- **head / tail**
  - `df.head(n)`: 위에서부터 n개 행 출력 (기본값 5)
  - `df.tail(n)`: 아래에서부터 n개 행 출력 (기본값 5)
- **shape**: `df.shape` — 데이터프레임의 크기를 (행, 열) 형태의 튜플로 반환
- **columns**: `df.columns` — 모든 컬럼의 이름 목록 반환
- **dtypes**: `df.dtypes` — 각 열의 데이터 타입만 요약 출력
- **info**: `df.info()` — 전체 행/열 개수, 데이터 타입, 결측치 개수, 메모리 사용량을 확인하는 종합 명세서

  > [!WARNING]
  > `df.info()`는 자체 출력 기능이 있어 `print(df.info())`로 쓰면 맨 아래에 `None`이 출력된다. 단독으로 호출해야 한다.

- **describe**: `df.describe()` — 수치형 데이터의 개수, 평균, 표준편차, 최솟값, 사분위수(25%, 50%, 75%), 최댓값을 자동 계산해 요약 출력
- **열 이름 바꾸기**: `df.rename(columns={'기존이름': '새이름'}, inplace=True)`. `inplace=True`를 주어야 원본 데이터프레임에 즉시 반영된다.
- **고유값 개수 및 종류 확인**
  - `df['열이름'].nunique()`: 중복을 제외한 고유값의 총 개수 반환
  - `df[['열1', '열2']].nunique()`: 대괄호 쌍으로 여러 열의 고유값 개수를 동시에 확인
  - `df['열이름'].unique()`: 고유값의 종류 자체를 배열로 반환
- **데이터 개수 세기**
  - `size`: 결측치 포함 여부와 관계없이 그룹 내 전체 행 개수를 단일 숫자로 반환
  - `count()`: 결측치를 제외한 유효 데이터 개수를 열별로 각각 반환
- **조합별 빈도수 구하기 (경우의 수)**
  - 방법 1: `pd.crosstab(df['열1'], df['열2'])` — 가로·세로 격자 형태의 교차표 출력
  - 방법 2: `df.groupby(['열1', '열2']).size()` — 세로로 길게 나열되는 리스트 형태 출력

### SQLite3

- **with 구문**: 에러가 없으면 자동으로 커밋, 에러가 있으면 자동 롤백

  ```python
  with sqlite3.connect('w1m2.db') as conn:
      cursor = conn.cursor()
      cursor.execute("""쿼리내용""")
  ```

- **TOP 50 PERCENT**: SQLite에는 퍼센트 기능이 없어 다음과 같은 방법을 사용한다.

  ```sql
  SELECT * FROM Customers
  LIMIT (SELECT COUNT(*) FROM Customers) / 2;
  ```

  기본적으로 소수점 아래는 버려진다. 파이썬에서 계산하려면 총 데이터 개수를 가져오고 계산한 뒤 다시 쿼리를 보내야 하므로, 쿼리에서 바로 계산하는 편이 더 효율적이다.

- **컬럼명 같이 출력**

  ```python
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

- **GROUP BY**: `GROUP BY`에 적은 컬럼과 집계함수 외에는 `SELECT` 하면 안 된다.

  ```sql
  SELECT MIN(Price) AS SmallestPrice, CategoryID
  FROM Products
  GROUP BY CategoryID;
  ```

  `HAVING`으로 조건을 걸 수 있다. `WHERE`는 그룹을 묶기 전, `HAVING`은 그룹을 묶은 후 필터링한다.

- **띄어쓰기가 있는 컬럼명**: 쌍따옴표(표준)나 대괄호(비표준)로 감싼다.

  ```sql
  SELECT COUNT(*) AS [Number of records]
  FROM Products;
  ```

- **LIKE**
  - 기본적으로 대소문자를 구분하지 않는다. 구분하려면 `PRAGMA case_sensitive_like = true;`
  - 와일드카드: `%`는 0글자 이상의 아무 문자, `_`는 1글자 고정 아무 문자
- **GLOB**: `LIKE`와 비슷하지만 대소문자를 구분하며 `[]` 연산을 지원한다.
  - 와일드카드: `*`는 `LIKE`의 `%`, `?`는 `LIKE`의 `_`
  - `[]`: 괄호 안의 문자에 해당하는지 확인. `[a-fA-F]`처럼 범위 지정도 가능하다.
- **IN**: 데이터 개수가 많아지면 `OR` 연산보다 `IN`이 훨씬 빠르다.
- **BETWEEN**: 문자열도 가능하다. 기본적으로 대소문자를 구분하며, 구분하지 않으려면 `COLLATE NOCASE`를 붙인다.
- **문자열 합치기**: `+`, `CONCAT`, `||`
- **JOIN**: `INNER JOIN`이 기본값이다.
- **`<>`**: 같지 않다(`!=`)
- **UNION**: `SELECT`문을 중복 제거하며 결합한다. 추출 컬럼의 데이터 타입/순서는 일치해야 한다. 아래처럼 컬럼명에 Alias를 지정하면 전체에 반영된다. 중복을 포함하려면 `UNION ALL`을 사용한다.

  ```sql
  SELECT 'Customer' AS Type, ContactName, City, Country
  FROM Customers
  UNION
  SELECT 'Supplier', ContactName, City, Country
  FROM Suppliers;
  ```

- **EXISTS**: 존재 여부를 확인한다. 아래와 같은 경우 안쪽 `SELECT`문의 컬럼명은 의미가 없다.

  ```sql
  SELECT SupplierName
  FROM Suppliers
  WHERE EXISTS (
      SELECT ProductName
      FROM Products
      WHERE Products.SupplierID = Suppliers.supplierID AND Price = 22
  );
  ```

- **ANY**: 하나라도 있는지 확인한다. SQLite에는 없으므로 `IN`으로 대체한다.
- **COALESCE**: 여러 값 중 NULL이 아닌 값을 처음 나온 순서로 반환한다.
- **IFNULL**: 인자 두 개만 쓸 수 있다. `IFNULL(A, B)`는 A가 NULL이면 B를 반환한다.
- **Stored Procedure**: SQLite에는 없다.
- **PARTITION BY**: 같은 값을 가진 것끼리 구역을 나눈다.

### matplotlib

- **모든 변수의 히스토그램 그리기**
  - `df.hist(figsize=(가로, 세로), bins=구간개수, grid=False)` — 가로/세로 단위는 인치
  - 전체 대제목 설정: `plt.suptitle('타이틀')`
  - 배치 최적화: `plt.tight_layout()` — 그래프와 글자가 겹치지 않게 조절
- **Scatter 차트 그리기**
  - `plt.scatter(x축_데이터, y축_데이터, color='색상', alpha=투명도, s=점크기)`
  - 타이틀 및 레이블 설정: `plt.title()`, `plt.xlabel()`, `plt.ylabel()`
  - 격자선 표시: `plt.grid(True)`
  - 화면 출력: `plt.show()`

### multiprocessing

- **Pool**: 워커 수를 미리 할당하는 방식
- **Process**: 수동으로 직접 프로세스를 생성하는 방식
- **Queue**
  - `get`: pop과 같은 역할. 데이터가 없으면 생길 때까지 대기한다.
  - `get_nowait`: 데이터가 없으면 `queue.Empty` 에러 발생 후 넘어간다. `get(block=False)`와 같다.

## 개념

### 상관계수

두 변수 간에 선형적 또는 비선형적 관계가 존재하는지, 그리고 그 관계가 얼마나 밀접한지를 -1부터 1 사이의 수치로 나타낸 지표.

- 1에 가까울수록 강한 양의 상관관계 (한쪽이 커지면 다른 쪽도 커짐)
- -1에 가까울수록 강한 음의 상관관계 (한쪽이 커지면 다른 쪽은 작아짐)
- 0에 가까울수록 두 변수 간에 아무런 상관관계가 없음

상관계수 표 구하기: `df.corr(numeric_only=True)` (문자열 열을 제외하고 수치형 데이터만 계산)

상관계수의 3가지 종류

- **피어슨(Pearson) 상관계수**: 두 변수 간의 선형적(직선형) 관계의 강도를 측정한다. 두 변수가 모두 연속형 숫자 데이터이고 정규분포를 따를 때 정확하지만, 이상치가 있으면 수치가 크게 왜곡된다.
- **스피어먼(Spearman) 순위 상관계수**: 데이터의 실제 값 대신 순위(Rank)를 매겨서 상관관계를 측정한다. 직선 형태가 아니더라도 한쪽이 증가할 때 다른 쪽도 같이 증가하는 monotonic 관계를 잘 잡아내며, 이상치에 영향을 거의 받지 않는다.
- **켄달(Kendall) 타우**: 스피어먼과 마찬가지로 순위 기반이지만, 데이터 쌍들이 서로 크고 작은 방향이 일치하는지 순서 관계를 따져서 계산한다. 샘플 개수가 적거나 같은 순위가 많을 때 스피어먼보다 더 안정적이고 정확하다.

활용법: `method` 옵션 변경 — `df.corr(method='pearson')`, `df.corr(method='spearman')`, `df.corr(method='kendall')`
