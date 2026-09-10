# Docker-Study

Docker 학습 기록 — 기본 개념, 명령어, 네트워크와 실습을 정리

## 학습 기록

| 일차 | 내용 |
| --- | --- |
| [1일차 — Docker 기본 개념](Day01-Docker-Basics/README.md) | 이미지·컨테이너, Hub·Registry·Desktop·Compose, 이미지 저장, 컨테이너 명령어, 네트워크와 포트 매핑 |
| [2일차 — Docker Compose, Network, Volume](Day02-Docker-Compose-Network-Volume/README.md) | Compose 실행과 프로젝트 설정, 서비스 이름 통신, 포트 바인딩, Bind Mount·Named Volume, MLOps 활용과 영속성 실습 |
| [3일차 — Docker Network와 DNS Alias](Day03-Docker-Network-DNS/README.md) | 네트워크 모드 비교, 사용자 정의 bridge, namespace·veth 흐름, embedded DNS와 alias 실습 |
| [4일차 — Compose 통합 복습](Day04-Compose-Review/README.md) | 1·2·3일차 연결, Nginx 여러 페이지, DB·Redis, 데이터 보존, 실행 순서와 scale | 

## 복습 방법

1. 개념을 읽고 각 명령어가 이미지와 컨테이너 중 무엇을 다루는지 구분합니다.
2. 실습 예제를 순서대로 따라 하며 상태 변화를 확인합니다.
3. 마지막 복습 질문에 답해 봅니다.

예제는 Docker Engine 또는 Docker Desktop이 실행 중인 Linux 컨테이너 환경을 기준으로 합니다. 각 일차 문서에 공식 참고 자료를 함께 기록합니다.
