# Fetch News

| 항목 | 값 |
|---|---|
| 파일 | `.github/workflows/fetch-news.yml` |
| 트리거 | `'50 22 * * *'` (KST 07:50) + `workflow_dispatch` |
| 데이터 소스 | Google News RSS (단지 키워드 union) |
| 스크립트 | `scripts/fetch-news.ts` |
| 출력 | `src/data/shared/news/{키워드}.json` |
| commit pattern | `src/data/shared/news/*.json` |
| concurrency | `data-commit` |
| matrix | 없음 (단일 job, 키워드 union 일괄 fetch) |

## 흐름

1. wooridanji 체크아웃 + `npm ci`
2. `npx tsx scripts/fetch-news.ts`
3. `git-auto-commit-action` push

## 메모

- Google News RSS 인증·rate limit 없음
- 시간 변경 시 wooridanji 의 `news.astro` 안내 문구도 같이 갱신

---

[← README](./README.md)
