# 2일차 — Docker Compose, Network, Volume

## 1. Docker Compose란?

Docker Compose는 여러 컨테이너를 하나의 프로젝트 단위로 정의하고 관리하는 도구다. `compose.yaml`에 서비스와 네트워크, 볼륨, 포트, 환경 변수 등의 설정을 적는다.

```text
Compose 프로젝트
├── Service A: Image A → Container A
├── Service B: Image B → Container B
└── 실행 설정: Network / Volume / Port / Environment
```

서비스는 컨테이너를 어떻게 실행할지 정의한 단위다. 위 그림은 서비스당 컨테이너 하나인 예시이며, 설정에 따라 여러 개로 늘릴 수도 있다.

## 2. 실행과 detached 모드

```bash
docker compose up
docker compose up -d
```

- `up`: 설정을 읽고 필요한 컨테이너 등을 생성·시작한다. 기존 컨테이너가 있으면 설정에 따라 재사용하거나 재생성한다.
- `-d`: **detached**. 백그라운드로 실행하고 터미널을 돌려준다.
- `up`만 쓰면 여러 서비스의 로그가 현재 터미널에 표시된다.
- `-d`여도 컨테이너의 주 프로세스가 종료되면 컨테이너는 정지한다. 재시작 여부는 별도의 재시작 정책에 달려 있다.
- 한 명령으로 시작한다고 해서 모든 서비스의 준비 완료를 동시에 보장하지는 않는다.

```bash
docker compose ps
docker ps
docker compose logs -f
docker compose stop
docker compose down
```

`compose ps`는 해당 프로젝트, `docker ps`는 Docker 환경 전체의 실행 중인 컨테이너를 확인한다. `logs -f`는 로그를 계속 따라간다. 로그 조회 중 Ctrl+C는 조회만 끝낸다. 반면 foreground의 `compose up`을 Ctrl+C로 중단하면 컨테이너들도 정지한다.

`stop`은 정지만 하고, `down`은 프로젝트 컨테이너와 관리 대상 네트워크 등을 제거한다. 명명된 볼륨은 기본적으로 남는다.

## 3. 특정 Compose 파일과 프로젝트 지정

```text
MLOps/
├── compose.yml
├── compose.train.yml
├── compose.clearml.yml
└── compose.monitoring.yml
```

```bash
docker compose up -d
docker compose -f compose.clearml.yml up -d
docker compose -p mlops -f compose.clearml.yml up -d
```

| 옵션 | 의미 |
| --- | --- |
| `-f` / `--file` | 사용할 Compose 파일 지정 |
| `-p` / `--project-name` | 프로젝트 이름 지정. 위 예제는 `mlops` |
| `-d` / `--detach` | 백그라운드 실행 |

기본 파일은 `compose.yaml`을 권장하며 `compose.yml`, 기존 `docker-compose.yaml/yml` 이름도 지원한다. 별도 파일·프로젝트를 지정했다면 조회와 정리에도 같은 옵션을 사용한다.

```bash
docker compose -p mlops -f compose.clearml.yml ps
docker compose -p mlops -f compose.clearml.yml logs -f
docker compose -p mlops -f compose.clearml.yml down
```

**명령마다 옵션 뜻이 다르다.** `docker run -p`는 포트 공개, `docker compose -p`는 프로젝트 이름이다. `compose -f`는 파일 지정, `compose logs -f`는 로그 따라가기다.

## 4. Docker Network와 여러 컨테이너

다음 IP는 설명용 예시이며, 일반적인 Linux bridge 네트워크를 기준으로 한다.

```text
Docker Host: 172.16.4.82
└── Compose Network
    ├── backend: 172.18.0.2:8000
    ├── db:      172.18.0.3:5432
    └── redis:   172.18.0.4:6379
```

컨테이너들은 네트워크에서 서로 다른 IP를 할당받는다. 네트워크 공간이 분리돼 있어 여러 컨테이너가 각각 내부 80번 포트를 써도 된다. 같은 호스트 IP·프로토콜의 호스트 포트를 중복 공개하면 충돌한다.

별도로 설정하지 않으면 Compose는 프로젝트의 기본 네트워크를 만들고 서비스를 연결한다. `network_mode: host` 등 다른 모드에는 같은 설명을 그대로 적용할 수 없다.

