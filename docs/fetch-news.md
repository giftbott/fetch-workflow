# Fetch News

단지별 검색 키워드 뉴스를 매일 1회 수집해 뉴스 페이지·홈 위젯에 제공.

| 항목 | 값 |
|---|---|
| 파일 | `.github/workflows/fetch-news.yml` |
| 트리거 | `schedule: '50 22 * * *'` (KST 07:50, UTC 22:50 전날) + `workflow_dispatch` |
| 데이터 소스 | Google News RSS (단지별 키워드 union) |
| 스크립트 | `scripts/fetch-news.ts` (wooridanji) |
| 출력 | `src/data/shared/news/{키워드}.json` |
| commit pattern | `src/data/shared/news/*.json` |
| concurrency | `data-commit` |
| matrix | 없음 (단일 job, 키워드 union 한 번에 fetch) |

## 흐름

1. wooridanji 체크아웃 → `npm ci`
2. `npx tsx scripts/fetch-news.ts` (모든 단지 키워드 중복 제거 후 일괄 수집)
3. `git-auto-commit-action` 으로 push

## 특이사항

- 이전엔 09:00 + 17:00 하루 2회 → **출근 직전 1회로 축소**. 새벽에 쌓인 새 기사를 출근 시간 직전에 갱신.
- Google News RSS 는 인증 / rate limit 거의 없음.
- UI 의 `news.astro` 안내 문구도 "매일 07:50 업데이트" 로 동기화돼 있음.

---

[← README](./README.md)
