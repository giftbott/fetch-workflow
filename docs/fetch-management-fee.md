# Fetch Management Fee Snapshot

UI "단지 공통" 탭 전용 데이터 (장충금/에너지/개별/공용관리비 26 endpoint) — 단지·월별 스냅샷.

| 항목 | 값 |
|---|---|
| 파일 | `.github/workflows/fetch-management-fee.yml` |
| 트리거 | `'0 0 23-30 * *'` (KST 09:00, 매달 23~30일) + `workflow_dispatch` |
| 데이터 소스 | 공공데이터포털 OpenAPI (26 endpoints) |
| 인증 | `OPEN_DATA_API_KEY` (누락 시 step 시작에서 명시 실패) |
| 스크립트 | `scripts/fetch-management-fee.ts` (`KAPTCODE` env + argv 둘 다 지원) |
| 대상 단지 | `apartments.json` 의 42단지 (`region` / `exclude` 입력으로 필터) |
| 시작월 (단지별) | `src/data/real-estate/area-cost/{kaptCode}/` 의 최초 월 (= 사용승인일 기준) |
| 종료월 (전역) | KST 24일 이전 → 전전월 / 24일 이후 → 전월 |
| 출력 | `src/data/real-estate/management-fee/{kaptCode}/{YYYYMM}.json` |
| commit pattern | `src/data/real-estate/management-fee/**/*.json` |
| concurrency | `management-fee` (별도 — backfill 시 시간 길어 data-commit 그룹 점유 회피) |
| matrix | 단지 × 2년 chunk (~120), `max-parallel: 5` |

## 입력 (workflow_dispatch)

| input | 기본 | 설명 |
|---|---|---|
| `region` | `''` | 송파/성남/하남 — 빈값이면 전체 |
| `exclude` | `''` | 제외 kaptCode 쉼표. region 필터 후 추가 적용 |
| `force` | `false` | 이미 수집된 월도 재수집 |

## 흐름

1. `setup` — apartments.json + area-cost/ + management-fee/ sparse-checkout → 누락 월 chunk 계산
2. `fetch` (matrix) — chunk 안 월별 순차 시도:
   - `KAPTCODE=$KAPT npx tsx scripts/fetch-management-fee.ts $MONTH`
   - exit 0 + 파일 생성 → 성공 + 5~15초 sleep
   - exit 0 + 파일 미생성 → empty (OpenAPI 미공개), 3초 sleep
   - exit 1 → 실패, 60초 sleep, 연속 3회면 chunk 조기 종료
3. `commit` — failures/*.txt 집계 + git-auto-commit + (실패 시) 텔레그램 요약

## API 호출량

| 시나리오 | 호출량 | 일 한도 |
|---|---|---|
| cron 매월 (전체 42 단지, 1개월) | 26 × 42 = 1,092 | 1만/일 안에 여유 |
| backfill (전체) | 약 86K (40단지 × 평균 ~83월 × 26) | 일 한도 1만 기준 ≥9일 분산 |

→ backfill 은 region 별로 단계 진행 추천 (송파→하남→성남). 막힌 chunks 는 다음 dispatch 에서 누락 월로 자동 재시도.

## 메모

- **idempotent**: 스냅샷 한 번 생성 후 불변. 재실행해도 file-exists skip (force=true 시 우회)
- **K-apt 미공개 vs OpenAPI 미공개**: OpenAPI 공개 시점은 K-apt 와 다를 수 있음. 단건 empty 는 다음 cron 자연 회복
- **silent 윈도우 없음**: area-cost 와 달리 운영 패턴 미확인 — 실패 시 항상 텔레그램. false-positive 패턴 보이면 추가
- **artifact path multi-line 필수** ([README → artifact path 규칙](./README.md#artifact-path-규칙-upload-artifactv4)):
  ```yaml
  path: |
    src/data/real-estate/management-fee/${{ matrix.chunk.kaptCode }}/
    failures/${{ matrix.chunk.kaptCode }}-${{ matrix.chunk.yearLabel }}.txt
  ```
  area-cost 와 같은 패턴 — failures 가 실제 존재 가능 (rename 트릭 불요)

---

[← README](./README.md)