## 5. 컨테이너끼리는 서비스 이름으로 통신

같은 Compose 네트워크에 있는 backend가 DB에 연결할 때는 보통 IP 대신 **`db:5432`**를 사용한다.

```text
backend → db:5432 → Docker의 이름 해석 → DB 컨테이너
```

컨테이너를 재생성하면 IP는 바뀔 수 있지만 서비스 이름은 유지할 수 있다. 애플리케이션은 끊긴 연결을 다시 맺을 수 있어야 한다.

- DB 주소: `db:5432`
- Redis 주소: `redis:6379`
- backend 컨테이너 안의 `localhost`: backend 자신
- 네트워크가 서로 분리돼 있으면 이름만 안다고 통신할 수 있는 것은 아니다.
- 같은 네트워크의 컨테이너끼리는 호스트에 `ports`를 공개하지 않아도 컨테이너 포트로 통신할 수 있다.

## 6. 내부 통신과 외부 접근

```text
내부: backend → db:5432

외부 PC → 호스트IP:8080 → 포트 매핑 → 웹 컨테이너:80
```

```bash
docker run -d --name day02-web -p 8080:80 nginx:alpine
```

이는 **호스트 8080번으로 들어오는 요청을 컨테이너 80번으로 전달**한다. 실제 프로그램이 해당 포트에서 요청을 받아야 한다. 외부 PC의 접근에는 호스트까지의 네트워크 연결과 방화벽 허용도 필요하다.

## 7. 특정 Host IP에 포트 바인딩

```bash
docker run -d --name day02-bound-web -p 172.16.4.82:8081:80 nginx:alpine
```

```text
-p 호스트IP:호스트포트:컨테이너포트
```

위 예제는 호스트에 실제로 `172.16.4.82` 주소가 있을 때 사용하는 설정이다. **앞의 IP는 접근을 허용할 사용자 PC의 IP가 아니다.** 호스트의 어느 주소에서 요청을 받을지 지정한다.

| 설정 | 질문 |
| --- | --- |
| 포트 바인딩 | 서비스를 호스트의 어디에 공개할까? |
| 방화벽 접근 정책 | 어떤 출발지의 접근을 허용할까? |

관리자 PC만 허용하려면 Docker의 네트워크 동작을 고려한 방화벽 정책이 필요하다. 로컬에서만 사용할 실습은 `-p 127.0.0.1:8081:80`처럼 지정할 수 있다. 호스트 IP를 생략한 포트 공개는 기본적으로 모든 인터페이스에 바인딩한다.

## 8. --link는 레거시 방식

```text
과거의 문법 예시: docker run --link db:mydb ...
현재의 일반적인 방식: 사용자 정의 네트워크 + Docker DNS + 서비스 이름
```

`--link`는 상대 컨테이너를 이름이나 별칭으로 참조하기 위해 사용하던 레거시 기능이다. 새 구성에서는 사용자 정의 네트워크를 사용한다. Compose에서는 같은 네트워크에 있는 서비스끼리 이름으로 통신하므로 별도 `--link`가 필요하지 않다.

## 9. Volume / Mount가 필요한 이유

컨테이너의 쓰기 계층에만 저장한 파일은 **컨테이너를 삭제하면 사라진다.** 단순 정지·재시작과 삭제는 다르다.

```text
컨테이너: 교체 가능한 실행 환경
볼륨 / 바인드 마운트: 컨테이너와 수명을 분리할 데이터
```

마운트한 경로에 저장해야 데이터가 외부 저장공간에 기록된다. 영속성은 백업과도 다르다. 볼륨 자체 삭제, 파일 삭제, 디스크 장애에 대비한 백업은 별도로 필요하다.

## 10. Bind Mount

호스트의 실제 경로를 컨테이너 경로에 연결한다.

```text
호스트 /host/data ⇄ 컨테이너 /container/data
```

아래는 Linux 호스트의 실습 예제다. 먼저 호스트에서 폴더를 만든다.

```bash
mkdir -p /tmp/day02-data
docker run --rm -v /tmp/day02-data:/container/data alpine:3.23 sh -c 'echo hello > /container/data/test.txt'
cat /tmp/day02-data/test.txt
```

