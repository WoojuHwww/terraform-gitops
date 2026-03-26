# Terraform AWS Infrastructure Project

Terraform로 AWS 인프라를 코드로 관리하고, GitHub Actions OIDC 기반으로 변경 사항을 검증 및 반영할 수 있도록 구성한 인프라 프로젝트입니다.  
단순 리소스 생성보다도, 실제 운영을 고려한 디렉터리 분리, 원격 상태 관리, 모듈화, 접근 제어 흐름을 설계하는 데 초점을 맞췄습니다.

## Overview

이 프로젝트는 `dev` 환경 기준으로 다음 인프라를 구성합니다.

- VPC, Public / Private App / Private DB 서브넷 분리
- NAT Gateway, Route Table을 포함한 네트워크 계층 구성
- 운영 접속용 Bastion Host
- EKS 클러스터와 Managed Node Group
- Private Subnet 기반 RDS
- 애플리케이션 이미지 저장용 ECR
- 업로더 권한과 공개 조회 정책을 분리한 S3 버킷
- Terraform remote state용 S3 + DynamoDB bootstrap 구성
- GitHub Actions와 AWS OIDC 연동을 통한 CI 기반 Terraform 실행

## Architecture

전체 구조는 크게 네 가지 층으로 나뉩니다.

1. **Bootstrap Layer**
   Terraform state를 로컬 파일이 아니라 AWS S3와 DynamoDB에서 안전하게 관리하기 위한 초기 리소스를 생성합니다.

2. **Environment Layer**
   `environments/dev`에서 실제 개발 환경 인프라를 조합합니다. 환경별 설정과 공통 모듈을 분리해 재사용성을 높였습니다.

3. **Module Layer**
   네트워크, 컴퓨팅, 데이터베이스, 레지스트리, 스토리지, IAM을 각각 독립 모듈로 구성해 유지보수성과 확장성을 확보했습니다.

4. **Delivery Layer**
   GitHub Actions가 Terraform `fmt`, `validate`, `plan`, `apply`를 수행하며, AWS 자격 증명은 Access Key 대신 OIDC 기반 Assume Role 방식으로 연결합니다.

## Key Design Points

### 1. Modular Terraform Structure

리소스를 한 파일에 몰아넣지 않고 역할별 모듈로 분리했습니다.

- `modules/vpc`: VPC, 서브넷, IGW, NAT Gateway, Route Table
- `modules/bastion`: 운영 접속용 EC2 및 보안 그룹
- `modules/eks`: EKS 클러스터, Node Group, 보안 그룹, 접근 정책
- `modules/rds`: DB Subnet Group, Parameter Group, Option Group, RDS 인스턴스
- `modules/ecr`: 컨테이너 이미지 레포지토리와 Lifecycle Policy
- `modules/s3`: 업로드 정책과 공개 읽기 정책을 가진 S3 버킷
- `modules/iam`: EKS OIDC 기반 Cluster Autoscaler용 IAM 리소스

### 2. Network Segmentation

퍼블릭과 프라이빗 계층을 분리해 역할에 맞는 네트워크 경계를 구성했습니다.

- Public Subnet: Bastion, NAT Gateway
- Private App Subnet: EKS Node Group
- Private DB Subnet: RDS

이 구조를 통해 외부 노출이 필요한 리소스와 내부 전용 리소스를 분리하고, DB는 Bastion 및 EKS 노드 보안 그룹만 접근할 수 있도록 제한했습니다.

### 3. Remote State Management

`bootstrap/backend`는 Terraform state 관리 전용 영역입니다.

- S3 Bucket: state 파일 저장
- DynamoDB Table: state lock 관리
- 버전 관리 및 서버 측 암호화 적용

Terraform 프로젝트를 실제 협업 가능한 형태로 운영하기 위해 가장 먼저 필요한 기반을 별도 레이어로 분리한 점이 이 프로젝트의 중요한 설계 포인트입니다.

### 4. GitHub Actions + OIDC

