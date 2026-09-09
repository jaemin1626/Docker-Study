# 3일차 — Docker Network 심화와 DNS Alias

Day02에서 익힌 Compose 서비스 이름 통신을 바탕으로, 네트워크가 실제로 연결되는 방식과 이름 해석을 정리한다.

## 학습 순서

1. 네트워크 드라이버와 namespace 공유 방식 비교
2. 사용자 정의 bridge 생성·연결·분리 및 패킷 흐름
3. host / none / container / overlay 활용
4. Embedded DNS와 network alias, Compose 서비스 이름
5. Macvlan과 LAN IP: bridge와 접근 방식 비교
6. 복습 질문으로 확인

예제는 Linux 컨테이너 기준이다. 명령은 각 실습 절 안에서 순서대로 실행하고, 여러 줄의 `\` 연결은 Bash 문법이다. Docker Desktop에서 Linux 호스트 내부 구조를 확인할 때는 VM 경계를 구분한다. 각 절의 공식 문서 링크에서 세부 조건을 확인할 수 있다.

## 1. 네트워크 드라이버와 공유 모드 비교

**Network namespace**는 인터페이스, IP, 라우팅 테이블, 포트 같은 네트워크 자원을 구분하는 공간이다. 아래는 일반적인 **Linux의 rootful Docker Engine** 기준이다. Docker Desktop의 Linux 컨테이너는 VM 안에서 실행되므로 `docker0` 등이 Windows/macOS 호스트에 그대로 보이지 않을 수 있다.

| 구분 | 네트워크 공간 / 연결 방식 | 대표 용도와 주의점 |
| --- | --- | --- |
| `bridge` | 컨테이너마다 별도 namespace와 IP, 호스트의 가상 bridge로 연결 | 한 Docker Host의 웹·DB 등. 같은 네트워크에서는 컨테이너 포트로 통신 |
| `host` | Host의 network namespace 공유, 별도 컨테이너 IP 없음 | 호스트 네트워크를 직접 사용. `-p` 불필요, 포트 충돌 가능 |
| `none` | 독립 namespace에 loopback(`lo`)만 존재 | 네트워크 없는 배치 작업, 오프라인 inference·전처리 |
| `container:<이름 또는 ID>` | 다른 컨테이너의 network namespace 공유 | 진단용 보조 컨테이너 등. IP·localhost·포트 공간 공유 |
| `overlay` | 여러 Docker Host에 걸친 논리 네트워크 | Swarm의 여러 노드에 분산된 서비스·컨테이너 연결 |

`container`는 독립적인 네트워크 드라이버가 아니라 **`--network`로 지정하는 공유 모드**다. `none` 네트워크의 드라이버는 `docker network ls`에서 `null`로 표시된다. [공식 네트워크 개요](https://docs.docker.com/engine/network/)

## 2. user-defined bridge network를 만드는 이유

`docker run`에서 네트워크를 지정하지 않으면 기본 `bridge`에 연결된다. 직접 만든 bridge와 같은 드라이버지만 사용 편의와 분리 단위가 다르다.

| 구분 | 기본 `bridge` | 사용자 정의 bridge |
| --- | --- | --- |
| Linux bridge 이름 | 보통 `docker0` | 보통 `br-<network ID>` |
| 컨테이너 이름 / alias 자동 DNS | 기본 제공하지 않음 | 같은 네트워크에서 제공 |
| 애플리케이션 구분 | 네트워크 미지정 컨테이너가 함께 연결 | 프로젝트·역할별로 묶어 분리 가능 |
| 설정 관리 | 기본 bridge 설정에 의존 | 네트워크별 subnet 등 설정 가능 |

이름으로 연결하면 IP 변경에 덜 의존하고, 관련 컨테이너끼리 통신 범위를 묶을 수 있다. 사용자 정의 bridge도 기본적으로 외부로 통신할 수 있으므로 **만들기만 하면 인터넷이 차단되는 것은 아니다.** Compose는 이 사용자 정의 네트워크 구성을 자동화한다. [Bridge 공식 문서](https://docs.docker.com/engine/network/drivers/bridge/)

## 3. create / connect / disconnect 실습

다음 명령은 순서대로 실행한다. `day03-net-*` 이름이 기존 컨테이너와 겹치지 않는 실습 환경을 사용한다.

```bash
# 네트워크 생성 (-d bridge는 기본값이므로 생략 가능)
docker network create --driver bridge day03-study-net
docker network ls

