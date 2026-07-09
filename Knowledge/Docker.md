# Docker

Docker를 사용하며 알게 된 정보를 정리하는 문서

---

## 설치

Docker Desktop 설치.

---

## Dockerfile & 이미지 빌드

**Dockerfile**은 이미지 생성을 위한 정보를 담는 파일이다.

### 이미지 빌드

```bash
docker build -t welcome-to-docker .
```

- `-t`: tag. 생성할 이미지 이름을 지정한다.
- `.`: 빌드 컨텍스트 경로. `.`은 현재 위치를 의미한다.

### 플랫폼 지정

서버 OS 아키텍처에 맞게 `--platform` 옵션을 지정해야 할 때가 있다.

```bash
docker build --platform linux/amd64 -t my-jupyter-image .
```

### 대화형으로 한 줄씩 작성하기

Dockerfile을 처음부터 완성하려 하지 않고, 베이스 이미지를 대화형으로 띄워 명령을 하나씩 직접 실행해본 뒤, 잘 동작하는 명령만 골라 Dockerfile에 한 줄씩 옮겨 적는 방식이다. Docker Desktop의 내장 터미널에서도 그대로 할 수 있다.

```bash
# 1. 베이스 이미지를 대화형 셸로 실행
docker run -it python:3.12-slim bash

# 2. 컨테이너 안에서 명령을 직접 실행해보며 확인
#    (예: apt-get update, pip install ... 등)

# 3. 정상 동작한 명령을 Dockerfile에 RUN 등으로 한 줄씩 옮겨 적는다
```

- `-i`: 컨테이너의 표준 입력을 열어둬 입력을 전달할 수 있게 한다.
- `-t`: 유사 터미널(TTY)을 붙여 대화형 셸처럼 쓰게 한다. 보통 `-it`로 함께 쓴다.

> [!TIP]
> 매번 전체를 빌드하지 않고 명령 단위로 검증하므로, 어느 단계에서 실패하는지 빠르게 파악할 수 있다. 셸에서 나가려면 `exit`.

---

## Docker Desktop

- 컨테이너 옵션에서 **포트 지정**이 가능하다.
- `Cmd + K`로 Docker Hub의 다른 이미지들을 이용할 수 있다. 검색어로 이미지를 찾은 뒤 선택해서 **Run**을 누르면 된다.

### Builds — 빌드 내역 확인 & 디버깅

대시보드 메뉴의 **Builds** 탭. Dockerfile을 작성하는 기능이 아니라, 실행한 빌드를 들여다보고 디버깅하는 도구다.

- `docker build` / `docker buildx build`로 시작한 빌드가 자동으로 목록에 나타난다.
- 각 빌드의 소요 시간, 캐시 사용량, Dockerfile 소스, 로그 등을 확인할 수 있다.
- **디버깅에 유용:** 실패한 단계 바로 옆에 스택 트레이스가 표시되어, 어느 명령에서 왜 실패했는지 바로 파악할 수 있다. 터미널 로그를 잃어버린 뒤에도 과거 빌드를 다시 열어볼 수 있다.
- Buildx 빌더 인스턴스 관리(빌더 설정, 스토리지 사용량 확인)도 이 화면에서 한다.

> [!NOTE]
> "한 줄씩 작성하고 실행"하는 건 위 [대화형으로 한 줄씩 작성하기](#대화형으로-한-줄씩-작성하기)를 참고. Builds 탭은 이미 실행한 빌드의 **결과를 분석**하는 용도라 목적이 다르다.

---

## Docker Compose

여러 컨테이너를 한 번에 실행해 서비스를 통째로 띄울 수 있다.

**compose.yaml**은 그 실행 방법을 Docker에게 알려주는 파일이다.

### 실행

```bash
docker compose up -d
```

- `-d`: detach 모드. 백그라운드로 실행한다.

### 개발 모드 (실시간 반영)

```bash
docker compose watch
```

코드 수정이 실시간으로 반영되는 개발 모드. 종료하려면 `Ctrl + C`.

> [!NOTE]
> 앱을 지워도 Docker Compose로 쉽게 재실행할 수 있다. 단, **DB 컨테이너가 지워지면 내용도 함께 지워진다.** 이를 방지하려면 아래 Volume을 사용한다.

---

## 데이터 유지: Volume & Bind Mount

Docker 컨테이너의 데이터는 로컬 파일시스템과 분리되어 있다. 목적에 따라 두 방식으로 연결한다.

### Volume — 데이터 영속화

Volume은 Docker가 관리하는 파일시스템이다. DB처럼 컨테이너가 지워져도 유지되어야 하는 데이터에 쓴다. `compose.yaml`에 다음을 추가한다.

```yaml
services:
  todo-database:
    volumes:
      # 공용 database 볼륨을 컨테이너의 /data/db에 연결
      # 문법: 원본(볼륨 이름):사용할 경로
      - database:/data/db

volumes:
  # 공용 공간에 database 볼륨 생성
  database:
```

### Bind Mount — 로컬 파일시스템 연결

컨테이너가 로컬 파일시스템에 접근하게 하려면 bind mount를 쓴다. 로컬 코드 수정을 컨테이너에 바로 반영할 때 유용하다.

```yaml
services:
  todo-app:
    # ...
    volumes:
      - ./app:/usr/src/app        # 로컬의 ./app을 컨테이너의 /usr/src/app에 연결
      - /usr/src/app/node_modules # 이 경로는 연결(덮어쓰기) 방지
```

> [!TIP]
> 이렇게 연결해두면 매번 빌드하지 않아도 내 컴퓨터의 코드 수정이 컨테이너에 자동 반영된다.

---

## 프로젝트 Containerize

기존 프로젝트에 Dockerfile과 compose.yaml을 만드는 방법.

```bash
docker init
```

이 명령어로 Dockerfile과 compose.yaml의 기본 틀을 생성할 수 있다. 부족한 부분은 아래 공식 문서를 참고한다.

- Dockerfile: <https://docs.docker.com/reference/dockerfile>
- Compose file: <https://docs.docker.com/reference/compose-file/>

이미지를 새로 만들며 실행하려면:

```bash
docker compose up --build
```