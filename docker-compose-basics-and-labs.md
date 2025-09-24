## Docker Compose 기본 개념과 동작 원리 (입문자용)

### Docker Compose란?
- **여러 컨테이너를 하나의 서비스 묶음(프로젝트)으로 관리**하는 도구입니다.
- 애플리케이션을 구성하는 요소(예: 웹, DB, 캐시 등)를 각각 컨테이너로 나누고, 이를 하나의 파일(`docker-compose.yml`)에 정의해 한 번에 생성·시작·중지할 수 있습니다.

### 기본 구성 요소 용어
- **컨테이너(Container)**: 이미지를 실행한 독립된 프로세스 환경. 애플리케이션이 실제로 돌아갑니다.
- **이미지(Image)**: 컨테이너를 만들기 위한 설계도(파일 시스템 스냅샷 + 실행 방법).
- **서비스(Service)**: 동일한 역할을 하는 컨테이너들의 논리적 그룹(예: `web`, `mysql`).
- **프로젝트(Project)**: 하나의 `docker-compose.yml`로 표현되는 전체 묶음. 디렉터리 이름이 기본 프로젝트 이름이 됩니다.
- **네트워크(Network)**: 컨테이너들이 서로 통신할 수 있는 가상 네트워크. 기본은 bridge 타입입니다.
- **볼륨(Volume)**: 컨테이너 재시작/삭제 후에도 유지되는 데이터 저장소.
- **서비스 디스커버리(Service Discovery)**: 같은 Compose 네트워크 안에서는 서비스 이름으로 서로 접근 가능합니다(예: `web`에서 `mysql:3306`).

### 동작 원리 한눈에 보기
1. `docker-compose.yml`을 읽습니다.
2. 정의된 순서와 의존 관계에 맞춰 컨테이너(서비스)를 생성합니다.
3. 프로젝트 이름과 서비스 이름을 조합해 컨테이너 이름을 만듭니다(예: `myproj_web_1`).
4. 같은 프로젝트 네트워크에서 서비스 이름으로 서로 통신합니다.
5. 스케일을 조정해 동일 서비스 컨테이너 수를 늘리거나 줄일 수 있습니다.

### 왜 쓰나요? (핵심 장점)
- **반복 작업 자동화**: `docker run` 옵션들을 YAML에 적어 한 번에 실행/중지.
- **여러 컨테이너 간 연동**: 포트, 네트워크, 볼륨, 의존성 등을 한 파일로 선언.
- **확장(Scale out)**: 동일 서비스 컨테이너를 여러 개 띄워 부하 분산(단일 Docker 호스트 기준).

### Compose 파일(YAML) 기본 구조
- 상위 4대 섹션: `version`, `services`, `volumes`, `networks`.
- 들여쓰기는 반드시 **스페이스(공백 2칸 이상)** 사용. 탭(Tab)은 오류를 유발합니다.
- 도구 이름 표기: 최신 도커는 `docker compose`(공백) 명령을 권장합니다. 구버전은 `docker-compose`(하이픈)입니다.

```yaml
version: '3.8'
services:
  web:
    image: nginx:latest
    ports:
      - '8080:80'
volumes: {}
networks: {}
```
- **version**: Compose 파일 포맷 버전(도커 엔진 버전에 따라 지원 범위가 다를 수 있음).
- **services**: 생성할 컨테이너 묶음 정의.
- **volumes/networks**: 공유 저장소와 네트워크를 정의(필요 시 사용).

