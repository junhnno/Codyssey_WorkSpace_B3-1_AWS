# AWS 웹 서비스 인프라 구축 (VPC + EC2 + Nginx)

VPC로 격리된 네트워크를 직접 구성하고, EC2에 Nginx를 배포해 인터넷에서 접속 가능한 웹 서비스를 만든 과제입니다.
보안 그룹과 IAM 최소권한을 적용했고, 실습 종료 후 모든 리소스를 정리합니다.

## 1. 외부 접속 증빙

| 항목 | 내용 |
|---|---|
| 선택한 방식 | **A. 브라우저로 `http://<퍼블릭IP>` 접속** |
| 접속 정보 | `http://13.125.200.239` |
| 확인 결과 | 고정 문구 `Hello Cloud - Codyssey` 페이지가 표시됨 |
| 스크린샷 | `docs/images/external-access.png` |

> 퍼블릭 IP는 인스턴스 실행 중에만 유효하며, 실습 종료 후 리소스를 삭제하면 접속할 수 없습니다.

## 2. 환경

| 항목 | 값 |
|---|---|
| 리전 | 서울 (ap-northeast-2) |
| 인스턴스 | EC2 1대, t3.micro |
| OS | Ubuntu Server LTS |
| 웹 서버 | Nginx 1.28.3 |
| 스토리지 | EBS 루트 볼륨 (기본 8GiB, 종료 시 삭제) |
| 접속 | SSH (키페어 사용) |

## 3. 아키텍처

```
Internet
   │
Internet Gateway (igw-09189f3a55fa268b7)
   │   Route Table: 0.0.0.0/0 → IGW
Public Subnet (10.0.1.0/24, ap-northeast-2a)
   │   Security Group: 80 ← 0.0.0.0/0, 22 ← 내 IP/32
EC2 (Nginx)  사설 IP 10.0.1.x + 퍼블릭 IP
   (VPC 10.0.0.0/16)
```

다이어그램: `docs/architecture.png`

> `architecture.png`는 VPC, Subnet, Route Table, Internet Gateway의 역할과 트래픽 흐름을 설명하는 개념도입니다. 그림의 프라이빗 서브넷과 데이터베이스(`10.0.2.0/24`)는 설명용 예시이며 이번 실습에서는 만들지 않았습니다. 실제 구성(Security Group, EC2 포함)은 위 도식과 4장을 기준으로 합니다.

## 4. 구성 내용

### 네트워크
| 리소스 | 이름 | 설정 |
|---|---|---|
| VPC | Codyssey-VPC | `10.0.0.0/16`, IPv6 미사용 |
| Public Subnet | Codyssey-public-subnet | `10.0.1.0/24`, 퍼블릭 IPv4 자동 할당 활성화 |
| Internet Gateway | - | VPC에 Attached |
| Route Table | Codyssey-public-rt | `10.0.0.0/16 → local`, `0.0.0.0/0 → Internet Gateway`, Public Subnet에 명시적 연결 |

- 외부 요청은 퍼블릭 IP → Internet Gateway → Route Table → Public Subnet → Security Group → EC2 순서로 도달합니다.
- 인스턴스에서 `curl https://example.com` 같은 아웃바운드 통신도 같은 경로(`0.0.0.0/0 → IGW`)로 나갑니다. 예를 들어 `apt-get install nginx`로 패키지를 내려받으려면 이 경로가 있어야 합니다.
- 라우팅은 가장 구체적인(prefix가 긴) 경로가 우선합니다. VPC 내부 `10.0.0.0/16`은 `local`로, 그 외 모든 목적지는 `0.0.0.0/0 → IGW`로 갑니다. 이번 구성에는 프라이빗 서브넷이 없어 NAT Gateway는 만들지 않았습니다.

### 보안 그룹 (Codyssey-web-sg)

| 방향 | 프로토콜/포트 | 소스 | 이유 |
|---|---|---|---|
| 인바운드 | TCP 80 | `0.0.0.0/0` | 누구나 웹 접속 |
| 인바운드 | TCP 22 | 내 공인 IP `/32` | 관리용 SSH는 본인 IP에서만 |
| 아웃바운드 | 전체 | `0.0.0.0/0` | 패키지 설치 등 아웃바운드 통신 (기본값) |