컨테이너가 종료·삭제된 뒤에도 호스트 파일에서 `hello`를 확인할 수 있다. 컨테이너에 저장한 뒤 호스트로 복사하는 과정이 아니라, **처음부터 연결된 호스트 저장공간에 접근**한 것이다.

기본적으로 양쪽 변경이 반영된다. 컨테이너에서 읽기만 하게 하려면 `:ro`를 붙인다.

```yaml
# 서비스 설정의 일부
volumes:
  - ./Dataset:/workspace/Dataset:ro
```

Compose의 상대 경로는 일반적으로 Compose 파일의 디렉터리를 기준으로 해석된다. 여러 `-f` 파일을 결합할 때는 기본적으로 첫 파일이 기준이다. 경로를 마운트하면 그 위치에 원래 있던 이미지 파일은 마운트 아래 가려진다.

## 11. Docker가 관리하는 Named Volume

Named volume은 사용자가 호스트 저장 경로를 직접 지정하는 대신 Docker에 저장 위치 관리를 맡기는 방식이다.

```bash
docker volume create day02-postgres-data
docker run -d --name day02-postgres -e POSTGRES_PASSWORD=day02-local-example -v day02-postgres-data:/var/lib/postgresql/data postgres:17
```

비밀번호는 학습용 예시다. 실사용 비밀번호를 그대로 공개 저장소에 기록하지 않는다.

```text
day02-postgres-data ⇄ PostgreSQL 컨테이너 /var/lib/postgresql/data
```

컨테이너를 삭제하더라도 볼륨을 지우지 않았다면 데이터는 남는다. 새 컨테이너에 같은 볼륨을 연결하면 호환되는 DB 버전에서 기존 데이터를 다시 사용할 수 있다. PostgreSQL 메이저 버전을 변경할 때는 별도 업그레이드 절차가 필요하다.

여기서는 저장 경로를 명확히 하기 위해 **`postgres:17`**을 사용했다. 공식 이미지의 PostgreSQL 18 이상은 기본 데이터 디렉터리·권장 마운트 경로가 달라졌으므로 태그만 바꿔 적용하지 않는다. 초기 비밀번호 등의 환경 변수는 빈 데이터 디렉터리를 처음 초기화할 때 적용된다.

## 12. Bind Mount와 Docker Volume 비교

| 구분 | Bind Mount | Named Volume |
| --- | --- | --- |
| 호스트 저장 경로 | 사용자가 지정 | Docker가 관리 |
| 호스트에서 직접 파일 관리 | 편리 | 직접 내부 파일을 조작하지 않는 방식이 일반적 |
| 컨테이너 삭제 후 유지 | 호스트 파일 유지 | 볼륨을 삭제하지 않으면 유지 |
| Dataset / Source / Config / Model | 호스트에서 직접 관리할 때 적합 | 애플리케이션 구조에 따라 사용 |
| DB 내부 데이터 | 사용 가능, 권한·경로 관리 필요 | 자주 사용하는 방식 |

Compose에서 named volume은 서비스에 연결하고 최상위에도 선언한다.

```yaml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_PASSWORD: day02-local-example
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

실제 볼륨 이름에는 기본적으로 프로젝트 접두사가 붙는다. 예: `day02_postgres-data`. 위의 수동 생성 볼륨과 자동으로 같은 볼륨이 되는 것은 아니다.

## 13. MLOps에서의 활용

```text
MLOps Host
└── Compose 프로젝트
    ├── Backend
    ├── DB
    └── ClearML 관련 서비스
         └── 공유한 Docker Network에서 서비스 이름으로 통신

호스트에서 직접 관리하는 Dataset / Model / Source / Config → Bind Mount
DB 등 서비스 내부 상태 데이터 → Named Volume
```

이는 역할을 이해하기 위한 개념도다. 실제 ClearML 구성은 여러 서비스로 나뉠 수 있다. Redis 데이터도 남기려면 볼륨 연결뿐 아니라 RDB/AOF 같은 Redis 자체 영속화 설정도 확인해야 한다.

## 14. 함께 확인하는 실습

다음은 **서로 다른 역할의 서비스 실행, 서비스 이름 통신, 볼륨 유지**를 연습하는 예제다. `web`은 Nginx 기본 페이지이며 DB를 사용하는 backend 코드를 구현한 것은 아니다.

아래 내용을 별도의 실습 폴더에 `compose.yaml`로 저장한다.

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "127.0.0.1:8082:80"

  db:
    image: postgres:17
    environment:
      POSTGRES_PASSWORD: day02-local-example
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:alpine

volumes:
  postgres-data:
```

