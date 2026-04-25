# Fetch Management Fee Snapshot

메인 단지의 **상세 관리비 스냅샷** (장충금/에너지/개별사용료/공용관리비 26 endpoint) 을 월간 수집. UI 의 **"단지 공통" 탭 전용 데이터**.

| 항목 | 값 |
|---|---|
| 파일 | `.github/workflows/fetch-management-fee.yml` |
| 트리거 | `schedule: '0 0 23-30 * *'` (KST 09:00, 매달 23~30일) + `workflow_dispatch` |
| 데이터 소스 | 공공데이터포털 K-apt Open API (26 endpoints) |
| 인증 | `OPEN_DATA_API_KEY` |
| 스크립트 | `scripts/fetch-management-fee.ts` (wooridanji) |
| 대상 월 | 1개월 전 (`date -d '1 month ago' +%Y%m`) |
| 대상 단지 | `complexes.json` 의 메인 단지 (현재 포레나송파 / 송파레이크힐) |
| 출력 | `src/data/complexes/{slug}/management-fee/{YYYYMM}.json` |
| commit pattern | `src/data/complexes/*/management-fee/*.json` |
| concurrency | `data-commit` |

## 흐름

1. `setup` — `complexes.json` sparse-checkout → matrix
2. `fetch` (matrix) — 대상 월 파일이 이미 있으면 즉시 `exit 0` (idempotent), 없으면 `npm run fetch:management-fee -- $MONTH $KAPTCODE`
3. `commit` — artifact merge → git-auto-commit

## 특이사항

- **idempotency**: 스냅샷은 한 번 만들면 변하지 않음. 같은 월에 cron 이 여러 번 돌아도 첫 성공 후 건너뜀.
- **데이터 미공개 처리**: K-apt API 가 해당 단지/월을 안 올렸으면 `[WARN] 데이터 없음 — 파일 저장 스킵 (26/26 엔드포인트 응답)` 출력 후 정상 종료. commit job 은 "Working tree clean" 으로 push 없이 끝남. 다음 cron 자동 재시도.
- **artifact path 는 multi-line**:
  ```yaml
  path: |
    src/data/complexes/${{ matrix.complex.slug }}/management-fee/
    failures-fee/${{ matrix.complex.slug }}.txt
  ```
  단일 path 면 `upload-artifact@v4` 가 prefix 를 trim 해서 commit 의 `file_pattern` 매칭 실패 → "Nothing to commit" 으로 끝나는 버그가 있었다 (수정 완료).
