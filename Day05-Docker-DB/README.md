# Day05 - Docker Login & Database 기초

## 1. 오늘의 학습 목표

Docker 환경에서 로그인 기능을 구현할 때 다음 구성요소들이 어떻게 연결되는지 이해한다.

```text
사용자
  │
  │ ID / Password
  ▼
Frontend
  │
  │ API 요청
  ▼
Backend (FastAPI)
  │
  │ SQL
  ▼
Database (PostgreSQL)
```

Docker Compose 환경에서는 다음과 같은 구조로 구성할 수 있다.

```text
Docker Compose
│
├── backend Service
│     └── FastAPI Container
│          ├── POST /signup
│          ├── POST /login
│          ├── POST /logout
│          └── GET  /profile
│
└── db Service
      └── PostgreSQL Container
           └── users Table
```

---

# 2. API란?

API는 단순히 하나의 기능을 의미하는 것이 아니다.

**다른 프로그램이 특정 프로그램의 기능을 사용할 수 있도록 제공하는 인터페이스**라고 이해할 수 있다.

예를 들어 로그인 서버에는 다음과 같은 기능이 있을 수 있다.

```text
회원가입
로그인
로그아웃
사용자 정보 조회
```

이를 API로 제공하면 다음과 같이 표현할 수 있다.

```text
POST /signup
POST /login
POST /logout
GET  /profile
```

각각의 `/login`, `/signup` 등을 **API Endpoint**라고 한다.

### 정리

| 용어 | 의미 |
|---|---|
| 기능 | 프로그램이 실제로 수행하는 작업 |
| API | 외부에서 기능을 사용할 수 있도록 제공하는 인터페이스 |
| Endpoint | API의 특정 기능에 접근하는 지점 |
| `/login` | 로그인 기능에 접근하기 위한 Endpoint |

---

# 3. API 하나마다 Docker Container를 만드는 것은 아니다

다음과 같은 FastAPI 애플리케이션이 있다고 가정한다.

```text
FastAPI Application
│
├── POST /signup
├── POST /login
├── POST /logout
└── GET /profile
```

이러한 **FastAPI 애플리케이션 전체를 하나의 Docker Container로 실행**할 수 있다.

```text
Docker Container
└── FastAPI
     ├── /signup
     ├── /login
     ├── /logout
     └── /profile
```

따라서 일반적으로

```text
API Endpoint 하나 = Docker Container 하나
```

가 아니다.

오히려 다음과 같이 이해하는 것이 좋다.

```text
하나의 애플리케이션/서버
        ↓
Docker Image
        ↓
Docker Container
        ↓
여러 API Endpoint 제공
```

Docker Compose에서는 이 애플리케이션이 하나의 `service`가 될 수 있다.

```yaml
services:

  backend:
    build: ./backend
    ports:
      - "8000:8000"

  db:
    image: postgres:17
```

구조는 다음과 같다.

```text
Docker Compose
│
├── backend Service
│      ↓
│   FastAPI Container
│      ├── /signup
│      ├── /login
│      └── /profile
│
└── db Service
       ↓
    PostgreSQL Container
```

---

# 4. 로그인 시스템의 기본 구조

로그인 기능을 만든다면 다음과 같은 흐름을 만들 수 있다.

```text
사용자
ID: jaemin
PW: ****
     │
     │ POST /login
     ▼
FastAPI
     │
     │ 사용자 조회
     ▼
PostgreSQL
     │
     │ users Table
     ▼
사용자 정보 반환
     │
     ▼
FastAPI에서 비밀번호 검증
     │
     ├── 일치 → 로그인 성공
     │
     └── 불일치 → 로그인 실패
```

---

# 5. DB 비밀번호와 사용자 비밀번호는 다르다

로그인 시스템에는 서로 다른 목적의 비밀번호가 존재할 수 있다.

## DB 접속 비밀번호

예:

```text
DB User     : jaemin
DB Password : pass0001
```

이 비밀번호는 **FastAPI가 PostgreSQL에 접속하기 위한 비밀번호**이다.

```text
FastAPI
   │
   │ jaemin / pass0001
   ▼
PostgreSQL
```

---

## 서비스 사용자 비밀번호

예:

```text
Username : user01
Password : 1234
```

이것은 **웹 서비스에 로그인하는 사용자의 비밀번호**이다.

```text
사용자
user01 / 1234
      │
      ▼
FastAPI
      │
      ▼
users Table 조회
```

따라서 두 비밀번호를 혼동하면 안 된다.

| 구분 | 예 | 용도 |
|---|---|---|
| DB Password | `pass0001` | Backend → PostgreSQL 접속 |
| User Password | `1234` | 사용자 → 서비스 로그인 |