```bash
docker compose -p day02 config -q
docker compose -p day02 up -d
docker compose -p day02 ps
docker compose -p day02 logs db
docker compose -p day02 exec web sh -c 'nc -z db 5432 && echo DB-port-open'
docker compose -p day02 exec redis redis-cli ping
```

DB가 준비된 뒤 연결 확인 명령을 실행한다. Nginx Alpine 이미지의 `nc`로 같은 네트워크의 `db:5432` TCP 포트에 접근하는 예시다. PostgreSQL 인증·쿼리 검증과는 다르다. Redis 응답은 `PONG`이다. 웹은 `http://localhost:8082`에서 확인한다.

볼륨 유지를 확인하려면 준비된 DB에 학습용 테이블을 만든다.

```bash
docker compose -p day02 exec db psql -U postgres -c "CREATE TABLE IF NOT EXISTS day02_note (message text); INSERT INTO day02_note VALUES ('volume persists');"
docker compose -p day02 down
docker compose -p day02 up -d
```

DB 시작 로그를 확인하고 준비된 뒤 조회한다.

```bash
docker compose -p day02 exec db psql -U postgres -c "SELECT * FROM day02_note;"
```

컨테이너가 다시 만들어져도 `volume persists` 행이 남아 있으면 영속성을 확인한 것이다. 실습 정리는 `docker compose -p day02 down`으로 한다. **`down -v`는 이 프로젝트의 명명된 볼륨과 연결된 익명 볼륨까지 제거하므로 데이터도 잃는다.** 외부(external) 볼륨은 Compose가 제거하지 않는다.

## 핵심 복습

| 개념 | 기억할 내용 |
| --- | --- |
| Compose | 컨테이너와 실행 설정을 프로젝트 단위로 관리 |
| `-d` | 터미널과 분리해서 실행 |
| Compose의 `-f` | 사용할 설정 파일 지정 |
| Docker Network | 컨테이너 통신 공간. 공유 네트워크에서 서비스 이름 사용 |
| Port Mapping | 호스트 주소·포트 → 컨테이너 포트 |
| Bind Mount | 사용자가 정한 호스트 경로 ⇄ 컨테이너 경로 |
| Named Volume | Docker 관리 저장공간 ⇄ 컨테이너 경로 |

1. `docker compose -p mlops`의 `-p`는 포트 번호일까?
2. backend가 DB에 접속할 때 왜 `localhost:5432` 대신 `db:5432`를 사용할까?
3. 포트 바인딩의 호스트 IP는 접속할 사용자 IP일까?
4. `down`과 `down -v`는 데이터 보존 측면에서 무엇이 다를까?
5. Bind Mount는 컨테이너 파일을 주기적으로 복사하는 기능일까?

<details>
<summary>정답 확인</summary>

1. 프로젝트 이름이다.
2. backend 안의 localhost는 자기 자신이고, db는 공유 네트워크에서 DB 서비스를 가리키기 때문이다.
3. 아니다. Docker 호스트에서 바인딩할 주소다.
4. down은 기본적으로 명명된 볼륨을 남기지만 down -v는 관리 대상 볼륨도 제거한다.
5. 아니다. 호스트 저장공간을 컨테이너 경로로 연결한다.

</details>

## 공식 참고 자료

- [Compose CLI](https://docs.docker.com/reference/cli/docker/compose/)
- [Compose up](https://docs.docker.com/reference/cli/docker/compose/up/)
- [Compose 네트워크](https://docs.docker.com/compose/how-tos/networking/)
- [포트 공개와 매핑](https://docs.docker.com/engine/network/port-publishing/)
- [레거시 링크](https://docs.docker.com/engine/network/links/)
- [Bind mounts](https://docs.docker.com/engine/storage/bind-mounts/)
- [Volumes](https://docs.docker.com/engine/storage/volumes/)
- [PostgreSQL 공식 이미지](https://hub.docker.com/_/postgres)