# A는 생성과 동시에 연결, B는 기본 bridge에서 시작
docker run -d --name day03-net-a --network day03-study-net alpine:3.23 sleep 3600
docker run -d --name day03-net-b alpine:3.23 sleep 3600

# 실행 중인 B를 사용자 정의 네트워크에도 연결
docker network connect day03-study-net day03-net-b
docker network inspect day03-study-net
docker exec day03-net-a ping -c 2 day03-net-b

# B에서 처음 연결됐던 기본 bridge 연결을 제거
docker network disconnect bridge day03-net-b

# 공유 네트워크에서도 B를 분리하면 A -> B 통신 경로가 사라짐
docker network disconnect day03-study-net day03-net-b
docker network inspect day03-study-net

# 다시 연결하면 같은 네트워크에서 이름으로 통신 가능
docker network connect day03-study-net day03-net-b
docker exec day03-net-a ping -c 2 day03-net-b
```

`connect`는 기존 연결을 교체하지 않고 **연결을 추가**한다. `disconnect`도 컨테이너를 삭제하지 않고 지정한 네트워크 연결만 끊는다. 연결 변경은 진행 중인 통신에 영향을 줄 수 있다. 재연결 뒤 실제 IP는 `inspect`로 확인한다. [명령어 참고](https://docs.docker.com/reference/cli/docker/network/connect/)

실습 컨테이너와 네트워크만 정리한다.

```bash
docker rm -f day03-net-a day03-net-b
docker network rm day03-study-net
```

## 4. eth0 - veth - docker0 - Host 흐름

`veth pair`는 양 끝이 연결된 가상 네트워크 인터페이스 쌍이다. 컨테이너 쪽 끝이 보통 `eth0`, 호스트 쪽 끝이 `veth...`다. `eth0` 뒤에 별도의 장치가 하나 더 있는 것으로 이해하지 않는다.

```text
기본 bridge, 외부 목적지로 나가는 흐름 (기본 NAT 구성)

+-- Container network namespace --+
| Application -> eth0             |
+-----------------|---------------+
                  | veth pair
+-- Host network namespace -------|------------------------+
|              veth...                                    |
|                 |                                       |
|              docker0 (Linux bridge / gateway)            |
|                 |                                       |
|       Host routing / firewall / NAT                     |
|                 |                                       |
|       Host NIC (예: ens3) -> 외부 네트워크                |
+---------------------------------------------------------+
```

**`docker0` 자체가 Host 안에 있다.** 기본 NAT 구성에서는 외부로 나갈 때 보통 출발지 컨테이너 IP가 호스트 주소로 변환된다. 같은 bridge의 컨테이너끼리는 물리 NIC나 외부 라우터를 거칠 필요가 없다.

```text
같은 네트워크 내부:
A eth0 <== veth pair ==> Host bridge <== veth pair ==> B eth0

