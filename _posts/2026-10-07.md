---
title: "[AWS] Route 53 + ACM + ALB로 HTTPS(SSL 인증서) 설정하기"
layout: single
date: 2026-10-07 00:00:00 +0900
categories:
  - AWS
tags:
  - AWS
  - Cloud Computing
  - SSL
  - ACM
  - ALB
  - Route 53
---

## 간략화 한 기존 프로젝트 아키텍쳐

```mermaid
    Client[사용자 Browser] -->|HTTPS:443| ALB[AWS ALB]
    ALB -->|HTTP:80| EC2[EC2 Security Group]
    
    subgraph EC2_Docker ["EC2 (Docker Host)"]
        EC2 --> Nginx[Nginx Container]
        subgraph Containers ["Containers"]
            Nginx -->|/| FE[Frontend]
            Nginx -->|/api| BE[Backend]
        end
    end
```

![AWS Architecture](/assets/images/aws_architecture_sleek.jpg)


현재 프로젝트의 구성은 위와 같다.(상세 구조는 일단 생략...)

이 블로그의 목표는 위와 같은 구성의 프로젝트에 SSL 인증서를 붙여서 HTTPS 통신이 가능하도록 세팅하는 것이다. 

![http로 통신이 되고있는 상태](/assets/images/10-07-aws-screenshots/Screenshot%202026-10-07%20at%208.33.42 AM.png)


## Step 1: Route 53을 통한 도메인 구매

AWS 콘솔에서 Route 53 을 통해 도메인을 먼저 구매 했다.
해당 과정이 필요한 이유는 SSL 인증서(ACM)는 IP 주소가 아닌 **소유권을 가진 도메인 이름**에만 발급되기 때문이다.

![AWS 화면 2](/assets/images/10-07-aws-screenshots/Screenshot%202026-10-07%20at%208.15.24 AM.png)

![AWS 화면 3](/assets/images/10-07-aws-screenshots/Screenshot%202026-10-07%20at%208.15.44 AM.png)

위와 같이 화면에 구매를 원하는 도메인명을 입력하고 클릭하면 도메인의 사용 가능 여부와 가격을 확인할 수 있다. (이미 구매를 한 후에 촬영했기 때문에 사용 불가능한 도메인이라고 뜬다.)

![AWS 화면 4](/assets/images/10-07-aws-screenshots/Screenshot%202026-10-07%20at%208.15.52 AM.png)

도메인의 .com, .net, .org, .io, .kr 같은 가장 끝부분을 TLD (Top-Level Domain, 최상위 도메인) 라고 한다고 하는데, 이 TLD 별 가격도 확인 할 수 있다. 도메인을 꼭 아마존에서 구매해야 할 필요는 없다. 

![AWS 화면 5](/assets/images/10-07-aws-screenshots/Screenshot%202026-10-07%20at%208.16.38 AM.png)

도메인을 선택 한 후 플랜(사용 기간) 선택 - 연락처 정보 입력 - 제출을 한 후 잠시 기다리면 도메인이 발급된다. 수 분 정도 소요되었다.


## Step 2: ACM을 이용한 SSL 인증서 발급

Certificate Manager(ACM)에서 내 도메인에 적용할 퍼블릭 SSL/TLS 인증서를 발급 받는다. 이 발급 과정에서 실제 도메인 소유자가 본인이 맞는지 확인을 하는 몇 가지 방법이 있는데, 이 블로그에서는 Route 53을 이용해 CNAME 레코드를 추가하여 인증하는 방식을 사용한다.

![AWS 화면 6](/assets/images/10-07-aws-screenshots/Screenshot%202026-10-07%20at%208.28.35 AM.png)

![AWS 화면 7](/assets/images/10-07-aws-screenshots/Screenshot%202026-10-07%20at%208.28.40 AM.png)

ACM 메뉴에서 "인증서 요청"을 클릭한다.

![AWS 화면 8](/assets/images/10-07-aws-screenshots/Screenshot%202026-10-07%20at%208.28.56 AM.png)

인증서를 요청 할 도메인에 아까 구매한 도메인을 입력한다. 

![AWS 화면 9](/assets/images/10-07-aws-screenshots/Screenshot%202026-10-07%20at%208.29.31 AM%20copy.png)