### 서비스에서 자주 쓰는 옵션 요약
- **depends_on**: 다른 서비스가 먼저 시작되어야 함을 명시(애플리케이션 준비 상태까지 확인하진 않음).
- **links**: 예전 방식의 연결 설정(대부분 필요 없음). Compose 네트워크에선 서비스 이름으로 접근 가능합니다.
- **ports**: 호스트포트:컨테이너포트 매핑. 예: `- '8000:80'` → 호스트 8000으로 들어오면 컨테이너 80으로 전달.
- **build**: Dockerfile로 이미지를 빌드하여 사용. `context`, `dockerfile`, `args`로 빌드 컨텍스트/파일/인자 지정.
- **extends**: 다른 Compose 파일의 서비스 정의를 상속.
- **networks**: 사용할 네트워크 지정. `driver`, `driver_opts`, `ipam`, `external` 등 세부 설정 가능.
- **volumes**: 컨테이너 경로에 호스트/네임드 볼륨을 연결. `driver`, `driver_opts`, `external` 지원.

### 유용한 명령
- `docker compose up -d`: 백그라운드로 서비스 생성/실행.
- `docker compose ps`: 현재 프로젝트 컨테이너 상태 확인.
- `docker compose up --scale web=3`: `web` 서비스 컨테이너를 3개로 확장.
- `docker compose down`: 프로젝트에 속한 컨테이너/네트워크(옵션에 따라 볼륨도) 정리.
- `docker compose -f <파일>`: 다른 파일 경로 지정.
- `docker compose -p <이름>`: 프로젝트 이름 지정(동일 호스트에서 여러 프로젝트를 겹치지 않게 실행).
- `docker compose config`: Compose 파일 문법·해석 결과를 검증/확인.

---

## 실습 1: 프라이빗 레지스트리 + 레지스트리 UI 구성

### 목표
- 도커 이미지를 저장하는 **프라이빗 레지스트리**와 이를 조회하는 **UI**를 Compose로 빠르게 띄웁니다.

### 사전 준비
- 원격 호스트(예: `container1`)에 접속할 수 있어야 합니다(SSH).
- 외부에서 접속 시 방화벽/보안그룹에 해당 포트를 개방하세요: 5000(레지스트리), 80(UI).
- Compose가 사용할 **외부 네트워크 `net2`**와 **외부 볼륨 `image-volume`**이 미리 존재해야 합니다. 없다면 먼저 만드세요.

```bash
# (필요 시) 외부 네트워크/볼륨 생성 예시
docker network create net2
docker volume create image-volume
```

### 절차
1) 원격 접속 및 작업 폴더 생성
```bash
ssh <user>@<container1-host>
mkdir -p ~/docker_workspace && cd ~/docker_workspace
```
- `ssh`: 원격 호스트에 접속합니다.
- `mkdir -p`: 폴더가 없으면 만들고, 있으면 그대로 둡니다.

2) Compose 파일 작성
```yaml
# docker-compose.yaml
version: '3.8'

services:
  registry:
    image: registry:2
    ports:
      - '5000:5000'
    volumes:
      - image-volume:/var/lib/registry
    networks:
      - net2

  registry_ui:
    image: joxit/docker-registry-ui:2.5.7
    ports:
      - '80:80'
    networks:
      - net2
    environment:
      - SINGLE_REGISTRY_MODE=true
      - REGISTRY_TITLE=MY Image Hub
      - DELETE_IMAGES=true
      - SHOW_CONTENT_DIGEST=true
      - NGINX_PROXY_PASS_URL=http://registry:5000
      - SHOW_CATALOG_NB_TAGS=true
      - TAGLIST_PAGE_SIZE=10
      - REGISTRY_SECURED=false
      - CATALOG_ELEMENTS_LIMIT=1000

volumes:
  image-volume:
    external: true

networks:
  net2:
    external: true
```
- `registry`: 표준 도커 레지스트리 v2 컨테이너.
- `volumes: image-volume:/var/lib/registry`: 이미지 데이터를 외부 볼륨에 보관(컨테이너 삭제에도 유지).
- `registry_ui`: 레지스트리 조회용 UI 컨테이너. `NGINX_PROXY_PASS_URL`로 레지스트리 주소를 지정.
- `networks: net2`: 두 서비스가 같은 네트워크에서 이름으로 통신.

3) 실행
```bash
sudo docker compose up -d
```
- `up -d`: 백그라운드로 컨테이너 생성 및 실행.

