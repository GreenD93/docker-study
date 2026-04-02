# Docker 스터디 정리

## 학습 흐름

```
ex03 (Django 단일 컨테이너)
  ↓
ex04 (Nginx 단일 컨테이너)
  ↓
ex05 (Django + Nginx 멀티 컨테이너)
  ↓
ex06 (PostgreSQL + Volume)
  ↓
ex07 (Docker Compose 풀스택)
```

---

## 사전 준비: Python 환경 세팅

```bash
brew install pyenv
brew install pyenv-virtualenv

pyenv install 3.11.6
pyenv virtualenv 3.11.6 py3_11_6   # 가상환경 생성
pyenv activate py3_11_6

pip install -r requirements.txt
django-admin startproject myapp
```

- Django 기본 프로젝트는 별도 DB 설치 없이 SQLite를 바로 사용
- `python manage.py runserver` — 개발 서버 실행
- `python manage.py dbshell` — SQLite 콘솔 접속 (auth_user, django_admin_log, sessions 테이블 확인)

---

## ex03: Django 단일 컨테이너
1.MVT

Model : 
```python
# models.py
from django.db import models

class Post(models.Model):
    title = models.CharField(max_length=100)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
```
View : 
```python
# views.py
from django.shortcuts import render
from .models import Post

def post_list(request):
    posts = Post.objects.all()  # DB 조회
    return render(request, 'post_list.html', {'posts': posts})
```

<!-- templates/post_list.html -->
Template :
```python
<!-- templates/post_list.html -->
<h1>게시글 목록</h1>
<ul>
  {% for post in posts %}
    <li>{{ post.title }}</li>
  {% endfor %}
</ul>
```

2.gunicorn -> 우리는 MSA로 worker수 설정(프로세스)

Flask/Django의 기본 서버(app.run(), manage.py runserver)는:
개발용 (debug, 단일 프로세스)
성능/안정성 부족
동시 요청 처리에 취약
그래서 운영 환경에서는 Gunicorn 같은 Production-grade 서버를 사용합니다.

[Client] → [NGINX] → [Gunicorn] → [Flask/Django App]

2-1. Gunicorn의 핵심 역할
① WSGI 서버 역할
Python 웹 앱은 WSGI 인터페이스로 동작
Gunicorn이 HTTP 요청을 받아 → WSGI로 변환 → 앱에 전달
[Client] → [NGINX] → [Gunicorn] → [Flask/Django App]
② 멀티 프로세스 처리
worker 프로세스를 여러 개 띄워서
동시에 많은 요청 처리 가능

예:
gunicorn -w 4 app:app

2-2. uvicorn <> gunicorn

WSGI (Web Server Gateway Interface)
→ 동기(synchronous) Python 웹 표준 : Django, Flask
ASGI (Asynchronous Server Gateway Interface)
→ 비동기(async)까지 지원하는 차세대 웹 표준  : FastAPI


**배운 것:** Dockerfile 기본 구조, 이미지 빌드, 컨테이너 실행

```dockerfile
FROM python:3.11.6
WORKDIR /usr/src/app
COPY . .
RUN pip install -r requirements.txt    # 이미지 빌드 시 실행
CMD python manage.py runserver 0.0.0.0:8000  # 컨테이너 실행 시 실행
EXPOSE 8000
```

**핵심 명령어:**
```bash
docker image build -t 이미지명 .
docker image ls
docker run -d -p 8000:8000 이미지명    # -p 외부포트:내부포트
```

**RUN vs CMD:**
| 구분 | RUN | CMD |
|------|-----|-----|
| 실행 시점 | 이미지 빌드 시 | 컨테이너 시작 시 |
| 용도 | 패키지 설치, 환경 구성 | 앱 실행 명령 |

---

## ex04: Nginx 단일 컨테이너

**배운 것:** 서비스 이미지 활용, foreground 프로세스의 중요성

```dockerfile
FROM nginx:1.25.3
CMD ["nginx", "-g", "daemon off;"]
```

**핵심 개념 — Docker 컨테이너는 foreground 프로세스가 필수:**
- Nginx 기본 동작: 메인 프로세스 시작 → 워커 생성 → 메인 종료(백그라운드화)
- Docker에서는 메인 프로세스가 종료되면 컨테이너도 종료됨
- `daemon off;` — Nginx를 foreground에서 실행하여 컨테이너가 살아있게 유지

---

## ex05: Django + Nginx 멀티 컨테이너

**배운 것:** 커스텀 네트워크, 서비스 디스커버리, 리버스 프록시

### Docker 네트워크

**기본 bridge 네트워크:**
```
Docker Host (Mac)
│
├── docker0 (bridge 네트워크)
│     └── 172.17.0.1  ← 게이트웨이
│
├── 컨테이너 A
│     └── 172.17.0.2
│
├── 컨테이너 B
│     └── 172.17.0.3
```

```bash
docker network inspect bridge | grep IPv4Address   # 컨테이너 내부 IP 확인
```

**서브넷 이해:**
- `172.17.0.2/16` → 범위: `172.17.0.0 ~ 172.17.255.255`
- `172.17.0.2/16`과 `192.168.0.5/16`은 다른 네트워크 → 직접 통신 불가, 라우터(게이트웨이) 필요

