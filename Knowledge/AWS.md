# AWS Tips

AWS로 서비스를 운영하며 알게 된 정보를 정리하는 문서

---

## EC2

서버 인스턴스 역할을 하는 서비스.

### 인스턴스 생성

- 인스턴스 생성 시 **퍼블릭 IP 할당을 켤 것**.

### user-data.sh

인스턴스 생성 시 등록해두면 **최초 부팅 때 한 번만** 실행되는 초기화 스크립트다. EC2의 cloud-init이 user data를 읽어 실행한다. 아래 스크립트는 부팅 시 Docker 환경을 자동으로 구성한다.

> [!WARNING]
> user data는 인스턴스 최초 시작 시에만 실행된다. 스크립트를 수정하려면 인스턴스를 삭제하고 다시 생성해야 한다.

#### 전체 스크립트

```bash
#!/bin/bash

# 2. 시스템 패키지 업데이트
apt-get update -y
apt-get upgrade -y

# 3. Docker GPG 키 등록
apt-get install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

# 4. Docker 저장소 등록
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

# 5. Docker 설치
apt-get update -y
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 6. 서비스 시작 및 권한 설정
systemctl start docker
systemctl enable docker
usermod -aG docker ubuntu
```

#### 1. 셔뱅 (shebang)

```bash
#!/bin/bash
```

이 파일을 bash 셸로 실행하라는 선언이다. cloud-init은 user data 첫 줄이 `#!`로 시작하면 스크립트로 인식한다. 이 줄이 없으면 스크립트로 취급되지 않으므로 반드시 첫 줄에 있어야 한다.

#### 2. 시스템 패키지 업데이트

```bash
apt-get update -y
apt-get upgrade -y
```

- `update`: 설치 가능한 패키지 목록(카탈로그)을 최신으로 갱신한다. 패키지를 설치하는 게 아니라 목록만 새로고침한다.
- `upgrade`: 이미 설치된 패키지를 실제 최신 버전으로 올린다. 갓 만들어진 AMI는 보안 패치가 밀려 있을 수 있어 필요하며, **부팅 시간의 대부분(수 분)을 차지**하는 단계다.
- `-y`: "계속할까요? [Y/n]" 프롬프트에 자동으로 yes. 무인 실행이므로 필수다.

#### 3. Docker 저장소 신뢰 설정 (GPG 키)

```bash
apt-get install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
```

Docker는 우분투 기본 저장소가 아닌 Docker 자체 저장소에서 설치한다(기본 저장소의 `docker.io`는 버전이 오래됨). 외부 저장소를 추가하려면 그 저장소의 패키지가 진짜 Docker가 서명한 것인지 검증할 공개키가 먼저 필요하다.

- `ca-certificates curl`: HTTPS 통신용 인증서 묶음과 파일 다운로드용 curl 설치
- `install -m 0755 -d`: 키 보관 디렉토리 `/etc/apt/keyrings`를 755 권한으로 생성 (mkdir + chmod를 한 번에)
- `curl ... -o docker.asc`: Docker GPG 공개키 다운로드 및 저장
  - `-f` HTTP 에러 시 실패 처리 / `-s` 진행바 숨김 / `-S` 에러는 표시 / `-L` 리다이렉트 따라가기
- `chmod a+r`: 모든 사용자가 읽을 수 있게. apt가 root가 아닌 내부 계정으로 키를 읽기 때문에 필요하다.

