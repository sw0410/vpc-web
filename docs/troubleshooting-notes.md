# 트러블슈팅 메모 (제출용 troubleshooting.md 재료)

## 2026-11-18 SSH·웹 모두 시간 초과

| 시각 | 단계 | 내용 |
|---|---|---|
| 17:29 | 증상 | `ssh -i vpc-web-key.pem ubuntu@퍼블릭IP` 무응답 → `Operation timed out` (port 22) |
| 17:30 | 가설 1 | 보안그룹 22번 소스(내 IP)와 지금 IP가 다르다 |
| 17:30 | 검증 1 | `curl https://checkip.amazonaws.com` = 22번 소스와 같음 → 가설 1 기각 |
| 17:31 | 검증 2 | 브라우저 `http://퍼블릭IP` (80은 0.0.0.0/0 허용, Nginx 설치 전) → 한참 돌다 시간 초과. 거부가 아니라 시간 초과 → 요청이 EC2까지 못 감. 80은 모두에게 열려 있으니 보안그룹은 원인 아님 |
| 17:31 | 가설 2 | A: 내 쪽(공용 컴퓨터 건물) 네트워크가 막음 / B: AWS 길 세 칸(IGW, 라우팅 줄, 퍼블릭 IP) 중 하나가 빠짐 |
| 17:33 | 검증 3 | 휴대폰 LTE로 `http://퍼블릭IP` → 역시 한참 돌다 시간 초과 → 다른 네트워크에서도 같음 → 가설 A 기각, B(AWS 길 세 칸)로 좁힘 |
| 17:34 | 검증 4 | 서브넷 쪽에서 거슬러 점검: 서브넷 vpc-web-public의 라우팅 테이블 탭 → 우리 표(vpc-web-public-rt)와 연결은 됨, 그런데 줄이 `10.0.0.0/16 → local` 하나뿐. `0.0.0.0/0 → IGW` 없음 → 응답이 인터넷으로 갈 줄이 없어 버려짐 |
| 17:34 | 원인 | 11-17 라우팅 편집 화면에서 줄을 넣고 **변경 사항 저장**을 누르지 않은 것으로 보임(편집 화면 캡처만 있었음) |
| 17:35 | 조치 | 라우팅 편집 → `0.0.0.0/0 → vpc-web-igw` 추가 → 저장 → 라우팅 (2), 두 줄 모두 상태 활성 |
| 17:36 | 결과 | 브라우저 `http://퍼블릭IP` → 바로 ERR_CONNECTION_REFUSED(Nginx 설치 전이라 정상: 요청이 EC2까지 감). SSH → 호스트 확인 질문(yes/no)까지 옴 = 22번 왕복 성공 |
| | 재발 방지 | 설정 저장 뒤 편집 화면이 아니라 **적용된 화면**(서브넷의 라우팅 테이블 탭)에서 줄 수를 확인한다 |

증빙 스크린샷: [줄 하나뿐인 표](images/trouble-01-route-table-missing-igw.png) → [고친 뒤 두 줄 활성](images/trouble-02-route-table-fixed.png) → [Nginx 설치 전 즉시 거부](images/trouble-03-browser-refused-before-nginx.png)

## 2026-11-18 일부러 끊어 보기 (라우팅 줄 삭제 실험)

| 확인 | 예측 | 실제 |
|---|---|---|
| 브라우저 새로고침 | 안 됨 | 시간 초과 (ERR_CONNECTION_TIMED_OUT) — [스크린샷](images/exp-02-browser-timed-out.png) |
| 열어 둔 SSH 창에서 엔터 | 안 나옴 | 멈춘 뒤 `Operation timed out`, 연결 끊김. 줄 복구 뒤에도 되살아나지 않음 — [스크린샷](images/exp-01-open-ssh-timed-out.png) |
| 새 SSH 접속 | 안 됨 | 시간 초과 |
| 줄 복구 뒤 | — | 브라우저 Nginx 페이지, 새 SSH 접속 성공 — [브라우저](images/exp-03-browser-restored.png), [SSH](images/exp-04-new-ssh-restored.png) |

정리: 라우팅 줄에는 목적지와 타겟만 있고 포트가 없다. 그래서 길이 빠지면 80과 22가 함께, 모두 시간 초과가 된다. 거부(즉시)가 아니라 시간 초과인 것은 패킷이 버려질 뿐 아무도 거절 답장을 보내지 않기 때문이다.
