# AWS 리소스 정리 체크리스트

> **현재 상태 (2026-11-21): 평가 대기 중 — 리소스 유지, 삭제 미실시.** 제출·평가자가 서버를 확인할 수 있도록 인스턴스를 유지하기로 했다. 모든 삭제 항목은 실제 AWS 콘솔에서 삭제 후에만 체크하며, 모든 리소스는 서울 리전(`ap-northeast-2`)에서 확인한다. EC2/EBS/공인 IPv4의 사용 요금 또는 크레딧 차감이 이어질 수 있으므로 예산 알림 및 비용을 점검할 것.

## 실습 리소스 목록

| 항목 | 생성 당시 정보 | 정리 상태 |
|---|---|---|
| EC2 | `cloud-mission-ec2` / `i-03b338bb34eb1bf11` | [ ] 확인 필요 |
| EBS gp3 | EC2 루트 8 GiB (볼륨 ID는 콘솔에서 확인) | [ ] 확인 필요 |
| Elastic IP | 별도 할당하지 않음으로 관찰됨. EIP 목록 재확인 | [ ] 확인 필요 |
| Public IPv4 | 당시 `3.36.115.9` (자동 할당, 삭제 후 재사용 불가) | [ ] 확인 필요 |
| Security Group | `cloud-mission-sg` | [ ] 확인 필요 |
| Public Route Table | `cloud-mission-public-rt` / `rtb-0d43d635bd21b8278` | [ ] 확인 필요 |
| Subnet | `cloud-mission-public-subnet` / `subnet-03b4e6fc304ef2d73` | [ ] 확인 필요 |
| Internet Gateway | `cloud-mission-igw` / `igw-0ec0d51aee9062b57` | [ ] 확인 필요 |
| VPC | `cloud-mission-vpc` / `vpc-082a156874c8d882c` | [ ] 확인 필요 |
| NAT Gateway, ELB/ALB, RDS | 생성한 기록 없음. 리전별 실제 목록 확인 | [ ] 확인 필요 |

## 안전한 삭제 순서

- [ ] README 및 스크린샷 증빙 저장 완료
- [ ] EC2 > 인스턴스 > `cloud-mission-ec2` > **인스턴스 종료(Terminate)** 실행, `Terminated` 확인. 단순 Stop은 과금 리소스를 모두 없애지 못함
- [ ] EC2 > 볼륨에서 루트 EBS가 삭제되었는지 확인. `available`로 남아 있는 실습용 볼륨이 있다면 연결 관계와 소유를 확인한 뒤 삭제
- [ ] EC2 > 탄력적 IP(Elastic IP)에서 실습 EIP가 남아 있다면 연결 해제 후 **주소 릴리스(Release)**. 별도 할당이 없었다면 없음으로 기록
- [ ] EC2 > 로드 밸런서 / VPC > NAT Gateway / RDS에서 실습용 리소스 유무 점검. 있다면 의존성 및 추가 리소스를 확인하여 먼저 삭제
- [ ] VPC > 보안 그룹에서 `cloud-mission-sg` 삭제 (EC2 네트워크 인터페이스 참조가 사라진 뒤)
- [ ] VPC > 라우팅 테이블에서 `cloud-mission-public-rt`의 서브넷 명시적 연결 해제, 비기본 라우팅 테이블 삭제
- [ ] VPC > 서브넷에서 `cloud-mission-public-subnet` 삭제
- [ ] VPC > 인터넷 게이트웨이에서 `cloud-mission-igw`를 VPC에서 분리(Detach)한 뒤 삭제
- [ ] VPC > VPC에서 **실습용** `cloud-mission-vpc`만 삭제 (기존 기본 VPC는 삭제하지 않음)
- [ ] 서울 리전 EC2, EBS, EIP, NAT Gateway, ALB, VPC 관련 리소스가 남지 않았는지 재확인
- [ ] Billing & Cost Management에서 비용 및 크레딧 차감 확인. Billing 지표는 반영이 지연될 수 있으므로 현재 미표시가 최종 0원 보장은 아님

## 평가 대기 중 운영·비용 점검

- [ ] 평가 기간 중 EC2가 `running`이고 Nginx `/`가 접속 가능한지 점검
- [ ] 비용/크레딧 현황을 **Billing & Cost Management**에서 확인 (권한이 없다면 계정 관리자가 확인)
- [ ] **AWS Budgets**의 예산 임계값과 알림 수신 이메일 설정 검토 (권한 및 플랜 제약 확인)
- [ ] `t3.micro` 사용시간, EBS gp3 8GiB, 퍼블릭 IPv4 사용시간 점검
- [ ] NAT Gateway, ALB, EIP, 스냅샷 등 추가 과금 리소스 생성 여부 점검
- [ ] 평가 완료 후 EC2와 연관 리소스 정리 실행 및 결과 캡처

### 확장 시 고려사항 (현재 미도입)

단일 EC2가 병목이 되면 CloudWatch CPUUtilization, CPUCreditBalance, OS 메모리, EBS I/O, Nginx 요청량/오류율과 응답시간을 확인한다. ALB를 도입한다면 두 개 이상의 가용 영역에 서브넷을 마련하고 Target Group 헬스 체크를 설정하는 등 추가 구성이 필요하며, ALB 사용료와 트래픽 관련 비용을 별도로 예산에 반영해야 한다. 이번 실습 요구사항은 EC2 1대와 Public Subnet 1개이므로 확장 구성은 **실제로 생성하지 않았다.**

## 실제 정리 결과 기록 (삭제 후 작성)

| 점검 일시(KST) | 수행 항목 | 최종 화면/근거 | 결과 |
|---|---|---|---|
| 2026-11-21 | 평가를 위해 서버와 네트워크 유지 | 사용자 확인: 삭제 보류 | **평가 대기 / 미완료** |

> 과금 방지를 위해 EC2와 퍼블릭 IPv4의 사용시간이 계속 증가하지 않도록 실습을 마친 즉시 정리할 것.