#### 4. Docker 저장소 등록

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
```

"이 주소에서도 패키지를 받아라"라고 apt에 알려주는 설정 파일(`/etc/apt/sources.list.d/docker.list`)을 만든다.

- `arch=$(dpkg --print-architecture)`: 현재 머신의 CPU 아키텍처 자동 감지 (t2.micro → `amd64`, t4g 같은 ARM → `arm64`)
- `signed-by=...docker.asc`: 이 저장소의 패키지는 3번에서 받은 키로 서명 검증하라는 의미
- `$(. /etc/os-release && echo "$VERSION_CODENAME")`: OS 정보를 읽어 우분투 코드네임을 자동 삽입 (예: 26.04 → `resolute`). 우분투 버전이 바뀌어도 스크립트 수정이 불필요하다.
- `stable`: 안정 버전 채널 사용

> [!NOTE]
> 원본에서 `sudo tee`를 쓴 이유는 `sudo echo > 파일`이 리다이렉트에 권한을 적용하지 못해 실패하기 때문이다. 지금은 전체가 root로 실행되므로 `echo "..." > /etc/apt/sources.list.d/docker.list`로 줄여도 되지만, 그대로 둬도 무해하다.

#### 5. Docker 설치

```bash
apt-get update -y
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

`update`를 한 번 더 하는 이유는 4번에서 저장소를 새로 추가했으니 apt가 그 목록을 읽어오게 해야 하기 때문이다. 빼면 "docker-ce를 찾을 수 없음" 에러가 난다.

| 패키지 | 역할 |
| --- | --- |
| `docker-ce` | Docker 엔진 본체 (데몬, CE = Community Edition) |
| `docker-ce-cli` | `docker run`, `docker pull` 등 명령어 도구 |
| `containerd.io` | 컨테이너를 실제로 생성·실행하는 저수준 런타임 (엔진이 내부적으로 사용) |
| `docker-buildx-plugin` | 이미지 빌드 확장 (멀티 아키텍처 빌드 등) |
| `docker-compose-plugin` | `docker compose` 명령 지원 (여러 컨테이너를 yaml 하나로 관리) |

#### 6. 서비스 시작 및 권한 설정

```bash
systemctl start docker
systemctl enable docker
usermod -aG docker ubuntu
```

- `start`: Docker 데몬을 지금 즉시 시작 ("지금")
- `enable`: 재부팅해도 Docker가 자동 시작되도록 등록 ("앞으로도") — 둘 다 필요하다.
- `usermod -aG docker ubuntu`: `ubuntu` 계정을 docker 그룹에 추가. Docker 데몬은 root 소유 소켓으로 통신하므로, 그룹에 넣지 않으면 일반 유저는 `docker ps`만 쳐도 permission denied가 난다. `-a`(append)가 없으면 기존 그룹이 전부 교체되니 주의.

> [!WARNING]
> 그룹 변경은 새 로그인 세션부터 적용된다. 이미 SSH로 접속 중이었다면 재접속해야 `sudo` 없이 docker 명령을 쓸 수 있다.

#### 전체 흐름 요약

시스템 최신화(2) → Docker 저장소 키 등록(3) → 저장소 주소 등록(4) → Docker 설치(5) → 부팅 시 자동 실행 + ubuntu 유저 권한 부여(6). 결과적으로 인스턴스가 뜨면 SSH 접속 후 바로 `docker pull`, `docker run`이 가능한 상태가 된다.

### 메모리 부족 (swap 설정)

램이 부족해 워드클라우드 분석 실행 시 커널이 죽는 문제가 있었다. EC2 스토리지 볼륨 크기를 늘린 뒤, 터미널에서 다음 과정으로 디스크 일부를 swap(대체 램)으로 등록한다.

**1. 파티션 크기 확장**

```bash
sudo growpart /dev/xvda 1
```

**2. 파일 시스템 확장**

```bash
sudo resize2fs /dev/root
```

**3. 디스크에서 swap 할당**

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab  # 재부팅 시에도 적용
```

---

## ECR

컨테이너 이미지를 관리하는 서비스. EC2에서 이미지를 받아오려면 [AWS CLI](#aws-cli) 설치가 필수다.

---

## AWS CLI

```bash
brew install awscli
```

> [!NOTE]
> AWS CLI 인증 오류가 반복될 경우, 임시 키(temporary credentials)를 발급받아 해결했다.