# Fetch Real Estate Data

단지별 매매·전월세 실거래가를 매일 갱신. 신규 거래 발견 시 텔레그램 알림.

| 항목 | 값 |
|---|---|
| 파일 | `.github/workflows/fetch-real-estate.yml` |
| 트리거 | `schedule: '0 22 * * *'` (KST 07:00, UTC 22:00 전날) + `workflow_dispatch` |
| 데이터 소스 | 국토교통부 실거래가 공공 API (매매 + 전월세) |
| 인증 | `OPEN_DATA_API_KEY` |
| 스크립트 | `scripts/fetch-real-estate.ts` (wooridanji) |
| 출력 | `public/data/real-estate/{아파트}.json` |
| commit pattern | `public/data/real-estate/*.json` |
| concurrency | `data-commit` |
| matrix | `complexes.json` 의 단지 |

## 흐름

1. `setup` — `complexes.json` sparse-checkout → matrix 출력
2. `fetch` — 단지별로 `scripts/fetch-real-estate.ts` 실행 (env: 매물 정보 + 텔레그램 토큰)
3. `commit` — artifact download → git-auto-commit

## 특이사항

- 알림은 워크플로우 step 이 아니라 **스크립트 내부에서** 직접 텔레그램 호출 (신규 거래 감지 시).
- 출력이 `public/` 이라 빌드 산출물에 그대로 포함 (정적 fetch 가능).

---

[← README](./README.md)
