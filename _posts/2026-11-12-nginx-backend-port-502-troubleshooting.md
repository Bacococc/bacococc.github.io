---
title: "[Server] 페이지는 열리는데 API만 502가 난다면? — Nginx와 Backend Port 불일치 장애 진단"
layout: single
date: 2026-11-12 02:00:00 +0900
categories:
  - Server
tags:
  - Server
  - Nginx
  - Docker
  - Backend
  - 502 Bad Gateway
  - Troubleshooting
  - 튜터링
---

튜터링 후반부에는 각자 하나씩 장애를 직접 만들어 두고 다른 팀원들이 원인을 모르는 상태에서 해결하는 실습을 진행하였다.

내가 만든 장애 중 하나는 Nginx가 Backend의 잘못된 Port로 요청하도록 설정하는 것이었다.

![튜터링 현장](/assets/images/튜터링현장_2.jpg){: .align-center}

사용자가 처음 마주하는 증상은 다음과 같았다.

```text
메인 페이지    ──> 정상 (200 OK)
게시글 목록    ──> 실패 (502 Bad Gateway)
API 주요 기능  ──> 실패 (502 Bad Gateway)
```

메인 페이지(정적 파일 / 프론트엔드)는 정상적으로 열리지만, `/api/posts`와 같은 백엔드 API 요청에서만 `502 Bad Gateway` 에러가 발생하도록 구성하였다.

---

## 1. 장애를 어떻게 만들었는가?

Backend Application 프로세스는 실제로 4000번 Port에서 동작하고 있다.

```text
Backend Process ──> Listen :4000
```

하지만 Nginx 설정 파일의 Upstream proxy 대상은 일부러 다음과 같이 잘못 설정해 두었다.

```nginx
proxy_pass http://backend:4001;
```

따라서 전체 요청 흐름은 다음과 같이 어긋나게 된다.

```text
Browser (Client)
     ↓
Nginx Reverse Proxy
     ↓
backend:4001 [연결 실패: 해당 포트 Listener 없음]
```

실제 Backend는 4000번 포트에서 요청을 기다리고 있기 때문에, Nginx가 4001번으로 TCP 연결을 시도하면 해당 포트에는 소켓을 열어둔 프로세스가 존재하지 않는다.

---

## 2. 처음부터 Nginx 설정을 보면 안 된다

이 장애를 분석할 때 가장 중요한 원칙은, 처음부터 `nginx.conf` 설정 파일부터 열어 정답을 찾으려 하지 않는 것이다.

먼저 사용자가 실패를 겪은 상황을 커맨드라인에서 동일하게 재현해 본다.

```bash
curl -v http://localhost/
```

메인 페이지 요청 결과:

```text
HTTP/1.1 200 OK
```

다음으로 실패가 발생한 API 엔드포인트를 호출해 본다.

```bash
curl -v http://localhost/api/posts
```

호출 결과:

```text
HTTP/1.1 502 Bad Gateway
```

이 두 가지 결과를 대조하면 다음과 같은 가설을 세울 수 있다.

```text
메인 화면은 정상 응답
→ Nginx 웹 서버 자체나 프론트엔드 컨테이너가 죽은 것은 아님

API 요청에서만 502 발생
→ Nginx에서 Backend 컨테이너로 넘어가는 프록시(Upstream) 구간에 문제 발생
```

---

## 3. Container 상태 확인

다음으로 백엔드 컨테이너 자체가 비정상 종료(Crash)되었는지 확인한다.

```bash
docker compose ps
```

조회 결과:

```text
NAME       COMMAND                  SERVICE    STATUS    PORTS
nginx      "/docker-entrypoint.…"   nginx      Up        0.0.0.0:80->80/tcp
frontend   "docker-entrypoint.s…"   frontend   Up        3000/tcp
backend    "docker-entrypoint.s…"   backend    Up        4000/tcp
```

모든 컨테이너가 정상적으로 실행(`Up`) 중인 상태이다.

하지만 여기서 기억해야 할 중요한 포인트가 있다.

```text
Container 상태 Up ≠ 서비스 간 네트워크 통신 정상
```

컨테이너 프로세스가 단순히 살아있다고 해서, Nginx가 백엔드의 올바른 포트로 정상 연결하고 있다는 보장은 없다.

---

## 4. Nginx 로그에서 결정적인 단서 찾기

장애의 실마리를 찾기 위해 Nginx의 에러 로그를 확인한다.

```bash
docker compose logs nginx --tail 100
```

로그 출력에서 다음과 같은 결정적인 오류 메시지를 발견할 수 있다.

```text
[error] 29#29: *1 connect() failed (111: Connection refused) while connecting to upstream, client: 172.18.0.1, server: localhost, request: "GET /api/posts HTTP/1.1", upstream: "http://172.18.0.3:4001/api/posts", host: "localhost"
```

이 로그에서 두 가지 핵심 정보를 도출할 수 있다.

1. **`host not found`가 아니다**:  
   Docker 내부 DNS가 `backend`라는 서비스 이름을 정상적으로 해석(해당 IP로 변환)했다는 뜻이다.
