# vpc-web

웹 서비스 인프라를 직접 구축하며 안정적인 서비스 환경을 설계 및 구현합니다.

AWS 서울 리전(`ap-northeast-2`)에 VPC, 퍼블릭 서브넷, 인터넷 게이트웨이(IGW), EC2(Nginx)를 직접 구성해 외부에서 웹 서버에 접속할 수 있게 만들고, 실습이 끝난 뒤 모두 정리했습니다.

## 외부 접속 검증

- 방식: **(A) 브라우저로 `http://<퍼블릭IP>` 접속**
- 접속 정보: `http://43.200.2.228` (EC2 퍼블릭 IP, 2026-11-19 기준. 리소스는 11-20에 정리함)
- 결과: 외부 브라우저에서 Nginx 기본 페이지 확인

![브라우저에서 Nginx 기본 페이지](docs/images/06-browser-welcome-nginx.png)

| 확인한 것 | 증빙 |
|---|---|
| 보안그룹 인바운드: 80 `0.0.0.0/0`, 22 내 IP `/32` | [01](docs/images/01-sg-inbound-rules.png) |
| EC2 실행 중, 퍼블릭 IP 할당 | [02](docs/images/02-ec2-running-public-ip.png) |
| SSH 접속 성공 | [03](docs/images/03-ssh-connected.png) |
| EC2 안에서 Nginx 응답 (`curl -I http://localhost` → 200) | [05](docs/images/05-nginx-localhost.png) |
| 외부 브라우저에서 Nginx 페이지 | [06](docs/images/06-browser-welcome-nginx.png) |

## 아키텍처

![아키텍처](docs/architecture.png)

## 요청과 응답 흐름

패킷 겉면에는 **받는 주소, 포트, 보낸 주소**가 적혀 있고, 지나가는 곳마다 보는 칸이 다릅니다. 단계별로 따라가는 인터랙티브 버전: https://claude.ai/artifact/E9CAXCq93XWQSAXcmoSjBp

| | 지점 | 일어나는 일 |
|---|---|---|
| 요청 | 브라우저 | 받는 주소 = EC2 퍼블릭 IP, 포트 = 80인 요청 패킷을 만든다 |
| | 인터넷 | 받는 주소를 따라 AWS로 보낸다 |
| | IGW | VPC 입구. 받는 주소를 퍼블릭 IP에서 프라이빗 IP(10.0.1.7)로 바꿔 넘긴다 |
| | 보안그룹 | 포트(80)와 보낸 주소를 보고 인바운드 규칙과 맞으면 들여보낸다 |
| | Nginx | EC2 안에서 80번으로 기다리다 받는다 |
| 응답 | 라우팅 테이블 | 받는 주소(내 노트북)가 VPC 밖이라 `0.0.0.0/0 → IGW` 줄을 따른다. 포트는 보지 않는다 |
| | IGW | 보낸 주소를 프라이빗 IP에서 퍼블릭 IP로 바꿔 내보낸다 |
| | 브라우저 | 인터넷을 거쳐 돌아온 응답으로 페이지를 띄운다 |

**한 칸이 빠지면**

| 빠진 것 | 어디서 막히나 | 증상 | 겪은 날 |
|---|---|---|---|
| 라우팅 `0.0.0.0/0 → IGW` 줄 | 라우팅 테이블 | 브라우저·SSH **모두** 시간 초과 | 11/18 |
| 보안그룹 80번 줄 | 보안그룹 | 브라우저만 시간 초과 (SSH는 됨) | 11/19 실험 |
| Nginx | EC2 안 | **즉시** 연결 거부 | 11/18 설치 전 |

자세한 과정은 [트러블슈팅](docs/troubleshooting.md)에 있습니다.

## 보안그룹과 IAM

| | 보안그룹 `vpc-web-sg` | IAM `mission-user` |
|---|---|---|
| 거르는 것 | EC2로 들어오는 **패킷** (포트 + 보낸 주소) | AWS 콘솔에서 하는 **작업** (누가 + 어떤 Action) |
| 설정 | 80 ← `0.0.0.0/0` (누구나 웹), 22 ← 내 IP `/32` (나만 SSH). 나머지 포트는 적지 않아 자동 차단 | FullAccess 대신 실제로 쓴 EC2·VPC 작업만 적은 [정책](docs/iam-policy.json) |
| 막히면 | 브라우저·터미널 시간 초과 | 콘솔 빨간 배너 (`s3:CreateBucket` 거부 [10](docs/images/10-iam-check-s3-denied.png)) |

- 22번 소스는 장소를 옮기면 그때 IP로 바꿨다 (11-18 학습장 → 11-19 집).
- 루트 사용자는 MFA를 걸고 계정 관리(IAM 설정, 결제 확인)에만 썼다.

## 리소스 추적과 정리

이름을 모두 `vpc-web-`로 시작하게 해서 기본 VPC와 구별하고, 만드는 즉시 아래 표에 적었습니다. 정리는 [체크리스트](docs/cleanup-checklist.md) 순서(돈이 나가는 EC2부터, 붙어 있는 것의 거꾸로)대로 하고 단계마다 근거 화면을 남겼습니다.

| 리소스 | 이름 | 있는 동안 돈? | 정리 |
|---|---|---|---|
| EC2 인스턴스 | vpc-web-server (t3.micro) | 예 | 11-20 종료 |
| EBS 볼륨 | 8 GiB (EC2와 함께 생김) | 예 (중지해도) | 종료와 함께 삭제 확인 |
| 퍼블릭 IP | 자동 할당 | 예 | 종료와 함께 반납 |
| Elastic IP | 만들지 않음 | — | 없음 확인 |
| 보안그룹 | vpc-web-sg | 아니오 | 삭제 |
| 서브넷 | vpc-web-public (10.0.1.0/24) | 아니오 | 삭제 |
| 라우팅 테이블 | vpc-web-public-rt | 아니오 | 삭제 |
| IGW | vpc-web-igw | 아니오 | 분리 후 삭제 |
| VPC | vpc-web (10.0.0.0/16) | 아니오 | 삭제 |
| 키 페어 | vpc-web-key | 아니오 | 삭제, 로컬 `.pem` 삭제 |

**비용**: 정리 다음 날(11-21) 크레딧 US$119.91, 실제 사용 약 9센트, 청구 0원 ([화면](docs/images/billing-01-credits-1121.png)).

## 문서

- [트러블슈팅](docs/troubleshooting.md)
- [리소스 정리 체크리스트](docs/cleanup-checklist.md)
- [IAM 정책](docs/iam-policy.json)