사용자 정의 bridge:
A eth0 <== veth pair ==> br-<network ID> -> Host routing -> 외부
```

사용자 정의 bridge 트래픽이 반드시 `docker0`도 통과하는 것은 아니다. 외부에서 시작한 접속을 컨테이너에 전달하는 `-p`는 [Day02의 내부 통신과 외부 접근](../Day02-Docker-Compose-Network-Volume/README.md#6-내부-통신과-외부-접근)의 포트 매핑과 연결해서 기억한다. [Bridge 구조 참고](https://docs.docker.com/engine/network/drivers/bridge/)

## 5. host: Host network namespace 공유

```text
+-- Host network namespace --------------------+
| Host process         :9000                   |
| Container process    :80                     |
| 같은 인터페이스 / IP / localhost / 포트 공간  |
+----------------------------------------------+
```

```bash
# Linux Host의 TCP 80번 포트가 비어 있을 때
docker run -d --name day03-host-web --network host nginx:alpine
curl http://127.0.0.1:80
docker rm -f day03-host-web
```

프로그램이 호스트의 네트워크 공간에서 직접 수신하므로 `-p 8080:80`을 추가해도 8080으로 변환되지 않는다. host 모드에서 포트 공개 옵션은 무시되며 경고가 표시된다. 같은 주소·프로토콜·포트에 이미 서버가 수신 중이면 충돌할 수 있다.

공유되는 것은 네트워크 공간이며 파일시스템이나 PID namespace까지 자동으로 공유하는 것은 아니다. Docker Desktop은 **4.34 이상에서 별도로 활성화하는 host networking 기능**을 제공하며, Linux Engine과 구현·지원 범위가 다르다. [Host 공식 문서와 제한 사항](https://docs.docker.com/engine/network/drivers/host/)

## 6. none: 네트워크 없이 실행하기

```bash
docker run --rm --network none alpine:3.23 ip addr
```

기본적으로 `lo`와 IPv4 loopback `127.0.0.1`만 보이고, 외부 통신용 `eth0`나 경로가 없다. [None 공식 문서](https://docs.docker.com/engine/network/drivers/none/)

학습 개념을 적용하면, 모델·라이브러리·입력 데이터를 미리 준비한 **오프라인 inference, 데이터 전처리, 파일 변환**에 활용할 수 있다.

```text
Host model / input -- mount --> [ Container: offline inference ]
Host output        <-- mount -- [ lo만 존재, 외부 API 연결 불가 ]
```

아래는 직접 만든 이미지와 파일이 있어야 하는 응용 예시다. `/srv/day03/model`, `input`, `output` 디렉터리를 미리 만들고 모델·입력 파일을 준비한다. 여러 줄 명령은 Bash 기준이다.

```bash
docker run --rm --network none \
  --mount type=bind,src=/srv/day03/model,dst=/model,readonly \
  --mount type=bind,src=/srv/day03/input,dst=/input,readonly \
  --mount type=bind,src=/srv/day03/output,dst=/output \
  my-offline-inference:local python /app/infer.py
```

`my-offline-inference:local`에는 Python·의존성·`/app/infer.py`가 포함되어 있고, 스크립트가 위 경로를 사용한다고 가정한다. 실행 중 모델 다운로드·패키지 설치·외부 API 호출은 할 수 없다. 마운트한 파일은 계속 접근할 수 있다. 또한 `--network none`은 **컨테이너의 실행 네트워크** 설정이므로 Docker 데몬이 실행 전에 이미지를 내려받는 것까지 차단하지는 않는다.

## 7. container: 다른 컨테이너의 네트워크 공유

```bash
docker run -d --name day03-shared-web nginx:alpine
docker run --rm --network container:day03-shared-web alpine:3.23 wget -qO- http://127.0.0.1:80
docker rm -f day03-shared-web
```

Nginx가 준비된 뒤 조회한다. 두 번째 컨테이너의 `127.0.0.1:80`으로 첫 번째 컨테이너의 Nginx에 접근한다.

```text
+-- day03-shared-web의 network namespace --------+
| nginx process -> :80                           |
| alpine wget   -> 127.0.0.1:80 -> nginx          |
| 공유: IP / interfaces / routes / port space    |
+-----------------------------------------------+
  파일시스템과 프로세스 공간은 기본적으로 각각 분리
```

두 컨테이너가 같은 bridge에 있는 것보다 더 강한 공유다. 별도 네트워크 IP를 받지 않고 포트 공간도 공유하므로 같은 포트 충돌이 가능하다. 보조 컨테이너의 `-p`, `--dns` 등은 지원되지 않으며, 포트 공개가 필요하면 네트워크를 제공하는 원래 컨테이너에 설정한다. [Container network mode](https://docs.docker.com/engine/network/#container-networks)

## 8. overlay: 여러 Docker Host를 연결

```text
Host A / Docker daemon A       Host B / Docker daemon B
  [Container A]                  [Container B]
        |                              |
        +====== overlay network =======+
              호스트 간 실제 네트워크 위에 구성