**커스텀 네트워크 생성:**
```bash
docker network create mynetwork02
docker run --network mynetwork02 --name djangotest 이미지명
docker run --network mynetwork02 --name nginxtest 이미지명
```
- 같은 커스텀 네트워크 내에서는 **컨테이너 이름으로 통신 가능** (서비스 디스커버리)

### Nginx 리버스 프록시 설정

```nginx
upstream myweb {
    server djangotest:8000;   # 컨테이너 이름으로 참조
}

server {
    listen 80;
    location / {
        proxy_pass http://myweb;
    }
}
```

### Gunicorn — 프로덕션 WSGI 서버
- Django의 `runserver`는 개발용 → 프로덕션에서는 Gunicorn 사용
- `gunicorn --bind 0.0.0.0:8000 myapp.wsgi:application`

---

## ex06: PostgreSQL + Volume

**배운 것:** 볼륨으로 데이터 영속화, 환경변수로 설정 주입

```bash
docker run -e POSTGRES_PASSWORD=mysecretpassword \
           --mount type=volume,source=볼륨명,target=/var/lib/postgresql/data \
           이미지명
```

**핵심 개념:**
- 컨테이너는 일시적(ephemeral) → 삭제되면 데이터도 사라짐
- **Volume**은 컨테이너 생명주기와 독립 → 데이터가 영속됨
- `-e KEY=VALUE` — 환경변수로 DB 비밀번호 등 설정 주입

**유용한 명령어:**
```bash
docker container exec -it 컨테이너ID /bin/bash   # 컨테이너 내부 진입
```

---

## ex07: Docker Compose 풀스택

**배운 것:** 선언적 멀티 컨테이너 오케스트레이션

### 아키텍처

```
Client
  ↓ :80
┌─ Nginx ─────────────────┐
│  reverse proxy           │
│  → proxy_pass :8000      │
└──────────┬──────────────┘
           ↓
┌─ Django (Gunicorn) ─────┐
│  WSGI application        │
│  → DB 연결: postgrestest │
└──────────┬──────────────┘
           ↓
┌─ PostgreSQL ────────────┐
│  Volume: mycomposevol01  │
│  데이터 영속화            │
└─────────────────────────┘

모두 composenet01 네트워크로 연결
```

### docker-compose.yml 핵심 구조

```yaml
version: "3"

services:
  djangotest:
    build: ./myDjango03          # Dockerfile 위치
    networks: [composenet01]
    depends_on: [postgrestest]   # 의존성 순서
    restart: always

  nginxtest:
    build: ./myNginx03
    ports: ["80:80"]             # 외부 노출은 Nginx만
    depends_on: [djangotest]
    restart: always

  postgrestest:
    build: ./myPostgres03
    environment:                 # 환경변수
      - POSTGRES_PASSWORD=mysecretpassword
    volumes:                     # 데이터 영속화
      - mycomposevol01:/var/lib/postgresql/data
    restart: always

networks:
  composenet01:                  # 커스텀 네트워크 자동 생성

volumes:
  mycomposevol01:                # Named volume 자동 생성
```

**ex05(수동) vs ex07(Compose) 비교:**

| 구분 | ex05 (수동) | ex07 (Compose) |
|------|-------------|----------------|
| 네트워크 | `docker network create` 직접 실행 | `networks:` 선언만 하면 자동 생성 |
| 볼륨 | `--mount` 플래그 직접 지정 | `volumes:` 선언만 하면 자동 생성 |
| 실행 순서 | 수동으로 순서 맞춰 실행 | `depends_on`으로 선언 |
| 시작/종료 | 컨테이너마다 개별 실행 | `docker compose up -d --build` 한 번 |
| 서비스 디스커버리 | 컨테이너 이름으로 | 서비스 이름으로 (동일 방식) |

**실행 명령어:**
```bash
docker compose up -d --build   # 빌드 + 백그라운드 실행
docker compose down             # 전체 종료
```

---

## 주요 개념 요약

| 개념 | 설명 |
|------|------|
| `FROM` | 베이스 이미지 지정 |
| `WORKDIR` | 작업 디렉터리 설정 |
| `COPY` | 호스트 → 이미지로 파일 복사 |
| `RUN` | 빌드 시 명령 실행 (패키지 설치 등) |
| `CMD` | 컨테이너 시작 시 실행할 명령 |
| `EXPOSE` | 컨테이너가 사용하는 포트 선언 |
| `-p 외부:내부` | 호스트 포트 ↔ 컨테이너 포트 매핑 |
| `-e KEY=VALUE` | 환경변수 주입 |
| `--network` | 컨테이너를 특정 네트워크에 연결 |
| `--mount type=volume` | 볼륨 마운트로 데이터 영속화 |
| `depends_on` | 서비스 시작 순서 제어 |
| `restart: always` | 컨테이너 비정상 종료 시 자동 재시작 |
| `daemon off;` | Nginx를 foreground로 실행 (Docker 필수) |
| Gunicorn | Django 프로덕션 WSGI 서버 |
