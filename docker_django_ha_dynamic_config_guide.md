## Docker 동적 설정을 이용한 Django 고가용성 아키텍처 구축 (최종판)

이 문서는 각 서비스를 독립 Docker 이미지로 빌드하고, 컨테이너 시작 시 `entrypoint.sh`와 환경 변수로 설정 파일을 동적으로 생성하여 운영 환경 수준의 유연성/재사용성/확장성을 확보하는 방법을 안내합니다. 초심자도 따라 할 수 있도록, 각 명령과 파일이 “무엇을, 왜” 하는지 간단명료하게 설명합니다.

- 대상 스택: Python 3.12, Django, Gunicorn, Nginx, Apache(httpd), HAProxy, MySQL 8.0, Docker/Compose
- 전제: Django 프로젝트(`community_board_project`)의 `DATABASES`, `STATIC_ROOT` 수정 및 `requirements.txt`의 `gunicorn` 추가가 완료되어 있다고 가정합니다.

---

## 1. 개요 및 최종 아키텍처

### 1.1. 가이드 목표
- 모든 서비스(Nginx/Apache/HAProxy/Django)를 “이미지” 단위로 나누고, 컨테이너 기동 시 환경 변수로 설정을 주입합니다.
- 설정 파일은 고정본이 아니라 “템플릿”을 사용하여, 동일 이미지를 여러 환경에서 재사용합니다.

### 1.2. 아키텍처 특징
- 완전한 컴포넌트화: 서비스마다 `Dockerfile`과 `entrypoint.sh`를 보유
- 완전한 자동화: 컨테이너 시작 시 템플릿 → 실제 설정으로 자동 변환
- 템플릿 기반 설정: `${VAR}` 형태의 변수를 포함한 설정 파일을 이미지에 포함

### 1.3. 요청 흐름(논리)
클라이언트 → HAProxy → (Nginx/Apache) → Django(Gunicorn) → DB(MySQL). 모든 단계의 서버가 기동 시점에 자신의 설정을 동적으로 생성해 사용합니다.

---

## 2. 사전 준비 및 폴더 구조

### 2.1. 최종 폴더 구조
```
/home/vagrant/docker_workspace/docker_lab2/
├── docker-compose.yml
│
├── apache/
│   ├── Dockerfile.apache
│   ├── entrypoint.sh
│   └── httpd.conf
│
├── django/
│   ├── Dockerfile.django
│   ├── entrypoint.sh
│   └── (Django 프로젝트 파일들...)
│
├── haproxy/
│   ├── Dockerfile.haproxy
│   ├── entrypoint.sh
│   └── haproxy.cfg
│
└── nginx/
    ├── Dockerfile.nginx
    ├── entrypoint.sh
    └── nginx.conf
```

- **디렉터리 역할**
  - `django/`: Django 앱과 웹서버(gunicorn) 컨테이너 이미지 정의
  - `nginx/`: Nginx 리버스 프록시 이미지와 설정 템플릿
  - `apache/`: Apache(httpd) 리버스 프록시 이미지와 설정 템플릿
  - `haproxy/`: L4/L7 로드밸런서 이미지와 설정 템플릿
  - `docker-compose.yml`: 전체 서비스를 한 번에 빌드/실행하는 오케스트레이션 정의