```

bridge는 한 Docker Host 범위, overlay는 여러 Docker Host에 걸친 연결에 사용한다. Docker의 overlay는 **Swarm 모드가 필요**하며, 일반 컨테이너를 연결하려면 `--attachable`을 사용한다.

```bash
# 이미 Swarm을 구성한 환경의 manager에서 실행
docker network create --driver overlay --attachable day03-overlay
docker run -d --name day03-overlay-a --network day03-overlay alpine:3.23 sleep 3600
```

다른 호스트도 같은 Swarm에 참여해야 한다. 단순히 네트워크 이름만 같게 만드는 것으로 연결되지 않는다. 다중 호스트 실습에는 노드 간 방화벽 설정 등이 추가로 필요하다. [Overlay 구성 조건](https://docs.docker.com/engine/network/drivers/overlay/)

## 9. Docker embedded DNS: 127.0.0.11

사용자 정의 네트워크의 컨테이너는 **Docker 내장 DNS**를 통해 같은 네트워크의 컨테이너 이름·서비스 이름·alias를 IP로 해석한다. 이 DNS의 주소가 `127.0.0.11`이다. 일반적인 기본 `bridge`는 컨테이너 이름 자동 DNS를 제공하지 않는다.

```text
backend: "db의 IP는?"
      |
      v
Docker embedded DNS (127.0.0.11)
      |
      +-- db -> 172.18.0.3 (예시)
      |
      v
backend가 172.18.0.3:5432로 직접 연결
```

DNS는 주소를 알려주며 DB 요청 자체를 중계하지 않는다. `127.0.0.11`은 컨테이너에서 사용하는 resolver 주소로, 호스트의 DB 주소가 아니다. 외부 도메인 질의는 설정된 상위 DNS로 전달한다. [DNS 공식 설명](https://docs.docker.com/engine/network/#dns-services)

## 10. --network-alias(--net-alias): 네트워크 안에서 쓰는 별명

`--network-alias`는 컨테이너에 **해당 네트워크 범위의 추가 DNS 이름**을 붙인다. `--net-alias`도 같은 옵션의 별칭이며, 예제에서는 공식 문서에 표시되는 `--network-alias`를 사용한다.

```bash
docker network create day03-alias-net
docker run -d --name day03-gpt-1 --network day03-alias-net --network-alias gpt-server alpine:3.23 sleep 3600
docker run -d --name day03-gpt-2 --network day03-alias-net --network-alias gpt-server alpine:3.23 sleep 3600

# 같은 네트워크에서 DNS 설정과 공통 별명 조회
docker run --rm --network day03-alias-net alpine:3.23 cat /etc/resolv.conf
docker run --rm --network day03-alias-net alpine:3.23 nslookup gpt-server 127.0.0.11
docker network inspect day03-alias-net
```

`resolv.conf`의 `nameserver 127.0.0.11`과 조회 결과의 컨테이너 IP들을 확인한다. 위 Alpine 컨테이너는 **DNS 실습용**이며 실제 GPT 모델이나 8000번 API를 실행하지 않는다.

이미 존재하는 컨테이너를 새 네트워크에 연결할 때는 `docker network connect --alias`를 사용한다. 예를 들어 위 실습을 유지한 상태에서:

```bash
docker network create day03-alias-extra
docker network connect --alias inference day03-alias-extra day03-gpt-1
```

이제 `day03-gpt-1`은 원래 네트워크에서는 `gpt-server`, 추가 네트워크에서는 `inference`로도 조회된다. [Run 옵션](https://docs.docker.com/reference/cli/docker/container/run/) · [Connect alias 옵션](https://docs.docker.com/reference/cli/docker/network/connect/#create-a-network-alias-for-a-container---alias)

## 11. 같은 alias에 여러 IP: DNS round-robin 성격

```text
동일한 day03-alias-net 안의 DNS 등록 (IP는 예시)

                     gpt-server
                          |
                Docker embedded DNS
                   /             \
          172.18.0.2              172.18.0.3
         day03-gpt-1             day03-gpt-2
```

별명은 네트워크 안에서 여러 컨테이너가 공유할 수 있다. 따라서 하나의 이름 질의에 여러 A 레코드(IPv4 주소)가 반환될 수 있다. DNS round-robin 성격으로 설명하지만, **정확히 1번·2번 순서대로 번갈아 요청을 처리한다는 보장은 없다.** 현재 Moby resolver 구현은 응답 주소 순서를 섞으며, 실제 선택은 클라이언트의 DNS 처리·캐시·연결 재사용 방식에도 달려 있다. [Alias 공유 규칙](https://docs.docker.com/reference/compose-file/services/#aliases) · [Moby DNS 구현](https://github.com/moby/moby/blob/master/daemon/libnetwork/resolver.go)

실제 GPT API가 각 컨테이너의 8000번에서 실행 중이라면 호출자는 `gpt-server:8000`이라는 이름을 사용할 수 있다. 그렇다고 사용자마다 컨테이너 하나를 독점 배정하거나, GPU 부하·모델 준비 상태를 보고 가장 한가한 인스턴스를 고르는 것은 아니다.

```text
요청 단위 분산이 필요한 구성 예시