- 0.0.0.0/0에 대한 전체 포트(0-65535) 인바운드 허용 규칙은 만들지 않았습니다.

### IAM
- 실습은 루트 계정이 아닌 IAM 사용자로 진행했습니다. (루트는 초기 설정에만 사용, MFA 적용)
- 사용자 그룹 `lab-ec2-vpc`에 AWS 관리형 정책 `AmazonEC2FullAccess`만 연결했습니다.
- AdministratorAccess, IAM, S3, RDS 등 과제와 무관한 권한은 부여하지 않았습니다.
- 참고: VPC, Subnet, Internet Gateway, Route Table, Security Group, Elastic IP, EBS는 콘솔에서는 별도 메뉴지만 API는 모두 `ec2:` 접두사를 쓰므로, `AmazonEC2FullAccess` 하나로 이 과제의 네트워크 구성까지 가능합니다. 반면 IAM, S3, RDS 등 다른 서비스는 접근되지 않습니다.
- 한계: 관리형 정책이라 필요한 동작과 리전으로 좁힌 고객 관리형 정책보다 범위가 넓습니다. 서울 리전 제한도 정책에는 없어서 작업 시 리전을 직접 확인했습니다. 실무에서는 필요한 동작과 리전으로 좁힌 정책을 사용합니다.

### Security Group과 IAM의 역할 차이

| | Security Group | IAM |
|---|---|---|
| 제어 대상 | 인스턴스로 들어오고 나가는 **네트워크 트래픽** | AWS **API 호출 권한** (누가 어떤 리소스에 어떤 작업을 할 수 있는가) |
| 판단 기준 | 프로토콜, 포트, 출발지/목적지 IP | 사용자/역할, 허용된 작업(Action), 대상 리소스 |
| 예 | 80은 모두 허용, 22는 내 IP만 허용 | 이 사용자는 EC2/VPC 작업만 가능하고 S3/RDS는 불가 |
| 못 막는 것 | 인스턴스를 삭제하려는 API 호출 | 80번 포트로 들어오는 웹 요청 |

- 한 문장 요약: **SG는 "어떤 트래픽이 서버에 닿는가", IAM은 "누가 AWS 리소스를 만들고 바꿀 수 있는가"를 제어합니다.** 둘은 서로 대체할 수 없어서 함께 적용했습니다.
- 최소권한 확대 절차(일반론, 이번 실습에는 적용하지 않음): 읽기 전용 권한으로 시작 → 작업에 필요한 동작만 단계적으로 추가 → CloudTrail 기록과 IAM Access Analyzer로 실제 사용한 권한만 남기고 축소.

### 네이밍 규칙과 태그

- 규칙: `프로젝트-역할-리소스종류` (예: `Codyssey-public-subnet`). 다음 실습부터는 환경(dev/prod)을 넣은 `프로젝트-환경-리소스` 형식으로 통일합니다. (예: `Codyssey-dev-web-sg`)
- 이번 실습의 리소스 이름:

| 리소스 | 이름 |
|---|---|
| VPC | `Codyssey-VPC` |
| Subnet | `Codyssey-public-subnet` |
| Route Table | `Codyssey-public-rt` |
| Security Group | `Codyssey-web-sg` |
| EC2 | `Codyssey-web` |

- 이번 실습에서는 `Name` 태그로 리소스를 구분했습니다. 일부 이름은 위 규칙을 완전히 따르지 않았습니다 (`Codyssey-VPC`, `Codyssey-web`에는 역할/종류 접미사가 없음).
- 실무 적용 시 권장하는 태그 필드 예시 (이번 실습 리소스에는 적용하지 않음):

| 태그 키 | 예시 값 | 용도 |
|---|---|---|
| `Name` | `Codyssey-dev-web` | 콘솔에서 식별 |
| `Project` | `Codyssey` | 프로젝트별 비용 집계 |
| `Env` | `dev` | 환경 구분 |
| `Owner` | `junhnno` | 담당자 식별, 정리 대상 추적 |

### 과금 요인과 비용 추적