---

# 6. PostgreSQL Docker Container 실행

먼저 PostgreSQL 서버를 Docker Container로 실행한다.

```bash
docker run -d \
  --name login-db \
  -e POSTGRES_USER=jaemin \
  -e POSTGRES_PASSWORD=pass0001 \
  -e POSTGRES_DB=login_db \
  -p 5432:5432 \
  postgres:17
```

각 옵션의 의미:

| 옵션 | 의미 |
|---|---|
| `-d` | 백그라운드 실행 |
| `--name login-db` | Container 이름 |
| `POSTGRES_USER` | PostgreSQL 사용자 |
| `POSTGRES_PASSWORD` | PostgreSQL 접속 비밀번호 |
| `POSTGRES_DB` | 최초 생성할 Database |
| `-p 5432:5432` | Host 5432 → Container 5432 |
| `postgres:17` | 사용할 Docker Image |

실행 후 구조:

```text
Docker Host
     │
     ▼
login-db Container
     │
     ▼
PostgreSQL Server
     │
     ▼
login_db Database
```

여기서 중요한 점은 **DB 서버를 실행했다고 해서 `users` 테이블까지 자동으로 만들어지는 것은 아니라는 것**이다.

---

# 7. PostgreSQL에 직접 접속하기

실행 중인 PostgreSQL Container에서 `psql` 프로그램을 실행한다.

```bash
docker exec -it login-db psql -U jaemin -d login_db
```

명령을 분해하면 다음과 같다.

```text
docker exec
   │
   │ 실행 중인 Container에서 명령 실행
   ▼
login-db
   │
   │ psql 실행
   ▼
PostgreSQL Client
   │
   ├── -U jaemin
   │       DB 사용자
   │
   └── -d login_db
           접속할 Database
```

정상적으로 접속하면:

```text
login_db=#
```

와 같은 화면을 볼 수 있다.

이제부터는 일반 Linux 명령을 입력하는 것이 아니라 **PostgreSQL에 SQL 명령을 입력하는 상태**이다.

---

# 8. users Table 생성

`login_db=#` 상태에서 다음 SQL을 입력한다.

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) NOT NULL
);
```

각 Column의 역할:

| Column | 설명 |
|---|---|
| `id` | 사용자 고유 ID |
| `username` | 로그인 ID |
| `password_hash` | 암호화/해시 처리된 비밀번호 |
| `role` | 사용자의 권한 |

---

# 9. Table 확인

PostgreSQL에서 Table 목록을 확인한다.

```text
\dt
```

예:

```text
         List of relations

 Schema | Name  | Type
--------+-------+-------
 public | users | table
```

---

# 10. 사용자 데이터 추가

연습용 데이터를 추가해본다.

```sql
INSERT INTO users (
    username,
    password_hash,
    role
)
VALUES (
    'jaemin',
    'test_hash',
    'ADMIN'
);
```

> `test_hash`는 학습용 예제이다.
> 실제 서비스에서는 사용자의 비밀번호를 안전한 password hashing 방식으로 처리해야 한다.

---

# 11. 사용자 조회

```sql
SELECT * FROM users;
```

결과 예:

```text
 id | username | password_hash | role
----+----------+---------------+-------
  1 | jaemin   | test_hash     | ADMIN
```

`id`는 `SERIAL`로 설정했기 때문에 자동으로 증가한다.

---

# 12. 기본 SQL 명령

DB를 다룰 때 자주 사용하는 명령:

```sql
-- 생성
CREATE TABLE ...;

-- 데이터 추가
INSERT INTO ...;

-- 조회
SELECT ...;

-- 수정
UPDATE ...;

-- 삭제
DELETE FROM ...;
```

PostgreSQL `psql` 자체 명령:

```text
\dt
```

Table 목록 확인.

```text
\q
```

PostgreSQL 접속 종료.

---

# 13. 로그인 권한 관리

로그인한 사용자마다 다른 권한을 부여할 수도 있다.

예를 들어 라벨링 시스템이라면:

```text
ADMIN
├── 사용자 관리
├── 작업 생성
├── 작업 할당
└── 전체 작업 조회

REVIEWER
├── 작업 검수
├── 승인
└── 반려

WORKER
├── 본인 작업 조회
└── 라벨링 작업
```

초기에는 `users` Table에 `role` Column을 두는 간단한 방법을 사용할 수 있다.

```text
users

id | username   | password_hash | role
---|------------|---------------|---------
1  | jaemin     | ...           | ADMIN
2  | reviewer01 | ...           | REVIEWER
3  | worker01   | ...           | WORKER
```

Backend에서는 로그인한 사용자의 `role`을 확인하여 API 접근을 제한할 수 있다.

```text
사용자 로그인
      ↓