Client -> Load balancer / inference router
                       |
              +--------+--------+
              |                 |
          GPT API 1          GPT API 2
```

건강 상태 확인·재시도·분배 정책은 별도로 구성한다. GPU 사용량이나 대기열 기반 선택은 그 지표를 이해하는 라우터와 설정이 필요하다. DNS alias만으로 애플리케이션 장애를 감지하거나 균등 분배할 수 있다고 가정하지 않는다.

위 alias 실습을 모두 마쳤다면 정리한다.

```bash
docker rm -f day03-gpt-1 day03-gpt-2
docker network rm day03-alias-net day03-alias-extra
```

## 12. container_name과 network alias 차이

| 구분 | `container_name` / `docker run --name` | Network alias |
| --- | --- | --- |
| 목적 | Docker가 관리하는 컨테이너 자체의 이름 | 특정 네트워크에서 사용할 추가 DNS 이름 |
| 범위 | 같은 Docker daemon에서 고유 | 네트워크별로 설정 |
| 동일 이름을 여러 컨테이너에 지정 | 불가 | 가능, 여러 IP로 해석될 수 있음 |
| 컨테이너당 개수 | 컨테이너 이름 하나 | 별명 여러 개 가능 |
| 사용 예 | `docker logs my-backend` | 같은 네트워크에서 `inference:8000`으로 접속 |

컨테이너 이름도 사용자 정의 네트워크에서는 DNS 조회에 사용할 수 있다. 다만 alias를 붙여도 Docker 관리 이름이 바뀌는 것은 아니므로 `docker logs <alias>`로 컨테이너를 지정할 수는 없다.

Compose의 `container_name`을 고정하면 그 서비스를 여러 컨테이너로 확장할 수 없다. 서비스 이름으로 통신하는 데 `container_name`은 필요하지 않다. [Compose container_name](https://docs.docker.com/reference/compose-file/services/#container_name)

## 13. Compose에서는 서비스 이름 DNS부터 사용

아래는 별도 폴더의 `compose.yaml`로 사용할 수 있는 DNS 실습 예시다.

```yaml
services:
  client:
    image: alpine:3.23
    command: ["sleep", "3600"]
  backend:
    image: nginx:alpine
    networks:
      default:
        aliases:
          - api
```

```bash
docker compose -p day03-dns up -d
docker compose -p day03-dns exec client nslookup backend
docker compose -p day03-dns exec client nslookup api
docker compose -p day03-dns exec client wget -qO- http://backend:80
docker compose -p day03-dns down
```

Nginx가 준비되면 `backend:80`으로 기본 페이지를 조회할 수 있다. `backend`는 기본 서비스 이름이고 `api`는 추가 별명이다. `networks` 설정을 생략해도 **서비스 이름 `backend`로 통신하는 기능은 기본 제공**된다. [Compose 네트워크 공식 문서](https://docs.docker.com/compose/how-tos/networking/)

따라서 Compose 중심 작업에서는 직접 `--net-alias`를 입력할 일이 상대적으로 적다. 이는 기능을 거의 사용하지 않는다는 통계가 아니라, 기본 서비스 이름으로 대부분의 내부 연결을 표현할 수 있다는 실무 관점이다. 다른 시스템이 기대하는 이름을 유지하거나 네트워크마다 다른 이름을 제공할 때 추가 alias를 고려한다.

## 14. Macvlan: LAN에서 독립된 기기처럼 접근하기

### 14.1 먼저 LAN과 IP부터 이해하기

**LAN(Local Area Network)**은 집이나 사무실처럼 가까운 범위의 기기들을 연결한 네트워크다. 집에서 같은 공유기에 연결된 노트북과 휴대폰을 떠올리면 된다. **IP 주소**는 네트워크에서 통신할 대상을 찾는 주소다.

```text
집 공유기: 192.168.0.1
    |
    +-- 노트북: 192.168.0.20
    +-- 휴대폰: 192.168.0.21
    +-- Docker 서버: 192.168.0.10
