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

---

## Docker Desktop

- 컨테이너 옵션에서 **포트 지정**이 가능하다.
- `Cmd + K`로 Docker Hub의 다른 이미지들을 이용할 수 있다. 검색어로 이미지를 찾은 뒤 선택해서 **Run**을 누르면 된다.

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