---
title: "[AWS] 서버 안에서는 되는데 밖에서는 안 된다면? — Security Group과 NACL 장애 실습"
layout: single
date: 2026-11-12 01:00:00 +0900
categories:
  - AWS
tags:
  - AWS
  - EC2
  - Network
  - Security Group
  - NACL
  - Troubleshooting
  - 튜터링
---

AWS EC2에 웹서비스를 직접 배포한 뒤 Network 설정과 관련된 장애도 실습하였다.

이번 실습에서 특히 기억에 남았던 것은 다음과 같은 상황이었다.

```text
EC2 내부: curl localhost ──> 200 OK
외부 PC : 서비스 접속     ──> Connection timed out
```

> **핵심 질문**: "서버 내부에서는 Application이 정상인데, 왜 외부에서는 접속할 수 없을까?"

이 상황을 통해 **Security Group(보안 그룹)**과 **Network ACL(NACL)**의 역할을 실제 장애와 연결하여 공부하였다.

---

## 1. 먼저 서버 내부가 정상인지 확인한다

웹사이트가 열리지 않는다고 무작정 AWS Console부터 뒤지는 것은 좋은 방법이 아니다.

먼저 EC2에 SSH로 접속하여 인스턴스 내부에서 동일한 HTTP 요청을 직접 보내 보았다.

```bash
curl -v http://localhost
```

응답 결과:

```text
HTTP/1.1 200 OK
```

Container도 정상적으로 실행되고 있었다.

```bash
docker compose ps
```

또한 80번 Port도 모든 인터페이스에서 정상적으로 Listen 중이었다.

```bash
ss -lntp | grep :80
```

응답 결과:

```text
LISTEN 0 4096 0.0.0.0:80
```

여기까지 확인하면 다음과 같이 판단할 수 있다.

```text
Application 정상
Docker Container 정상
Nginx 웹서버 정상
EC2 내부 80 Port LISTEN 정상
```

> **진단 원칙**: 서버 내부(Application, Docker, Nginx, Port)는 완전히 정상 동작하고 있다.  
> 그런데 외부에서만 접속할 수 없다면, 조사 범위를 EC2 내부가 아닌 **EC2 밖의 네트워크 경계 영역**으로 옮겨야 한다.

---

## CASE 1: Security Group의 80번 Port가 열려 있지 않은 경우

EC2를 처음 배포했을 때 실제로 내부 `curl`은 정상인데 외부에서는 접속되지 않는 상황이 발생하였다.

Security Group 설정을 확인해 보니 HTTP 80번 Port에 대한 **Inbound 규칙**이 등록되어 있지 않았다.

Security Group은 AWS Resource(인스턴스/ENI) 바로 앞단에서 어떤 Traffic을 허용할 것인지를 결정하는 가상 방화벽이다.

따라서 Nginx가 80번 Port에서 정상적으로 Listen 중이더라도, Security Group이 외부 Traffic을 인바운드에서 허용하지 않으면 사용자의 패킷은 EC2 내부로 들어오지 못한다.

```text
Client (외부 사용자)
     │
     ▼
Security Group ❌ (80번 허용 규칙 없음 → 차단)
     │
    (패킷 유실 / 도달 불가)
     │
    EC2
     │
 Nginx (:80)
```

즉, Nginx나 컨테이너가 잘못된 것이 아니라 **Nginx까지 요청 패킷 자체가 도착하지 못한 것**이다.

Security Group 인바운드 규칙에 HTTP 80번(0.0.0.0/0)을 추가한 뒤 외부에서도 즉시 정상적으로 웹페이지에 접속할 수 있었다.

이때 에러 메시지의 차이를 다시 확실하게 정리할 수 있었다.

```text
Connection refused
→ 목적지 Host에는 정상 도달했으나, 해당 Port에서 연결을 받아주는 Listener(프로세스/소켓)가 없음

Connection timed out
→ 패킷 자체가 목적지 Host까지 도달하지 못하고 중간(방화벽/보안그룹/라우팅)에서 차단/유실됨
```

---

## CASE 2: NACL에서 80번 Port를 차단해 보기

이후에는 내가 직접 Network ACL(NACL) 장애를 만들어 다른 팀원들이 원인을 찾아보는 실습을 진행하였다.

서브넷에 연결된 NACL의 Inbound Rule에 다음과 같은 차단 규칙을 임의로 추가하였다.

```text
Rule 50 : TCP 80 ──> DENY
```

그 아래에는 기존의 전체 허용 규칙이 그대로 존재하도록 구성했다.

```text
Rule 100: ALL Traffic ──> ALLOW
```

처음 보면 아래에 `Rule 100: ALL ALLOW`가 있기 때문에 전체 트래픽이 허용될 것처럼 보일 수 있다.

하지만 **NACL은 번호가 낮은 Rule Number부터 순서대로 평가**한다.

```text
패킷 인입 (TCP 80)
     ↓
Rule 50 평가  ──> TCP 80 차단(DENY) 일치! ──> [즉시 차단 및 종료]
     ↓ (평가 진행되지 않음)
Rule 100 (ALLOW)
```

따라서 80번 Port Traffic은 `Rule 50`에서 먼저 DENY 처리되고, 아래에 있는 `Rule 100`까지는 도달조차 하지 못하고 즉시 드롭된다.