### 2.2. 작업 환경 준비 및 파일 재배치
```bash
mkdir -p /home/vagrant/docker_workspace/docker_lab2
cd /home/vagrant/docker_workspace/docker_lab2
sudo mkdir apache django haproxy nginx
# 기존 Django 프로젝트 폴더명이 django-community-board라고 가정
sudo mv django-community-board/* django/
sudo rmdir django-community-board
```
- **mkdir -p**: 중간 경로가 없으면 함께 생성
- **sudo mkdir …**: 권한이 필요한 디렉터리를 루트 권한으로 생성
- **mv … django/**: 기존 Django 소스와 설정을 `django/`로 이동
- **rmdir**: 더 이상 쓰지 않는 빈 디렉터리를 제거

---

## 3. 애플리케이션 및 서버 설정 파일 작성(템플릿 + 엔트리포인트)

> 공통 원리: 설정 파일을 “템플릿(.template 동등)”으로 이미지에 넣고, 컨테이너 시작 시 `envsubst` 또는 스크립트에서 환경 변수를 치환해 “최종 설정”을 만든 뒤 본 프로세스를 실행합니다.

### 3.1. Django (`django/`)

#### Dockerfile.django
```dockerfile
FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
WORKDIR /app
RUN apt-get update && apt-get install -y \
    pkg-config \
    default-libmysqlclient-dev \
    build-essential \
    && rm -rf /var/lib/apt/lists/*
COPY requirements.txt /app/
RUN pip install -r requirements.txt
COPY . /app/
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```
- **FROM**: Python 3.12 슬림 이미지 사용
- **ENV PYTHONDONTWRITEBYTECODE**: `.pyc` 생성 방지로 I/O 감소
- **ENV PYTHONUNBUFFERED**: 로그가 즉시 출력되게 버퍼링 비활성화
- **RUN apt-get …**: `mysqlclient` 컴파일 등에 필요한 빌드 도구 설치
- **COPY requirements.txt → pip install**: 의존성 사전 설치로 빌드 캐시 효율↑
- **ENTRYPOINT**: 컨테이너 시작 시 `entrypoint.sh`를 항상 실행

#### entrypoint.sh
```bash
#!/bin/sh
export DJANGO_SETTINGS_MODULE=community_board_project.settings
echo "Applying database migrations..."
python manage.py migrate
echo "Collecting static files..."
python manage.py collectstatic --no-input
exec gunicorn community_board_project.wsgi:application --bind 0.0.0.0:8000
```
- **DJANGO_SETTINGS_MODULE**: Django가 사용할 설정 모듈 지정
- **migrate**: DB 스키마를 현재 코드 상태로 반영
- **collectstatic**: 정적 파일을 `STATIC_ROOT`로 모아 Nginx/Apache가 서빙 가능
- **exec gunicorn**: PID 1을 Gunicorn으로 교체하여 올바른 신호 처리

> 참고: DB 접속 정보(`DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`)는 컨테이너 환경 변수로 주입되고, Django settings에서 읽어 사용합니다.

---

### 3.2. Nginx (`nginx/`)

#### nginx.conf (템플릿)
```nginx
server {
    listen ${NGINX_PORT};
    location /static/ { alias /app/staticfiles/; }
    location / {
        proxy_pass http://${BACKEND_HOST}:${BACKEND_PORT};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
- **listen ${NGINX_PORT}**: 환경 변수로 포트 선택(예: 80)
- **location /static/**: Django가 수집한 정적 파일을 직접 서빙
- **proxy_pass**: 백엔드인 Django(Gunicorn)로 요청 전달

#### entrypoint.sh
```bash
#!/bin/sh
envsubst '${NGINX_PORT} ${BACKEND_HOST} ${BACKEND_PORT}' < /etc/nginx/conf.d/default.conf.template > /etc/nginx/conf.d/default.conf
exec nginx -g "daemon off;"
```
- **envsubst**: 템플릿의 `${…}`를 실제 환경 변수 값으로 치환
- **daemon off**: 포그라운드 실행으로 컨테이너가 종료되지 않게 유지

#### Dockerfile.nginx
```dockerfile
FROM nginx:latest
RUN apt-get update && apt-get install -y gettext-base && rm -rf /var/lib/apt/lists/*
COPY nginx.conf /etc/nginx/conf.d/default.conf.template
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```
- **gettext-base**: `envsubst`를 제공하는 패키지
- **.template**: 컨테이너 시작 전까지는 변수 미치환 상태 유지

---

### 3.3. Apache (`apache/`)

#### httpd.conf (템플릿)
```apache
ServerRoot "/usr/local/apache2"
PidFile "/usr/local/apache2/logs/httpd.pid"
Listen ${APACHE_PORT}
LoadModule mpm_event_module modules/mod_mpm_event.so
LoadModule log_config_module modules/mod_log_config.so
# ... 필요한 LoadModule 지시어들 ...
LoadModule alias_module modules/mod_alias.so
LoadModule dir_module modules/mod_dir.so
User www-data
Group www-data
ErrorLog /proc/self/fd/2
LogLevel warn
CustomLog /proc/self/fd/1 common
<VirtualHost *:${APACHE_PORT}>
    ServerName localhost
    <Directory "/usr/local/apache2/htdocs">
        DirectoryIndex disabled
        Require all granted
    </Directory>
    Alias /static/ /usr/local/apache2/htdocs/
    ProxyPass /static/ !
    ProxyPass / http://${BACKEND_HOST}:${BACKEND_PORT}/
    ProxyPassReverse / http://${BACKEND_HOST}:${BACKEND_PORT}/
</VirtualHost>
```
- **Listen ${APACHE_PORT}**: 수신 포트를 환경 변수로 제어
- **Alias /static/**: 정적 파일은 로컬에서 바로 서빙하도록 경로 매핑
- **ProxyPass / …**: 동적 요청을 Django로 전달, `ProxyPassReverse`로 응답 헤더의 경로도 되돌림

#### entrypoint.sh
```bash
#!/bin/sh
envsubst '${APACHE_PORT} ${BACKEND_HOST} ${BACKEND_PORT}' < /usr/local/apache2/conf/httpd.conf.template > /usr/local/apache2/conf/httpd.conf
exec httpd-foreground
```
- **httpd-foreground**: Apache를 포그라운드로 실행

#### Dockerfile.apache
```dockerfile
FROM httpd:2.4
RUN apt-get update && apt-get install -y gettext-base && rm -rf /var/lib/apt/lists/*
COPY httpd.conf /usr/local/apache2/conf/httpd.conf.template
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

---

### 3.4. HAProxy (`haproxy/`)
> 원리만 동일하게 이해하면 됩니다. `haproxy.cfg`를 템플릿으로 두고, `entrypoint.sh`에서 환경 변수 치환 후 `haproxy -f`로 실행합니다. 아래는 최소 예시입니다.

#### haproxy.cfg (템플릿 예)
```haproxy
global
    daemon
    maxconn 2048

defaults
    mode http
    timeout connect 5s
    timeout client  30s
    timeout server  30s

frontend fe_http
    bind *:80
    default_backend be_web

backend be_web
    balance roundrobin
    server nginx nginx_container:80 check
    server apache apache_container:80 check
```

#### entrypoint.sh (예)
```bash
#!/bin/sh
# 필요 시 envsubst 사용하도록 구성 가능
exec haproxy -f /usr/local/etc/haproxy/haproxy.cfg -db
```

#### Dockerfile.haproxy (예)
```dockerfile
FROM haproxy:2.9
COPY haproxy.cfg /usr/local/etc/haproxy/haproxy.cfg
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

---

## 4. docker-compose.yml 최종본

```yaml
services:
  haproxy:
    build: { context: ./haproxy, dockerfile: Dockerfile.haproxy }
    container_name: haproxy_container
    restart: always
    ports: [ "80:80", "8888:8888" ]
    depends_on: [ nginx, apache ]

  apache:
    build: { context: ./apache, dockerfile: Dockerfile.apache }
    container_name: apache_container
    restart: always
    ports: [ "8001:80" ]
    volumes: [ static_volume:/usr/local/apache2/htdocs/ ]
    environment:
      - APACHE_PORT=80
      - BACKEND_HOST=app
      - BACKEND_PORT=8000
    depends_on: [ app ]

  nginx:
    build: { context: ./nginx, dockerfile: Dockerfile.nginx }
    container_name: nginx_container
    ports: [ "8000:80" ]
    volumes: [ static_volume:/app/staticfiles ]
    environment:
      - NGINX_PORT=80
      - BACKEND_HOST=app
      - BACKEND_PORT=8000
    depends_on: [ app ]

  app:
    build: { context: ./django, dockerfile: Dockerfile.django }
    container_name: django_container
    restart: always
    volumes:
      - ./django:/app
      - static_volume:/app/staticfiles
    expose: [ "8000" ]
    depends_on: [ db ]
    environment:
      - DB_HOST=db
      - DB_NAME=my_project_db
      - DB_USER=my_db_user
      - DB_PASSWORD=my_db_password

  db:
    image: mysql:8.0
    container_name: db_container
    ports: [ "3306:3306" ]
    environment:
      MYSQL_DATABASE: 'my_project_db'
      MYSQL_USER: 'my_db_user'
      MYSQL_PASSWORD: 'my_db_password'
      MYSQL_ROOT_PASSWORD: 'my_root_password'
    volumes: [ ./db-data:/var/lib/mysql ]

volumes:
  static_volume:
```

- **haproxy**: Nginx/Apache로 트래픽을 분산. `depends_on`으로 두 웹서버 이후 기동
- **apache/nginx**: 각각 리버스 프록시. `environment`로 템플릿 변수를 주입
- **app**: Django + Gunicorn. 정적파일 볼륨을 공유해 프록시가 `/static/`을 서빙 가능
- **db**: MySQL 8.0. 볼륨에 실제 데이터가 저장되어 재기동해도 유지
- **volumes.static_volume**: Django 수집 정적파일을 프록시와 공유

---

## 5. 실행 및 확인

### 5.1. 빌드 및 기동
```bash
sudo docker compose up --build -d
```
- **--build**: 기존 이미지를 재사용하지 않고 새로 빌드
- **-d**: 백그라운드(detached) 실행

### 5.2. build args vs environment 그리고 up --build의 동작
- **build.args**: 이미지 “빌드 시점”에만 쓰는 값. Dockerfile 내부에서만 필요할 때 사용
- **environment**: 컨테이너 “실행 시점”에 주입되는 값. 애플리케이션 동작에 직접 영향

실행 순서 요약(간단)
1) Compose 파일 읽기 → 2) 각 서비스 이미지 빌드(파일 복사, 템플릿 포함, ENTRYPOINT 예약) → 3) 네트워크/볼륨 생성 → 4) `depends_on` 순서로 컨테이너 시작 → 5) ENTRYPOINT 스크립트가 템플릿을 치환하고 본 서버 프로세스 실행

### 5.3. 관리자 계정 생성(선택)
```bash
sudo docker compose run --rm app python manage.py createsuperuser
```
- **run --rm**: 임시 컨테이너로 명령 실행 후 자동 제거
- **createsuperuser**: Django 관리자 계정 생성

### 5.4. 간단 점검
```bash
sudo docker ps
curl -I http://localhost:8000
curl -I http://localhost:8001
curl -I http://localhost/
```
- 8000(Nginx), 8001(Apache), 80(HAProxy) 응답 상태코드를 확인합니다.

---

## 6. 파일별 핵심 역할 요약
- **django/Dockerfile.django**: Django 런타임 이미지 정의(의존성, 작업 디렉터리, 엔트리포인트 등록)
- **django/entrypoint.sh**: 마이그레이션/정적 수집 후 Gunicorn 실행
- **nginx/nginx.conf**: `${…}` 변수를 가진 Nginx 템플릿
- **nginx/entrypoint.sh**: 환경 변수로 템플릿 치환 후 Nginx 포그라운드 실행
- **nginx/Dockerfile.nginx**: `envsubst`를 위한 `gettext-base` 설치, 템플릿 복사
- **apache/httpd.conf**: `${…}` 변수를 가진 Apache 템플릿
- **apache/entrypoint.sh**: 템플릿 치환 후 Apache 포그라운드 실행
- **apache/Dockerfile.apache**: 템플릿 복사 및 엔트리포인트 등록
- **haproxy/haproxy.cfg**:(예시) 백엔드(Nginx/Apache)로 라우팅하는 HAProxy 설정
- **haproxy/entrypoint.sh**: 설정으로 HAProxy 실행
- **haproxy/Dockerfile.haproxy**: 설정/엔트리포인트 포함한 이미지 정의
- **docker-compose.yml**: 전체 컴포넌트를 빌드/연결/기동하는 오케스트레이션

---

## 7. 자주 겪는 이슈 간단 체크
- 컨테이너는 떠 있으나 502/504: 백엔드 호스트/포트 환경 변수 값 확인(`BACKEND_HOST`, `BACKEND_PORT`)
- 정적 파일 404: `collectstatic` 실행 여부와 정적 볼륨 마운트 경로 확인
- DB 연결 실패: `DB_HOST`, 계정/암호, MySQL 기동 상태 확인
- 포트 충돌: 호스트 포트 중복 사용 여부 확인(8000/8001/80 등)

---

이 가이드를 그대로 적용하면, 모든 서버 컨테이너가 자신의 설정을 “동적으로” 생성하여 실행됩니다. 동일 이미지를 다양한 환경 변수 조합으로 재사용할 수 있어 운영 환경에서도 견고하고 확장 가능한 구조를 갖출 수 있습니다.