GitHub Actions 워크플로우는 `environments/dev`와 `modules` 변경을 감지해 Terraform 검증을 수행합니다.

- Pull Request: `terraform plan`
- Main 브랜치 Push: `terraform plan`
- Manual Dispatch: `plan` 또는 `apply`

AWS 인증은 장기 Access Key를 저장하지 않고, GitHub Actions가 OIDC로 IAM Role을 Assume 하도록 구성했습니다.  
즉, 인프라 자동화와 보안 요구사항을 함께 고려한 구조입니다.

## Directory Structure

```text
.
|-- .github/
|   `-- workflows/
|       |-- terraform-dev.yml
|       `-- terraform-destroy.yml
|-- bootstrap/
|   `-- backend/
|       |-- main.tf
|       |-- variables.tf
|       `-- outputs.tf
|-- environments/
|   `-- dev/
|       |-- backend.tf
|       |-- main.tf
|       |-- variables.tf
|       `-- outputs.tf
|-- modules/
|   |-- bastion/
|   |-- ecr/
|   |-- eks/
|   |-- iam/
|   |-- rds/
|   |-- s3/
|   `-- vpc/
`-- README.md
```

## Provisioned Resources

### VPC

- DNS hostnames / DNS support 활성화
- Public / Private App / Private DB 서브넷 분리
- Internet Gateway 및 NAT Gateway 구성
- EKS 연동을 위한 subnet tagging 적용

### Bastion

- Public Subnet에 배치된 Amazon Linux 기반 EC2
- 허용된 CIDR에 대해서만 SSH 접근 허용
- Private RDS 접근을 위한 운영 점프 서버 역할

### EKS

- 별도 IAM Role을 가진 EKS Cluster
- Launch Template 기반 Managed Node Group
- Access Entry와 Cluster Admin Policy Association 구성
- Cluster Autoscaler 연동을 고려한 태그 부여

### RDS

- Private DB Subnet에 배치
- DB Subnet Group, Parameter Group, Option Group 지원
- Bastion / EKS Node Security Group만 접근 가능
- 백업, 성능 옵션, 삭제 보호 등 운영 속성 확장 가능

### ECR

- 복수 레포지토리 생성 가능
- Push 시 이미지 스캔
- Lifecycle Policy를 통한 이미지 정리 자동화

### S3

- 서버 측 암호화 적용
- 특정 업로더 ARN에 `ListBucket`, `PutObject`, `DeleteObject` 권한 부여
- Object Public Read 정책 구성

## Workflow

이 레포는 다음 순서로 운영되도록 설계되었습니다.

1. `bootstrap/backend`에서 remote state 인프라를 먼저 생성
2. `environments/dev`에서 실제 개발 환경 인프라 구성
3. GitHub Actions가 변경 사항을 검증하고 필요 시 수동 적용
4. 애플리케이션은 ECR, EKS, S3 등과 연동해 배포 기반을 활용

## What This Repository Shows

포트폴리오 관점에서 이 레포는 다음 역량을 보여주기 위한 프로젝트입니다.

- Terraform 모듈 설계 및 환경 분리 능력
- AWS 네트워크, 컴퓨팅, 데이터베이스 리소스 설계 경험
- 원격 state 및 협업 가능한 IaC 구조에 대한 이해
- GitHub Actions와 OIDC를 활용한 안전한 인프라 자동화 구성
- 단순 실습이 아니라 운영 관점의 보안 경계와 배포 흐름을 고려한 설계

## Future Improvements

- `staging`, `prod` 환경 추가
- Helm / ArgoCD 기반 GitOps 확장
- Monitoring stack(Prometheus, Grafana) 연동
- Secrets 관리 체계 개선
- 재사용 가능한 공통 CI 파이프라인 템플릿화

## Tech Stack

- Terraform
- AWS
- Amazon VPC
- Amazon EKS
- Amazon RDS
- Amazon ECR
- Amazon S3
- AWS IAM / OIDC
- GitHub Actions