인증서 요청을 완료하면 상태가 검증 대기 중으로 변경된다. 이 과정도 수 분 기다림이 필요했다. 

## Step 3: 호스팅 영역 및 레코드 생성

ACM 인증서 검증과 도메인-ALB 트래픽 연결을 위해 Route 53 호스팅 영역에 레코드를 생성해 주는 단계이다.

![AWS 화면 10](/assets/images/10-07-aws-screenshots/Screenshot%202026-10-07%20at%208.36.02 AM.png)

Route 53 메뉴에서 '호스팅 영역'을 클릭한 후 구매한 도메인에 대한 호스팅 영역을 생성한다. 생성이 완료되면 해당 호스팅 영역으로 이동하여 '레코드 생성'을 클릭한다.

![AWS 화면 11](/assets/images/10-07-aws-screenshots/Screenshot%202026-10-07%20at%208.38.07 AM.png)

ACM 콘솔 화면 우측 상단의 [Route 53에서 레코드 생성] 버튼을 클릭하면, 일일이 CNAME 이름과 값을 입력할 필요 없이 Route 53 호스팅 영역에 검증용 CNAME 레코드가 자동으로 등록된다.

*(NS와 SOA 레코드는 Route 53에서 호스팅 영역을 생성하면 기본으로 자동 생성되는 레코드이다. NS: 네임서버 레코드, 이 도메인에 대한 정보를 어떤 서버가 관리하는지 알려주는 레코드. SOA: 이 도메인 영역의 관리 정보 및 기본 규칙을 담고 있는 레코드.)*

![AWS 화면 12](/assets/images/10-07-aws-screenshots/Screenshot%202026-10-07%20at%208.50.37 AM.png)

다음으로 구매한 도메인(`gachimura.com`)으로 들어오는 트래픽을 실제 서비스 중인 ALB로 라우팅하기 위해 A 레코드를 생성한다. 레코드 유형은 **'A - IPv4 주소'**를 선택하고, **'별칭(Alias)' 토글 스위치를 활성화**해 준다.

트래픽 라우팅 대상에서 **'Application/Classic Load Balancer에 대한 별칭'**을 선택하고 본인의 리전(서울)과 미리 생성해 둔 ALB를 지정하여 레코드를 생성해 준다.

## Step 4: ALB 리스너에 ACM 인증서 적용

![AWS 화면 13](/assets/images/10-07-aws-screenshots/Screenshot%202026-10-07%20at%208.45.28 AM.png)

다음으로는 ALB 리스너에 아까 발급받은 ACM 인증서를 적용해야 한다.

먼저 적용을 할 프로젝트의 ALB에서 리스너 추가를 클릭한다. 

![AWS 화면 14](/assets/images/10-07-aws-screenshots/Screenshot%202026-10-07%20at%208.46.14 AM.png)

리스너 정책에서 아까 만들어둔 ACM 으로 인증을 설정 해준다. 

![AWS 화면 15](/assets/images/10-07-aws-screenshots/Screenshot%202026-10-07%20at%208.56.42 AM.png)

이렇게 하면 ALB의 리스너에 인증서 적용은 끝이났다!

마지막으로 HTTP 통신을 하는 리스너의 기본 동작을 HTTPS 통신으로 포워딩하도록 변경해야 한다.

## Step 5: HTTP to HTTPS 리다이렉션 설정

대상 ALB의 리스너 편집 메뉴에 들어가준다. 

여기서 HTTP:80 리스너의 기본 동작을 리다이렉션으로 변경한다. 리다이렉션 규칙을 HTTPS:443 으로 변경해주면 된다.

![AWS 화면 16](/assets/images/10-07-aws-screenshots/Screenshot%202026-10-07%20at%208.57.25 AM.png)

![AWS 화면 17](/assets/images/10-07-aws-screenshots/Screenshot%202026-10-07%20at%208.57.38 AM.png)

이렇게 해주면 모든 설정이 끝났다. HTTPS 를 붙여서 접속하면 정상적으로 사이트에 접속이 가능하고, 
HTTP 로 접속 시에는 HTTPS 로 리다이렉션 되는 것을 확인 할 수 있다!

![https로 접속이 된 화면](/assets/images/10-07-aws-screenshots/Screenshot%202026-10-07%20at%208.55.37 AM.png)