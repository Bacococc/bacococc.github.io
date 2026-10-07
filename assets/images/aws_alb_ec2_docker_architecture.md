# AWS ALB - EC2 - Docker - Nginx - Frontend/Backend 아키텍처 다이어그램

````carousel
![AWS Architecture Diagram (상세)](/Users/hyojin/.gemini/antigravity-ide/brain/7a7c83be-7147-4b70-8886-29430382e4c9/aws_architecture_diagram_1791368835433.jpg)
<!-- slide -->
![AWS Architecture Diagram (심플 2D)](/Users/hyojin/.gemini/antigravity-ide/brain/7a7c83be-7147-4b70-8886-29430382e4c9/aws_architecture_simple_1791368904766.jpg)
````

---

## 1. 세부 컴포넌트 흐름 (Mermaid Sequence Diagram)

```mermaid
sequenceDiagram
    autonumber
    actor User as 📱/💻 사용자 (Client)
    participant ALB as ☁️ AWS ALB (Port 443)
    participant SG as 🛡️ EC2 Security Group
    participant Nginx as 🌐 Nginx Container (Port 80)
    participant FE as 🎨 Frontend (React/Vue/Next)
    participant BE as ⚙️ Backend API (Node/Spring)
    participant DB as 🗄️ Database (RDS)

    %% 1. 웹 사이트 접속
    rect rgb(240, 248, 255)
    note right of User: [시나리오 1] 정적 페이지 접속 (https://example.com/)
    User->>ALB: HTTPS GET / (Port 443)
    ALB->>ALB: SSL Offloading (HTTPS -> HTTP 해제)
    ALB->>SG: HTTP GET / (Target Group 포트 전달)
    SG->>Nginx: HTTP 요청 허용 및 전달
    Nginx->>FE: 정적 파일 요청 (index.html, main.js)
    FE-->>Nginx: HTML/JS/CSS 반환
    Nginx-->>ALB: 응답 반환
    ALB-->>User: 브라우저에 화면 렌더링
    end

    %% 2. API 데이터 요청
    rect rgb(255, 245, 238)
    note right of User: [시나리오 2] 백엔드 API 요청 (https://example.com/api/v1/data)
    User->>ALB: HTTPS POST /api/v1/data
    ALB->>SG: HTTP POST /api/v1/data
    SG->>Nginx: HTTP 요청 전달
    Nginx->>Nginx: location /api/ 라우팅 룰 매칭 (proxy_pass)
    Nginx->>BE: http://backend:8080/api/v1/data 요청
    BE->>DB: SQL Query (Data 조회/저장)
    DB-->>BE: Query Result
    BE-->>Nginx: JSON Data Response
    Nginx-->>ALB: HTTP Response
    ALB-->>User: HTTPS JSON Response (200 OK)
    end
```

---

## 2. 계층별 핵심 역할 및 특징 정리

| 구분 | 컴포넌트 | 위치 / 네트워크 | 주요 역할 및 기술 스택 |
| :--- | :--- | :--- | :--- |
| **L7 Load Balancer** | **AWS ALB** | Public Subnet | • SSL/TLS Certificate (ACM) 종단<br>• 헬스체크(Health Check) 및 타겟 그룹 라우팅<br>• AWS WAF 연동으로 웹 보안 통제 |
| **Compute Host** | **AWS EC2** | Private/Public Subnet | • Docker Engine 구동 호스트<br>• 보안그룹(Security Group)으로 ALB 외부 IP만 수신 제한 |
| **Reverse Proxy** | **Nginx** | Docker Container (`bridge`) | • 단일 진입점 역할 (`:80`)<br>• `/api/` 요청 백엔드 프록시 (`proxy_pass`)<br>• 정적 파일(Frontend 빌드본) 초고속 직접 서빙 |
| **Web App** | **Frontend** | Docker Container | • React, Vue, Svelte, Next.js 등 사용자 인터페이스<br>• Static SPA 빌드 자원 제공 |
| **API Server** | **Backend** | Docker Container | • Spring Boot, Node.js, FastAPI, Django 등<br>• 비즈니스 로직 처리, JWT 인증, DB 연동 |

---

## 3. 핵심 아키텍처 포인트

> [!NOTE]
> **1. SSL Offloading (보안 및 연산 효율화)**
> 외부 인터넷 통신은 `HTTPS(443)`를 사용하지만, ALB가 SSL을 해제해 주므로 내부 EC2 및 Docker Nginx는 `HTTP(80)` 통신만 수행하여 암호화 연산 부담을 줄입니다.

> [!TIP]
> **2. CORS 문제 근본 해결**
> Frontend와 Backend가 모두 동일한 Nginx 역프록시 뒤에 위치하므로, 브라우저는 도메인과 포트가 동일하다고 인식하여 CORS(Cross-Origin Resource Sharing) 설정 이슈를 피할 수 있습니다.

> [!IMPORTANT]
> **3. 보안그룹 (Security Group) 최우선 격리**
> EC2 보안그룹의 인바운드 규칙을 `Anywhere(0.0.0.0/0)`가 아닌 **ALB의 보안그룹 ID**로만 지정함으로써, 인터넷에서 EC2로 직접 접속하려는 무단 침입을 차단합니다.
