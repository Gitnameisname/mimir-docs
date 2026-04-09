# Mimir 장애 대응 절차 (Incident Response)

---

## 1. 장애 심각도 분류

| 등급 | 기준 | 대응 SLA |
|------|------|----------|
| P1 (Critical) | 서비스 전체 다운, 데이터 유실 위험 | 15분 이내 대응 시작 |
| P2 (High) | 주요 기능 장애 (로그인 불가, 문서 조회 불가) | 1시간 이내 |
| P3 (Medium) | 부분 기능 저하 (검색 지연, RAG 오류) | 4시간 이내 |
| P4 (Low) | UI 버그, 미미한 성능 저하 | 다음 배포 사이클 |

---

## 2. 장애 대응 흐름

```
알림 수신 (Grafana Alert / 사용자 제보)
    │
    ▼
① 심각도 분류 (5분 이내)
    │
    ▼
② 초기 조사 — 로그/메트릭 확인 (15분)
    │
    ├── 빠른 수정 가능 → ③-A 즉시 수정
    │
    └── 복잡한 원인  → ③-B 우회 조치 후 근본 원인 분석
    │
    ▼
④ 서비스 복구 확인
    │
    ▼
⑤ Post-mortem 작성 (P1/P2)
```

---

## 3. 초기 조사 체크리스트

### 서비스 다운 (P1)

- [ ] `docker compose ps` — 컨테이너 상태 확인
- [ ] `curl http://localhost:8000/api/v1/system/health` — 헬스체크
- [ ] `docker compose logs --tail 100 backend` — 최근 로그 확인
- [ ] `docker compose exec db pg_isready` — DB 연결 확인
- [ ] `docker compose exec valkey valkey-cli ping` — 캐시 연결 확인
- [ ] 디스크 공간 확인: `df -h`
- [ ] 메모리 확인: `free -h`

### 응답 지연 (P2/P3)

- [ ] `/api/v1/system/metrics`에서 P95 응답 시간 확인
- [ ] DB 슬로우 쿼리: `pg_stat_statements` 조회
- [ ] 활성 연결 수: `pg_stat_activity` 조회
- [ ] Valkey 메모리: `valkey-cli info memory`

---

## 4. 일반적인 장애 시나리오 및 해결

### 4-1. 백엔드 OOM 재시작 반복

**증상**: 백엔드 컨테이너가 주기적으로 재시작, 메모리 사용량 증가

**해결**:
```bash
# 메모리 현황 확인
docker stats mimir-backend --no-stream

# 임시 조치: 메모리 제한 완화 (docker-compose.yml 수정)
# 근본 해결: 메모리 릭 원인 프로파일링
docker compose exec backend python3 -c "import tracemalloc; ..."
```

### 4-2. DB 연결 풀 고갈

**증상**: `connection pool exhausted` 오류 다수 발생

**해결**:
```bash
# 현재 연결 수 확인
docker compose exec db psql -U master -d mimir -c \
  "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"

# 유휴 연결 강제 종료
docker compose exec db psql -U master -d mimir -c \
  "SELECT pg_terminate_backend(pid) FROM pg_stat_activity
   WHERE state='idle' AND state_change < now() - interval '10 minutes';"

# 임시 조치: 백엔드 재시작 (연결 풀 초기화)
docker compose restart backend
```

### 4-3. Valkey 연결 실패

**증상**: 캐시 관련 오류 로그, 세션 인증 실패

**해결**:
```bash
# Valkey 재시작 (세션/캐시 초기화)
docker compose restart valkey
# 주의: 세션 데이터 유실 — 모든 사용자 재로그인 필요
```

### 4-4. 데이터 부정합 / 마이그레이션 오류

**해결**:
```bash
# 최근 백업으로 복구
./scripts/backup/restore_db.sh /var/backups/mimir/mimir_최신.dump
```

---

## 5. Post-mortem 템플릿 (P1/P2 필수)

```markdown
## 장애 Post-mortem: [날짜] [간략 제목]

### 요약
- 발생 시각:
- 복구 시각:
- 영향 범위:
- 근본 원인:

### 타임라인
| 시각 | 이벤트 |
|------|--------|
| HH:MM | 알림 수신 |
| HH:MM | 원인 파악 |
| HH:MM | 우회 조치 시작 |
| HH:MM | 서비스 복구 |

### 근본 원인 분석 (RCA)
...

### 재발 방지 액션 아이템
| 액션 | 담당자 | 기한 |
|------|--------|------|
| ... | ... | YYYY-MM-DD |
```
