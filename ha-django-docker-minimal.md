## Docker 기반 고가용성 Django 배포 가이드 (Minimal Files, 최종판)

### 1) 개요와 목표
- **목표**: Python 3.12 + Django 앱을 Docker Compose만으로 고가용성/로드밸런싱 구조로 배포. 별도 `haproxy.cfg`, `nginx.conf`, `httpd.conf` 파일을 만들지 않고, 컨테이너 시작 시 환경변수와 명령으로 설정을 동적으로 생성합니다.
- **최종 스택**: HAProxy(정문 로드 밸런서) → Nginx & Apache(정적/동적 프록시) → Django+Gunicorn(WSGI 앱 서버) → MySQL(DB)
- **핵심 원칙**: 
  - conf 파일 바인드 마운트 없이 `docker-compose.yml`만으로 설정
  - 컨테이너 시작 시 `sh -c` 스크립트로 각 서버 설정 파일 생성 후 데몬 실행
  - 환경변수만으로 비밀/파라미터 주입

### 2) 최종 요청 흐름
- **HAProxy(80/tcp)**가 모든 요청을 수신 후 URL 패턴으로 라우팅
  - **/static/**: `static_backend`로 전달 → Nginx 또는 Apache가 정적 파일 응답
  - **그 외**: `dynamic_backend`로 전달 → Nginx 또는 Apache가 Django로 프록시 → Gunicorn 처리 → 응답 반환

### 3) 사전 준비
- **작업 디렉터리**: 예) `/home/vagrant/docker_workspace/docker_lab1` 또는 원하는 프로젝트 루트
- **Django 설정 전제**:
  - DB 접속 정보는 환경변수로 읽도록 구성 (예: `os.environ.get('DB_HOST')` 등)
  - 배포용 정적 수집 경로 지정: `STATIC_ROOT = BASE_DIR / "staticfiles"`
  - (권장) `ALLOWED_HOSTS`에 배포 호스트를 포함하거나 와일드카드 허용

필요 시 참고 예시:
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

STATIC_ROOT = BASE_DIR / 'staticfiles'
```

### 4) 단일 Compose 파일만으로 구성하기
- 아래 파일 하나만 생성합니다: `docker-compose.yml`
- conf 파일 바인드 마운트 없이, 각 컨테이너가 부팅 시 설정 파일을 생성합니다.

```yaml
services:
  haproxy:
    image: haproxy:2.9
    restart: always
    ports:
      - "80:80"
      - "8888:8888"
    environment:
      HAPROXY_STATS_USER: admin
      HAPROXY_STATS_PASS: your_secure_password
    depends_on:
      - nginx
      - apache
    command: >
      sh -c 'cat > /usr/local/etc/haproxy/haproxy.cfg <<EOF
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
        server nginx nginx:80 check
        server apache apache:80 check

      backend dynamic_backend
        balance roundrobin
        server nginx nginx:80 check
        server apache apache:80 check

      listen stats
        bind *:8888
        stats enable
        stats uri /stats
        stats refresh 10s
        stats auth ${HAPROXY_STATS_USER}:${HAPROXY_STATS_PASS}
      EOF
      && exec haproxy -W -db -f /usr/local/etc/haproxy/haproxy.cfg'

  nginx:
    image: nginx:1.25-alpine
    container_name: nginx
    restart: always
    ports:
      - "8000:80"
    volumes:
      - static_volume:/app/staticfiles:ro
    depends_on:
      - app
    command: >
      sh -c 'cat > /etc/nginx/conf.d/default.conf <<'\''EOF'\''
      server {
        listen 80;

        location /static/ {
          alias /app/staticfiles/;
          access_log off;
          expires 1h;
        }

        location / {
          proxy_pass http://app:8000;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
        }
      }
      EOF
      && exec nginx -g "daemon off;"'

  apache:
    image: httpd:2.4
    container_name: apache
    restart: always
    ports:
      - "8001:80"
    volumes:
      - static_volume:/usr/local/apache2/htdocs:ro
    depends_on:
      - app
    command: >
      sh -c 'set -eu;
      CONF=/usr/local/apache2/conf/httpd.conf;
      # 필요한 모듈이 비활성화된 경우에만 추가 로드
      grep -qF "mod_proxy.so" "$CONF" || echo "LoadModule proxy_module modules/mod_proxy.so" >> "$CONF";
      grep -qF "mod_proxy_http.so" "$CONF" || echo "LoadModule proxy_http_module modules/mod_proxy_http.so" >> "$CONF";
      grep -qF "mod_alias.so" "$CONF" || echo "LoadModule alias_module modules/mod_alias.so" >> "$CONF";
      grep -qF "mod_rewrite.so" "$CONF" || echo "LoadModule rewrite_module modules/mod_rewrite.so" >> "$CONF";
      grep -qF "ServerName" "$CONF" || echo "ServerName localhost" >> "$CONF";
      cat >> "$CONF" <<\'EOF\'
      <VirtualHost *:80>
        Alias /static/ "/usr/local/apache2/htdocs/"
        ProxyPass /static/ !
        ProxyPass / http://app:8000/
        ProxyPassReverse / http://app:8000/
        <Directory "/usr/local/apache2/htdocs">
          Require all granted
          DirectoryIndex disabled
        </Directory>
      </VirtualHost>
      EOF
      exec httpd-foreground'

  app:
    image: python:3.12-slim
    container_name: app
    restart: always
    working_dir: /app
    environment:
      PYTHONDONTWRITEBYTECODE: "1"
      PYTHONUNBUFFERED: "1"
      # Django/DB 설정 (settings.py에서 os.environ.get으로 읽음)
      DB_HOST: db
      DB_NAME: my_project_db
      DB_USER: my_db_user
      DB_PASSWORD: my_db_password
      DJANGO_DEBUG: "0"
      DJANGO_SECRET_KEY: replace_me_with_a_secure_value
      # 런타임 패키지 목록(파일 없이 설치)
      PIP_REQUIREMENTS: |
        Django==4.2.7
        asgiref==3.7.2
        sqlparse==0.4.4
        mysqlclient==2.2.4
        pkgconfig==1.5.5
        gunicorn
      # Gunicorn 튜닝(옵션)
      GUNICORN_WORKERS: "3"
      GUNICORN_THREADS: "2"
      GUNICORN_TIMEOUT: "60"
    volumes:
      - ./:/app
      - static_volume:/app/staticfiles
    expose:
      - "8000"
    depends_on:
      - db
    command: >
      sh -c 'set -eux;
      apt-get update;
      DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        default-libmysqlclient-dev build-essential pkg-config;
      rm -rf /var/lib/apt/lists/*;
      python -m pip install --no-cache-dir --upgrade pip;
      printf "%s" "$PIP_REQUIREMENTS" > /tmp/requirements.txt;
      pip install --no-cache-dir -r /tmp/requirements.txt;
      python manage.py migrate --noinput;
      python manage.py collectstatic --no-input;
      exec gunicorn community_board_project.wsgi:application \
        --bind 0.0.0.0:8000 \
        --workers ${GUNICORN_WORKERS:-3} \
        --threads ${GUNICORN_THREADS:-2} \
        --timeout ${GUNICORN_TIMEOUT:-60}'

  db:
    image: mysql:8.0
    container_name: db
    restart: always
    ports:
      - "3306:3306"
    environment:
      MYSQL_DATABASE: my_project_db
      MYSQL_USER: my_db_user
      MYSQL_PASSWORD: my_db_password
      MYSQL_ROOT_PASSWORD: my_root_password
    volumes:
      - db_data:/var/lib/mysql

volumes:
  static_volume:
  db_data:
```

설명 요약:
- **HAProxy/Nginx/Apache**: 컨테이너 시작 시 `cat <<EOF`로 설정 파일을 생성하고 즉시 포그라운드 실행
- **Django 앱**: `python:3.12-slim`을 바로 사용해 런타임에 시스템 패키지 및 파이썬 패키지 설치, 마이그레이션/정적 수집까지 자동 실행 후 Gunicorn으로 서비스 시작
- **MySQL**: 데이터는 `db_data` 볼륨에 영구 저장
- **정적 파일**: `static_volume` 볼륨을 Nginx/Apache, Django 간 공유 (읽기전용으로 서빙)

### 5) 실행 절차 (간단 버전)
1. 작업 디렉터리로 이동 후 백그라운드 실행 및 최초 셋업 자동화
```bash
docker compose up -d
```
  - 앱 컨테이너가 자동으로 패키지 설치 → migrate → collectstatic → gunicorn 순서로 실행합니다.

2. 상태 확인 및 로그 확인
```bash
docker compose ps
docker compose logs -f | cat
```

3. 접속 확인
- **HAProxy**: `http://<서버IP>` 또는 `http://localhost`
- **Nginx 직접**: `http://<서버IP>:8000`
- **Apache 직접**: `http://<서버IP>:8001`
- **HAProxy 통계**: `http://<서버IP>:8888/stats` (기본 `admin/your_secure_password`)

### 6) 운영 명령어
- **중지 및 제거(데이터 유지)**: 
```bash
docker compose down
```

- **완전 초기화(데이터 포함 삭제)**:
```bash
docker compose down --volumes
```

### 7) 커스터마이징 팁 (파일 없이 환경변수만으로)
- **HAProxy 인증 변경**: `HAPROXY_STATS_USER`, `HAPROXY_STATS_PASS`
- **Django/DB 정보**: `DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`, `DJANGO_DEBUG`, `DJANGO_SECRET_KEY`
- **Gunicorn 스케일**: `GUNICORN_WORKERS`, `GUNICORN_THREADS`, `GUNICORN_TIMEOUT`
- **정적 파일 경로**: Django `settings.py`의 `STATIC_ROOT`만 맞으면 됩니다. (컨테이너는 `/app/staticfiles` 기준으로 공유)

### 8) 자주 묻는 질문(FAQ)
- **Q. conf 파일이 전혀 없는데도 안전한가요?**
  - **A.** 운영 중 재시작에도 동일 설정이 재생성됩니다. 버전관리 측면에서 conf를 파일로 둘 수도 있지만, 본 가이드는 "파일 최소화"에 초점을 둔 방식입니다.

- **Q. 최초 부팅이 오래 걸립니다.**
  - **A.** 앱 컨테이너가 패키지를 런타임에 설치하기 때문입니다. 빌드 시간을 런타임으로 옮긴 대가입니다. 재시작 시에는 레이어 캐시가 없어도 Python 패키지 캐시는 컨테이너마다 재설치가 필요하니, 성능이 중요하면 별도의 Dockerfile 빌드 방식을 권장합니다.

- **Q. Nginx/Apache 중 하나만 쓰고 싶어요.**
  - **A.** HAProxy `backend`에서 원치 않는 서버 한 쪽을 주석 처리하거나 서비스 자체를 제거하면 됩니다.

- **Q. HTTPS(SSL/TLS)도 가능한가요?**
  - **A.** 가능합니다. 본 예시에서는 단순화를 위해 80 포트만 다뤘습니다. 실제 운영에서는 HAProxy 혹은 Nginx에 인증서를 주입하는 방식으로 확장하세요.

---

이 가이드는 conf 파일 생성 없이, `docker-compose.yml` 하나로 고가용성 Django 스택을 재현하는 데 초점을 뒀습니다. 필요 시 보안/성능/옵저버빌리티(예: 로그/메트릭) 항목을 추가해 운영 환경에 맞게 확장하십시오.