4) 확인
```bash
docker compose ps | cat
```
- 서비스 상태가 `Up`이면 정상. 브라우저에서 `http://<host>:80` 접속해 UI 확인.

---

## 실습 2: Nginx 이미지를 Compose로 빌드하고 용량 비교

### 목표
- Ubuntu 기반으로 Nginx 이미지를 빌드하고, **APT 캐시 정리** 전후로 이미지 크기를 비교합니다.

### 초기 버전 작성 및 빌드
1) 폴더와 파일 생성
```bash
mkdir -p ~/docker_workspace/nginx-build && cd ~/docker_workspace/nginx-build
```

2) `docker-compose.yaml`
```yaml
version: '3.8'

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    image: container.company.co.kr/nginx-server:latest
    ports:
      - '8000:80'
```
- `build.context`: 빌드에 사용할 디렉터리.
- `build.dockerfile`: 사용할 Dockerfile 이름.
- `image`: 빌드 결과에 붙일 태그.
- `ports`: 호스트 8000 → 컨테이너 80.

3) `Dockerfile`
```dockerfile
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y nginx

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```
- `DEBIAN_FRONTEND=noninteractive`: 설치 중 대화형 질문 방지.
- `EXPOSE`: 컨테이너가 사용하는 포트의 메타데이터 표시.
- `CMD`: 컨테이너 시작 시 실행할 명령.

4) 빌드 및 실행
```bash
sudo docker compose up -d --build
```
- `--build`: 이미지가 없거나 변경되면 빌드 후 실행.

5) 이미지 크기 확인(비교용 기준 캡처)
```bash
docker images | grep nginx-server | cat
```

### Dockerfile 최적화(캐시 정리) 후 재빌드
1) `Dockerfile` 수정: APT 캐시 삭제 추가
```dockerfile
RUN apt-get update && apt-get install -y nginx && rm -rf /var/lib/apt/lists/*
```

2) 다시 빌드/실행
```bash
sudo docker compose up -d --build
```

3) 이미지 크기 재확인 및 비교
```bash
docker images | grep nginx-server | cat
```
- `rm -rf /var/lib/apt/lists/*`: 패키지 인덱스 캐시를 제거해 이미지 크기를 줄입니다.

---

## 실습 3: 빌드 인자와 템플릿으로 Nginx 설정 파라미터화

### 목표
- Compose `build.args`와 `envsubst`를 사용해 Nginx 포트/문서 루트를 빌드 시 주입합니다.

### Compose 파일 수정
```yaml
version: '3.8'

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    args:
      - NGINX_SERVER_PORT=8080
      - NGINX_SERVER_ROOT=/var/www/html
    image: container.company.co.kr/nginx-server:latest
    ports:
      - '8000:8080'
```
- `args`: 빌드 시 전달되는 변수. Dockerfile의 `ARG`로 받습니다.
- `ports 8000:8080`: 컨테이너 내부 Nginx 포트를 8080으로 바꾸므로 매핑도 8080에 맞춥니다.

### Dockerfile 수정
```dockerfile
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y nginx gettext-base && rm -rf /var/lib/apt/lists/*

ARG NGINX_SERVER_PORT
ARG NGINX_SERVER_ROOT

ENV NGINX_SERVER_PORT=${NGINX_SERVER_PORT}
ENV NGINX_SERVER_ROOT=${NGINX_SERVER_ROOT}

COPY nginx/sites-available/default /etc/nginx/sites-available/default
RUN envsubst "$NGINX_SERVER_PORT $NGINX_SERVER_ROOT" < /etc/nginx/sites-available/default | tee /etc/nginx/sites-available/default

EXPOSE ${NGINX_SERVER_PORT}

CMD ["nginx","-g", "daemon off;"]
```
- `gettext-base`: `envsubst` 명령 제공 패키지.
- `ARG`/`ENV`: 빌드 인자 → 환경 변수로 승격해 후속 명령에서 사용.
- `envsubst`: 템플릿 파일에서 `${변수}` 자리를 실제 값으로 치환.

