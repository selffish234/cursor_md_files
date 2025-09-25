## Docker를 이용한 고가용성 Django 웹 애플리케이션 구축 가이드 (최종 완성판)

이 문서는 Python 3.12 기반 Django 애플리케이션을 Docker Compose로 구성해 HAProxy, Nginx, Apache, Django(Gunicorn), MySQL을 하나의 고가용성 웹 스택으로 구축하는 전체 과정을 초심자 관점에서 친절하게 안내합니다. 각 단계에서 명령어와 설정이 어떤 의미를 가지는지 간결히 설명합니다.

---

## 1) 개요 및 최종 아키텍처

### 1.1 목표
- **목표**: Docker Compose로 다중 서버(로드 밸런서, 2개의 웹 서버, 앱 서버, DB) 환경을 구성하고, 정적/동적 요청을 분리 처리하여 안정성과 확장성을 높입니다.

### 1.2 구성 요소 역할
- **HAProxy(정문/로드 밸런서)**: 외부 요청(80/tcp)을 먼저 수신하고, URL 특성에 따라 Nginx/Apache로 분배합니다.
- **Nginx & Apache(웹 서버)**: 정적 파일을 직접 서빙하고, 동적 요청은 Django로 프록시합니다.
- **Django + Gunicorn(애플리케이션 서버)**: 비즈니스 로직을 처리하고 DB와 통신합니다.
- **MySQL(DB)**: 애플리케이션 데이터 저장소입니다.
- **Docker Compose(통합 오케스트레이션)**: 모든 컨테이너를 정의, 연결, 실행합니다.

### 1.3 요청 흐름
1. 사용자가 `http://서버IP`로 접속합니다.
2. **HAProxy**가 80 포트에서 요청을 수신합니다.
3. **HAProxy**가 URL을 판별합니다.
   - **정적 요청**(`/static/` 시작): `static_backend`로 라우팅 → Nginx 또는 Apache가 정적 파일을 직접 응답
   - **동적 요청**(그 외 경로): `dynamic_backend`로 라우팅 → Nginx/Apache가 요청을 **Django(app)** 로 프록시
4. **Django**가 요청을 처리하고 필요 시 **MySQL**과 통신합니다.
5. 결과가 웹 서버를 통해 사용자에게 응답됩니다.

---

## 2) 사전 준비 및 폴더 구조

### 2.1 최종 폴더 구조

다음은 예시 폴더 구조입니다. 실습에서는 이 구조를 기준으로 파일을 배치합니다.

```bash
/home/vagrant/docker_workspace/docker_lab1/
├── docker-compose.yml
├── Dockerfile
├── apache/
│   └── httpd.conf
├── haproxy/
│   └── haproxy.cfg
├── nginx/
│   └── nginx.conf
│
├── community_board_project/
├── attendance/
├── static/
├── manage.py
└── requirements.txt
```

### 2.2 작업 환경 준비 및 파일 재배치

```bash
cd /home/vagrant/docker_workspace/docker_lab1
```
- **cd(변경)**: 현재 작업 디렉터리를 이동합니다. `docker-compose.yml`이 있는 폴더에서 명령을 실행해야 합니다.

```bash
sudo mv django-community-board/* .
```
- **sudo**: 관리자 권한으로 실행해 권한 문제를 방지합니다.
- **mv**: `django-community-board` 안의 파일을 현재 폴더로 이동해 구조를 단순화합니다.

```bash
sudo rmdir django-community-board
sudo mkdir apache nginx haproxy
```
- **rmdir**: 비어 있는 디렉터리를 삭제합니다.
- **mkdir**: 서버별 설정 파일을 보관할 디렉터리를 생성합니다.

---

## 3) 애플리케이션 설정 파일 수정

### 3.1 `requirements.txt` 수정
- **의도**: `gunicorn`(WSGI 서버)을 추가해 Nginx/Apache와 Django를 연결합니다.

```txt
Django==4.2.7
asgiref==3.7.2
sqlparse==0.4.4
mysqlclient==2.2.4
pkgconfig==1.5.5
gunicorn
```

### 3.2 `community_board_project/settings.py` 수정
- **의도**: DB를 MySQL로 전환하고, 배포 시 정적 파일 수집 위치를 지정합니다.