| 과금 요인 | 실행 중 | 중지 상태 | 이번 구성 |
|---|---|---|---|
| EC2 인스턴스 | 시간당 과금 (서울 t3.micro 약 $0.013/h) | 과금 없음 | 사용 |
| EBS 볼륨 | 용량 기준 과금 | **중지해도 과금** | 사용 (8GiB) |
| 퍼블릭 IPv4 | 시간당 약 $0.005 | 자동 할당 IP는 반납됨 | 자동 할당 사용 |
| Elastic IP | 위와 동일, 인스턴스에 붙어 있지 않아도 과금 | 중지 인스턴스에 붙어 있어도 과금 | 사용하지 않음 |
| NAT Gateway / ELB / RDS | 시간당 과금 | - | 생성하지 않음 |

- 단가는 2026-11 확인한 참고값이며 변동될 수 있습니다.
- 비용 추적: AWS Budgets로 소액 임계값(예: $1) 알림 설정, Cost Explorer로 서비스별 확인, 태그(`Project`)를 비용 할당 태그로 활성화하면 프로젝트별 집계가 가능합니다. 실습 종료 후 Billing Dashboard 확인은 [`cleanup-checklist.md`](docs/cleanup-checklist.md)에 포함했습니다.

### 확장 포인트 (이번 과제에서는 구현하지 않음)

- **수직 확장**: t3.micro의 CPU/메모리가 부족하면 더 큰 인스턴스 타입으로 변경 (인스턴스 중지 후 변경).
- **수평 확장**: 인스턴스를 여러 대로 늘리고 ALB를 앞에 두는 구조. 여러 가용 영역에 분산하면 장애 대응에도 유리합니다.
- **ALB 적용 판단 기준**: 인스턴스가 2대 이상이 될 때, 무중단 배포나 헬스체크 기반 트래픽 분산이 필요할 때, TLS를 앞단에서 종료해야 할 때. 단일 인스턴스 실습에서는 ALB의 시간당 과금이 이점보다 커서 만들지 않았습니다.
- **병목 확인**: CloudWatch의 `CPUUtilization`, T 계열은 `CPUCreditBalance`를 확인하고, Nginx `access.log`로 요청량을 봅니다.

### 공개 포트 모니터링 (제안, 이번 실습에는 미적용)

- 80번처럼 공개된 포트는 접근 기록을 남기는 것이 좋습니다. VPC Flow Logs로 ACCEPT/REJECT 트래픽을 기록하고, CloudWatch Logs/알람과 연동해 비정상 급증을 탐지할 수 있습니다.
- 인스턴스 수준에서는 Nginx `access.log`와 `error.log`를 확인합니다.
- SG 규칙이 의도치 않게 바뀌는 것은 CloudTrail 기록으로 추적할 수 있습니다.

## 5. 검증 결과

| 확인 항목 | 방법 | 결과 |
|---|---|---|
| SSH 접속 | `ssh -i <키>.pem ubuntu@13.125.200.239` | 성공 |
| 웹 서버 실행 | `systemctl status nginx` | 시작 로그 확인 |
| localhost 응답 | `curl -I http://localhost` | `HTTP/1.1 200 OK` |
| 아웃바운드 통신 | EC2에서 `curl -I https://example.com` | `HTTP/2 200` (`docs/images/outbound-curl.png`) |
| 외부 접속 | 브라우저에서 `http://13.125.200.239` | 페이지 표시 (섹션 1) |

## 6. 트러블슈팅

실제로 겪은 오류와 해결 과정은 [`docs/troubleshooting.md`](docs/troubleshooting.md)에 정리했습니다.

- IAM 사용자 콘솔 로그인 실패 (계정 이름을 계정 ID/별칭 칸에 입력)
- SSH/HTTP `Connection timed out` (라우팅 테이블의 서브넷 연결 누락)

## 7. 리소스 정리

과금 방지를 위해 실습 종료 후 EC2, EBS, Internet Gateway, Subnet, Route Table, Security Group, VPC를 삭제합니다.
삭제 순서와 확인 근거는 [`docs/cleanup-checklist.md`](docs/cleanup-checklist.md)에 기록합니다.

## 8. 파일 구성

```
README.md
docs/
├── architecture.png
├── troubleshooting.md
├── cleanup-checklist.md
└── images/
    ├── external-access.png
    └── outbound-curl.png
```