FastAPI
      ↓
users Table 조회
      ↓
role 확인
      ↓
ADMIN / REVIEWER / WORKER
      ↓
API 접근 허용/거부
```

이러한 방식을 **Role-Based Access Control(RBAC)**의 기본 형태로 볼 수 있다.

---

# 14. Docker 명령과 SQL 명령 구분하기

이번 학습에서 가장 중요한 부분이다.

## Docker가 담당

```bash
docker run
docker exec
docker stop
docker start
docker rm
```

Docker는:

```text
Container 생성
Container 실행
Container 중지
Container 내부 프로그램 실행
```

등을 담당한다.

---

## PostgreSQL이 담당

```sql
CREATE TABLE
INSERT
SELECT
UPDATE
DELETE
```

PostgreSQL은:

```text
Database 관리
Table 생성
데이터 저장
데이터 조회
데이터 수정
```

등을 담당한다.

즉:

```text
Docker
   │
   │ PostgreSQL Container 실행
   ▼
PostgreSQL Server
   │
   │ Database 접속
   ▼
login_db
   │
   │ SQL
   ▼
users Table
```

---

# 15. 전체 실행 순서

이번 학습의 핵심 흐름:

```text
① PostgreSQL Container 실행

docker run ...
        │
        ▼
PostgreSQL Server
        │
        ▼
login_db 생성


② DB 접속

docker exec -it login-db psql ...
        │
        ▼
login_db=#


③ Table 생성

CREATE TABLE users ...
        │
        ▼
users Table


④ 사용자 데이터

INSERT INTO users ...
        │
        ▼
사용자 정보 저장


⑤ Backend 연결

FastAPI
   │
   │ db:5432
   ▼
PostgreSQL
   │
   ▼
users Table


⑥ 로그인

사용자
   │
   │ POST /login
   ▼
FastAPI
   │
   │ 사용자 조회
   ▼
users Table
   │
   ▼
Password 검증
   │
   ├── 성공
   └── 실패
```

---

# 16. 실제 프로젝트에서는?

학습 단계에서는 직접:

```bash
docker exec -it login-db psql ...
```

로 들어가서:

```sql
CREATE TABLE ...
```

을 입력해보는 것이 DB 동작을 이해하기 좋다.

하지만 실제 프로젝트에서는 매번 사람이 직접 Table을 생성하지 않는다.

예를 들어:

```text
project/
├── compose.yml
├── backend/
│   ├── Dockerfile
│   └── ...
└── db/
    └── init.sql
```

처럼 `init.sql`을 두어 최초 DB 초기화를 자동화할 수 있다.

프로젝트가 커지면 다음과 같은 Database Migration 도구도 사용할 수 있다.

```text
FastAPI
   ↓
SQLAlchemy
   ↓
Alembic
   ↓
PostgreSQL
```

이를 통해 DB Schema 변경 이력을 코드로 관리할 수 있다.

---

# 17. 비밀번호 보안

실제 서비스에서는 다음과 같이 비밀번호 원문을 DB에 저장하면 안 된다.

```text
username | password
---------|---------
jaemin   | 1234    ❌
```

대신 password hashing을 적용한다.

```text
사용자 비밀번호
      │
      ▼
Password Hashing
      │
      ▼
users Table

username | password_hash
---------|-------------------------
jaemin   | $2b$12$........
```

로그인할 때도 원문 비밀번호를 DB에서 꺼내 비교하는 것이 아니라 입력된 비밀번호와 저장된 hash를 안전한 검증 함수로 비교한다.

---

# 핵심 복습

```text
API
= 프로그램의 기능을 외부에서 사용할 수 있도록 제공하는 인터페이스

API Endpoint
= /login, /signup처럼 특정 기능에 접근하는 지점

Docker Container
= API 하나를 담는 것이 아니라
  FastAPI 같은 애플리케이션 전체를 실행할 수 있음

DB Password
= Backend가 DB에 접속하기 위한 비밀번호

User Password
= 사용자가 서비스에 로그인하기 위한 비밀번호

docker run
= PostgreSQL Server를 Container로 실행

docker exec ... psql
= 실행 중인 Container에서 psql을 실행하여 DB에 접속

CREATE TABLE
= Docker 명령이 아니라 PostgreSQL SQL 명령

role
= ADMIN / REVIEWER / WORKER 등의 사용자 권한
```

## 오늘의 한 줄 정리

> Docker로 PostgreSQL 서버를 실행하고, `docker exec`를 통해 `psql`로 DB에 접속한 뒤 SQL 명령으로 Table과 데이터를 관리한다. FastAPI는 이 DB를 조회하여 로그인 및 권한 기능을 제공할 수 있다.