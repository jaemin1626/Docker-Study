
# Docker 기반 AI 학습 서버 아키텍처 및 네트워크 통신

Docker와 Kubernetes를 활용한 다중 서버 인프라의 네트워크 통신, API 요청, 로드밸런싱, AI 학습 작업 분배 및 데이터셋 공유 구조를 정리한 학습 문서입니다.

## 목차

1. [네트워크 통신의 기본 개념](#1-네트워크-통신의-기본-개념)
2. [Docker 네트워크와 컨테이너 간 통신](#2-docker-네트워크와-컨테이너-간-통신)
3. [Nginx 기반 로드밸런싱](#3-nginx-기반-로드밸런싱)
4. [다중 서버 환경에서의 API 통신](#4-다중-서버-환경에서의-api-통신)
5. [Kubernetes 기반 AI 학습 서버](#5-kubernetes-기반-ai-학습-서버)
6. [GPU 학습 작업 분배](#6-gpu-학습-작업-분배)
7. [NFS 기반 데이터셋 공유](#7-nfs-기반-데이터셋-공유)
8. [Docker 운영 명령어](#8-docker-운영-명령어)
9. [아키텍처 선택 기준](#9-아키텍처-선택-기준)

---

## 1. 네트워크 통신의 기본 개념

### 1.1 IP와 Port

IP는 통신 대상 서버를 식별하고, Port는 해당 서버에서 실행되는 서비스를 구분합니다.

예를 들어 다음 주소는 다음과 같은 의미를 가집니다.

```text
http://192.168.0.10:8000/train

192.168.0.10  → 서버 IP
8000          → 서비스 Port
/train        → API Endpoint
```

### 1.2 IP Binding

IP Binding은 서버 애플리케이션이 어떤 로컬 네트워크 인터페이스에서 요청을 수신할지 지정하는 설정입니다.

| Binding IP | 의미 |
|---|---|
| 127.0.0.1 | 서버 내부에서만 접근 |
| 192.168.0.10 | 해당 로컬 IP에서 요청 수신 |
| 0.0.0.0 | 모든 IPv4 네트워크 인터페이스에서 요청 수신 |

FastAPI 예시:

```python
import uvicorn

uvicorn.run(
    "main:app",
    host="0.0.0.0",
    port=8000
)
```

`0.0.0.0`은 모든 클라이언트의 접근을 허가한다는 의미가 아닙니다. 실제 외부 접근 여부는 방화벽, 네트워크 경로, 포트 공개 및 인증 정책에 따라 달라집니다.

특정 클라이언트 IP만 허용하려면 방화벽, Reverse Proxy, API 인증 등의 별도 접근 제어가 필요합니다.

### 1.3 Docker Port Mapping

Docker Port Mapping은 호스트의 포트와 컨테이너 내부 포트를 연결하는 기능입니다.

```bash
docker run -d -p 8080:8000 my-api
```

```text
Client
   |
   | HTTP Request
   v
Host IP:8080
   |
   | Docker Port Mapping
   v
Container:8000
   |
   v
API Application
```

포트 매핑 형식:

```text
-p HOST_PORT:CONTAINER_PORT
```

로컬 접근만 허용하려면 다음과 같이 설정할 수 있습니다.

```bash
docker run -d -p 127.0.0.1:8080:8000 my-api
```

### 1.4 API Request

API 요청은 클라이언트가 서버의 IP, Port, Endpoint로 요청을 보내고 응답을 받는 통신 방식입니다.

```python
import requests

response = requests.post(
    "http://192.168.0.10:8080/train",
    json={
        "dataset": "dataset_v1",
        "epochs": 100
    },
    timeout=30
)

print(response.json())
```

요청을 보내는 클라이언트는 별도의 수신 포트를 직접 바인딩할 필요가 없습니다. 일반적인 HTTP 클라이언트는 운영체제가 할당하는 임시 포트를 이용해 연결을 생성하고 응답을 받습니다.

**핵심 차이**

- IP Binding: 서버가 요청을 수신할 로컬 주소 지정
- Port Mapping: 호스트 포트와 컨테이너 포트 연결
- API Request: 클라이언트가 서버에 요청 전송

---

## 2. Docker 네트워크와 컨테이너 간 통신

Docker는 컨테이너마다 격리된 네트워크 환경을 제공합니다.

### 2.1 Docker Bridge Network

동일한 사용자 정의 Docker Bridge Network에 연결된 컨테이너는 서로 통신할 수 있습니다.

```text
           Docker Host
                |
       Docker Bridge Network
                |
       +--------+--------+
       |        |        |
       v        v        v
    API       Redis     Database
    :8000     :6379      :5432
```

사용자 정의 네트워크에서는 Docker DNS를 통해 서비스 이름으로 통신할 수 있습니다.

예시:

```text
http://api:8000
redis:6379
database:5432
```

### 2.2 Docker Compose Network

Docker Compose는 기본적으로 동일 프로젝트의 서비스들을 하나의 default 네트워크에 연결합니다.

```yaml
services:
  api:
    image: my-api
    ports:
      - "8080:8000"

  redis:
    image: redis:7
```

API 컨테이너는 Redis에 다음 주소로 접근할 수 있습니다.

```text
redis:6379
```

Redis에 외부 포트 매핑을 설정하지 않아도 같은 Docker 네트워크 안에서 통신할 수 있습니다.

### 2.3 서로 다른 물리 서버의 Docker 통신

서로 다른 물리 서버에서 실행되는 Docker 컨테이너는 기본 Bridge Network만으로 직접 연결되지 않습니다.

예시:

```text
[MLOps Server]                     [Train Server]
192.168.0.10                       192.168.0.20

Docker Container                   Docker Container
MLOps API                          Train API
      |                                 ^
      |                                 |
      +---- HTTP Request :8000 --------+
```

일반적인 통신 방법:

1. 학습 서버 컨테이너에서 API 서버 실행
2. Docker Port Mapping으로 API 포트 공개
3. MLOps 서버에서 학습 서버의 호스트 IP와 Port로 요청
4. 학습 서버에서 요청 검증 후 작업 실행
5. 작업 상태 및 결과 반환

서로 다른 서버는 미리 Docker 네트워크에 연결되어 있어야만 HTTP 통신이 가능한 것은 아닙니다.

서버 간 IP 라우팅과 방화벽 정책이 허용되면 일반적인 네트워크 통신이 가능합니다.

---

## 3. Nginx 기반 로드밸런싱

### 3.1 로드밸런싱 개념

로드밸런싱은 여러 서버에 요청을 분산하여 특정 서버에 트래픽이 집중되지 않도록 하는 기술입니다.

```text
                   Client
                      |
                      v
                 Nginx :80
                      |
        +-------------+-------------+
        |             |             |
        v             v             v
     Server1       Server2       Server3
      :8000         :8000         :8000
```

사용자는 개별 서버에 접근하지 않고 Nginx의 대표 주소로 요청을 보냅니다.

### 3.2 Nginx 설정 예시

```nginx
upstream backend {
    server server1:8000;
    server server2:8000;
    server server3:8000;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
    }
}
```

기본적인 Nginx upstream 로드밸런싱은 Round Robin 방식으로 요청을 분산합니다.

### 3.3 Docker Compose 예시

```yaml
services:
  nginx:
    image: nginx:stable
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - server1
      - server2
      - server3

  server1:
    image: my-api:latest

  server2:
    image: my-api:latest

  server3:
    image: my-api:latest
```

백엔드 컨테이너는 외부 포트 매핑 없이 Nginx와 내부 네트워크를 통해 통신합니다.

단, 기본 Bridge 기반 Compose 네트워크는 동일한 Docker 호스트 범위에서 동작합니다.

서로 다른 물리 서버에 위치한 백엔드를 연결하려면 실제 서버 IP, DNS, 라우팅 및 방화벽 구성이 필요합니다.

### 3.4 AI 학습 서버에서의 한계

일반적인 HTTP 로드밸런싱은 요청 분산에 초점을 맞춥니다.

그러나 AI 학습 작업은 다음 특징이 있습니다.

- 작업 실행 시간이 길 수 있음
- GPU 메모리를 지속적으로 점유함
- 서버별 GPU 성능이 다를 수 있음
- 하나의 작업이 장시간 실행될 수 있음
- 학습 중인 서버에 추가 작업을 할당하면 OOM이 발생할 수 있음

따라서 AI 학습 작업은 단순 Round Robin보다 GPU 자원을 고려한 작업 스케줄링이 적합합니다.

---

## 4. 다중 서버 환경에서의 API 통신

### 4.1 MLOps 서버와 학습 서버 분리

MLOps 서버와 학습 서버의 역할을 분리하면 서버별 자원을 독립적으로 관리할 수 있습니다.

```text
[User]
   |
   v
[MLOps Server]
   |
   | POST /train
   v
[Train Server API]
   |
   | Validate Request
   v
[GPU Training Process]
   |
   | Save Result
   v
[Dataset / Model Storage]
```

MLOps 서버의 역할:

- 사용자 요청 관리
- 학습 작업 생성
- 작업 상태 조회
- 모델 및 데이터셋 관리
- 학습 결과 제공

학습 서버의 역할:

- 학습 API 수신
- 요청 및 권한 검증
- GPU 자원 확인
- 학습 프로세스 실행
- 체크포인트 및 결과 저장
- 상태 및 결과 보고

### 4.2 학습 서버 API 예시

FastAPI를 이용한 단순 요청 수신 예시입니다.

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class TrainRequest(BaseModel):
    dataset: str
    epochs: int

@app.post("/train")
def train(request: TrainRequest):
    return {
        "status": "accepted",
        "dataset": request.dataset,
        "epochs": request.epochs
    }
```

위 코드는 요청 수신 예시입니다. 실제 학습을 수행하거나 작업을 큐에 저장하지는 않습니다.

실제 학습 시스템에서는 요청 검증, 비동기 작업 실행, Job ID 생성, 상태 저장, 실패 복구 등의 기능이 필요합니다.

### 4.3 MLOps 서버 요청 예시

```python
import requests

TRAIN_SERVER = "http://192.168.0.20:8000"

response = requests.post(
    f"{TRAIN_SERVER}/train",
    json={
        "dataset": "dataset_v1",
        "epochs": 100
    },
    timeout=10
)

response.raise_for_status()
print(response.json())
```

### 4.4 IP 기반 접근 제한

학습 서버 API를 특정 MLOps 서버에서만 호출하도록 제한할 수 있습니다.

권장 방법은 다음과 같습니다.

- 방화벽에서 MLOps 서버 IP만 허용
- 내부 전용 네트워크 사용
- API 토큰 또는 서비스 간 인증 적용
- 필요한 경우 TLS 적용

특히 Docker Published Port는 외부 네트워크에 노출될 수 있으므로 호스트 방화벽과 Docker 네트워크 규칙을 함께 확인해야 합니다.

---

## 5. Kubernetes 기반 AI 학습 서버

### 5.1 Kubernetes 개념

Kubernetes는 여러 물리 서버 또는 가상 서버에 분산된 컨테이너 워크로드를 관리하는 플랫폼입니다.

Docker Compose가 주로 단일 Docker 호스트의 다중 컨테이너 구성을 관리한다면, Kubernetes는 여러 Node에 걸친 워크로드 관리 기능을 제공합니다.

### 5.2 주요 구성 요소

| 구성 요소 | 역할 |
|---|---|
| Cluster | Kubernetes 전체 환경 |
| Node | 실제 워크로드가 실행되는 서버 |
| Pod | 컨테이너 실행의 기본 단위 |
| Deployment | 지속 실행 서비스의 복제 및 배포 관리 |
| Service | Pod 집합에 대한 네트워크 접근 제공 |
| Job | 완료형 작업 실행 |
| Scheduler | Pod 실행에 적합한 Node 선택 |
| PersistentVolume | 스토리지 자원 |
| PersistentVolumeClaim | 스토리지 사용 요청 |

### 5.3 Kubernetes 구조

```text
             Kubernetes Cluster
                      |
               Control Plane
                      |
                 Scheduler
                      |
        +-------------+-------------+
        |             |             |
        v             v             v
      Node1         Node2         Node3
        |             |             |
       Pod1          Pod2          Pod3
        |             |             |
      GPU 0         GPU 0         GPU 0
```

### 5.4 Kubernetes Service

Service는 여러 Pod에 접근할 수 있는 안정적인 네트워크 엔드포인트를 제공합니다.

```text
Client
   |
   v
Service
   |
   +--------+--------+
   |        |        |
   v        v        v
  Pod1     Pod2     Pod3
```

Service가 여러 Pod에 트래픽을 분배할 수 있지만, 반드시 엄격한 순차 Round Robin을 보장하는 것은 아닙니다.

또한 Service 자체가 GPU 점유율이나 AI 학습 작업의 상태를 이해하여 작업을 분배하는 것은 아닙니다.

### 5.5 Readiness Probe

Readiness Probe는 Pod가 요청을 받을 준비가 되었는지를 확인하는 기능입니다.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8000
  periodSeconds: 5
```

주의 사항:

- Kubernetes가 GPU 사용량을 자동으로 해석하지 않음
- 학습 서버가 현재 작업 상태를 별도로 관리해야 함
- /ready 엔드포인트에서 준비 상태를 반환해야 함
- 긴 학습 작업의 스케줄링을 Readiness Probe에만 의존하는 것은 적합하지 않음

### 5.6 Kubernetes Job

AI 학습처럼 시작과 종료가 명확한 작업은 Kubernetes Job을 활용할 수 있습니다.

예시:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ai-training
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: trainer
          image: my-trainer:latest
          command: ["python", "train.py"]
          resources:
            limits:
              nvidia.com/gpu: 1
  backoffLimit: 2
```

위 예시는 NVIDIA GPU를 Kubernetes에 노출하는 장치 플러그인 등이 이미 설정되었다는 전제하에 동작합니다.

또한 실제 실행을 위해서는 유효한 학습 이미지, 데이터셋 접근 경로 및 학습 설정이 필요합니다.

---

## 6. GPU 학습 작업 분배

### 6.1 일반 API 로드밸런싱과 학습 작업 분배

일반 API 서버는 요청을 받고 비교적 빠르게 응답하는 경우가 많습니다.

반면 AI 학습 서버는 작업 하나가 수 시간 이상 실행될 수 있습니다.

따라서 GPU 학습 환경에서는 HTTP 요청 분배와 실제 학습 작업 할당을 구분해야 합니다.

### 6.2 Message Queue 기반 구조

```text
                    User
                     |
                     v
                 MLOps API
                     |
                     v
                 Job Queue
                     |
               Job Scheduler
                     |
         +-----------+-----------+
         |           |           |
         v           v           v
      Worker1     Worker2     Worker3
       GPU0        GPU0        GPU0
         |           |           |
         +-----------+-----------+
                     |
                     v
               Shared Storage
```

작업 흐름:

1. 사용자가 학습 요청
2. MLOps API가 작업 정보 검증
3. 데이터베이스에 작업 ID 및 상태 기록
4. 작업 큐에 학습 요청 등록
5. Worker 또는 Scheduler가 실행 가능한 작업 선택
6. GPU 자원 확인 및 할당
7. 학습 프로세스 실행
8. 학습 로그 및 결과 저장
9. 최종 작업 상태 업데이트

메시지 큐만 도입한다고 GPU 자원 관리가 자동 해결되는 것은 아닙니다.

GPU 수량, 메모리, 동시 실행 제한 등을 고려하는 Worker 로직 또는 Scheduler가 필요합니다.

### 6.3 작업 요청 데이터

```json
{
  "job_id": "train-001",
  "model": "yolo",
  "dataset": "dataset_v1",
  "epochs": 100,
  "gpu_count": 1
}
```

실제 운영에서는 Job ID를 기준으로 학습 상태를 관리합니다.

예시:

```text
PENDING
   |
   v
QUEUED
   |
   v
RUNNING
   |
   +----> COMPLETED
   |
   +----> FAILED
   |
   +----> CANCELLED
```

### 6.4 독립 학습과 분산 학습

두 가지 개념은 다릅니다.

**독립 학습**

```text
GPU 1 → Model A 학습
GPU 2 → Model B 학습
GPU 3 → Model C 학습
```

서로 다른 학습 작업이 독립적으로 실행됩니다.

**분산 학습**

```text
             Model A
                |
       Distributed Training
                |
       +--------+--------+
       |        |        |
      GPU1     GPU2     GPU3
```

하나의 모델을 여러 GPU가 협력하여 학습합니다.

분산 학습에는 PyTorch Distributed Data Parallel(DDP), FSDP 등의 프레임워크가 사용될 수 있습니다.

Kubernetes에서 Pod를 여러 개 생성했다고 모델의 분산 학습이 자동으로 구현되지는 않습니다.

---

## 7. NFS 기반 데이터셋 공유

### 7.1 데이터셋 공유가 필요한 이유

학습 서버가 여러 대라면 동일한 데이터셋을 각 서버에 별도로 복사하는 것은 저장 공간을 낭비할 수 있습니다.

중앙 데이터 저장소를 이용하면 여러 학습 서버가 동일한 원본 데이터에 접근할 수 있습니다.

```text
            NFS Dataset Server
                    |
          +---------+---------+
          |         |         |
          v         v         v
       Train1    Train2    Train3
```

### 7.2 NFS Mount

예시 디렉토리:

```text
NFS Server
/data/datasets

Train Server
/mnt/datasets
```

학습 서버에서 NFS 마운트:

```bash
sudo mount -t nfs -o ro \
192.168.0.30:/data/datasets \
/mnt/datasets
```

위 명령은 NFS 서버의 Export 설정과 클라이언트의 NFS 지원 패키지가 준비되어 있다는 전제입니다.

읽기 전용 마운트를 사용하면 원본 데이터셋의 의도하지 않은 수정을 방지하는 데 도움이 됩니다.

### 7.3 Docker Volume Mount

학습 서버 호스트에 NFS가 마운트되어 있다면 해당 경로를 Docker 컨테이너에 전달할 수 있습니다.

```yaml
services:
  trainer:
    image: my-trainer:latest
    volumes:
      - /mnt/datasets:/app/datasets:ro
```

```text
NFS Server
    |
    v
Train Server /mnt/datasets
    |
    | Docker Bind Mount
    v
Container /app/datasets
```

### 7.4 스토리지 종류 비교

| 저장소 | 특징 | 사용 목적 |
|---|---|---|
| NFS | 네트워크 파일시스템 | 여러 서버의 데이터셋 공유 |
| NAS | 네트워크 연결 스토리지 장비 또는 시스템 | 중앙 파일 저장 |
| S3 | 객체 스토리지 | 원본 데이터 및 모델 아티팩트 보관 |
| Local SSD | 서버 내부 디스크 | 학습 데이터 캐시 |
| Kubernetes PVC | 클러스터 스토리지 사용 요청 | Pod의 영구 스토리지 연결 |

S3는 기본적으로 NFS와 같은 POSIX 파일시스템이 아닙니다. S3 API를 통한 데이터 접근이 일반적이며, 필요한 경우 별도의 마운트 도구를 활용합니다.

### 7.5 학습 결과 저장 전략

권장 분리:

```text
Shared Storage
|
+-- datasets/
|   +-- dataset_v1/
|   +-- dataset_v2/
|
+-- models/
|   +-- train-001/
|   +-- train-002/
|
+-- checkpoints/
    +-- train-001/
    +-- train-002/
```

원본 데이터셋은 읽기 전용으로 공유하고, 학습 체크포인트 및 결과물은 작업별 디렉토리에 저장하는 것이 좋습니다.

---

## 8. Docker 운영 명령어

### 8.1 컨테이너 및 이미지 확인

```bash
docker ps
docker ps -a
docker images
docker network ls
docker volume ls
```

### 8.2 전체 컨테이너 중지

```bash
docker ps -q | xargs -r docker stop
```

### 8.3 전체 컨테이너 삭제

```bash
docker ps -aq | xargs -r docker rm -f
```

### 8.4 사용하지 않는 리소스 정리

```bash
docker container prune
docker image prune -a
docker network prune
docker system prune -a
```

주의: prune 명령은 복구하기 어려운 리소스 삭제를 수행할 수 있습니다. 특히 볼륨 삭제 전에는 데이터 백업 여부를 확인해야 합니다.

`docker system prune -a`는 실행 중인 컨테이너가 사용하는 이미지를 일반적으로 삭제하지 않습니다.

### 8.5 Docker Compose 관리

```bash
docker compose up -d
docker compose ps
docker compose logs -f
docker compose down
```

### 8.6 네트워크 및 마운트 확인

```bash
docker inspect <container_name>
docker network inspect <network_name>
findmnt
df -h
mount | grep nfs
```

---

## 9. 아키텍처 선택 기준

| 환경 | 추천 구성 |
|---|---|
| 단일 서버 API 서비스 | Docker |
| 단일 호스트 다중 서비스 | Docker Compose |
| HTTP 요청 로드밸런싱 | Nginx + Docker Compose |
| MLOps 서버 + 학습 서버 1대 | Docker + API + NFS |
| 학습 서버 여러 대 | Docker + Message Queue + GPU Scheduler |
| 다중 Node 워크로드 관리 | Kubernetes |
| 완료형 GPU 학습 작업 | Kubernetes Job |
| 하나의 모델을 다중 GPU로 학습 | DDP / FSDP 등 분산 학습 프레임워크 |

### 최종 정리

Docker는 컨테이너 실행과 격리를 담당하고, Docker Compose는 다중 컨테이너를 구성합니다.

Nginx는 네트워크 요청을 여러 백엔드 서버로 분산하는 데 활용할 수 있습니다.

Kubernetes는 여러 Node에 걸친 컨테이너 워크로드의 배포, 스케줄링 및 운영을 지원합니다.

AI 학습 서버에서는 단순 HTTP 로드밸런싱보다 작업 큐, GPU 자원 관리, 학습 상태 추적 및 공유 스토리지 설계가 중요합니다.

최종적으로 다중 서버 MLOps 환경은 다음 역할을 분리하여 설계할 수 있습니다.

- MLOps API: 사용자 요청 및 작업 관리
- Job Queue: 비동기 학습 작업 보관
- Scheduler: 사용 가능한 서버 및 GPU 자원 판단
- Train Worker: 실제 AI 학습 수행
- Shared Storage: 데이터셋 및 모델 아티팩트 공유
- Database: 학습 상태와 메타데이터 저장

이러한 구성은 소규모 Docker 기반 환경에서 시작하여, 서비스 규모와 운영 복잡도가 증가할 때 Kubernetes 기반 환경으로 확장할 수 있습니다.

---

## 참고 문서

- [Docker 공식 문서](https://docs.docker.com/)
- [Docker Compose Networking](https://docs.docker.com/compose/how-tos/networking/)
- [Nginx HTTP Load Balancing](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)
- [Kubernetes 공식 문서](https://kubernetes.io/docs/)
- [Kubernetes Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [Kubernetes Storage](https://kubernetes.io/docs/concepts/storage/)
- [PyTorch Distributed](https://docs.pytorch.org/docs/stable/distributed.html)