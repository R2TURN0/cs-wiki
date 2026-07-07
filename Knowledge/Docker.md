# Docker 사용법에 대해 정리

설치 - docker desktop

dockerfile: 이미지에 생성을 위한 정보

이미지 빌드
docker build -t welcome-to-docker .
-t: tags(생성할 이미지 이름)
. : 경로, .은 현재 위치

도커 데스크탑 기준 컨테이너 옵션에서 포트 지정 가능

cmd+k 로 도커 허브의 다른 이미지들 이용 가능, 검색어 검색 후 이미지 선택해서 run 누르면 됨

docker compose
여러 컨테이너를 한번에 실행시켜서 서비스 한번에 띄울 수 있음

compose.yaml: 이를 위한 실행 방법을 도커에게 알려주는 파일

도커 컴포즈 실행
docker compose up -d
-d: detach 모드로 실행. 백그라운드 실행 모드임

docker compose watch
실시간 수정 반영되는 개발 모드
끝내려면 ctrl+c

앱 지워도 도커 컴포즈로 쉽게 재실행 가능

단 db컨테이너가 지워지면 내용은 지워진다

이를 방지하기 위해선 volume을 쓸것
volume은 도커가 관리하는 파일시스템임

사용하려면 compose.yaml에 아래 코드 추가

todo-database:

    volumes: # 공용 database를 todo-database에서 사용할 /data/db로 연결함
      - database:/data/db #도커 컴포즈 문법임. 연결할 원본 경로:사용할 경로 이름

volumes: #공용 공간에 database 공간 생성
  database:
              
도커 컨테이너 데이터들은 로컬 파일시스템과 분리되어있다.
도커 컨테이너에서 로컬 파일시스템 접근하게 하고 싶으면 bind mount 사용

todo-app:
    # ...
    volumes:
      - ./app:/usr/src/app # 로컬 파일시스템의 ./app 경로를 /usr/src/app에 연결
      - /usr/src/app/node_modules # 해당 경로는 연결(덮어쓰기) 방지
이런식으로 연결해두면 매번 빌드 안해도 내 컴퓨터의 코드 수정하면 컨테이너에 자동 반영되어서 좋다

프로젝트 containerize
도커 파일/컴포즈 빌드하는법
docker init #이 명령어로 도커파일과 컴포즈야믈의 기본적인 필요한 것들 생성 가능
부족하다면 아래 문서들 참고
- https://docs.docker.com/reference/dockerfile
- https://docs.docker.com/reference/compose-file/


docker compose up --build # 이미지 새로 만들기
