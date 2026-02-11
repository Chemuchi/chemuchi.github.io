---
title: 오늘한일 정리
date: "2026-02-11"
draft: false
tags:
description:
---
## DEV 환경을 위한 ECR을 테라폼으로 구축
개발(Dev) 환경 구축을 위해 AWS의 Route53과 ECR을 도입했다.

서버는 팀원의 미니 PC를 활용하며, ECR은 개발 환경에서 사용할 이미지를 저장하고 관리하는 용도로 사용한다.  
CI 파이프라인을 통해 빌드가 완료되면 이미지를 ECR에 푸시하도록 구성했다.

전체적인 프로젝트 구조는 다음과 같다.

```plaintext
terraform/
├── environments/
│   └── dev/
│   └── shared/
│       └── iam.tf
└── modules/
    ├── ecr/
    └── route53/
```

### ECR 모듈 구현
ECR의 설정과 생애주기 정책은 다음과 같이 설정하였다.

```hcl
# ECR Repository 생성
resource "aws_ecr_repository" "this" {
  name = var.repository_name
  image_tag_mutability = "IMMUTABLE"
  force_delete = false

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "KMS"
  }

  tags = var.tags
}

# ECR 생애주기 정책
resource "aws_ecr_lifecycle_policy" "this" {
  policy     = jsonencode({
    rules = [
      {
        rulePriority = 1
        description = "최근 50개의 이미지만 유지"
        selection = {
          tagStatus = "any"
          countType = "imageCountMoreThan"
          countNumber = 50
        }
        action = {
          type = "expire"
        }
      }
    ]
  })
  repository = aws_ecr_repository.this.name
}
```

### Dev 환경 테라폼 세팅 및 ECR 생성

Dev 환경에 대한 테라폼 코드를 처음으로 작성하기 떄문에 백엔드를 연결하고, ECR 을 생성하였다.

```hcl
# Dev 환경 테라폼 설정
terraform {
  backend "s3" {
    bucket = "goormgb-tf-state-bucket"
    key = "dev/terraform.tfstate"
    region = "ap-northeast-2"
    dynamodb_table = "goormgb-tf-lock"
    encrypt = true
  }
}

provider "aws" {
  region = "ap-northeast-2"
}

locals {
  environments = "dev"
  project = "goormgb"

  services = [
    "auth-guard",
    "order-core",
    "queue",
    "recommendation",
    "seat"
  ]
}

module "ecr_services" {
  source = "../../modules/ecr"

  for_each = toset(local.services)

  # 네이밍 규칙: 환경/프로젝트/서비스명 (ex: dev/goormgb/auth-guard)
  repository_name = "${local.environments}/${local.project}/${each.key}"

  tags = {
    Environment = local.environments
    Project = local.project
    Service = each.key
    ManagedBy = "Terraform"
  }
}
```

### 개발팀이 만든 서버를 위한 Dockerfile 만들기
프로젝트 루트에 서비스가 나누어져 있어 각 서비스 폴더 루트에 Dockerfile을 추가하였다
```dockerfile
# Auth-Guard 서비스의 Dockerfile
FROM gradle:jdk21-alpine AS builder
WORKDIR /app

# 1. 프로젝트 설정 파일 복사 (Gradle 래퍼 불필요)
COPY build.gradle .
COPY settings.gradle .

# 2. 모든 모듈의 build.gradle 복사 (Gradle 멀티 모듈 구조 유지 필수)
COPY Auth-Guard/build.gradle Auth-Guard/
COPY common-core/build.gradle common-core/
COPY Order-Core/build.gradle Order-Core/
COPY Queue/build.gradle Queue/
COPY Seat/build.gradle Seat/
COPY Recommendation/build.gradle Recommendation/

# 3. 필요한 소스 코드만 복사 (공통 모듈 + 현재 서비스)
COPY common-core/src common-core/src
COPY Auth-Guard/src Auth-Guard/src

# 4. 빌드 수행 (테스트 제외)
RUN gradle :Auth-Guard:bootJar -x test --no-daemon

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/Auth-Guard/build/libs/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```
또한 프로젝트 루트에 .dockerignore 파일을 추가하여 불필요한 파일이 컨테이너에 포함되지 않도록 하였다.

레포지토리를 클론하여 브랜치를 따서 작업했기 때문에 push 후 PR을 생성하여 머지까지 완료하였다.

정말 연결 테스트 같은건 없이 이미지 밀드만 문제없이 되도록 하였기 때문에, 추후 수정이 계속 필요할? 수도 있다. 아마도.

빌드시에는 프로젝트 루트에서 다음형태로 명령어 실행 필요
```bash
# 프로젝트 루트(102-goormgb-backend)에서 실행
docker build -t auth-guard -f Auth-Guard/Dockerfile .
```

## 예정된 할일
### Teamcity로 Dev CI 환경 구축
### Dev 환경에 ArgoCD 구축
### 민감정보 관리
다음와 같이 설계를 해보았다
1. 테라폼으로 공용환경으로 S3 버킷 생성
2. 생성된 버킷에 .env 파일 업로드
3. 테라폼에 모듈로 SSM Parameter Store 또는 Secrets Manager 모듈 추가
4. dev/staging/prod 환경에 맞게 SSM Parameter Store 또는 Secrets Manager를 생성하도록 함
    + 여기서 키는 직접 생성, 값은 S3에 저장된 .env 참조하도록
