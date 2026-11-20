# AWS 클라우드 인프라 구축 및 트러블슈팅 — 미션 3-1

> AWS 서울 리전(ap-northeast-2)에서 VPC, Public Subnet, Internet Gateway, EC2, Security Group 및 IAM 최소권한을 적용하고, Nginx 웹 서비스의 외부 접속을 검증한 실습 기록.

## 실습 개요

| 구성 요소 | 설정 |
|---|---|
| Region | `ap-northeast-2` (서울) |
| VPC | `cloud-mission-vpc` — `10.0.0.0/16` |
| Public Subnet | `cloud-mission-public-subnet` — `10.0.1.0/24` (`ap-northeast-2a`) |
| Internet Gateway | `cloud-mission-igw` |
| Route Table | `cloud-mission-public-rt`; `10.0.0.0/16 → local`, `0.0.0.0/0 → IGW` |
| EC2 | `cloud-mission-ec2`, Amazon Linux 2023, `t3.micro`, EBS 8 GiB gp3 |
| Security Group | `cloud-mission-sg`: TCP 22 = 개인 IP /32, TCP 80 = `0.0.0.0/0` |
| 서버 | Nginx 1.30.7 |
| 인증 | IAM 사용자 `cloud-mission-user`, `CloudMissionLeastPrivilegePolicy` 사용 |
| 실습일 | 2026-11-20 |

## 외부 접속 검증 — 방식 A

- **검증 방식:** (A) 외부 브라우저에서 HTTP 페이지 열기
- **당시 확인 URL:** http://3.36.115.9
- **검증 결과:** 외부 Safari 브라우저에서 **Welcome to nginx!** 페이지 확인
- **주의:** 퍼블릭 IPv4는 인스턴스 중지/재시작 시 달라질 수 있으며, 실습 정리 후 URL은 더 이상 유효하지 않음

### 추가 확인

```bash
# EC2 SSH 로그인: 성공
ssh -i cloud-mission-key.pem ec2-user@3.36.115.9
# 서버 내부
curl -I http://localhost
# HTTP/1.1 200 OK
curl -I https://example.com
# HTTP/2 200
```

### 보안 그룹

| 구분 | 프로토콜/포트 | 허용 대상 | 사유 |
|---|---|---|---|
| SSH | TCP/22 | 실습 당시 개인 공인 IP `121.135.181.35/32` | 관리용 SSH 제한 |
| HTTP | TCP/80 | `0.0.0.0/0` | 외부 웹 서비스 접속 |

보안 그룹은 인스턴스 트래픽을 제어하고, IAM 정책은 AWS API/콘솔 작업 권한을 제어한다. `AdministratorAccess` 없이 실습 범위를 한정하는 IAM 정책을 생성했다. **정책의 실제 전체 작업 목록에 대한 최종 감사는 별도 확인 필요**.

## 제출 자료

- [아키텍처 다이어그램](docs/architecture.pdf) — 인터넷 → IGW → 라우팅 → Public Subnet → SG → EC2/Nginx 트래픽 구조
- [트러블슈팅 보고서](docs/troubleshooting.md) — 실제 CIDR 중복 오류 및 검증 내용
- [리소스 정리 체크리스트](docs/cleanup-checklist.md) — 삭제 후 실제 완료 여부를 체크할 것
- 외부 접속 증빙: `docs/screenshots/external-access.png` **추가 필요** (대화에서 확인된 실제 스크린샷을 이 경로에 업로드)

## 사용 명령

```bash
sudo dnf update -y
sudo dnf install nginx -y
sudo systemctl enable --now nginx
sudo systemctl status nginx --no-pager
curl -I http://localhost
curl -I https://example.com
```

## 운영 및 비용

EC2, EBS, 퍼블릭 IPv4 등은 사용량에 따라 요금이나 무료 크레딧 차감이 발생할 수 있다. **문서 작성만으로 리소스가 삭제되는 것은 아니다.** 반드시 [정리 체크리스트](docs/cleanup-checklist.md)에 따라 서울 리전의 실제 AWS 리소스를 삭제·검증해야 한다.

## 비밀 정보 관리

`.pem` 개인키, 비밀번호, 액세스 키, 세션 토큰은 레포에 올리지 않는다.