### Nginx 템플릿 파일 추가
```bash
mkdir -p nginx/sites-available
```
```nginx
# nginx/sites-available/default
server {
    listen       ${NGINX_SERVER_PORT};
    server_name  _;

    root ${NGINX_SERVER_ROOT};
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### 빌드/실행
```bash
sudo docker compose up -d --build
```

---

## 실습 4: Apache 서비스 추가(템플릿 기반 VirtualHost)

### 목표
- Nginx 서비스와 함께 Apache 서비스를 추가로 빌드/실행하고, 포트/문서 루트를 템플릿로 주입합니다.

### Compose에 Apache 서비스 추가
```yaml
services:
  apache-server:
    build:
      context: .
      dockerfile: Dockerfile.apache
      args:
        - APACHE_SERVER_PORT=8080
        - APACHE_SERVER_ROOT=/var/www/html
    image: container.company.co.kr/apache-server:latest
    container_name: apache-server
    ports:
      - '8001:8080'
```
- `container_name`: 컨테이너 이름을 고정(디버깅이나 `docker cp`에 편리).

### Apache 기본 설정 가져오기(참고)
이미 동작 중인 Apache 컨테이너에서 기본 설정을 복사해 템플릿용으로 사용할 수 있습니다.
```bash
# 컨테이너 실행 후 설정 디렉터리 복사
sudo docker compose up -d --build
sudo docker cp apache-server:/etc/apache2 ./
```

### 가상호스트 템플릿 작성
`./apache2/sites-available/000-default.conf` 파일을 아래처럼 변수 기반으로 수정합니다.
```apache
<VirtualHost *:${APACHE_SERVER_PORT}>
    ServerAdmin webmaster@localhost
    DocumentRoot ${APACHE_SERVER_ROOT}

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

### `Dockerfile.apache` 작성
```dockerfile
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y apache2 gettext-base && rm -rf /var/lib/apt/lists/*

ARG APACHE_SERVER_PORT
ARG APACHE_SERVER_ROOT

ENV APACHE_SERVER_PORT=${APACHE_SERVER_PORT}
ENV APACHE_SERVER_ROOT=${APACHE_SERVER_ROOT}

COPY apache2/sites-available/000-default.conf /etc/apache2/sites-available/000-default.conf

RUN echo "Listen ${APACHE_SERVER_PORT}" >> /etc/apache2/ports.conf

RUN envsubst '$APACHE_SERVER_PORT $APACHE_SERVER_ROOT' < /etc/apache2/sites-available/000-default.conf | tee /etc/apache2/sites-available/000-default.conf

EXPOSE ${APACHE_SERVER_PORT}

CMD ["apache2ctl","-D", "FOREGROUND"]
```
- `Listen`: Apache가 바인딩할 포트를 설정 파일에 추가.
- `envsubst`: 템플릿의 `${변수}`를 실제 값으로 치환.

### 빌드/실행 및 확인
```bash
sudo docker compose up -d --build
docker compose ps | cat
```
- 브라우저에서 `http://<host>:8001` 접속하여 Apache 서비스 응답을 확인합니다.

---

## 부록: 스케일/프로젝트/검증 명령 빠른 참고
- 스케일: `docker compose up --scale mysql=3`
- 특정 서비스만 실행: `docker compose up -d mysql`
- 특정 서비스만 실행(의존성 무시): `docker compose up --no-deps web`
- 프로젝트 이름 지정: `docker compose -p myproj up -d`
- 중지/정리: `docker compose down`
- 파일 검증: `docker compose config`

---

## 마무리
- Compose는 여러 컨테이너로 구성된 애플리케이션을 **선언적으로** 관리합니다.
- 본 문서의 실습을 통해 레지스트리/UI, Nginx 빌드·최적화, 파라미터화, Apache 추가까지 핵심 흐름을 익힐 수 있습니다.

