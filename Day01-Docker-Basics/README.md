# 1일차 — Docker 기본 개념과 명령어

## 1. Docker, 이미지, 컨테이너

Docker는 애플리케이션을 필요한 파일·라이브러리와 함께 이미지로 묶고, 컨테이너로 실행하는 도구입니다.

| 개념 | 의미 | 비유 |
| --- | --- | --- |
| 이미지(Image) | 실행에 필요한 파일과 기본 설정을 담은 읽기 전용 템플릿 | 붕어빵 틀 |
| 컨테이너(Container) | 이미지를 바탕으로 만들어진 격리된 실행 환경. 정지 상태로도 존재 | 틀로 만든 붕어빵 |
| Docker Engine | 이미지를 관리하고 컨테이너를 생성·실행하는 핵심 구성 요소 | 실제 작업 담당자 |

하나의 이미지로 여러 컨테이너를 만들 수 있습니다. 각 컨테이너는 고유 ID와 쓰기 가능한 파일 계층을 가집니다. 컨테이너 안에서 파일을 바꿔도 원본 이미지가 자동으로 바뀌지는 않습니다.

컨테이너는 일반적인 가상 머신처럼 각각 별도 커널을 부팅하지 않습니다. Linux 컨테이너는 실행 환경의 Linux 커널을 공유하며 프로세스·파일시스템·네트워크 등을 격리합니다. Docker Desktop에서는 Linux VM이 그 실행 환경을 제공할 수 있습니다.

