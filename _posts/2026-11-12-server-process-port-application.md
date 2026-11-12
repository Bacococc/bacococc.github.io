---
title: "[Server] 서버가 켜져 있으면 서비스도 정상일까? — Process, Port, Application 구분하기"
layout: single
date: 2026-11-12 00:00:00 +0900
categories:
  - Server
tags:
  - Server
  - Linux
  - Process
  - Port
  - Socket
  - Troubleshooting
  - 튜터링
---

백호 튜터링 첫 주에는 웹서비스에 요청을 보냈을 때 서버 내부에서는 어떤 일이 일어나는지를 공부하였다.

나는 그중 Linux의 Process와 Port를 중심으로 발표를 준비하였다.

![튜터링 현장](/assets/images/IMG_1543.jpg){: .align-center}

처음에는 서버가 켜져 있고 프로그램이 실행 중이면 서비스도 당연히 정상이라고 생각하기 쉬웠다.

하지만 실제로는 다음 세 가지를 구분해서 확인해야 한다.

```text
서버가 켜져 있다
       ≠
Process가 살아 있다
       ≠
Application이 정상적으로 요청에 응답한다
```

> **핵심 원칙**: "서버가 켜져 있다"는 것은 단 하나의 통합된 상태가 아니다.  
> `Process(실행)` ≠ `Port(소켓 바인딩 & Listen)` ≠ `Application(정상 응답)` 세 단계를 반드시 분리해서 점검해야 온전한 정상 여부를 알 수 있다.

이번 글에서는 이 차이를 중심으로 정리해보려고 한다.

---

## 1. Program과 Process의 차이

* **Program**: 디스크에 저장되어 있는 실행 가능한 명령어나 소프트웨어 코드의 집합 (레시피)
* **Process**: 그 Program이 실제로 메모리에 적재되어 실행되고 있는 상태 (요리 중)

예를 들어 NestJS Application 코드가 서버 파일 시스템에 존재한다고 해서 서비스가 동작하고 있는 것은 아니다. 실제로 **Node Process**가 실행되어 메모리에 올라가 있어야 한다.

```text
Program (디스크 저장, 코드 파일)
   ↓ 실행 (npm run start)
Process (메모리 적재, PID 발급)
```

Linux에서는 다음과 같은 명령어로 실행 중인 Process를 확인할 수 있다.

```bash
ps aux
```

또는 특정 Process만 찾고 싶다면 `grep`과 함께 파이프라인으로 연결하여 사용할 수 있다.

```bash
ps aux | grep node
```

Process가 정상적으로 실행 중이라면 OS로부터 부여받은 고유 번호인 **PID(Process ID)**도 확인할 수 있다.

> **주의**: 하지만 여기서 `ps` 명령어로 Process가 보인다는 것만으로는 아직 서비스가 정상이라고 말할 수 없다.

---

## 2. Process가 실행되면 바로 요청을 받을 수 있을까?

Network 요청을 받기 위해서는 Process가 네트워크 창구인 **Socket**을 열고, 특정 **Port**에 연결되어 있어야 한다.

이를 단계별로 정리하면 다음과 같다.

```text
Program 실행
     ↓
Process 생성
     ↓
 Socket 생성
     ↓
    Bind
     ↓
   Listen
     ↓
Client의 연결 대기
```

Process가 백그라운드에서 살아 있어도 원하는 Port에서 **`Listen`**하고 있지 않다면 사용자는 해당 서비스에 접근할 수 없다.

현재 서버에서 어떤 Port가 Listen 중인지 확인할 때는 다음 명령어를 사용할 수 있다.

```bash
ss -lntp
```

예를 들어 다음과 같은 결과가 조회된다.

```text
LISTEN 0 4096 0.0.0.0:80
```

이 경우 모든 네트워크 인터페이스의 80번 Port에서 클라이언트의 연결을 기다리는 Process가 정상 대기 중이라는 의미이다.

> **포인트**: Process 생성 후 Socket 생성 → Bind → Listen 단계까지 마쳐야 외부 클라이언트와의 TCP 핸드셰이크를 받아들일 준비가 완료된다.

---

## 3. 127.0.0.1과 0.0.0.0의 차이

Port를 확인할 때는 단순히 Port 번호만 볼 것이 아니라, 소켓이 **어떤 주소(Listen Address)에 Bind되어 있는가**도 반드시 확인해야 한다.

```text
127.0.0.1 (Loopback)
vs
0.0.0.0 (Wildcard)
```

* **`127.0.0.1:4000`**:  
  자기 자신(Loopback)만을 가리킨다. 해당 서버 내부의 localhost에서만 접근할 수 있으며 외부 클라이언트는 절대 접근할 수 없다.
* **`0.0.0.0:80`**:  
  서버에 연결된 모든 네트워크 인터페이스(공인 IP, 사설 IP, 루프백 등)로 들어오는 요청을 전부 수용하도록 Bind되어 있다는 뜻이다.

> **팁**: 외부 접속 문제를 조사할 때는 단순히 포트 방화벽만 열려 있는지 볼 것이 아니라, Process의 Listen Address가 `127.0.0.1`로 갇혀 있는지 `0.0.0.0`으로 열려 있는지를 함께 확인해야 한다.

