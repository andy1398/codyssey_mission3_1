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

### 외부 접속 증빙 스크린샷

![external-access](https://github.com/user-attachments/assets/eb089327-d5fc-40a8-851e-172fa8839458)

[증빙 이미지 원본 보기](https://github.com/user-attachments/assets/eb089327-d5fc-40a8-851e-172fa8839458)
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

보안 그룹은 인스턴스 트래픽을 제어하고, IAM 정책은 AWS API/콘솔 작업 권한을 제어한다. `AdministratorAccess` 없이 실습 범위를 한정하는 IAM 정책을 생성했다. 2026-11-21에 확인한 사용자 지정 정책 JSON 기준으로, `ec2:Describe*` 및 실습에 필요한 생성·연결·보안 그룹·인스턴스·태그·정리 작업을 허용하고 `aws:RequestedRegion = ap-northeast-2` 조건을 적용했다. **다른 사용자·그룹 연결 정책까지 포함한 유효 권한 전체 감사 및 리소스별 세분화는 수행하지 않았다.**

## IAM 최소권한 검토

- 정책: `CloudMissionLeastPrivilegePolicy` (그룹 `cloud-mission-group`에 연결)
- EC2 서비스 관련 조회·생성·수정·삭제를 명시적으로 허용하며 서울 리전 조건을 적용함
- S3, RDS, `AdministratorAccess` 허용 문은 **확인한 JSON에 없음**
- `ec2:Describe*` 및 `Resource: "*"`는 일부 조회에 필요할 수 있지만, 자원 단위의 최협의 최소권한은 아님
- 권한 관리 역할과 네트워크 보안 그룹의 역할이 다름: IAM은 AWS API 조작 권한, Security Group은 서버 TCP 트래픽 접근 제어

## 제출 자료

- [아키텍처 다이어그램](docs/architecture.pdf) — 인터넷 → IGW → 라우팅 → Public Subnet → SG → EC2/Nginx 트래픽 구조
- [트러블슈팅 보고서](docs/troubleshooting.md) — 실제 CIDR 중복 오류 및 검증 내용
- [리소스 정리 체크리스트](docs/cleanup-checklist.md) — 삭제 후 실제 완료 여부를 체크할 것
- 외부 접속 증빙: [README에 삽입된 GitHub 이슈 첨부 이미지](https://github.com/user-attachments/assets/eb089327-d5fc-40a8-851e-172fa8839458) (저장소 내부의 별도 PNG 파일이 아닌 GitHub 첨부 이미지 URL 사용)

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

**2026-11-21 현재 평가 대기 중으로 인스턴스를 유지하기로 결정했다.** EC2, EBS, 퍼블릭 IPv4 등은 사용량에 따라 요금이나 무료 크레딧 차감이 발생할 수 있다. **문서 작성만으로 리소스가 삭제되는 것은 아니다.** 반드시 [정리 체크리스트](docs/cleanup-checklist.md)에 따라 서울 리전의 실제 AWS 리소스를 삭제·검증해야 한다.

## 비밀 정보 관리

`.pem` 개인키, 비밀번호, 액세스 키, 세션 토큰은 레포에 올리지 않는다.
