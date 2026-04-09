# Mimir 운영 Runbook

> Phase 13 산출물. 담당자 없이도 장애 대응이 가능하도록 작성된 운영 절차서.

---

## 1. 서비스 개요

| 항목 | 내용 |
|------|------|
| 서비스명 | Mimir 문서/지식 플랫폼 |
| Backend | FastAPI (Python 3.11), 포트 8000 |
| Frontend | Next.js 14, 포트 3000 |
| DB | PostgreSQL 16 + pgvector |
| 캐시 | Valkey 8 (Redis 호환) |
| 배포 | Docker Compose / Kubernetes |

---

## 2. 서비스 상태 확인

### 2-1. 전체 상태 확인

```bash
# 컨테이너 상태
docker compose ps

# 헬스체크
curl -s http://localhost:8000/api/v1/system/health | python3 -m json.tool

# 메트릭 확인
curl -s http://localhost:8000/api/v1/system/metrics | head -50
```

### 2-2. 로그 확인

```bash
# 백엔드 실시간 로그 (JSON)
docker compose logs -f backend | python3 -m json.tool

# 오류 로그만 필터
docker compose logs backend | grep '"level":"ERROR"'

# 특정 시간대 로그
docker compose logs --since 30m backend
```

### 2-3. DB 연결 상태

```bash
# PostgreSQL 상태
docker compose exec db pg_isready -U master

# 연결 수 확인
docker compose exec db psql -U master -d mimir -c \
  "SELECT count(*) FROM pg_stat_activity WHERE datname='mimir';"
```

### 2-4. Valkey 상태

```bash
docker compose exec valkey valkey-cli ping
docker compose exec valkey valkey-cli info memory | grep used_memory_human
```

---

## 3. 서비스 재시작 절차

### 3-1. 백엔드만 재시작 (기동 순서 유지)

```bash
docker compose restart backend
# 헬스체크 통과 확인 (30초 대기)
sleep 15
curl -s http://localhost:8000/api/v1/system/health
```

### 3-2. 전체 스택 재시작

```bash
docker compose down
docker compose up -d
# 모든 서비스 헬스체크 확인
docker compose ps
```

### 3-3. DB 재시작 (데이터 보존)

```bash
# 반드시 백업 먼저
./scripts/backup/backup_db.sh

docker compose restart db
# 연결 복구 확인
docker compose exec db pg_isready -U master
```

---

## 4. 배포 절차 (Rolling Update)

### 4-1. 이미지 업데이트

```bash
# 새 이미지 Pull
docker compose pull backend frontend

# 백엔드 먼저 배포
docker compose up -d --no-deps backend
sleep 30

# 헬스체크
curl -sf http://localhost:8000/api/v1/system/health || echo "FAILED"

# 프론트엔드 배포
docker compose up -d --no-deps frontend
```

### 4-2. 롤백

```bash
# 이전 이미지 태그로 롤백
IMAGE_TAG=sha-previous docker compose up -d --no-deps backend
```

---

## 5. DB 백업 및 복구

### 5-1. 수동 백업 실행

```bash
./scripts/backup/backup_db.sh
# 백업 파일 확인
ls -lh /var/backups/mimir/
```

### 5-2. 자동 백업 설정 (crontab)

```
# 매일 새벽 2시 백업
0 2 * * * /opt/mimir/scripts/backup/backup_db.sh >> /var/log/mimir_backup.log 2>&1
```

### 5-3. 복구 절차 (RTO < 4시간)

```bash
# 1단계: 최근 백업 파일 확인
ls -lt /var/backups/mimir/*.dump | head -5

# 2단계: 체크섬 검증
sha256sum --check /var/backups/mimir/mimir_YYYYMMDD_HHMMSS.dump.sha256

# 3단계: 복구 실행 (기존 데이터 유지하면서 복구)
./scripts/backup/restore_db.sh /var/backups/mimir/mimir_YYYYMMDD_HHMMSS.dump

# 4단계: 복구 확인
docker compose exec db psql -U master -d mimir -c "SELECT count(*) FROM documents;"
```

---

## 6. 성능 문제 대응

### 6-1. 응답 시간 지연 (P95 > 500ms)

```bash
# 1. 메트릭으로 느린 경로 확인
curl -s http://localhost:8000/api/v1/system/metrics | grep "http_request_duration_ms_bucket"

# 2. DB 슬로우 쿼리 확인
docker compose exec db psql -U master -d mimir -c \
  "SELECT query, mean_exec_time, calls FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;"

# 3. DB 인덱스 사용 현황
docker compose exec db psql -U master -d mimir -c \
  "SELECT schemaname, tablename, indexname, idx_scan FROM pg_stat_user_indexes ORDER BY idx_scan ASC LIMIT 20;"

# 4. Valkey 캐시 히트율 확인
docker compose exec valkey valkey-cli info stats | grep keyspace_hits
```

### 6-2. 메모리 과다 사용

```bash
# 컨테이너별 메모리 사용량
docker stats --no-stream

# Python 메모리 릭 의심 시 백엔드 재시작
docker compose restart backend
```

---

## 7. 보안 사고 대응

### 7-1. 비정상 인증 시도 감지

```bash
# 인증 실패 로그 확인 (최근 100건)
docker compose logs backend | grep '"result":"denied"' | tail -100

# IP별 요청 수 집계
docker compose logs backend | grep '"result":"denied"' | \
  python3 -c "import sys, json; [print(json.loads(l).get('actor_id','')) for l in sys.stdin if l.strip().startswith('{')]" | \
  sort | uniq -c | sort -rn | head -20
```

### 7-2. Rate Limit 임계값 조정

Rate limit은 `app/api/rate_limit.py`에서 설정. 변경 후 백엔드 재시작 필요.

### 7-3. API 키 긴급 비활성화

```bash
docker compose exec db psql -U master -d mimir -c \
  "UPDATE api_keys SET status='REVOKED' WHERE key_prefix='<PREFIX>';"
```

---

## 8. 데이터 보존 정책 실행

```bash
# Dry-run으로 영향도 확인
python3 scripts/data_retention.py --dry-run --verbose

# 실제 실행
python3 scripts/data_retention.py

# crontab (매월 1일 새벽 3시)
0 3 1 * * /opt/mimir/scripts/data_retention.py >> /var/log/mimir_retention.log 2>&1
```

---

## 9. SLO 모니터링

| 지표 | 목표 | 확인 방법 |
|------|------|----------|
| API P95 응답 | < 500ms | Grafana / `/api/v1/system/metrics` |
| RAG TTFT P95 | < 2000ms | Grafana |
| 가용성 | 99.5% | Uptime 모니터링 |
| 오류율 (5xx) | < 0.1% | `http_errors_total / http_requests_total` |
| DB 백업 RPO | < 24시간 | `/var/backups/mimir/` 파일 날짜 확인 |
| 복구 RTO | < 4시간 | 복구 테스트 기록 |
