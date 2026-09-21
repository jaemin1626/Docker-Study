# Docker DB Image 선택 가이드

Docker에서는 데이터베이스를 직접 설치하기보다 `PostgreSQL`, `MySQL`, `Redis`, `MongoDB` 등의 이미지를 사용하여 DB 환경을 구성할 수 있다.

중요한 것은 단순히 DB 이미지를 실행하는 방법보다 **서비스의 데이터 특성과 목적에 따라 적절한 DB를 선택하는 것**이다.

## 1. PostgreSQL

**관계형 데이터와 복잡한 데이터 처리가 필요한 서비스에 적합한 범용 RDBMS**

PostgreSQL은 사용자, 권한, 프로젝트, 작업 이력처럼 데이터 간 관계가 명확한 서비스에 적합하다. 일반적인 SQL 기반 관계형 데이터뿐만 아니라 JSON 계열 데이터도 유연하게 다룰 수 있어 현대적인 백엔드 시스템에서 많이 활용된다.

### 적합한 상황

- 사용자 및 권한 관리
- 프로젝트 및 작업 상태 관리
- 데이터 간 관계가 중요한 서비스
- 복잡한 SQL Query가 필요한 경우
- JSON 형태의 데이터도 함께 관리해야 하는 경우
- FastAPI, Django, Spring 등의 백엔드 시스템
- AI/MLOps 플랫폼의 메타데이터 관리

### 예시

```text
MLOps System
│
├── User
├── Group
├── Permission
├── Dataset
├── Training Job
└── Deployment History
        │
        ▼
   PostgreSQL
```

> **선택 기준:** 일반적인 신규 백엔드 시스템에서 관계형 DB가 필요하고 특별한 제약이 없다면 우선적으로 고려할 수 있는 DB이다.

---

## 2. MySQL

**범용적인 웹서비스와 기존 시스템에서 널리 사용되는 관계형 DB**

MySQL은 오랜 기간 웹서비스에서 사용되어 온 대표적인 RDBMS로, 관련 자료와 생태계가 매우 크다.

사용자 정보, 게시글, 상품, 주문 등 구조화된 데이터를 관리하는 일반적인 웹서비스에 적합하다.

### 적합한 상황

- 일반적인 웹서비스
- 사용자 / 게시판 / 상품 / 주문 데이터
- 기존 시스템이 MySQL 기반인 경우
- PHP 기반 웹서비스
- MySQL을 사용하는 기존 프로젝트와의 호환성이 중요한 경우

### 예시

```text
Web Service
│
├── User
├── Product
├── Order
└── Payment
      │
      ▼
    MySQL
```

> **선택 기준:** 기존 인프라나 서비스가 MySQL을 사용하고 있거나, 범용적인 웹서비스용 RDBMS가 필요한 경우 적합하다.

---

## 3. MariaDB

**MySQL 계열의 오픈소스 RDBMS**

MariaDB는 MySQL과 유사한 사용 경험을 제공하는 관계형 데이터베이스이다.

MySQL 계열의 SQL 및 구조를 유지하면서 MariaDB의 기능이나 오픈소스 생태계를 활용하고 싶은 경우 선택할 수 있다.

### 적합한 상황

- MySQL 계열 DB가 필요한 경우
- 기존 MySQL 기반 시스템을 이전하는 경우
- 오픈소스 중심 환경
- MariaDB를 표준 DB로 사용하는 조직

### 예시

```text
Existing MySQL Service
        │
        ▼
   MariaDB 검토
```

> **선택 기준:** 단순히 "무료 MySQL"로 보기보다는 MySQL 계열 호환성과 MariaDB 자체 기능 및 운영 환경을 고려해서 선택한다.

---

## 4. Redis

**빠른 접근이 필요한 임시 데이터, Cache, Session 등에 적합한 In-Memory 데이터 저장소**

Redis는 PostgreSQL이나 MySQL과 목적이 다르다.

주로 데이터를 메모리에서 빠르게 처리하기 때문에 중요한 영구 데이터를 저장하는 메인 DB보다는 **Cache, Session, Queue, 실시간 상태 관리** 등에 많이 활용된다.

### 적합한 상황

- Cache
- 로그인 Session
- API 응답 Cache
- 작업 Queue
- 실시간 상태 관리
- Rate Limiting
- 분산 Lock
- 반복적으로 조회되는 데이터의 임시 저장

