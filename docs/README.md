# Workflows

워크플로우 정의 전용 레포. 스크립트는 wooridanji 의 `scripts/`. 결과는 wooridanji main 에 직접 push.

## 일람

| 워크플로우 | 주기 (KST) | 데이터 소스 | 출력 (wooridanji) |
|---|---|---|---|
| [Fetch News](./fetch-news.md) | 매일 07:50 | Google News RSS | `src/data/shared/news/` |
| [Fetch Real Estate Data](./fetch-real-estate.md) | 매일 07:00 | 국토부 실거래가 OpenAPI | `public/data/real-estate/` |
| [Fetch Management Fee Snapshot](./fetch-management-fee.md) | 매달 23~30일 09:00 | K-apt Open API | `src/data/complexes/{slug}/management-fee/` |
| [Fetch Management Fee by Area](./fetch-area-cost.md) | 매달 24~28일 08:00 | K-apt 웹 (Playwright) | `src/data/real-estate/area-cost/{kaptCode}/` |

## 시크릿

| 이름 | 용도 |
|---|---|
| `GH_PAT` | wooridanji push (classic PAT, repo 권한) |
| `OPEN_DATA_API_KEY` | 부동산 OpenAPI / K-apt Open API |
| `TELEGRAM_BOT_TOKEN` / `TELEGRAM_CHAT_ID` | 실패 알림 (미설정 시 알림 step 자동 skip) |

## Concurrency

| 그룹 | 워크플로우 | 이유 |
|---|---|---|
| `data-commit` | news / real-estate / management-fee | wooridanji main push 직렬화 |
| `area-cost` | area-cost | 실행 30분~1시간, 별도 큐 |

## artifact path 규칙 (`upload-artifact@v4`)

- 단일 path → artifact root 가 그 디렉토리 (prefix trim)
- 다중 path → 공통 prefix 가 root (원본 구조 보존)

commit job 의 `file_pattern` 이 `src/data/...` 같이 절대 prefix 면 **다중 path 필수**. management-fee/area-cost 가 이를 위해 multi-line.

## 알림

실패 시 텔레그램 1건. 성공 침묵. area-cost 만 별도 silent 윈도우 (해당 문서 참조).

## gh 명령

```bash
gh workflow run "<name>" --repo giftbott/fetch-workflow [-f input=value]
gh run list   --repo giftbott/fetch-workflow --workflow "<name>" --limit 5
gh run view   <id>     --repo giftbott/fetch-workflow --log-failed
gh run watch  <id>     --repo giftbott/fetch-workflow --exit-status
```

## 매년 12월 말 — `HOLIDAYS_KR` 갱신

`fetch-area-cost.yml` 의 두 곳에 같은 `YYYY-MM-DD` 리스트:

| 위치 | 용도 |
|---|---|
| `jobs.fetch.steps[Fetch missing months].run` | chunk silent 판정 |
| `jobs.commit.steps[Decide notification policy].run` | 알림 silent 판정 |

둘이 어긋나면 chunk silent 인데 알림 가거나 반대.

## 트러블슈팅

| 증상 | 의심 |
|---|---|
| "Working tree clean. Nothing to commit" | upload-artifact path 가 단일 |
| 텔레그램 알림 안 옴 | silent 윈도우 / `TELEGRAM_BOT_TOKEN` 누락 |
| chunk 다수 실패 | `--log-failed` → `[DIALOG:alert]` (K-apt 미공개) / `combo=false` (rate-limit) |
| main push 실패 | `GH_PAT` 만료/권한 |
