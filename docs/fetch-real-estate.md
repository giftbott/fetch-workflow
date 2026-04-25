# Fetch Real Estate Data

| 항목 | 값 |
|---|---|
| 파일 | `.github/workflows/fetch-real-estate.yml` |
| 트리거 | `'0 22 * * *'` (KST 07:00) + `workflow_dispatch` |
| 데이터 소스 | 국토부 실거래가 OpenAPI (매매 + 전월세) |
| 인증 | `OPEN_DATA_API_KEY` |
| 스크립트 | `scripts/fetch-real-estate.ts` |
| 출력 | `public/data/real-estate/{아파트}.json` |
| commit pattern | `public/data/real-estate/*.json` |
| concurrency | `data-commit` |
| matrix | `complexes.json` 단지별 |

## 흐름

1. `setup` — `complexes.json` sparse-checkout → matrix
2. `fetch` — 단지별 `scripts/fetch-real-estate.ts`
3. `commit` — artifact download + git-auto-commit

## 메모

- 알림은 스크립트 내부에서 직접 텔레그램 호출 (신규 거래 감지 시)
- `public/` 출력 → 빌드 산출물에 정적 포함

---

[← README](./README.md)
