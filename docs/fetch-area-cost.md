# Fetch Management Fee by Area

42 단지 면적별 관리비 (단위면적당 breakdown). K-apt 엑셀 → JSON.

| 항목 | 값 |
|---|---|
| 파일 | `.github/workflows/fetch-area-cost.yml` |
| 트리거 | `'0 23 23-27 * *'` (KST 08:00, 매달 24~28일) + `workflow_dispatch` |
| 데이터 소스 | K-apt 웹 (https://www.k-apt.go.kr, 인증 불요) |
| 방식 | Playwright + HTTP hybrid → Excel 다운 → JSON |
| 스크립트 | `scripts/fetch-area-cost.ts`, `scripts/lib/xls-parser.ts` |
| 대상 단지 | `apartments.json` 의 42단지 (`exclude` 입력으로 제외 가능) |
| 출력 | `src/data/real-estate/area-cost/{kaptCode}/{YYYYMM}.json` |
| commit pattern | `src/data/real-estate/area-cost/**/*.json` |
| concurrency | `area-cost` (별도) |
| matrix | 단지 × 2년 chunk (~180), `max-parallel: 5` |

## 입력 (workflow_dispatch)

| input | 기본 | 설명 |
|---|---|---|
| `exclude` | `''` | 제외 kaptCode 쉼표 |
| `force` | `false` | 이미 수집된 월도 재수집 |

## 흐름

1. `setup` — apartments.json + 기존 area-cost 디렉터리 sparse-checkout → 누락 월 chunk 계산
2. `fetch` (matrix) — chunk 안 월별 순차 시도:
   - `goto main` → `_csrf` 추출
   - `request.post('/cmmn/selectKapt.do', {kapt_code, _csrf})` (302)
   - `page.goto('/apiinfo/goApiInfoSearch.do?searchDate=YYYYMM&kaptCode=')`
   - 콤보 선택 + 엑셀 다운로드 → JSON 변환
3. `commit` — failures/*.txt 집계 + git-auto-commit + (조건부) 텔레그램 요약

## 대상 월 계산 (KST)

- 1~23일 cron: 전전월까지
- 24~말일 cron: 전월까지
- 단지별 `startMonth` / `occuFirstDate` / `endMonth` 고려
- 기존 파일 존재 월 제외 (`force=true` 면 무시)
- 2년 chunk (1년이면 ~336개로 matrix 256 한도 초과)

## Rate-limit

- chunk 단위 60초 쿨다운
- 연속 3회 rate-limit → chunk 조기 종료
- 실패 월 → `failures/{kaptCode}-{yearLabel}.txt`

## Silent 윈도우 (cron 만, dispatch 무시)

| KST 일자 | 실패 시 |
|---|---|
| 24일 | silent (롤링 공개 첫날) |
| 25일 | silent (진행 중) |
| 26일 | 토·일 또는 공휴일 → silent / 평일 → 알림 |
| 27·28일 | 알림 |

> **롤링 공개**: K-apt 는 24일부터 데이터 공개를 시작하지만 등록 자체는 단지 관리사무소가 하므로 24~28일 분산. 25일 새벽 시점 17/42 미등록도 정상 범위.

공휴일 = `HOLIDAYS_KR` 인라인 (yaml 두 곳, 매년 갱신 → [README](./README.md#매년-12월-말--holidays_kr-갱신)).

## 메모

- Playwright 브라우저 cache 활용 (cache miss 시 `playwright install --with-deps` 추가)
- K-apt 미공개 시 페이지 alert: `관리비 데이터가 존재해야 엑셀 다운로드가 가능합니다` → 스크립트가 fail 처리
- artifact path 는 실제 두 path (`area-cost/...` + `failures/...`) — multi-line 자연스럽게 보존

---

[← README](./README.md)