```

위 주소처럼 해당 LAN 안에서 사용하는 IP를 여기서는 **LAN IP**라고 부른다. IP는 공유기가 자동으로 할당할 수도 있고, 충돌하지 않도록 직접 설정할 수도 있다. LAN IP는 인터넷 전체에서 바로 접근할 수 있는 공인 IP라는 뜻이 아니다.

### 14.2 기존 컨테이너도 IP를 받는데 무엇이 다를까?

**bridge 컨테이너의 IP도 실제 통신에 쓰이는 주소다.** 가짜 주소와 진짜 주소의 차이가 아니라, **어느 네트워크에 속하고 다른 기기에서 어떤 경로로 접근하느냐**의 차이다.

일반적인 Docker bridge의 기본 설정을 보면:

```text
집 LAN
    +-- 노트북: 192.168.0.20
    |
    +-- Docker 서버: 192.168.0.10
            |
            +-- Docker 내부 bridge 네트워크
                    +-- 컨테이너: 172.18.0.2:80

노트북 -> 서버 192.168.0.10:8080 -> 포트 매핑 -> 컨테이너 172.18.0.2:80
```

같은 bridge의 컨테이너끼리는 내부 IP로 통신하지만, LAN의 다른 PC는 기본 설정에서 그 내부 IP로 바로 접근할 수 있도록 구성되어 있지 않다. 따라서 보통 서버 IP와 공개한 포트를 이용한다. 별도 라우팅·방화벽 설정으로 bridge IP 직접 접근을 구성할 수도 있으므로, bridge IP가 원천적으로 접근 불가능한 주소라는 뜻은 아니다. [Docker 포트 공개와 직접 라우팅](https://docs.docker.com/engine/network/port-publishing/)

Macvlan은 호스트의 물리 랜카드를 통해 **컨테이너가 LAN에 별도 기기처럼 나타나게 한다.** 컨테이너 네트워크 인터페이스마다 별도의 **MAC 주소**를 사용한다. MAC은 같은 LAN에서 통신할 때 쓰는 인터페이스의 식별 주소이며, 여기서 Mac은 Apple 컴퓨터를 뜻하지 않는다.

```text
집 LAN에서 보이는 모습
    +-- 노트북:       192.168.0.20
    +-- Docker 서버:  192.168.0.10
    +-- 컨테이너 A:   192.168.0.201:80  (별도 MAC A)
    +-- 컨테이너 B:   192.168.0.202:80  (별도 MAC B)

실제 실행 위치와 연결
Docker 서버의 물리 랜카드
    +-- Macvlan 인터페이스 A -- 컨테이너 A
    +-- Macvlan 인터페이스 B -- 컨테이너 B

노트북 -> 컨테이너 A 192.168.0.201:80
```

컨테이너는 여전히 Docker 서버 안에서 실행되지만, 같은 LAN의 다른 PC는 컨테이너 IP로 직접 접근할 수 있다. 이 경로에는 별도 `-p`가 필요하지 않다. 실제 서비스가 해당 주소·포트에서 수신 중이어야 하고 네트워크 정책도 통신을 허용해야 한다. [Macvlan 공식 문서](https://docs.docker.com/engine/network/drivers/macvlan/)

| 방식 | 다른 PC에서 접근하는 주소 예시 | 컨테이너의 네트워크 공간 |
| --- | --- | --- |
| bridge + 포트 매핑 | 서버 `192.168.0.10:8080` | Docker 내부 네트워크에 별도 IP |
| host | 서버 `192.168.0.10:80` | Host network namespace 공유 |
| macvlan | 컨테이너 `192.168.0.201:80` | 별도 namespace와 LAN에서 사용할 IP·MAC |

`172`로 시작해서 가상이고 `192`로 시작해서 실제인 것은 아니다. **주소 숫자만으로 bridge와 Macvlan을 구분하지 않는다.** 위 IP들은 설명용 예시다.

### 14.3 생성 명령 예시

아래는 **Linux rootful Docker Engine과 유선 LAN**을 가정한 예시다. 실제 환경의 주소와 랜카드 이름으로 바꿔야 한다.

- `parent=eth0`: Docker 서버에서 LAN에 연결된 물리 랜카드 이름. 실제 이름은 `ip addr`로 확인한다.
- `--subnet`: 사용할 네트워크 주소 범위. 여기서는 `192.168.0.0/24` LAN을 가정한다.
- `--gateway`: 다른 네트워크로 나갈 때 사용하는 출구 주소. 여기서는 공유기 `192.168.0.1`이다.
- `--ip-range`: Docker가 컨테이너에 할당할 범위. 예제의 `192.168.0.200/29` 범위는 **공유기 DHCP 자동 할당에서 제외하고 기존 장비가 사용하지 않도록 미리 확보**했다고 가정한다. Docker가 공유기의 DHCP와 자동으로 조정하지는 않는다.

```bash
docker network create -d macvlan \
  --subnet=192.168.0.0/24 \
  --gateway=192.168.0.1 \
  --ip-range=192.168.0.200/29 \
  -o parent=eth0 \
  day03-macvlan

