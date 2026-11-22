# 트러블슈팅 (vpc-web)

> 단계별로 따라가는 인터랙티브 버전: https://claude.ai/artifact/Mv1tg9PXZAjTQU9DnECCaV (공유 설정 후 열람 가능)

시각과 원본 메모는 [troubleshooting-notes.md](troubleshooting-notes.md)에 있습니다. 이 문서는 그중 두 건을 `증상 → 가설 → 검증 → 조치 → 결과 → 재발 방지` 형식으로 정리한 것입니다.

## 사례 1. SSH와 웹이 모두 시간 초과 (2026-11-18)

| 단계 | 내용 |
|---|---|
| **증상** | EC2를 띄운 직후 `ssh -i vpc-web-key.pem ubuntu@<퍼블릭IP>`가 `Operation timed out`. 브라우저 `http://<퍼블릭IP>`도 한참 돌다가 시간 초과. 휴대폰 LTE로 열어도 똑같이 시간 초과 |
| **가설** | ① 보안그룹 22번 소스(내 IP)가 지금 IP와 다르다 → ② 내 쪽 네트워크가 막는다 → ③ AWS 쪽 길(IGW, 라우팅 줄, 퍼블릭 IP) 중 한 칸이 빠졌다 |
| **검증** | ① `curl https://checkip.amazonaws.com` 결과가 22번 소스와 같음 → 기각. 80번은 `0.0.0.0/0`인데 웹도 시간 초과라 보안그룹은 원인이 아님 ② 다른 네트워크(휴대폰 LTE)에서도 같음 → 기각 ③ 서브넷 `vpc-web-public`의 라우팅 테이블 탭에 줄이 `10.0.0.0/16 → local` **하나뿐**이고 `0.0.0.0/0 → IGW`가 없음 ([화면](images/trouble-01-route-table-missing-igw.png)) |
| **조치** | 라우팅 편집 → `0.0.0.0/0 → vpc-web-igw` 추가 → **변경 사항 저장** |
| **결과** | 라우팅 2줄 모두 '활성' ([화면](images/trouble-02-route-table-fixed.png)). 브라우저가 바로 `ERR_CONNECTION_REFUSED`(Nginx 설치 전이라 정상, 요청이 EC2까지 도착) ([화면](images/trouble-03-browser-refused-before-nginx.png)), SSH 접속 성공 |
| **재발 방지** | 원인은 전날 라우팅 편집 화면에서 줄을 넣고 **저장을 누르지 않은 것**. 설정을 바꾼 뒤에는 편집 화면이 아니라 **적용된 화면**(서브넷의 라우팅 테이블 탭)에서 줄 수를 확인한다. "전부 시간 초과"는 포트와 상관없는 **길** 문제의 신호이므로 길 세 칸부터 본다 |

## 사례 2. 집에서 SSH만 시간 초과 (2026-11-19)

| 단계 | 내용 |
|---|---|
| **증상** | EC2를 다시 시작한 뒤(새 퍼블릭 IP) 집 랩탑에서 브라우저는 Nginx가 열리는데 SSH만 `Connection timed out` ([화면](images/trouble-04-ssh-timeout-from-home.png)) |
| **가설** | ① 키 파일을 나중에 설정해서 생긴 인증 문제 → ② 보안그룹 22번 줄이 지금 출발지를 허용하지 않는다 |
| **검증** | ① 키가 틀리면 서버가 `Permission denied (publickey)`로 **대답**한다. 지금은 대답 없는 시간 초과라 요청이 서버 프로그램까지 가지 못함 → 기각 ② 웹(80)은 되므로 길은 멀쩡함. 22번 줄 소스는 전날 **학습장에서 넣은 IP**인데 오늘 출발지는 **집 IP** → 맞는 줄 없음 |
| **조치** | 22번 소스를 '내 IP'(집 IP)`/32`로 교체. `0.0.0.0/0`으로 넓히지 않음 |
| **결과** | SSH 접속 성공. 호스트 지문이 전날과 같아 IP는 바뀌어도 같은 서버임을 확인 |
| **재발 방지** | 장소를 옮겨 접속하기 전에 22번 소스를 그때 IP로 바꾸고, README 기록표에 바꾼 시각을 적는다 |

## 증상으로 칸 찾기

| 막힌 곳 | 증상 |
|---|---|
| 길 (라우팅 줄, IGW, 퍼블릭 IP) | 브라우저와 SSH **모두** 시간 초과 |
| 보안그룹 | **그 포트만** 시간 초과, 다른 포트는 됨 |
| 서버 프로그램 (Nginx) | **즉시 거부** (connection refused) |
| IAM | 콘솔에 **빨간 배너**와 Action 이름 (예: `s3:CreateBucket`) |

일부러 막아 본 실험(라우팅 줄 삭제, 보안그룹 80번 줄 삭제)의 결과는 [troubleshooting-notes.md](troubleshooting-notes.md)에 있습니다.