```python
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': os.environ.get('DB_NAME'),
        'USER': os.environ.get('DB_USER'),
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': os.environ.get('DB_HOST'),
        'PORT': '3306',
    }
}

STATIC_ROOT = BASE_DIR / "staticfiles"
```

- **환경 변수 사용**: 민감한 DB 정보를 코드에 하드코딩하지 않고, 컨테이너 환경 변수에서 읽습니다.
- **`STATIC_ROOT`**: `collectstatic` 결과물을 모으는 최종 경로입니다.

---

## 4) Docker 및 서버 설정 파일 작성

### 4.1 `Dockerfile`
- **의도**: 앱 컨테이너 이미지에 Python, 빌드 도구, 의존성, 소스코드를 포함합니다.

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
```

- **핵심 포인트**
  - **빌드 도구 설치**: `mysqlclient` 컴파일을 위해 시스템 헤더/라이브러리가 필요합니다.
  - **레이어 최적화**: `requirements.txt`를 먼저 복사해 캐시 활용도를 높입니다.

### 4.2 `docker-compose.yml`
- **의도**: 모든 컨테이너의 연결 관계, 볼륨, 포트, 환경 변수를 정의합니다.

```yaml
services:
  haproxy:
    image: haproxy:latest
    container_name: haproxy_container
    restart: always
    ports:
      - "80:80"
      - "8888:8888"
    volumes:
      - ./haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    depends_on:
      - nginx
      - apache

  apache:
    image: httpd:2.4
    container_name: apache_container
    restart: always
    ports:
      - "8001:80"
    volumes:
      - static_volume:/usr/local/apache2/htdocs/
      - ./apache/httpd.conf:/usr/local/apache2/conf/httpd.conf:ro
    depends_on:
      - app

  nginx:
    image: nginx:latest
    container_name: nginx_container
    restart: always
    ports:
      - "8000:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - static_volume:/app/staticfiles
    depends_on:
      - app

  app:
    build: .
    container_name: django_container
    command: gunicorn community_board_project.wsgi:application --bind 0.0.0.0:8000
    volumes:
      - .:/app
      - static_volume:/app/staticfiles
    expose:
      - "8000"
    depends_on:
      - db
    environment:
      - DB_HOST=db
      - DB_NAME=my_project_db
      - DB_USER=my_db_user
      - DB_PASSWORD=my_db_password

  db:
    image: mysql:8.0
    container_name: db_container
    ports:
      - "3306:3306"
    environment:
      MYSQL_DATABASE: 'my_project_db'
      MYSQL_USER: 'my_db_user'
      MYSQL_PASSWORD: 'my_db_password'
      MYSQL_ROOT_PASSWORD: 'my_root_password'
    volumes:
      - ./db-data:/var/lib/mysql

volumes:
  static_volume:
```

- **핵심 포인트**
  - **포트**: 외부 접근이 필요한 서비스만 호스트에 노출합니다.
  - **볼륨**: `static_volume`로 정적 파일을 공유해 Nginx/Apache가 동일 파일을 서빙합니다.
  - **환경 변수**: `settings.py`에서 DB 연결 정보로 사용됩니다.
  - **depends_on**: 컨테이너 시작 순서를 지정합니다.

### 4.3 `apache/httpd.conf`
- **의도**: 정적 파일 서빙과 Django 프록시를 동시에 처리합니다.

```apache
ServerRoot "/usr/local/apache2"
PidFile "/usr/local/apache2/logs/httpd.pid"
Listen 80

LoadModule mpm_event_module modules/mod_mpm_event.so
LoadModule log_config_module modules/mod_log_config.so
LoadModule unixd_module modules/mod_unixd.so
LoadModule authz_core_module modules/mod_authz_core.so
LoadModule rewrite_module modules/mod_rewrite.so
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
LoadModule alias_module modules/mod_alias.so
LoadModule dir_module modules/mod_dir.so

# httpd 기본 이미지의 기본 사용자/그룹은 daemon 입니다.
User daemon
Group daemon

ErrorLog /proc/self/fd/2
LogLevel warn
CustomLog /proc/self/fd/1 common

<VirtualHost *:80>
    ServerName localhost

    <Directory "/usr/local/apache2/htdocs">
        # 기본 index 자동 탐지를 사실상 비활성화
        DirectoryIndex disabled
        Require all granted
    </Directory>

    Alias /static/ /usr/local/apache2/htdocs/

    ProxyPass /static/ !
    ProxyPass / http://app:8000/
    ProxyPassReverse / http://app:8000/