참고: [Docker 개요](https://docs.docker.com/get-started/docker-overview/)

## 2. Docker Hub, Private Registry, Docker Desktop, Compose

| 이름 | 하는 일 | 기억할 점 |
| --- | --- | --- |
| Docker Hub | Docker가 제공하는 이미지 저장·배포 서비스 | 공개 이미지와 비공개 저장소를 제공 |
| Private Registry | 접근을 제한해 사용하는 이미지 레지스트리 | 회사 내부 서버나 관리형 서비스 등으로 운영 가능 |
| Docker Desktop | PC에서 Docker를 설치·실행·관리하는 앱 | Engine, CLI, Compose와 관리 화면 등을 제공 |
| Docker Compose | 여러 서비스의 실행 설정을 YAML로 정의하고 함께 관리 | 이미지 저장소가 아니라 애플리케이션 실행 관리 도구 |

Registry는 여러 이미지 repository를 보관하는 서버/서비스이고, repository는 특정 이미지의 태그들을 모아 놓은 단위입니다. Docker Hub도 registry의 한 종류입니다. Docker Hub의 비공개 repository와 직접 운영하는 private registry는 운영 방식이 다릅니다.

```text
Registry / Docker Hub
       │ docker pull (이미지 내려받기)
       ▼
로컬 이미지 ── docker run ──▶ 컨테이너
       │
       └── docker push ──▶ Registry
```

`docker push`에는 대상 repository에 대한 쓰기 권한과 필요한 로그인이 있어야 합니다.

참고: [Docker 개요 — Registries](https://docs.docker.com/get-started/docker-overview/), [Docker Desktop](https://docs.docker.com/desktop/), [Compose](https://docs.docker.com/compose/)

## 3. 이미지 이름 구조와 태그

간단히 `[registry/repository]:tag`로 생각할 수 있습니다. 대괄호는 설명용이며 명령에 입력하지 않습니다. 선택 항목까지 표현하면 다음과 같습니다.

```text
[registry[:port]/][namespace/]repository[:tag]

ubuntu:24.04
docker.io/library/ubuntu:24.04
registry.example.com:5000/team/myapp:1.0
```

- `registry`: 이미지를 받을 서버. 생략하면 기본적으로 Docker Hub(`docker.io`).
- `namespace`: 사용자/조직 등의 구분. `ubuntu` 같은 Docker 공식 이미지는 `library`를 사용.
- `repository`: 이미지 저장소 이름. 예: `ubuntu`, `myapp`.
- `tag`: 버전이나 변형을 구분하는 이름. 예: `24.04`, `1.0`, `alpine`.
- 태그를 생략하면 `latest`를 사용합니다. `latest`는 태그 이름이며 최신 버전이라는 보장은 없습니다. 태그가 가리키는 이미지도 바뀔 수 있습니다.

```bash
docker pull ubuntu:24.04
docker image ls
```

`docker tag`는 이미지에 다른 이름을 붙이는 명령입니다. 자체적으로 이미지를 업로드하지 않습니다.

참고: [이미지 이름과 태그](https://docs.docker.com/reference/cli/docker/image/tag/)

## 4. docker save — 이미지를 파일로 보관

```bash
docker pull ubuntu:24.04
docker save -o ubuntu-24.04.tar ubuntu:24.04
docker load -i ubuntu-24.04.tar
```

- `save`: 로컬 이미지의 계층과 이미지 정보를 tar 파일로 저장합니다.
- `load`: 저장한 이미지 파일을 Docker로 읽어들입니다.
- `save`는 실행 중인 컨테이너의 메모리, 컨테이너에서 새로 바꾼 파일, 볼륨 데이터를 통째로 백업하는 기능이 아닙니다.
- `docker export`는 컨테이너 파일시스템을 내보내는 별도 명령입니다. 이미지 계층·설정을 보존하는 `save/load`와 목적이 다르고, 볼륨 데이터 백업도 별도로 해야 합니다.

참고: [docker image save](https://docs.docker.com/reference/cli/docker/image/save/)

## 5. 컨테이너 ID와 이름

```bash
docker ps
docker ps -a
docker ps -a --no-trunc
```

`docker ps`는 실행 중인 컨테이너, `docker ps -a`는 정지된 컨테이너까지 보여 줍니다. `--no-trunc`를 붙이면 전체 ID를 확인할 수 있습니다.

컨테이너 ID는 각 컨테이너의 고유 식별자입니다. 보통 목록에는 앞 12자리가 표시됩니다. 충돌 없이 구분되는 ID 접두사나 `--name`으로 지정한 이름으로 명령을 실행할 수 있습니다. 이미지 ID와 컨테이너 ID는 서로 다릅니다.

## 6. create / start / attach / run / exec

| 명령 | 역할 | 새 컨테이너 생성? |
| --- | --- | --- |
| `docker create` | 이미지로 컨테이너만 생성 | O |
| `docker start` | 기존 컨테이너 시작 | X |
| `docker attach` | 실행 중인 컨테이너의 주 프로세스 입출력에 연결 | X |
| `docker run` | 새 컨테이너를 생성하고 시작 | O |
| `docker exec` | 실행 중인 컨테이너 안에서 추가 프로세스 실행 | X |

`run`은 큰 흐름에서 `create + start`입니다. 필요한 이미지가 로컬에 없으면 기본 설정에서 내려받기도 합니다. 기존 컨테이너를 다시 켤 때는 `run` 대신 `start`를 사용합니다.

### -i, -t, -d

- `-i`: 표준 입력을 열어 둡니다.
- `-t`: 가상 터미널(TTY)을 할당합니다.
- `-it`: 셸과 대화하듯 작업할 때 두 옵션을 함께 사용합니다.
- `-d`: 백그라운드에서 실행합니다.

`-it`가 셸 자체를 설치하거나 실행하는 것은 아닙니다. `bash` 또는 `sh` 등 실제 실행할 프로그램이 이미지에 있어야 합니다.

### 생성과 시작을 나눠 보기

아래 예제는 같은 이름의 컨테이너가 없는 상태에서 순서대로 실행합니다.

```bash
docker create -it --name day01-shell ubuntu:24.04 bash
docker ps -a
docker start day01-shell
docker attach day01-shell
```

이제 컨테이너의 `bash`에 연결됩니다. `exit`로 주 프로세스인 셸을 끝내면 컨테이너도 정지합니다. 이처럼 `-it`로 만든 컨테이너에서 실행을 유지한 채 연결만 끊으려면 기본 분리 키인 **Ctrl+P, 이어서 Ctrl+Q**를 사용합니다.

정지했다면 다음 명령으로 다시 시작하면서 입력과 출력을 연결할 수 있습니다.

```bash
docker start -ai day01-shell
```

### 한 번에 생성·실행하기

```bash
docker run -it --name day01-run ubuntu:24.04 bash
```

위 명령은 `day01-run`이라는 새 컨테이너를 만듭니다. 컨테이너가 정지했더라도 이름은 남아 있으므로 같은 이름으로 `run`을 반복하면 충돌합니다.

### attach와 exec의 핵심 차이

실행 중인 `day01-shell`에 별도의 셸을 열려면 호스트의 다른 터미널에서 실행합니다.

```bash
docker exec -it day01-shell bash
```

`attach`는 기존 주 프로세스에 연결하고, `exec`는 추가 프로세스를 실행합니다. `exec`로 연 셸에서 `exit`하면 그 셸만 종료되며, 원래 주 프로세스가 살아 있다면 컨테이너는 계속 실행됩니다. 정지된 컨테이너에는 `exec`를 실행할 수 없습니다.

참고: [컨테이너 실행](https://docs.docker.com/engine/containers/run/), [attach](https://docs.docker.com/reference/cli/docker/container/attach/), [exec](https://docs.docker.com/reference/cli/docker/container/exec/)

## 7. 컨테이너 삭제 — rm -f와 prune

다음은 삭제 명령 설명용 예제입니다. 필요한 데이터를 확인한 다음 대상으로 지정한 컨테이너만 정리합니다.

```bash
docker stop day01-run
docker rm day01-run
```

| 명령 | 삭제 범위 |
| --- | --- |
| `docker rm day01-run` | 지정한 정지 컨테이너 |
| `docker rm -f day01-shell` | 지정한 컨테이너를 실행 중이어도 강제로 종료·삭제 |
| `docker container prune` | Docker 환경의 모든 정지 컨테이너. 기본적으로 확인 질문 표시 |
| `docker image prune` | 기본적으로 dangling 이미지 정리 |
| `docker system prune` | 정지 컨테이너 외에 미사용 네트워크, dangling 이미지, 빌드 캐시 등도 정리 |

`rm -f`의 `-f`는 강제 삭제이고, `container prune -f`의 `-f`는 확인 질문을 생략합니다. `prune`은 하위 명령에 따라 대상이 달라집니다.

컨테이너를 삭제해도 원본 이미지는 남습니다. 컨테이너의 쓰기 계층에만 있던 파일은 사라지므로 계속 쓸 데이터는 볼륨이나 바인드 마운트로 보관해야 합니다. 위 컨테이너 삭제 명령이 볼륨까지 자동으로 백업하거나 모두 삭제하는 것은 아닙니다.

참고: [container prune](https://docs.docker.com/reference/cli/docker/container/prune/)

## 8. 컨테이너 가상 IP, eth0, lo

다음 설명은 일반적인 Linux bridge 네트워크를 기준으로 합니다. `host`, `none` 등의 네트워크 모드에서는 구성이 달라집니다.

컨테이너는 연결된 Docker 네트워크에서 내부 IP를 할당받습니다. 예를 들어 `172.17.0.2` 같은 주소가 나올 수 있지만 고정된 값은 아닙니다. 재생성이나 네트워크 연결 변경 등에 따라 바뀔 수 있으므로 주소를 외워서 의존하지 않습니다.

```bash
docker inspect --format '{{json .NetworkSettings.Networks}}' day01-shell
```

| 인터페이스 | 의미 |
| --- | --- |
| `eth0` | 보통 컨테이너의 첫 번째 가상 네트워크 인터페이스. Docker 네트워크와 통신 |
| `lo` | 루프백 인터페이스. `127.0.0.1` / `localhost`로 자기 자신과 통신 |

일반적인 격리 네트워크에서 컨테이너 안의 `localhost`는 **그 컨테이너 자신**입니다. 호스트나 다른 컨테이너를 뜻하지 않습니다. 인터페이스 이름과 개수는 네트워크 구성에 따라 달라질 수 있습니다.

컨테이너에 `ip` 명령이 설치돼 있다면 `docker exec day01-shell ip addr`로 인터페이스를 볼 수 있습니다. 최소 이미지에는 이 도구가 없을 수 있으므로 IP 확인만 필요하면 위의 `docker inspect`를 사용합니다.

같은 사용자 정의 bridge 네트워크에 연결한 컨테이너끼리는 이름으로 통신할 수 있습니다. 외부 PC에서는 내부 IP 대신 호스트의 공개된 포트로 접속하는 것이 일반적입니다. Docker Desktop의 Linux 컨테이너 IP는 VM 내부 주소라 호스트에서 직접 접근할 수 없는 경우도 있습니다.

참고: [Docker 네트워크](https://docs.docker.com/engine/network/)

## 9. -p 8080:80 — 포트 매핑

수업에서 본 명령:

```bash
docker run -i -t --name mywebserver -p 8080:80 ubuntu:14.04
```

여기서 `8080`은 서버 IP 주소가 아니라 **호스트 포트 번호**이고, `80`은 **컨테이너 안에서 프로그램이 요청을 기다리는 포트 번호**입니다.

```text
-p 호스트포트:컨테이너포트

브라우저: http://호스트IP:8080
                  │
                  ▼
           호스트의 8080번 포트
                  │ Docker가 전달
                  ▼
           컨테이너의 80번 포트
                  │
                  ▼
           실행 중인 웹 서버
```

HTTP 서버는 관례적으로 80번 포트를 사용하지만 프로그램 설정에 따라 다른 포트를 사용할 수도 있습니다. 매핑의 오른쪽 번호는 실제 프로그램이 듣고 있는 포트와 맞아야 합니다.

`ubuntu:14.04`는 과거 수업 명령을 해석하기 위한 예시입니다. Ubuntu 이미지 자체가 웹 서버를 자동으로 설치·실행하지는 않습니다. 따라서 위 명령의 포트 매핑만으로 웹 페이지가 뜨지는 않습니다.

### 실제 웹 서버로 확인하기

```bash
docker run -d --name day01-web -p 8080:80 nginx:alpine
docker port day01-web
docker logs day01-web
```

Docker가 현재 PC에서 실행 중이면 브라우저에서 `http://localhost:8080`으로 접속합니다. 원격 서버에서 실행 중이라면 `http://서버IP:8080`을 사용하며 해당 서버의 네트워크·방화벽 접근이 가능해야 합니다. `nginx:alpine`은 Nginx 웹 서버를 실행하는 이미지입니다.

포트 매핑은 기본적으로 호스트의 모든 네트워크 인터페이스에 바인딩합니다. 로컬 PC에서만 접속할 별도 실습 예제는 다음과 같습니다.

```bash
docker run -d --name day01-web-local -p 127.0.0.1:8081:80 nginx:alpine
```

이 경우 `http://localhost:8081`로 접속합니다. 컨테이너의 프로그램도 컨테이너 네트워크 인터페이스에서 요청을 받아야 하며, 컨테이너 내부 `127.0.0.1`에만 바인딩돼 있으면 일반적인 포트 매핑으로 접근할 수 없습니다.

### 컨테이너마다 80번을 써도 되는 이유

```text
같은 호스트
  :8080 ──▶ 컨테이너 A :80
  :8081 ──▶ 컨테이너 B :80
  :8082 ──▶ 컨테이너 C :80
```

컨테이너별 네트워크 공간이 분리돼 있어 내부 포트는 같아도 됩니다. 같은 호스트 IP와 프로토콜에서 사용할 호스트 포트는 서로 겹치면 안 됩니다.

`EXPOSE 80`은 이미지가 사용하는 포트를 알리는 설정이며, 그 자체로 호스트에 포트를 공개하지 않습니다. 실제 공개에는 `-p` 등의 설정이 필요합니다.

참고: [포트 공개](https://docs.docker.com/get-started/docker-concepts/running-containers/publishing-ports/), [포트 매핑](https://docs.docker.com/engine/network/port-publishing/)

## 10. Docker Compose로 실행 설정 기록하기

여러 개의 긴 `docker run` 명령 대신 `compose.yaml`에 서비스·포트·볼륨·네트워크 설정 등을 적을 수 있습니다. 아래는 문법을 익히기 위한 한 서비스 예제입니다.

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "127.0.0.1:8082:80"
```

위 내용을 `compose.yaml`로 저장하고 그 폴더에서 실행합니다. 예제의 8082번 포트는 다른 프로그램이 사용하지 않아야 합니다.

```bash
docker compose up -d
docker compose ps
docker compose logs web
docker compose exec web sh
```

셸에서 `exit`한 뒤 정리하려면:

```bash
docker compose down
```

`up -d`는 설정에 맞게 컨테이너를 생성·시작하고 백그라운드로 실행합니다. `down`은 해당 Compose 프로젝트의 서비스 컨테이너와 관리 대상 네트워크 등을 제거합니다. 명명된 볼륨은 기본적으로 남으며, 볼륨 제거 옵션인 `-v`는 데이터 삭제를 의도할 때만 사용합니다.

Docker Desktop에는 Compose가 포함됩니다. 여기서는 공백이 있는 `docker compose` 명령을 사용합니다. 예전 자료의 `docker-compose`는 별도의 구형 실행 방식입니다.

참고: [Compose 개요](https://docs.docker.com/compose/), [Compose 설치 방식](https://docs.docker.com/compose/install/)

## 11. 복습 질문

1. 같은 이미지로 컨테이너 3개를 만들면 ID도 같을까?
2. 정지된 컨테이너를 다시 켜는 명령은 `run`일까, `start`일까?
3. `attach`와 `exec -it ... bash`는 무엇이 다를까?
4. `docker save`가 컨테이너의 변경 파일과 볼륨 데이터까지 보관할까?
5. `-p 8080:80`에서 브라우저가 접속하는 호스트 포트는 무엇일까?
6. 컨테이너 안의 `localhost`는 일반적으로 누구를 가리킬까?
7. Docker Hub와 Docker Compose의 역할은 어떻게 다를까?

<details>
<summary>정답 확인</summary>

1. 다르다. 컨테이너마다 고유 ID를 가진다.
2. `docker start`. `run`은 새 컨테이너를 만든다.
3. `attach`는 기존 주 프로세스의 입출력에 연결하고, `exec`는 추가 프로세스를 실행한다.
4. 아니다. 이미지 저장과 컨테이너·볼륨 데이터 백업은 구분해야 한다.
5. `8080`. Docker가 컨테이너의 `80`으로 전달한다.
6. 그 컨테이너 자신이다. 일반적인 격리 네트워크 기준이다.
7. Hub는 이미지를 저장·배포하는 서비스, Compose는 서비스 실행 설정과 생명주기를 관리하는 도구다.

</details>
