인스턴스 생성시 user-data.sh 등록해둬야 함. 처음 시작할때만 실행되기에 수정하려면 인스턴스를 삭제하고 재시작해야함

## user-data.sh 설명

1. 셔뱅 (shebang)
#!/bin/bash
이 파일을 bash 셸로 실행하라는 선언입니다. EC2의 cloud-init이 user data를 읽을 때 첫 줄이 #!로 시작하면 스크립트로 인식해서 실행합니다. 이 줄이 없으면 스크립트로 취급되지 않으니 반드시 첫 줄에 있어야 합니다.

2. 시스템 패키지 업데이트
apt-get update -y
apt-get upgrade -y
update: 우분투가 알고 있는 "설치 가능한 패키지 목록"을 최신으로 갱신합니다. 패키지 자체를 설치하는 게 아니라 카탈로그만 새로고침하는 것입니다.
upgrade: 이미 설치된 패키지들을 실제로 최신 버전으로 올립니다. 갓 만들어진 AMI는 보안 패치가 밀려 있을 수 있어서 해주는 것이고, 이 단계가 부팅 시간의 대부분(수 분)을 차지합니다.
-y: 중간에 "계속할까요? [Y/n]" 하고 물어볼 때 자동으로 yes. user data는 사람이 응답할 수 없는 무인 실행이라 필수입니다.
3. Docker 저장소 신뢰 설정 (GPG 키)
apt-get install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
Docker는 우분투 기본 저장소가 아닌 Docker 자체 저장소에서 설치합니다(기본 저장소의 docker.io는 버전이 오래됨). 외부 저장소를 추가하려면 "이 저장소에서 온 패키지가 진짜 Docker가 서명한 것인지" 검증할 공개키가 먼저 필요합니다.

ca-certificates curl: HTTPS 통신에 필요한 인증서 묶음과, 파일을 내려받을 curl 설치
install -m 0755 -d ...: 키를 보관할 /etc/apt/keyrings 디렉토리를 755 권한으로 생성 (mkdir + chmod를 한 번에 하는 명령)
curl ... -o docker.asc: Docker의 GPG 공개키를 내려받아 저장. 옵션은 -f(HTTP 에러 시 실패 처리) -s(진행바 숨김) -S(에러는 표시) -L(리다이렉트 따라가기)
chmod a+r: 모든 사용자가 읽을 수 있게 — apt가 root가 아닌 내부 계정으로 키를 읽기 때문에 필요합니다.
4. Docker 저장소 등록
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
"이 주소에서도 패키지를 받아라"라고 apt에 알려주는 설정 파일을 만드는 부분입니다. 한 줄짜리 저장소 정의를 /etc/apt/sources.list.d/docker.list에 써 넣습니다.

arch=$(dpkg --print-architecture): 현재 머신의 CPU 아키텍처를 자동 감지 (t2.micro면 amd64, t4g 같은 ARM 인스턴스면 arm64)
signed-by=...docker.asc: 이 저장소의 패키지는 3번에서 받은 키로 서명 검증하라는 의미
$(. /etc/os-release && echo "$VERSION_CODENAME"): OS 정보 파일을 읽어 우분투 코드네임을 자동으로 넣습니다. 26.04면 resolute가 들어갑니다. 이 부분 덕분에 우분투 버전이 바뀌어도 스크립트 수정이 필요 없습니다.
stable: 안정 버전 채널 사용
tee ... > /dev/null: echo의 출력을 파일에 기록. 원본 스크립트에서 sudo tee를 쓴 것은 sudo echo > 파일이 리다이렉트에는 권한이 적용되지 않아 실패하기 때문인데, 지금은 전체가 root로 돌아서 사실 echo "..." > /etc/apt/sources.list.d/docker.list로 줄여도 됩니다. 그대로 둬도 무해합니다.
5. Docker 설치
apt-get update -y
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
update를 한 번 더 하는 이유: 방금 4번에서 저장소를 새로 추가했으니, apt가 그 저장소의 패키지 목록을 읽어오게 해야 합니다. 이걸 빼면 "docker-ce를 찾을 수 없음" 에러가 납니다.
설치하는 5개 패키지:
docker-ce: Docker 엔진 본체 (데몬, CE = Community Edition)
docker-ce-cli: docker run, docker pull 같은 명령어 도구
containerd.io: 컨테이너를 실제로 생성·실행하는 저수준 런타임 (Docker 엔진이 내부적으로 사용)
docker-buildx-plugin: 이미지 빌드 확장 (멀티 아키텍처 빌드 등, docker build가 내부적으로 사용)
docker-compose-plugin: docker compose 명령 지원 (여러 컨테이너를 yaml 하나로 관리)
6. 서비스 시작 및 권한 설정
systemctl start docker
systemctl enable docker
usermod -aG docker ubuntu
start: Docker 데몬을 지금 즉시 시작
enable: 인스턴스를 재부팅해도 Docker가 자동으로 시작되도록 등록. start는 "지금", enable은 "앞으로도"라서 둘 다 필요합니다.
usermod -aG docker ubuntu: ubuntu 계정을 docker 그룹에 추가. Docker 데몬은 root 소유의 소켓으로 통신하기 때문에 기본적으로 일반 유저는 docker ps만 쳐도 permission denied가 납니다. 그룹에 넣어두면 SSH 접속 후 sudo 없이 docker 명령을 쓸 수 있습니다. -aG의 -a(append)가 중요한데, 없으면 기존 그룹들이 전부 교체돼 버립니다.
주의: 그룹 변경은 새 로그인 세션부터 적용되므로, 이미 SSH로 접속해 있었다면 재접속해야 합니다.
전체 흐름 요약
시스템 최신화(2) → Docker 저장소를 믿을 수 있게 키 등록(3) → 저장소 주소 등록(4) → 그 저장소에서 Docker 설치(5) → 부팅 시 자동 실행 + ubuntu 유저 권한 부여(6). 결과적으로 인스턴스가 뜨면 SSH 접속해서 바로 docker pull, docker run을 할 수 있는 상태가 됩니다.

EC2 인스턴스 만들때 public IP 킬것

EC2: 서버 인스턴스

ECR: 컨테이너 관리하는 서비스

ECR 사용시 aws cli 설치 필수

brew install awscli

aws cli 인증 오류 계속 남 - 임시 키 발급받아 해결

램이 부족해서 워드클라우드 분석 실행시 커널이 죽음

ec2 스토리지 볼륨 크기를 늘리고 터미널에서 아래 과정을 통해 대체 램 등록

1. 파티션 크기 확장

`sudo growpart /dev/xvda 1`

2. 파일 시스템 확장

`sudo resize2fs /dev/root`

3. 디스크에서 대체 램 할당

```
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab # 재부팅 시에도 적용
```