</VirtualHost>
```

- **핵심 포인트**
  - **Alias**: URL `/static/` → 실제 정적 파일 경로 매핑
  - **ProxyPass**: 정적 제외 모든 경로를 Django로 전달

### 4.4 `nginx/nginx.conf`
- **의도**: 정적 파일을 직접 서빙하고, 나머지는 Django로 프록시합니다.

```nginx
server {
    listen 80;

    location /static/ {
        alias /app/staticfiles/;
    }

    location / {
        proxy_pass http://app:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

- **핵심 포인트**
  - **alias**: 컨테이너 내부 정적 파일 경로와 URL 매핑
  - **proxy_set_header**: 원본 클라이언트 정보 전달

### 4.5 `haproxy/haproxy.cfg`
- **의도**: 정적/동적 요청 분리 라우팅과 통계 페이지를 설정합니다.

```haproxy
global
    log stdout format raw local0
defaults
    log     global
    mode    http
    timeout connect 5s
    timeout client  50s
    timeout server  50s

frontend http_in
    bind *:80
    acl url_static path_beg /static/
    use_backend static_backend if url_static
    default_backend dynamic_backend

backend static_backend
    balance roundrobin
    server nginx_static_server nginx:80 check
    server apache_static_server apache:80 check

backend dynamic_backend
    balance roundrobin
    server nginx_dynamic_server nginx:80 check
    server apache_dynamic_server apache:80 check

listen stats
    bind *:8888
    stats enable
    stats uri /stats
    stats refresh 10s
    stats auth admin:your_secure_password
```

- **핵심 포인트**
  - **acl/use_backend**: `/static/` 경로를 정적 백엔드로 분기
  - **roundrobin**: Nginx/Apache로 균등 분산
  - **stats**: 운영 중 상태 확인 페이지 제공

---

## 5) 시스템 실행 및 초기 설정

### 5.1 컨테이너 빌드 및 실행
```bash
sudo docker compose up --build -d
```
- **--build**: 이미지가 없거나 변경되었을 때 새로 빌드합니다.
- **-d**: 백그라운드(detached) 모드로 실행합니다.

### 5.2 DB 마이그레이션
```bash
sudo docker compose run --rm app python manage.py migrate
```
- **run --rm**: 일회성 실행 후 컨테이너를 자동 삭제합니다.
- **migrate**: DB에 필요한 테이블을 생성/갱신합니다.

### 5.3 정적 파일 수집
```bash
sudo docker compose run --rm app python manage.py collectstatic --no-input
```
- **collectstatic**: 각 앱의 정적 파일을 `STATIC_ROOT`로 모읍니다.
- **--no-input**: 비대화식 모드로 진행합니다.

---

## 6) 최종 확인

- **HAProxy 경유**: `http://서버IP` 또는 `http://localhost`
- **Nginx 직접 접속**: `http://서버IP:8000` 또는 `http://localhost:8000`
- **Apache 직접 접속**: `http://서버IP:8001` 또는 `http://localhost:8001`
- **HAProxy 통계 페이지**: `http://서버IP:8888/stats` (인증 필요)

---

## 7) 시스템 관리 명령어

```bash
sudo docker compose ps
```
- **상태 확인**: 실행 중인 서비스 목록 및 상태를 확인합니다.

```bash
sudo docker compose down
```
- **종료**: 컨테이너와 네트워크를 중지/삭제합니다.(데이터 유지)

```bash
sudo docker compose down --volumes
sudo rm -rf db-data
```
- **완전 초기화**: 볼륨까지 삭제하여 DB 데이터를 포함해 모두 초기화합니다.

---

## 부록) 자주 묻는 질문(간단)

- **정적 파일이 404인 경우**: `collectstatic`을 실행했는지, `static_volume`이 Nginx/Apache에 마운트됐는지 확인합니다.
- **DB 연결 오류**: `DB_HOST/DB_NAME/DB_USER/DB_PASSWORD` 환경 변수가 `settings.py`와 일치하는지 확인합니다.
- **포트 충돌**: 호스트에서 80/8000/8001/8888/3306 포트를 이미 사용하는 프로세스가 없는지 확인합니다.

