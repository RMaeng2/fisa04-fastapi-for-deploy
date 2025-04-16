# AWS를 사용한 FastAPI 기반 실시간 머신러닝 서빙 애플리케이션 클라우드 마이그레이션

이 실습 자료는 FastAPI 기반의 실시간 머신러닝 서빙 애플리케이션을 AWS 환경에 배포하는 과정을 다룹니다. Docker를 사용하여 이미지화하고, .env 파일을 통해 환경 변수 관리를 하며, EC2 인스턴스 상에서 실행할 수 있도록 구성합니다.

### 📁프로젝트 구조
```
├── app/
│   ├── main.py             # FastAPI 진입점
│   ├── ...
│   └── logs/               # 로그 저장 디렉토리 (호스트와 마운트)
├── .env                    # 환경 변수 파일
├── Dockerfile              # 컨테이너 이미지 정의
├── requirements.txt        # Python 의존성 명세
├── .gitignore              # Git 추적 제외 파일
├── .dockerignore           # Docker 이미지 제외 설정
└── README.md               # 현재 문서
```

### 🐳 이미지 빌드와 실행
```
$ docker build -t fisa04-fastapi .

# 윈도우인 경우
$ docker run -d -p 80:8000 --env-file .env -v %cd%/app/logs:/app/logs --name fastapi-docker fisa04-fastapi

# 리눅스에서
$ docker run -d -p 80:8000 --env-file .env -v ./app/logs:/app/logs --name fastapi-docker fisa04-fastapi
```

### 📡 EC2에서의 배포 참고사항
- 본인의 VPC를 생성하고 해당 VPC 안에 서버를 배치합니다.
- EC2 인스턴스는 보안 그룹에서 80번 포트 인바운드를 허용해야 합니다.
- EC2에 Docker(필요하다면 Git도) 설치되어 있어야 합니다.
- .env 파일과 로그 디렉토리는 EC2 내부에 위치해야 하며, docker run 시 경로를 정확히 지정해야 합니다.
- FastAPI 애플리케이션 로그는 ./app/logs 디렉토리에 저장되며, 컨테이너 외부 볼륨으로 마운트되어 유지됩니다.

### 🔎 접속 확인
FastAPI 서버 실행 후, 브라우저에서 EC2 인스턴스의 퍼블릭 IP를 입력하면 기본 라우트에 접속할 수 있습니다.

```
http://<EC2_PUBLIC_IP>/
http://<EC2_PUBLIC_IP>/docs  # Swagger 문서
```

### 🛠 트러블슈팅 & 향후 개선
#### AWS
1. VPC에서 생성한 보안 그룹이 EC2 인스턴스 생성 시 목록에 나타나지 않음  <br>
문제 : 동일한 리전과 VPC에 생성한 보안 그룹이 EC2 인스턴스 생성 시 선택 목록에 나타나지 않음  <br>
해결 : 기존 보안 그룹을 삭제하고, EC2 인스턴스 생성 중 새로 보안 그룹을 생성함으로써 문제 해결  <br>

2. AMI로 인스턴스를 생성하지 않고 시작 템플릿을 생성함  <br> 
문제 : "AMI를 템플릿으로 사용"하라는 말을 보고, Launch Template 생성이 필요한 것으로 착각  <br>
원인 : AMI(이미지)와 Launch Template(설정 템플릿)의 개념 혼동  <br>
해결 : 생성한 시작 템플릿 삭제하고 AMI로 다시 인스턴스 생성함  <br>
📘 정확한 의미  <br>
AMI (Amazon Machine Image) = EC2 인스턴스의 스냅샷. OS, 설정, 패키지 포함.  <br>
Launch Template = EC2를 만들기 위한 템플릿 문서 (AMI + 인스턴스 타입 + 키페어 + 태그 + 유저데이터 등 세팅 포함 가능).  <br>
👉 즉, AMI는 “무엇을 띄울지”, 템플릿은 “어떻게 띄울지”를 정의  <br>

3. 배포용 EC2에 퍼블릭 IP를 할당하지 않음  <br>
퍼블릭 IP : EC2 중단 및 종료 시 변경되거나 사라짐  <br>
탄력적 IP : 고정 IP, 지속적 서비스 운영에 적합  <br>
해결 : 탄력적 IP를 할당함  <br>

#### jenkins
1. Jenkins 플러그인 설치 오류 <br>
문제 : 네트워크 이슈 혹은 권한 문제로 Jenkins 플러그인 설치 실패. <br>
해결 : 필요한 .hpi 플러그인 파일을 직접 다운로드하여 jenkins에서 플러그인 직접 설치 <br>

2. Jenkins EC2에 Docker 설치 누락 <br>
문제 : Jenkins 서버에서 배포 이미지를 생성하기 위한 Docker 명령 실행 불가. <br>
해결 : Jenkins 서버에 수동으로 Docker 설치 <br>

3. Jenkins 사용자에게 sudo 권한이 없어서 Docker 명령 실행 실패 <br>
문제 : Jenkins가 sudo docker build, sudo docker run 같은 명령을 실행할 때 비밀번호 입력 요구로 자동화 실패 발생. <br>
해결 : sudo visudo 명령으로 편집기 열고 아래 줄 추가하여 비밀번호 입력 없이 sudo 명령 실행 가능하도록 설정함: <br>
`jenkins ALL=(ALL) NOPASSWD: ALL` : jenkins 사용자에게 모든 명령어를 비밀번호 없이 sudo로 실행할 수 있도록 허용 <br>