---

## 3. 사용자에게 보이는 증상

이 NACL 장애 상황에서도 다음 항목들은 여전히 모두 정상 동작하고 있었다.

```text
EC2 Instance Running
Docker Container Up
localhost /health ──> 200 OK
80 Port LISTEN
Security Group 인바운드 80 허용 정상
Route Table / Internet Gateway 정상
SSH(22) 접속 정상
```

그런데 외부 브라우저에서는 서비스에 접속할 수 없고, ALB 대상 그룹(Target Group)의 Target 인스턴스는 **Unhealthy** 상태로 변했다.

여기서 가장 중요한 단서는 **SSH(22번 Port) 접속은 정상적으로 유지된다**는 점이었다.

```text
22번 Port (SSH) ──> 정상 허용 (원격 접속 가능)
80번 Port (HTTP) ──> 차단 (타임아웃 발생)
```

> **단서 분석**: 인스턴스나 네트워크 경로 전체가 완전히 단절된 것이라면 SSH 접속도 불가능해야 한다.  
> SSH는 멀쩡히 접속되는데 HTTP(80)만 연결되지 않는다면, **특정 Port에 대한 트래픽만 선별적으로 차단**하는 방화벽/NACL 규칙이 개입되었음을 직관적으로 유추할 수 있다.

---

## 4. 정상인 영역을 하나씩 제거한다 (소거법)

장애를 해결할 때는 처음부터 감으로 NACL 콘솔을 열어보지 않았다.

다음과 같이 가장 안쪽(애플리케이션)부터 바깥쪽(네트워크)으로 하나씩 점검하며 정상인 부분을 지워 나갔다.

```text
1. docker compose ps
   ↓ Container 정상 가동 확인

2. curl localhost/health
   ↓ 내부 애플리케이션 200 OK 응답 확인

3. ss -lntp
   ↓ 0.0.0.0:80 포트 정상 LISTEN 확인

4. Target Group 상태 확인
   ↓ Unhealthy 상태 확인 (외부 트래픽 인입 문제)

5. Security Group 인바운드 규칙 확인
   ↓ HTTP 80 허용 정상 확인

6. Route Table / Internet Gateway 확인
   ↓ VPC 인터넷 라우팅 정상 확인

7. Network ACL (NACL) 규칙 확인
   ↓ Rule 50: TCP 80 DENY 규칙 발견! 원인 특정 완료
```

이 순서대로 확인하면 처음에는 원인 후보군이 너무 많아 보여도, **정상인 부분을 하나씩 제외(소거법)**하면서 조사 범위를 확실하게 좁힐 수 있다.

---

## 5. Security Group과 NACL의 차이

직접 실습으로 장애를 유발하고 추적해보면서 두 보안 계층의 핵심 차이를 명확히 정리할 수 있었다.

| 구분 | Security Group (보안 그룹) | Network ACL (NACL) |
| :--- | :--- | :--- |
| **적용 단위** | **인스턴스 (ENI)** 단위 | **서브넷 (Subnet)** 단위 |
| **상태 유지** | **Stateful** (응답 트래픽 자동 허용) | **Stateless** (In/Outbound 별도 허용 필요) |
| **규칙 유형** | **Allow(허용)** 규칙만 정의 가능 | **Allow 및 Deny(차단)** 모두 정의 가능 |
| **평가 방식** | 모든 규칙을 종합하여 평가 | **Rule Number가 낮은 순서**대로 순차 평가 |

* **Security Group (Stateful)**:  
  인바운드로 들어온 요청이 허용되었다면, 그에 대한 나가는 응답 트래픽은 아웃바운드 규칙과 무관하게 자동으로 허용된다.
* **Network ACL (Stateless)**:  
  상태를 기억하지 않기 때문에 인바운드 트래픽뿐만 아니라 되돌아가는 아웃바운드(Ephemeral Port 등) 트래픽도 명시적으로 허용되어 있어야 통신이 완성된다. 또한 DENY 규칙을 직접 생성할 수 있어 Rule Number의 배치 순서가 매우 중요하다.

---

## 6. 이번 실습에서 배운 점

처음에는 웹사이트가 열리지 않으면 무조건 애플리케이션 코드나 웹 서버(Nginx)에 문제가 생겼을 것이라 생각하기 쉬웠다.

하지만 다음 두 결과를 비교해 보면 장애의 위치가 완전히 달라진다.

```text
내부 curl: http://localhost ──> 200 OK (애플리케이션 정상)
외부 브라우저 요청           ──> Connection timed out (네트워크 차단)
```

이 두 가지 결과만 확인하더라도 조사 대상은 애플리케이션 내부가 아니라 **서버와 외부를 연결하는 네트워크 경계(Security Group, Route Table, NACL)**로 순식간에 좁혀진다.

> **마무리 교훈**: 중요한 것은 AWS 콘솔의 수많은 메뉴나 설정을 맹목적으로 외우는 것이 아니다.  
> **"사용자의 요청 패킷이 네트워크 계층 중 어디까지 도달했는가?"**를 단계별로 검증하는 사고방식이 인프라 트러블슈팅의 핵심이라는 것을 배울 수 있었다.