2. **`Connection refused`가 발생했다**:  
   목적지 IP(Backend 컨테이너)까지는 도달했으나, 해당 대상의 **4001번 Port에서 TCP 핸드셰이크를 수락하는 Listener 프로세스가 없다**는 명백한 증거이다.

---

## 5. Backend는 실제로 어느 Port를 사용하는가?

이제 백엔드 컨테이너가 실제로 어떤 포트 번호를 바라보고 실행 중인지 환경 변수를 확인한다.

```bash
docker compose exec backend printenv | grep '^PORT='
```

확인 결과:

```text
PORT=4000
```

두 정보를 나란히 비교해 보면 원인이 명확해진다.

```text
Nginx가 프록시 요청을 보내는 곳 ──> backend:4001
Backend 프로세스가 대기 중인 곳  ──> backend:4000
```

여기서 원인이 사실상 확정된다.

---

## 6. Docker DNS 문제도 배제해 보기

혹시 모를 변수를 차단하기 위해 Docker 내부 DNS 해석 문제가 아닌지도 확실하게 검증해 둔다.

Nginx 컨테이너 내부에서 `backend` 호스트명에 대한 조회를 수행한다.

```bash
docker compose exec nginx getent hosts backend
```

조회 결과:

```text
172.18.0.3 backend
```

백엔드 컨테이너의 내부 IP가 정상적으로 반환된다.

```text
Nginx → backend DNS 이름 해석: 정상 (IP 매핑 완료)
```

따라서 DNS나 도커 네트워크 브릿지 문제가 아니라, 순수하게 **Port 불일치 문제**임을 완전히 확정할 수 있다.

---

## 7. 마지막으로 실행 중인 Nginx 설정 확인

충분한 로그와 증거를 수집한 뒤, 비로소 Nginx에 실제 로드된 설정을 확인한다.

```bash
docker compose exec nginx nginx -T
```

설정 내용 중 문제가 된 블록을 발견할 수 있다.

```nginx
location /api/ {
    proxy_pass http://backend:4001;
}
```

반면 백엔드는 4000번 포트에서 구동되고 있으므로 최종 원인은 다음과 같이 정리된다.

```text
[Nginx Upstream Port 오설정]

Nginx 프록시 전달 대상 ──> backend:4001 (오설정)
Backend 실제 리슨 포트  ──> backend:4000 (정상 가동)
```

---

## 8. 복구

Nginx 설정 파일에서 Backend의 실제 포트인 `4000`번으로 연결하도록 수정한다.

```nginx
location /api/ {
    proxy_pass http://backend:4000;
}
```

Nginx 설정을 재반영(`reload` 또는 컨테이너 재시작)한 뒤, 실패했던 API 요청으로 다시 검증한다.

```bash
curl -v http://localhost/api/posts
```

응답 결과가 `200 OK`로 정상 반환되면서 장애 복구가 완료된다.

---

## 9. 502를 보고 어디부터 확인해야 할까?

이번 장애 추적 과정을 요약하면 다음과 같다.

```text
메인 페이지 정상이나 API만 실패
     ↓
API 직접 호출 확인 (502 Bad Gateway 재현)
     ↓
docker compose ps (컨테이너 생존 여부 점검 → 모두 Up)
     ↓
Nginx error log 확인 (Connection refused & upstream backend:4001 포착)
     ↓
Backend 실제 환경변수 및 LISTEN 포트 확인 (PORT=4000 확인)
     ↓
Nginx 설정(nginx.conf) 확인 (proxy_pass backend:4001 불일치 확인)
     ↓
포트 수정 및 200 OK 정상 복구 검증
```

이번 실습을 통해 `502 Bad Gateway`라는 HTTP 상태 코드 하나만 보고 섣불리 원인을 단정 지어서는 안 된다는 것을 깊이 이해할 수 있었다.

`502 Bad Gateway`는 **리버스 프록시(Nginx)가 업스트림 서버로부터 올바른 응답을 받지 못했다는 결과**를 나타낼 뿐이며, 그 이면의 실제 원인은 다양하다.

* 백엔드 애플리케이션 프로세스가 크래시되어 완전히 종료된 경우
* 도커 네트워크나 호스트 네트워크 연결이 단절된 경우
* 이번 실습처럼 Nginx가 엉뚱한 포트로 접속을 시도하여 포트가 닫혀 있는 경우

따라서 실제 장애 대응에서는 단순히 상태 코드에만 의존하지 않고,

> "어느 계층까지는 정상이고, 어느 구간부터 응답이 끊어지는가?"

를 단계별 로그와 네트워크 명령어를 통해 확인하며 조사 범위를 점진적으로 좁혀 나가는 체계적인 접근이 가장 중요하다.
특히 이번 실습에서는 `Connection refused`와 `upstream backend:4001`이라는 두 가지 단서를 통해 Nginx와 Backend 사이의 Port 불일치를 명확하게 찾아낼 수 있었다.
