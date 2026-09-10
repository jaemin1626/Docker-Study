# 4일차 — Compose로 복습하는 Docker 1·2·3일차

## 학습 목표와 참고 글

참고한 글: 브라더댄, [Docker 강의 6강: Docker Compose를 활용한 멀티 컨테이너 환경 구성](https://brotherdan.tistory.com/56) (2025-01-27).

참고 글은 웹 서버·데이터베이스·캐시를 Compose로 함께 실행하며 YAML 설정, 로그, 데이터 보존, 실행 순서와 확장 시 주의점을 익히는 강의다. 이 문서는 그 학습 주제를 바탕으로 **우리 저장소의 기존 노트와 질문을 연결한 독립적인 복습 자료**다. 예제와 설명은 아래 공식 문서를 기준으로 새로 작성했다.

| 이전 학습 | 이번에 확인할 내용 |
| --- | --- |
| [Day01: 기본 개념](../Day01-Docker-Basics/README.md) | 이미지와 컨테이너, exec, 포트 매핑 |
| [Day02: Compose·저장공간](../Day02-Docker-Compose-Network-Volume/README.md) | 프로젝트 실행, Bind Mount, Named Volume, down |
| [Day03: Network·DNS](../Day03-Docker-Network-DNS/README.md) | 사용자 정의 bridge, 서비스 이름, alias와 요청 분배의 차이 |

## 1. 먼저 전체 구조 읽기

```text
브라우저 → 호스트 localhost:8080 → nginx 컨테이너:80
                                      │
                                      └── html 폴더의 페이지 제공

Compose의 기본 네트워크
├── nginx :80
├── db    :3306
└── redis :6379

호스트 ./html  ⇄ nginx /usr/share/nginx/html
Named Volume ⇄ db    /var/lib/mysql
```

- 이미지 3종을 사용해 서비스 3개를 실행한다. 이미지와 컨테이너는 서로 다른 개념이다.
- Nginx 하나로 HTML 페이지 여러 개를 제공할 수 있다.
- 이 실습의 Nginx는 정적 UI 파일을 제공한다. DB·Redis를 사용하는 백엔드 프로그램은 포함하지 않는다.
- 같은 네트워크에 연결했다고 Nginx가 DB를 자동으로 조회하거나 API를 자동 생성하지는 않는다.

## 2. 복습용 Compose 설정

별도 실습 폴더에 다음 파일들을 준비한다. Docker Engine 또는 Docker Desktop의 Linux 컨테이너 환경을 기준으로 한다.

```text
day04-review/
├── compose.yaml
└── html/
    ├── index.html
    └── about.html
```

`compose.yaml`:

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "127.0.0.1:8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro

  db:
    image: mysql:8.4
    environment:
      MYSQL_ROOT_PASSWORD: day04-local-example
      MYSQL_DATABASE: study
    volumes:
      - mysql-data:/var/lib/mysql

  redis:
    image: redis:alpine

volumes:
  mysql-data:
```

비밀번호는 로컬 실습용 예시다. 실서비스 비밀번호를 GitHub에 올리지 않는다. DB와 Redis는 내부 통신만 연습하므로 호스트 포트 공개를 생략했다. 태그도 시간이 지나면 내용이 바뀔 수 있어 완전히 동일한 이미지 재현에는 digest 고정이 필요하다.

설정 읽기:

| 항목 | 해석 |
| --- | --- |
| `services.nginx` | 서비스 이름. Compose 명령에서도 이 이름 사용 |
| `image` | 컨테이너 생성에 사용할 이미지 |
| `127.0.0.1:8080:80` | 호스트 로컬 주소 8080 → 컨테이너 80 |
| `./html:...:ro` | 호스트 폴더를 읽기 전용으로 마운트 |
| `environment` | 프로그램에 전달할 환경 변수 |
| `mysql-data:...` | 명명된 볼륨 연결 |
| 최상위 `volumes` | 이 프로젝트가 사용할 named volume 선언 |

현재 Compose에서는 최상위 `version`이 obsolete이므로 생략한다. `container_name`도 필수가 아니며, 고정하면 해당 서비스를 여러 컨테이너로 확장할 수 없다. [Compose version](https://docs.docker.com/reference/compose-file/version-and-name/), [서비스 설정](https://docs.docker.com/reference/compose-file/services/)

## 3. 하나의 Nginx에서 여러 페이지 제공

`html/index.html`:

```html
<!doctype html>
<html lang="ko">
<head><meta charset="utf-8"><title>Docker 복습</title></head>
<body>
  <h1>4일차 메인 페이지</h1>
  <p>Nginx 컨테이너 하나가 여러 파일을 제공합니다.</p>
  <a href="/about.html">소개 페이지로 이동</a>
</body>
</html>
```

`html/about.html`:

```html
<!doctype html>
<html lang="ko">
<head><meta charset="utf-8"><title>소개</title></head>
<body>
  <h1>Docker 학습 소개</h1>
  <p>이미지, 네트워크, 볼륨을 Compose로 복습합니다.</p>
  <a href="/">메인으로 돌아가기</a>
</body>
</html>
```

```text
localhost:8080/           → nginx → html/index.html
localhost:8080/about.html → nginx → html/about.html
```

**포트는 Nginx로 들어가는 입구이고, URL 경로가 파일을 구분한다.** HTML 파일마다 포트나 컨테이너를 추가할 필요가 없다. 포트 공개만으로 호스트 폴더가 연결되지는 않으며 마운트가 별도로 필요하다.

HTML을 수정하고 새로고침하면 변경을 확인할 수 있다. 캐시가 남으면 강력 새로고침으로 확인한다. 로그인 HTML을 추가해도 실제 인증 기능은 별도 백엔드 구현이 필요하다.

## 4. 실행·조회·컨테이너 안 확인

이하 명령은 `compose.yaml`이 있는 폴더에서 실행한다.

```bash
docker compose version
docker compose -p day04 config -q
docker compose -p day04 up -d
docker compose -p day04 ps
docker compose -p day04 logs --tail 30
docker compose -p day04 exec nginx sh
```

`config -q`는 설정 해석·검증, `up -d`는 백그라운드 생성·실행이다. 이미 실행 중이면 기존 상태와 설정에 따라 처리한다. `-p day04`는 프로젝트 이름이며 포트 설정이 아니다.

새 셸에서는 다음을 확인한다.

```sh
ls /usr/share/nginx/html
cat /usr/share/nginx/html/about.html
exit
```

`exec`는 추가 프로세스이므로 여기서 `exit`해도 주 프로세스인 Nginx는 계속 실행된다. `attach`는 기존 주 프로세스의 입출력에 연결하므로 Nginx에 붙어도 새 셸이 생기지 않는다. Compose의 `exec`는 대화형 터미널 연결이 기본값이다.

## 5. Day03 연결: DNS는 주소를 알려준다

```bash
docker network inspect day04_default
docker compose -p day04 exec nginx cat /etc/resolv.conf
```

이 예제의 기본 네트워크 이름은 `day04_default`다. 사용자 정의 네트워크에서 Docker embedded DNS 주소 `127.0.0.11`을 확인할 수 있다.

```text
클라이언트: "db의 IP가 무엇이지?"
    → Docker DNS가 주소 응답
    → 클라이언트가 db의 IP:3306에 연결
```

- 내부 주소는 `db:3306`, `redis:6379`다.
- Nginx 안의 `localhost`는 Nginx 컨테이너 자신이다.
- bridge 네트워크를 공유해도 각 컨테이너의 network namespace는 분리돼 있다.
- DNS는 이름을 IP로 바꾸며 요청을 대신 처리하거나 부하를 측정해 분배하지 않는다.
- 이 구성은 한 호스트의 bridge 네트워크다. host나 container 네트워크 공유 모드와 구분한다.

Redis에 서비스 이름으로 접근하는 임시 클라이언트를 실행해 볼 수 있다.

```bash
docker run --rm --network day04_default redis:alpine redis-cli -h redis ping
```

준비된 Redis의 예상 응답은 `PONG`이다. 호스트에 6379를 공개하지 않아도 같은 네트워크에서는 접근할 수 있다. [Compose 네트워크](https://docs.docker.com/compose/how-tos/networking/)

## 6. 시작 순서와 준비 완료는 다르다

정적 파일만 제공하는 현재 Nginx에는 DB 의존성을 억지로 추가하지 않았다. DB를 사용하는 backend를 추가할 때는 준비 여부를 따로 다뤄야 한다.

아래는 전체 실행 파일이 아닌 **설정 원리 예시**다.

```yaml
services:
  backend:
    image: my-backend:local
    depends_on:
      db:
        condition: service_healthy
```

`my-backend:local`은 직접 준비해야 하는 이미지이고, `db`에는 적절한 `healthcheck`가 정의되어 있어야 한다.

짧은 `depends_on: [db]`는 시작 순서만 다룬다. `service_healthy`는 정의된 healthcheck가 통과하기를 기다린다. 이것도 실행 중 장애를 모두 해결해 주지는 않으므로 애플리케이션에는 재연결·재시도 처리가 필요하다. [시작 순서와 healthcheck](https://docs.docker.com/compose/how-tos/startup-order/)

## 7. 데이터가 남는지 확인

DB 초기화 완료는 로그에서 확인한다.

```bash
docker compose -p day04 logs db
docker compose -p day04 exec db mysql -uroot -p study
```

비밀번호 질문에 위의 학습용 비밀번호를 입력하고 SQL을 실행한다.

```sql
CREATE TABLE IF NOT EXISTS review_note (message VARCHAR(100));
INSERT INTO review_note VALUES ('Day04 volume check');
SELECT * FROM review_note;
exit
```

이제 컨테이너를 제거하고 재생성한다.

```bash
docker compose -p day04 down
docker compose -p day04 up -d
```

DB가 다시 준비되면 같은 클라이언트 명령으로 접속해 `SELECT * FROM review_note;`를 실행한다. 기존 행이 남아 있으면 볼륨의 영속성을 확인한 것이다. 초기 비밀번호 환경 변수를 바꿔도 기존 DB 사용자의 비밀번호가 자동 변경되는 것은 아니다. [MySQL 공식 이미지](https://hub.docker.com/_/mysql)

| 저장 위치 | `down` | `down -v` |
| --- | --- | --- |
| 컨테이너 쓰기 계층 | 컨테이너와 함께 제거 | 제거 |
| 호스트 `./html` Bind Mount | 유지 | 유지 |
| Compose 관리 Named Volume | 유지 | 제거 |
| 컨테이너에 연결된 익명 볼륨 | 기본적으로 남지만 재생성 때 자동 재연결 안 됨 | 제거 |
| external 선언 볼륨 | 유지 | 유지 |

**`down -v`는 데이터를 버릴 의도가 있을 때만 실행한다.** 바인드 마운트 파일을 컨테이너 안에서 직접 삭제하면 호스트에도 반영되지만, 이번 HTML 마운트는 `:ro`라 컨테이너에서 수정·삭제할 수 없다. [down 공식 동작](https://docs.docker.com/reference/cli/docker/compose/down/)

## 8. --scale nginx=3의 정확한 의미

```bash
docker compose -p day04 up -d --scale nginx=3
```

이 명령은 nginx 서비스를 **총 3개 컨테이너로 맞추는 것**이다. 사용자 3명에게 하나씩 배정하는 기능도, 페이지 3개를 만드는 기능도 아니다.

다만 **위 기본 실습 설정에 그대로 실행하면 8080 포트 충돌이 난다.** 복제 실습을 하려면 별도 폴더의 `compose.scale.yaml`을 아래처럼 준비한다.

```yaml
services:
  nginx:
    image: nginx:alpine
```

```bash
docker compose -p day04-scale -f compose.scale.yaml up -d --scale nginx=3
docker compose -p day04-scale -f compose.scale.yaml ps
docker compose -p day04-scale -f compose.scale.yaml down
```

호스트 포트와 고정 `container_name`을 지정하지 않았으므로 복제 수 확인이 가능하다. 이 예제는 외부 브라우저 접속이나 요청 분배를 구성한 것은 아니다.

```text
실제 요청 분배 구성:
여러 사용자의 요청 → 별도 로드 밸런서 → 복제된 서비스들
```

컨테이너 하나도 여러 사용자를 처리한다. DNS alias로 IP 여러 개를 반환하는 것과 부하·상태를 고려하는 로드 밸런싱도 다르다. 사용자 세션을 복제본 전체에서 유지하려면 공유 저장소 등 별도 설계가 필요하다. [container_name과 scale 제약](https://docs.docker.com/reference/compose-file/services/#container_name)

## 9. 오류가 나면 확인할 순서

| 현상 | 먼저 확인할 것 |
| --- | --- |
| YAML 오류 | 들여쓰기, `docker compose -p day04 config -q` |
| 8080 포트 충돌 | 기존 사용 서비스 확인 후 실습 포트를 8084 등으로 변경 |
| 페이지 404 | `html` 폴더와 파일명, URL 경로 확인 |
| 파일 Permission denied | 파일 읽기 권한과 상위 폴더 접근 권한, 마운트 경로 확인 |
| DB 접속 실패 | 초기화 로그, 실제 인증 정보, DB 준비 상태 확인 |
| 이름을 찾지 못함 | 두 컨테이너가 같은 네트워크에 있는지 확인 |
| 복제 실패 | `container_name`과 고정 호스트 포트 확인 |

DB 접속 오류만 보고 `down -v`부터 실행하지 않는다. 데이터가 필요한 환경에서는 로그와 인증 설정을 먼저 점검한다. 본 문서는 학습용 명령을 제공하며 실제 실행 결과는 환경에서 확인해야 한다.

## 10. 복습 문제

1. Nginx 한 개로 메인·소개·로그인 페이지를 제공할 수 있을까?
2. 8080:80은 HTML 폴더를 연결하는 설정일까?
3. Nginx와 DB가 같은 네트워크면 자동으로 DB 기반 웹앱이 될까?
4. db와 localhost는 Nginx 컨테이너 안에서 같은 대상일까?
5. `exec nginx sh`에서 exit하면 Nginx도 종료될까?
6. down -v 뒤에도 ./html은 남을까? mysql-data는?
7. scale=3은 사용자당 컨테이너 한 개를 배정할까?
8. version: "3.8"과 container_name은 필수일까?

<details>
<summary>정답 확인</summary>

1. 가능하다. URL 경로로 파일을 구분한다.
2. 아니다. 포트는 요청 전달, 마운트는 파일 경로 연결이다.
3. 아니다. DB를 사용하는 애플리케이션 코드가 필요하다.
4. 아니다. db는 서비스 이름, localhost는 자기 컨테이너다.
5. 아니다. 추가 셸만 종료된다.
6. html은 남고 Compose 관리 mysql-data는 삭제된다.
7. 아니다. 서비스 복제 수를 맞추며 요청 분배는 별도다.
8. 아니다. version은 obsolete이고 고정 container_name은 확장을 제한한다.

</details>