---

## 4. 마지막에는 실제 Application에 요청을 보내야 한다

Process와 Port가 모두 정상이라고 하더라도 Application 자체가 정상이라는 보장은 없다.  
포트는 정상적으로 열려 있어 TCP 연결은 맺어지지만, DB 연결 실패나 내부 런타임 오류로 인해 `500 Internal Server Error`를 반환할 수도 있기 때문이다.

따라서 마지막에는 실제 Application에 요청을 보내 보아야 한다.

```bash
curl -v http://localhost/health
```

정상적인 경우 다음과 같이 200 응답을 확인할 수 있다.

```text
HTTP/1.1 200 OK
```

이를 통해 다음 세 단계를 각각 명확하게 확인할 수 있다.

```text
ps   ──> Process 존재 여부 (프로세스가 살아있는가?)
ss   ──> Port Listen 여부 (포트가 열려 연결을 대기하는가?)
curl ──> 실제 Application 응답 여부 (비즈니스 로직이 200 OK로 응답하는가?)
```

> **핵심 요약**: `Process` → `Port` → `Application` 3단계 진단 흐름을 통과해야 비로소 서비스가 완전히 정상이라고 판단할 수 있다.

---

## 5. Connection refused는 무엇을 의미할까?

장애 실습을 진행하며 `Connection refused` 에러를 자주 확인하였다.

예를 들어 다음 요청이 실패했다고 가정한다.

```bash
curl http://localhost:4000
```

결과:

```text
curl: (7) Failed to connect to localhost port 4000: Connection refused
```

이 메시지는 보통 목적지 Host까지는 네트워크 패킷이 정상 도달했지만, 해당 Port에서 연결을 받아주는 Process(Listener)가 없다는 강한 단서가 된다.

즉, 다음 두 가지 상황을 명확히 구분해야 한다.

```text
서버 자체가 없는 것 (호스트 통신 불가)
       vs
서버는 있지만 해당 Port가 열려 있지 않은 것 (Listener 없음)
```

반면 `Connection timed out`은 일정 시간 동안 응답 자체가 없었다는 뜻이므로, Security Group(보안 그룹), Firewall(방화벽), 라우팅 경로 등에서 패킷 자체가 유실/차단되었을 가능성을 먼저 의심할 수 있다.

```text
Connection refused
→ 목적지 호스트까지 패킷 도달 완료
→ 해당 Port의 Listener(프로세스/소켓) 문제 가능성

Connection timed out
→ Packet이 목적지까지 도달하지 못함
→ Security Group, 방화벽, 라우팅 테이블 차단 의심
```

> **구분 포인트**: `Connection refused`가 발생하면 네트워크 회선보다 서버 내부의 Process 실행 여부와 Port Listen 상태(`ps`, `ss`)를 먼저 들여다보아야 한다.

---

## 6. 실제 EC2에서도 같은 방법으로 확인하기

이후 AWS EC2에 직접 웹서비스를 배포하면서 이 내용을 다시 사용하였다.

서비스 접속에 문제가 생겼을 때 먼저 서버 내부에서 Process와 Port를 확인하였다.

```bash
# 1. 프로세스 생존 여부 확인
ps aux | grep node
```

```bash
# 2. 포트 바인딩 및 LISTEN 상태 확인
ss -lntp
```

그다음 EC2 내부에서 직접 Application에 HTTP 요청을 발송하였다.

```bash
# 3. 로컬 루프백 헬스체크 요청
curl -v http://localhost
```

이렇게 확인하면 외부 네트워크 설정을 점검하기 전에 먼저 **서버 내부 인프라가 온전히 정상인지**를 확실하게 판단할 수 있다.

```text
EC2 내부: curl http://localhost ──> 200 OK (정상 작동)
외부 접속: 브라우저 접속 불가 (타임아웃 / 먹통)

↓ 원인 분석
서버 내부(Application, Docker, Nginx)는 정상이므로,
앞단의 AWS Security Group, Route 53, ALB 등 외부 네트워크 설정을 먼저 의심!
```

예를 들어 EC2 내부에서는 `200 OK`가 잘 나오는데 외부에서만 접속되지 않는다면,

```text
Application
Docker
Nginx
```

보다 앞단의 네트워크 설정(Security Group 인바운드 규칙 등)을 먼저 의심할 수 있다.

---

## 정리

이번 학습을 통해 서버 장애를 볼 때 단순히 "서버가 켜져 있는가"만 확인해서는 안 된다는 것을 알게 되었다.

앞으로 장애를 확인할 때는 다음 순서를 기본 원칙으로 적용하려고 한다.

```text
Process (프로세스 확인: ps)
   ↓
 Port   (포트 리슨 확인: ss)
   ↓
Application (응답 상태 확인: curl)
```

> **마무리 교훈**: 각 단계가 정상이라는 것을 직접 확인해야 다음 단계로 넘어갈 수 있다는 것이 이번 학습에서 가장 중요하게 배운 부분이었다.
