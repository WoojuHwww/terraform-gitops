# 🚀 Terraform GitOps Project

> **Last Updated:** 2026-03-18
> Terraform + AWS + GitHub Actions(OIDC)를 활용한 **GitOps 기반 인프라 자동화 프로젝트**

---

## 📌 프로젝트 개요

이 프로젝트는 다음을 목표로 합니다:

* Terraform을 통한 AWS 인프라 코드화 (IaC)
* GitHub Actions + OIDC 기반 **보안 자동 배포**
* S3 + DynamoDB를 활용한 **State 관리 및 Locking**
* GitOps 방식으로 **인프라 변경 이력 관리**

---

## ⚠️ 0. 시작 전 주의사항

* `terraform.tfvars` 파일의 값을 **본인 환경에 맞게 수정**해야 합니다.
* Terraform 실행 위치:

  ```bash
  environments/dev/*
  ```
* Bastion Host 설정:

  * 본인의 **공인 IP**
  * 생성한 **Key Pair 이름**
    → 반드시 수정 필요

* bootstrap/backend/terraform.tfvars 설정:

  * s3 버킷 이름 **본인 ID or 본인이 사용할 버킷 이름으로 변경**

* .github/workflows 파일 설정:

  * workflow 설정 **본인 github 및 oidc 설정 이후 role에 맞게 설정**

---

## 🏗️ 1. Backend 인프라 구성 (State 관리)

Terraform State를 안전하게 관리하기 위한 구성입니다.

### 📦 구성 요소

* **S3 Bucket**

  * Terraform State 저장
* **DynamoDB Table**

  * State Lock 관리 (동시 작업 충돌 방지)

---

### 🔄 작업 순서

1. `bootstrap` 폴더에서 초기 인프라 생성
2. S3 Bucket 생성 (State 저장용)
3. DynamoDB Table 생성 (Lock 관리)
4. `environments/dev`에 backend 설정 추가
5. `terraform init` 실행 (State Migration)

---

### 💡 State Migration 설명

* 기존: 로컬 `tfstate`
* 변경: S3 기반 원격 저장

👉 이후:

* 로컬 & GitHub Actions가 동일한 State 공유

---

### ⚙️ 실행 명령어

```bash
cd bootstrap/backend

terraform init
terraform fmt -recursive
terraform validate
terraform apply -var-file="terraform.tfvars"
```

> ⚠️ 이미 로컬에서 인프라를 생성한 경우에만 Migration 필요

---

## 🔐 2. OIDC 인증 설정 (GitHub Actions)

AWS Access Key 없이 GitHub Actions에서 AWS 접근을 가능하게 합니다.

---

### 🔁 인증 흐름

```
GitHub Actions 실행
→ OIDC 토큰 발급
→ AWS IAM 검증
→ IAM Role Assume
→ 임시 자격증명 발급
```

---

### 🧩 설정 단계

#### 1. OIDC Provider 생성

* Provider URL:

  ```
  token.actions.githubusercontent.com
  ```
* Audience:

  ```
  sts.amazonaws.com
  ```

---

#### 2. IAM Role 생성 (Trust Policy)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GitHubActionsOIDC",
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::398875891485:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": [
            "repo:WoojuHwww/terraform-gitops:ref:refs/heads/main",
            "repo:WoojuHwww/terraform-gitops:pull_request"
          ]
        }
      }
    }
  ]
}
```

---

### 📌 핵심 설정 요약

* GitHub OIDC Provider를 신뢰
* `sts.amazonaws.com` Audience만 허용
* 특정 Repository만 허용
* `main` 브랜치 및 PR 이벤트만 허용

---

## ⚙️ 3. Terraform 운영 명령어

### 🎯 특정 리소스만 생성 (Target Apply)

```bash
terraform apply -target=module.vpc -var-file="terraform.tfvars"
terraform apply -target=module.rds -var-file="terraform.tfvars"
terraform apply -target=module.bastion -var-file="terraform.tfvars"
terraform apply -target=module.eks -var-file="terraform.tfvars"
terraform apply -target=module.ecr -var-file="terraform.tfvars"
```

---

## 🛢️ 4. RDS 접속 방법

### 📍 Step 1. Bastion Host 접속

* Putty 또는 SSH 사용

---

### 📦 Step 2. MySQL Client 설치

```bash
sudo dnf install -y wget

wget https://repo.mysql.com/mysql80-community-release-el9-1.noarch.rpm

sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm
sudo dnf install -y mysql-community-client
```

---

### 🔌 Step 3. RDS 접속

```bash
mysql -h dev-mysql.cxisck4ewfvd.ap-northeast-2.rds.amazonaws.com \
      -P 3306 \
      -u admin \
      -p
```

---

## 🧠 추가 참고

* GitOps 기반으로 모든 인프라 변경은 **코드 → PR → Apply** 흐름으로 관리
* State 충돌 방지를 위해 반드시 DynamoDB Lock 사용
* OIDC를 통해 **Access Key 없이 안전한 인증 구조 유지**

---

## 📎 디렉터리 구조 (예시)

```
.
├── bootstrap/
│   └── backend/
├── environments/
│   └── dev/
├── modules/
├── .github/workflows/
└── terraform.tfvars
```

---

## ✅ TODO (확장 가능)

* [ ] Helm Chart 연동
* [ ] ArgoCD GitOps 확장
* [ ] Monitoring (Prometheus / Grafana)
* [ ] CI/CD Pipeline 고도화

---
