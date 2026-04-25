# Fetch Management Fee by Area

42 단지 (메인 + 비교) 면적별 관리비 (단위면적당 breakdown) 를 k-apt 에서 엑셀로 받아 JSON 변환.

| 항목 | 값 |
|---|---|
| 파일 | `.github/workflows/fetch-area-cost.yml` |
| 트리거 | `schedule: '0 23 23-27 * *'` (KST 08:00, 매달 24~28일) + `workflow_dispatch` (`exclude`, `force`) |
| 데이터 소스 | k-apt.go.kr (인증 불요, Playwright + HTTP hybrid → Excel → JSON) |
| 스크립트 | `scripts/fetch-area-cost.ts` (wooridanji) |
| 대상 단지 | `apartments.json` 의 42 단지 (필요 시 `exclude` 입력으로 제외) |
| 출력 | `src/data/real-estate/area-cost/{kaptCode}/{YYYYMM}.json` |
| commit pattern | `src/data/real-estate/area-cost/**/*.json` |
| concurrency | `area-cost` (다른 워크플로우와 분리, 실행 시간 길어서) |
| matrix | 단지 × 2년 chunk (~180 chunk), `max-parallel: 5` |

## 입력 (workflow_dispatch)

| input | 기본 | 설명 |
|---|---|---|
| `exclude` | (빈값) | 제외할 kaptCode 쉼표 구분. 비우면 전 단지 |
| `force`   | `false` | `true` 면 이미 수집된 월도 재수집 |

## 흐름

1. **setup** — apartments.json + 기존 area-cost 디렉터리 sparse-checkout → 누락 월만 chunks emit (`force=true` 면 모든 월)
2. **fetch** (matrix, max-parallel 5) — chunk 안에서 월별 순차 시도
   - `goto main` → `_csrf` 추출
   - `request.post('/cmmn/selectKapt.do', {kapt_code, _csrf})` (302)
   - `page.goto('/apiinfo/goApiInfoSearch.do?searchDate=YYYYMM&kaptCode=')` 직진입
   - 콤보 선택 + 엑셀 다운로드 → `scripts/lib/xls-parser.ts` 가 JSON 변환
3. **commit** — failures/*.txt 집계 → git-auto-commit + (조건부) 텔레그램 요약

## 대상 월 계산 (setup, KST)

- **24일 이전** (1~23일) cron: 전전월까지 (예: 4/23 → 202602)
- **24일 이후** (24~말일) cron: 전월까지 (예: 4/24 → 202603)
- 단지별 `startMonth`, `occuFirstDate` (입주 시점), `endMonth` 고려해 가능한 월 범위 계산
- 기존 `area-cost/{kaptCode}/{YYYYMM}.json` 존재 월은 제외 (`force=true` 면 무시)
- 2년 단위 chunk (1년 단위면 ~336개로 matrix 256 한도 초과)

## Rate-limit 처리

- chunk 단위 60초 쿨다운, 연속 3회 rate-limit 시 chunk 조기 종료
- 실패 월은 `failures/{kaptCode}-{yearLabel}.txt` 에 기록 → commit job 이 집계

## Silent 윈도우 (cron 만 적용, dispatch 는 무시)

| KST 일자 | 실패 시 동작 |
|---|---|
| 24일 | 무조건 silent (chunk green, 알림 skip) — k-apt 롤링 공개 첫날 |
| 25일 | 무조건 silent — 롤링 공개 진행 중 |
| 26일 | 토·일 또는 한국 공휴일 → silent / 평일·평상 → 알림 |
| 27·28일 | 항상 알림 (실제 이슈 가능성) |
| 수동 dispatch | 항상 알림 |

공휴일은 yaml 안 인라인 `HOLIDAYS_KR` (2026 리스트). **매년 12월 말 갱신 필요** (chunk fail 분기 + 알림 분기 양쪽 동일 리스트).

## 특이사항

- **artifact path 는 multi-line** (`src/data/...` + `failures/...`) → 공통 prefix repo root 보존.
- **Playwright 브라우저 cache** 활용 (cache miss 시 `playwright install --with-deps` 추가 시간).
- k-apt 가 단지/월 데이터를 안 올렸으면 페이지 alert: `관리비 데이터가 존재해야 엑셀 다운로드가 가능합니다`. 스크립트가 이 alert 을 감지해 fail 처리. 25일 새벽까지도 17/42 미공개인 경우 흔하다 (silent 윈도우 25일 추가 이유).
- 새 코드와 데이터 모두 wooridanji `main` 에 있음. 과거에 yml 에 `ref: feat/management-fee-comparison` 가 있었으나 머지 완료로 제거됨.
