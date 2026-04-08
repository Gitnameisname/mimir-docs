# Task 13-7. CI/CD 배포 파이프라인

## 1. 작업 목적

코드 변경이 안전하게 자동 빌드/테스트/배포되는 CI/CD 파이프라인을 구축한다.

이 작업의 목표는 다음과 같다.

- PR 단계 자동 테스트 (단위 + 통합 + 보안 스캔)
- 스테이징/운영 환경 자동 배포
- 롤백 절차 자동화
- Docker 이미지 빌드 및 컨테이너 레지스트리 관리
- 시크릿 관리 체계 구축

---

## 2. 작업 범위

### 포함 범위

- GitHub Actions (또는 GitLab CI) 파이프라인 구현
- Docker 이미지 빌드 최적화
- 스테이징 자동 배포
- 운영 배포 (수동 승인 후 자동)
- 롤백 절차
- 시크릿 관리 정책

### 제외 범위

- 인프라 프로비저닝 (Kubernetes 클러스터 구성 등)
- 멀티 리전 배포

---

## 3. 주요 구현 대상

### 3-1. 파이프라인 전체 흐름

```
PR 생성
  └── CI 파이프라인 (자동)
        ├── 코드 품질 검사 (lint, type check)
        ├── 단위 테스트 + 커버리지
        ├── 통합 테스트 (pgvector 포함)
        ├── 보안 스캔 (Bandit, npm audit)
        ├── Docker 이미지 빌드 테스트
        └── 결과 → PR 코멘트 및 블로킹

main 브랜치 머지
  └── 스테이징 배포 파이프라인 (자동)
        ├── Docker 이미지 빌드 및 태그 (SHA)
        ├── 이미지 레지스트리 푸시
        ├── 스테이징 배포
        ├── 스모크 테스트 (/health + 핵심 API)
        └── Slack 배포 완료 알림

운영 배포 (수동 승인 필요)
  └── 운영 배포 파이프라인
        ├── 릴리즈 태그 생성 (v1.2.3)
        ├── 승인자 확인 → 승인 후 진행
        ├── 블루/그린 배포 또는 롤링 업데이트
        ├── 스모크 테스트
        ├── 이상 감지 → 자동 롤백
        └── 배포 완료 알림
```

---

### 3-2. GitHub Actions 파이프라인 구조

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install ruff && ruff check app/
      - run: cd frontend && npm run lint

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: pgvector/pgvector:pg16
        env:
          POSTGRES_PASSWORD: test
    steps:
      - run: pytest tests/ --cov=app --cov-fail-under=80

  security:
    runs-on: ubuntu-latest
    steps:
      - run: bandit -r app/ -ll
      - run: pip install safety && safety check
      - run: cd frontend && npm audit --audit-level=high

  build:
    runs-on: ubuntu-latest
    steps:
      - uses: docker/build-push-action@v5
        with:
          push: false
          tags: mimir:test
```

---

### 3-3. Docker 이미지 구성

```dockerfile
# Dockerfile (Backend)
FROM python:3.12-slim AS base

# 의존성 레이어 (캐시 최적화)
FROM base AS deps
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 애플리케이션 레이어
FROM deps AS app
COPY app/ /app/app/
WORKDIR /app

# 비루트 사용자 실행
RUN adduser --disabled-password --gecos "" appuser
USER appuser

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

### 3-4. 시크릿 관리 정책

| 환경 | 시크릿 저장 방법 |
|------|--------------|
| 로컬 개발 | .env 파일 (gitignore) |
| CI/CD | GitHub Secrets / GitLab CI Variables |
| 스테이징/운영 | Kubernetes Secrets 또는 AWS Secrets Manager |

```
절대 금지:
  - 코드에 시크릿 하드코딩
  - .env 파일 git 커밋
  - 로그에 시크릿 출력
```

---

### 3-5. 롤백 절차

```bash
# 롤백 스크립트
#!/bin/bash
PREV_IMAGE="mimir-api:${PREVIOUS_SHA}"

# 이전 이미지로 재배포
kubectl set image deployment/mimir-api mimir-api=${PREV_IMAGE}
kubectl rollout status deployment/mimir-api

# DB 마이그레이션 롤백 (필요 시)
alembic downgrade -1

echo "롤백 완료: ${PREV_IMAGE}"
```

---

## 4. 산출물

1. CI 파이프라인 (.github/workflows/ci.yml)
2. 스테이징 배포 파이프라인 (.github/workflows/deploy-staging.yml)
3. 운영 배포 파이프라인 (.github/workflows/deploy-prod.yml)
4. Dockerfile (Backend + Frontend)
5. 롤백 스크립트
6. 시크릿 관리 정책 문서

---

## 5. 완료 기준

- PR 생성 시 단위/통합/보안 테스트가 자동 실행된다
- main 머지 후 스테이징에 자동 배포된다
- 운영 배포는 수동 승인 후 자동 실행된다
- 배포 실패 시 롤백 절차가 문서화되어 있다
- 모든 시크릿이 환경 변수로 관리된다

---

## 6. Codex 작업 지침

- Docker 이미지 빌드 시 비루트(non-root) 사용자로 실행하여 컨테이너 보안을 강화한다
- 이미지 태그는 `latest` 사용 금지 — 항상 Git SHA 또는 릴리즈 버전 태그를 사용한다
- 스테이징 스모크 테스트 실패 시 자동으로 이전 버전으로 롤백한다
- DB 마이그레이션은 배포와 독립적으로 먼저 실행하며 (backwards-compatible), 롤백 가능한 마이그레이션만 허용한다