docker run -d --name day03-macvlan-web \
  --network day03-macvlan \
  --ip 192.168.0.201 \
  nginx:alpine

docker network inspect day03-macvlan
```

Nginx가 준비되면 **Docker 서버 자신이 아닌 같은 LAN의 다른 PC**에서 확인한다.

```bash
curl http://192.168.0.201:80
```

실습 자원 정리는 Docker 서버에서 실행한다.

```bash
docker rm -f day03-macvlan-web
docker network rm day03-macvlan
```

### 14.4 활용과 제한

컨테이너가 LAN의 독립 장비처럼 보여야 하는 기존 서비스나 네트워크 도구에 활용한다. 일반적인 Backend·DB 내부 통신에는 사용자 정의 bridge를 먼저 고려할 수 있다.

- **Host와 Macvlan 컨테이너는 기본적으로 직접 통신하지 못한다.** Linux 커널의 제한이다. Host에도 Macvlan 인터페이스를 만들거나 컨테이너에 bridge 연결을 추가하는 등 별도 구성이 필요하다.
- 스위치 등 네트워크 장비가 물리 랜카드 하나 뒤의 여러 MAC 주소를 허용해야 한다.
- Docker Desktop의 Windows/macOS, Windows Docker Engine, rootless 모드에서는 지원하지 않는다. 대부분의 클라우드 제공 환경에서도 제한된다.
- Macvlan의 기본 모드 이름도 `bridge`지만, 앞서 배운 Docker `bridge` 드라이버와 같은 구성이라는 뜻은 아니다.

지원 조건과 Host 통신 제한은 [Macvlan 공식 문서](https://docs.docker.com/engine/network/drivers/macvlan/)를 참고한다. 이 예제는 환경별 설정이 필요한 학습용 명령이며 실제 실행 검증은 하지 않았다.

### 14.5 헷갈렸던 부분 복습

1. LAN IP는 인터넷 어디서나 접속 가능한 공인 IP일까?
2. bridge IP는 가짜이고 Macvlan IP만 진짜일까?
3. Macvlan은 host 모드처럼 Host의 네트워크 공간을 공유할까?
4. Macvlan 웹 서버에 접속하는 실습을 왜 다른 PC에서 할까?

<details>
<summary>정답 확인</summary>

1. 아니다. 여기서는 집·회사 내부 LAN에서 쓰는 주소라는 뜻이다.
2. 둘 다 실제 통신에 쓰는 주소다. 속한 네트워크와 접근 경로가 다르다.
3. 아니다. Macvlan 컨테이너는 별도 네트워크 공간과 IP·MAC을 사용한다.
4. Host와 Macvlan 컨테이너는 기본적으로 직접 통신하지 못하기 때문이다.

</details>

## 핵심 복습

| 개념 | 기억할 내용 |
| --- | --- |
| bridge | 한 호스트의 독립된 컨테이너 네트워크 공간을 가상 bridge로 연결 |
| 사용자 정의 bridge | 이름 기반 DNS와 애플리케이션별 네트워크 구성 |
| host | Host network namespace 공유, `-p` 불필요, 포트 충돌 가능 |
| none | loopback만 있는 격리 환경, 준비된 파일로 오프라인 작업 |
| container 모드 | 다른 컨테이너의 IP·localhost·포트 공간 공유 |
| overlay | Swarm의 여러 Docker Host를 논리 네트워크로 연결 |
| macvlan | 컨테이너에 별도 MAC과 LAN IP를 구성해 다른 LAN 기기에서 직접 접근 |
| bridge IP와 LAN IP | 가짜·진짜의 차이가 아니라 속한 네트워크와 접근 경로의 차이 |
| Embedded DNS | 사용자 정의 네트워크의 `127.0.0.11` resolver |
| Network alias | 네트워크 범위의 추가 이름, 여러 IP 가능, 요청 분배 보장 없음 |
| Compose 서비스 이름 | 기본 내부 DNS 이름이므로 별도 alias나 container_name은 필수가 아님 |

### 네트워크 추가 복습

1. 기본 `bridge`와 사용자 정의 bridge는 이름 해석에서 어떻게 다를까?
2. `docker network connect`는 기존 네트워크를 교체할까?
3. 사용자 정의 bridge의 패킷도 반드시 `docker0`를 통과할까?
4. host 모드에서 `-p 8080:80`을 붙이면 8080으로 접속할 수 있을까?
5. none 모드에서 미리 마운트한 모델로 추론할 수 있을까? 모델 다운로드는 어떨까?
6. 같은 bridge 연결과 `--network container:...`는 localhost 관점에서 어떻게 다를까?
7. `127.0.0.11`은 DB 주소일까, DNS resolver 주소일까?
8. 같은 alias의 GPT 컨테이너 두 개는 요청을 항상 절반씩 처리할까?
9. `container_name` 없이도 Compose 서비스 이름으로 통신할 수 있을까?
10. 여러 Docker Host를 overlay로 연결하려면 어떤 조건이 필요할까?

<details>
<summary>네트워크 정답 확인</summary>

1. 기본 bridge는 컨테이너 이름 자동 DNS를 제공하지 않고, 사용자 정의 bridge는 같은 네트워크의 이름·alias를 해석한다.
2. 연결을 추가한다. 기존 연결 제거는 `disconnect`로 따로 수행한다.
3. 아니다. 보통 해당 네트워크의 `br-<network ID>`를 사용한다.
4. 아니다. host 모드에서는 `-p`가 무시된다. 프로그램의 실제 수신 포트를 사용한다.
5. 코드·의존성·모델·입력이 준비되어 있으면 가능하다. 컨테이너 실행 중 네트워크 다운로드는 불가능하다.
6. 같은 bridge의 컨테이너들은 각자의 localhost를 갖지만 container 모드는 localhost와 포트 공간까지 공유한다.
7. Docker embedded DNS resolver 주소다. DB는 이름 해석 결과의 IP와 DB 포트로 접속한다.
8. 아니다. DNS 응답 순서·클라이언트 캐시·연결 재사용 등의 영향을 받으며 부하 기반 분배가 아니다.
9. 가능하다. 기본 서비스 이름 DNS가 제공된다. 고정 `container_name`은 여러 복제본으로 확장하는 데 제약이 된다.
10. 같은 Swarm에 참여한 호스트와 노드 간 통신 설정이 필요하다. 일반 컨테이너 연결에는 `--attachable`도 필요하다.

</details>

## 공식 참고 자료

- [Docker 네트워크와 DNS](https://docs.docker.com/engine/network/)
- [Bridge](https://docs.docker.com/engine/network/drivers/bridge/), [Host](https://docs.docker.com/engine/network/drivers/host/), [None](https://docs.docker.com/engine/network/drivers/none/), [Overlay](https://docs.docker.com/engine/network/drivers/overlay/)
- [Network create](https://docs.docker.com/reference/cli/docker/network/create/), [connect](https://docs.docker.com/reference/cli/docker/network/connect/), [disconnect](https://docs.docker.com/reference/cli/docker/network/disconnect/)
- [Compose 네트워크](https://docs.docker.com/compose/how-tos/networking/), [aliases](https://docs.docker.com/reference/compose-file/services/#aliases), [container_name](https://docs.docker.com/reference/compose-file/services/#container_name)
- [Docker run](https://docs.docker.com/reference/cli/docker/container/run/)
- [Docker CLI의 --net-alias 정의](https://github.com/docker/cli/blob/master/cli/command/container/opts.go)
- [Moby DNS resolver 구현](https://github.com/moby/moby/blob/master/daemon/libnetwork/resolver.go)
