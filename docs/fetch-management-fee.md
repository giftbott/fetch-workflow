# Fetch Management Fee Snapshot

UI "단지 공통" 탭 전용 데이터 (장충금/에너지/개별/공용관리비 26 endpoint).

| 항목 | 값 |
|---|---|
| 파일 | `.github/workflows/fetch-management-fee.yml` |
| 트리거 | `'0 0 23-30 * *'` (KST 09:00, 매달 23~30일) + `workflow_dispatch` |
| 데이터 소스 | K-apt Open API (26 endpoints) |
| 인증 | `OPEN_DATA_API_KEY` |
| 스크립트 | `scripts/fetch-management-fee.ts` |
| 대상 월 | 1개월 전 (`date -d '1 month ago' +%Y%m`) |
| 대상 단지 | `complexes.json` 메인 단지만 |
| 출력 | `src/data/complexes/{slug}/management-fee/{YYYYMM}.json` |
| commit pattern | `src/data/complexes/*/management-fee/*.json` |
| concurrency | `data-commit` |

## 흐름

1. `setup` — matrix 출력
2. `fetch` — 대상 파일 있으면 `exit 0` (idempotent), 없으면 fetch
3. `commit` — artifact merge + git-auto-commit

## 메모

- **idempotent**: 스냅샷 한 번 생성 후 불변, 재실행해도 skip
- **K-apt 미공개 시**: `[WARN] 데이터 없음 — 파일 저장 스킵 (26/26 응답)` → "Nothing to commit" → 다음 cron 재시도
- **artifact path multi-line 필수** ([README → artifact path 규칙](./README.md#artifact-path-규칙-upload-artifactv4)):
  ```yaml
  path: |
    src/data/complexes/${{ matrix.complex.slug }}/management-fee/
    failures-fee/${{ matrix.complex.slug }}.txt   # dummy, 실제 안 만듦
  ```

---

[← README](./README.md)