### 예시

```text
Backend
│
├── PostgreSQL
│     └── 영구 데이터
│
└── Redis
      ├── Cache
      ├── Session
      └── Queue
```

> **선택 기준:** PostgreSQL/MySQL을 대체하는 DB라기보다는 메인 DB를 보조하여 빠른 처리가 필요한 기능을 담당하는 용도로 생각하는 것이 좋다.

---

## 5. MongoDB

**데이터 구조가 유동적인 Document 형태의 데이터를 관리할 때 적합한 NoSQL DB**

MongoDB는 데이터를 Table과 Row 중심으로 관리하는 관계형 DB와 달리 Document 형태로 관리한다.

JSON과 유사한 형태로 데이터를 저장할 수 있기 때문에 데이터 구조가 자주 변경되거나 모든 데이터가 동일한 Schema를 가질 필요가 없는 서비스에서 유용하다.

### 적합한 상황

- Document 형태의 데이터
- 데이터 구조가 자주 변경되는 경우
- 다양한 형태의 Metadata 저장
- JSON 중심의 데이터를 자연스럽게 저장하고 싶은 경우
- 고정된 Table Schema가 불편한 서비스

### 예시

```json
{
  "dataset": "fire_detection",
  "model": "VLM",
  "options": {
    "epochs": 10,
    "gpu": [0, 1],
    "prompt": "Detect fire"
  },
  "custom_metadata": {
    "site": "seoul",
    "camera": "CCTV-01"
  }
}
```

> **선택 기준:** 데이터 간 관계보다 각각의 Document 자체가 중요하고, Schema의 유연성이 필요한 경우 고려한다.

---

# DB Image 선택 기준

| 요구사항 | 적합한 DB |
|---|---|
| 일반적인 관계형 데이터 | **PostgreSQL / MySQL** |
| 신규 백엔드 시스템 | **PostgreSQL** |
| 기존 MySQL 기반 시스템 | **MySQL** |
| MySQL 계열 오픈소스 환경 | **MariaDB** |
| 사용자 / 권한 / 작업 이력 관리 | **PostgreSQL** |
| Cache | **Redis** |
| Session | **Redis** |
| 작업 Queue | **Redis** |
| 빠른 임시 데이터 처리 | **Redis** |
| Document / JSON 중심 데이터 | **MongoDB** |
| Schema가 자주 변경되는 데이터 | **MongoDB** |

## DB는 하나만 선택할 필요가 없다

실제 서비스에서는 하나의 DB가 모든 역할을 담당하기보다 각 데이터 저장소의 장점을 조합해서 사용하는 경우가 많다.

예를 들어 웹 기반 AI/MLOps 시스템이라면 다음과 같이 구성할 수 있다.

```text
                    Backend
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
     PostgreSQL      Redis      File/NAS
          │            │            │
       영구 데이터   Cache/Queue   Dataset
          │
    ┌─────┴──────┐
    │            │
 User/권한    Training Job
 Dataset 정보   History
```

즉,

```text
PostgreSQL
→ 서비스의 핵심 영구 데이터

Redis
→ 빠른 Cache / Session / Queue

File System / NAS / Object Storage
→ 이미지 / 영상 / Dataset 등의 대용량 파일
```

처럼 역할을 분리할 수 있다.

## 핵심 정리

DB 이미지를 선택할 때는 단순히 가장 유명한 DB를 선택하는 것이 아니라 **저장하려는 데이터의 특성과 사용 목적**을 먼저 판단해야 한다.

```text
데이터 간 관계가 중요한가?
        │
        └── Yes → PostgreSQL / MySQL

빠른 임시 저장이나 Cache가 필요한가?
        │
        └── Yes → Redis

Document 형태이고 Schema가 유동적인가?
        │
        └── Yes → MongoDB

기존 MySQL 생태계를 유지해야 하는가?
        │
        └── Yes → MySQL / MariaDB
```

Docker는 이러한 DB들을 각각 독립된 Container로 구성할 수 있기 때문에 서비스의 목적에 따라 여러 DB를 조합하여 사용할 수 있다.

> **Docker DB Image 선택의 핵심은 "어떤 DB가 가장 좋은가?"가 아니라 "이 데이터는 어떤 방식으로 저장하고 접근해야 하는가?"를 먼저 결정하는 것이다.**
