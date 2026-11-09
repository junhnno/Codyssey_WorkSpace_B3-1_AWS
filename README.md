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

## 4. 구성 내용

### 네트워크
| 리소스 | 이름 | 설정 |
|---|---|---|
| VPC | Codyssey-VPC | `10.0.0.0/16`, IPv6 미사용 |
| Public Subnet | Codyssey-public-subnet | `10.0.1.0/24`, 퍼블릭 IPv4 자동 할당 활성화 |
| Internet Gateway | - | VPC에 Attached |
| Route Table | Codyssey-public-rt | `10.0.0.0/16 → local`, `0.0.0.0/0 → Internet Gateway`, Public Subnet에 명시적 연결 |

- 외부 요청은 퍼블릭 IP → Internet Gateway → Route Table → Public Subnet → Security Group → EC2 순서로 도달합니다.
- 인스턴스에서 `curl https://example.com` 같은 아웃바운드 통신도 같은 경로(`0.0.0.0/0 → IGW`)로 나갑니다.

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
- 한계: 관리형 정책이라 필요한 동작과 리전으로 좁힌 고객 관리형 정책보다 범위가 넓습니다. 서울 리전 제한도 정책에는 없어서 작업 시 리전을 직접 확인했습니다. 실무에서는 필요한 동작과 리전으로 좁힌 정책을 사용합니다.

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
