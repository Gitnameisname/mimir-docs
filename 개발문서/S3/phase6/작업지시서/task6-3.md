# task6-3 — Content Sanitize

**Phase**: S3 Phase 6 / FG 6-3
**작성일**: 2026-05-18
**Handoff Level**: `extended` (정책 — 입력 검증 / 출력 정규화)
**Approver**: @최철균

---

## 1. 의도

annotation content / mention payload 에 들어올 수 있는 저수준 바이트 위험
(null byte, BOM, U+202E, ANSI escape, 기타 control byte) 을 일괄 정규화.
prompt-injection (Phase 4) 과 결이 다른 텍스트 byte layer.

R-O3 정책:
  - **write 시 reject** — 명백한 위험 문자 (null/BOM/RLO/surrogate) 는 저장 거부.
  - **read 시 sanitize** — ANSI escape / C0/C1 control char (tab/newline/CR 제외) 는
    응답 직렬화 단계에서 제거. DB 원본은 raw 보존.

## 2. 산출물

| 파일 | 변경 |
|------|------|
| `backend/app/utils/content_sanitizer.py` | 신설 — `reject_dangerous_chars` + `sanitize_for_response` |
| `backend/app/services/annotations_service.py` | `_validate_content()` 가 `reject_dangerous_chars` 호출 (write 거부) |
| `backend/app/api/v1/annotations.py` | `_to_response()` 가 `sanitize_for_response(content)` 적용 (응답 정규화) |
| `backend/app/services/notifications_service.py` | mention `snippet` payload 에 `sanitize_for_response` 적용 (저장 정합) |
| `backend/tests/unit/test_content_sanitizer_fg63.py` | 회귀 13건 |

## 3. ANSI control strict 도

본 라운드 결정: **모든 C0/C1 control char 제거, 단 `\t` `\n` `\r` 보존**.

- 이유: 사용자가 자연스럽게 입력하는 화이트스페이스는 보존. 그 외 ESC / 0x01~0x08 /
  0x0B / 0x0C / 0x0E~0x1F / 0x7F (DEL) 은 안전 의미 없음.
- ANSI escape sequence (ESC[ ... letter / ESC] ... BEL / ESC X) 는 3개의 regex 로
  정밀 매칭 후 제거 — 색 코드 / OSC / Fp escapes.

## 4. R-O3 (sanitize 정책) 준수 확인

| 항목 | 위치 | 검증 |
|---|---|---|
| write 시 null byte → 400 | `_validate_content` → `reject_dangerous_chars` | S1 회귀 |
| write 시 BOM/RLO → 400 | 같음 | S2 회귀 |
| 정상 한글/이모지 통과 | 같음 | S3 회귀 |
| read 시 ANSI 제거 | `_to_response()` | S4 회귀 |
| read 시 control 제거 (tab/newline 보존) | 같음 | S5 회귀 |
| 10,000 자 초과 reject | 같음 | S6 회귀 |

## 5. 회귀 (`tests/unit/test_content_sanitizer_fg63.py`)

총 13건 — `.venv/bin/python -m pytest tests/unit/test_content_sanitizer_fg63.py` → 13 passed.

| ID | 시나리오 |
|---|---|
| S1 | null byte reject |
| S2 | BOM reject |
| S3 | RLO override reject |
| S4 | surrogate reject |
| S5 | 길이 overflow reject |
| S6 | 한글 + 이모지 통과 |
| S7 | tab/newline/CR 통과 |
| S8 | None → "" |
| S9 | ANSI CSI 제거 (red) |
| S10 | ANSI OSC 제거 (window title) |
| S11 | C0 control 제거 |
| S12 | tab/newline/CR 보존 |
| S13 | sanitize 후 한글/이모지 유지 |

## 6. 충돌 점검 (Phase 6 §7 R-07)

- Phase 4 prompt-injection layer 와 본 sanitize 는 별 layer.
- Phase 4 는 LLM 명령 해석 — `detected_risks` annotation.
- Phase 6 FG 6-3 은 byte/character layer — DB 직렬화 안전.
- 두 layer 가 같은 string 에 동시 적용되어도 idempotent (제거 후 다시 제거).

## 7. 범위 밖

- HTML tag sanitize — 기존 `html_sanitizer.py` (XSS) 가 별 layer 로 담당.
- mention 본문 안 멘션 패턴 정합성 — `extract_mentions` 가 담당 (별 라운드).
