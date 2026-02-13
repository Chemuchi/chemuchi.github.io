---
title: 2026-02-13 정리
date: "2026-02-13"
draft: false
tags:
description:
---

## 오늘 한일 

### Docker Compose 환경별 설정


## 예정된 할일
### Dev 환경에 ArgoCD 구축
### 민감정보 관리
다음와 같이 설계를 해보았다
1. 테라폼으로 공용환경으로 S3 버킷 생성
2. 생성된 버킷에 .env 파일 업로드
3. 테라폼에 모듈로 SSM Parameter Store 또는 Secrets Manager 모듈 추가
4. dev/staging/prod 환경에 맞게 SSM Parameter Store 또는 Secrets Manager를 생성하도록 함
    + 여기서 키는 직접 생성, 값은 S3에 저장된 .env 참조하도록