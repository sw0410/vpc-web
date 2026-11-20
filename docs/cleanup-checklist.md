# 리소스 정리 체크리스트 (vpc-web)

- 리전: 서울 `ap-northeast-2` (콘솔은 고른 리전의 것만 보여 주므로 리전을 먼저 확인)
- 정리한 사람: `mission-user` (정책 `vpc-web-mission-policy`에 종료·삭제 Action 포함)
- 원칙: **돈이 나가는 것부터, 붙어 있는 것의 거꾸로** 지운다. 붙어 있는 대상이 남아 있으면 AWS가 삭제를 거부한다.

## 계정에 있던 것 (만든 것 + 딸려 생긴 것)

| 종류 | 이름 | 만듦 / 딸려 생김 | 있는 동안 돈? | 무엇에 붙어 있나 |
|---|---|---|---|---|
| EC2 인스턴스 | vpc-web-server (t3.micro) | 만듦 | 예 (실행 중일 때) | 서브넷, 보안그룹 |
| EBS 볼륨 | 8 GiB 루트 디스크 | 딸려 생김 | 예 (중지해도) | EC2 |
| 퍼블릭 IPv4 | 자동 할당 | 딸려 생김 | 예 (붙어 있는 동안, 청구서에서는 VPC 아래) | EC2 (중지하면 반납) |
| 네트워크 인터페이스 | 자동 | 딸려 생김 | 아니오 | EC2, 서브넷 |
| 키 페어 | vpc-web-key | 만듦 | 아니오 | — |
| 보안그룹 | vpc-web-sg | 만듦 | 아니오 | VPC (EC2가 사용) |
| 서브넷 | vpc-web-public | 만듦 | 아니오 | VPC |
| 라우팅 테이블 | vpc-web-public-rt | 만듦 | 아니오 | VPC, 서브넷과 연결 |
| 인터넷 게이트웨이 | vpc-web-igw | 만듦 | 아니오 | VPC에 Attach |
| VPC | vpc-web | 만듦 | 아니오 | — |
| 기본 라우팅 테이블·보안그룹·NACL | 자동 | 딸려 생김 | 아니오 | VPC (VPC 삭제 시 함께 삭제) |
| Elastic IP | — | 만들지 않음 | (만들었다면 예) | — |
| NAT Gateway / ELB / RDS | — | 만들지 않음 | (만들었다면 예) | — |

## 정리 순서와 근거

| 순서 | 할 일 | 왜 이 자리인가 | 완료 | 근거 (스크린샷) |
|---|---|---|---|---|
| 1 | EC2 인스턴스 **종료** | 돈이 나가는 것의 중심. 서브넷·보안그룹이 EC2에 묶여 있어 먼저 지워야 풀린다 | ☑ 11-20 22:08 | [종료됨](images/cleanup-01-ec2-terminated.png) |
| 2 | EBS 볼륨 목록 비었는지 확인 | 종료 시 함께 지워지는 설정이지만, 남으면 인스턴스 없이도 과금 | ☑ 11-20 22:09 | [볼륨 없음](images/cleanup-02-ebs-empty.png) |
| 3 | Elastic IP 목록 비었는지 확인 | 만들지 않았지만 확인. 남으면 과금 | ☑ 11-20 22:10 | [없음](images/cleanup-03-eip-empty.png) |
| 4 | 보안그룹 `vpc-web-sg` 삭제 | EC2가 쓰는 동안에는 삭제 거부 | ☑ 11-20 22:10 | [default 두 개만 남음](images/cleanup-04-sg-only-default.png) |
| 5 | 서브넷 `vpc-web-public` 삭제 | 안에 네트워크 인터페이스가 남아 있으면 거부 | ☑ 11-20 22:11 | [삭제 완료](images/cleanup-05-subnet-deleted.png) |
| 6 | 라우팅 테이블 `vpc-web-public-rt` 삭제 | 서브넷 연결이 풀린 뒤 삭제 가능. 처음엔 '연결이 있어 삭제 불가'로 거부됨(서브넷 삭제 직후라 화면 정보가 늦음) → 새로고침 후 삭제 | ☑ 11-20 22:14 | [거부 화면](images/cleanup-06-rt-delete-blocked.png), [기본 표 두 개만 남음](images/cleanup-06b-rt-only-main.png) |
| 7 | IGW `vpc-web-igw` **분리(Detach)** 후 삭제 | VPC에 붙어 있는 동안 삭제 불가. 처음에 바로 삭제를 눌러 'VPC에서 먼저 분리하세요'로 거부됨 → 분리 후 삭제 | ☑ 11-20 22:15 | [거부 문구](images/cleanup-07-igw-detach-first.png), [기본 VPC의 IGW만 남음](images/cleanup-07b-igw-only-default.png) |
| 8 | VPC `vpc-web` 삭제 | 마지막. 기본 라우팅 테이블·보안그룹·NACL이 함께 사라짐 | ☑ 11-20 22:16 | [기본 VPC만 남음](images/cleanup-08-vpc-only-default.png) |
| 9 | 키 페어 `vpc-web-key` 삭제, 노트북의 `.pem` 삭제 | 돈은 안 나가지만 쓰지 않는 열쇠는 남기지 않음 | ☑ 11-20 22:21 | [키 페어 없음](images/cleanup-09-keypair-empty.png), 랩탑 `Remove-Item` 완료 |
| 10 | Billing 확인 (정리 후 하루 이상 지나서) | 청구 화면은 약 하루 늦게 반영 | ☐ | |
