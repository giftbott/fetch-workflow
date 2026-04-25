# Workflows

이 레포(`giftbott/fetch-workflow`)는 우리단지(`giftbott/wooridanji`) 데이터 수집을 GitHub Actions 로 위임한다. **여기엔 워크플로우 정의(.yml)만**, 스크립트 본체는 wooridanji 의 `scripts/` 에 산다. 모든 워크플로우가 wooridanji 를 체크아웃해 거기 안의 스크립트를 실행하고, 결과 JSON 을 wooridanji main 에 push.

## 일람

| 워크플로우 | 주기 (KST) | 데이터 소스 | 출력 (wooridanji) |
|---|---|---|---|
| [Fetch News](./fetch-news.md) | 매일 07:50 | Google News RSS | `src/data/shared/news/` |
| [Fetch Real Estate Data](./fetch-real-estate.md) | 매일 07:00 | 국토부 실거래가 OpenAPI | `public/data/real-estate/` |
| [Fetch Management Fee Snapshot](./fetch-management-fee.md) | 매달 23~30일 09:00 | K-apt Open API (26 EP) | `src/data/complexes/{slug}/management-fee/` |
| [Fetch Management Fee by Area](./fetch-area-cost.md) | 매달 24~28일 08:00 | k-apt.go.kr (Playwright) | `src/data/real-estate/area-cost/{kaptCode}/` |

## 공통 사항

**시크릿**: `GH_PAT`, `OPEN_DATA_API_KEY`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`.

**Concurrency 그룹**:
- `data-commit` — fetch-news / fetch-real-estate / fetch-management-fee 공유. 셋 다 짧고(5분 내) wooridanji main 에 push 하므로 한 번에 하나만 돌게 묶음 → push 충돌 / race 방지.
- `area-cost` — fetch-area-cost 만 별도. 한 번 실행에 30분~1시간 걸려서 같은 그룹에 두면 다른 cron 들이 그동안 큐에서 대기.

**알림**: 실패 시에만 텔레그램 1건. 성공은 침묵. fetch-area-cost 만 별도 silent 윈도우 (개별 문서 참조).

**artifact path 트릭**: `upload-artifact@v4` 는 path 가 **단일** 이면 그 디렉토리를 artifact root 로 만들어 prefix 를 잘라낸다. download 후 commit job 의 `file_pattern` (`src/data/...`) 매칭 실패 → "Nothing to commit". `path:` 에 **두 개 이상** 적으면 공통 prefix 가 repo root 가 되어 원본 디렉토리 구조 보존. 그래서 fetch-management-fee 는 dummy 두 번째 path (`failures-fee/{slug}.txt`, 실제 파일 안 만듦) 를 같이 적는다. fetch-area-cost 는 실제 두 path (`area-cost/...` + `failures/...`) 라 자연스럽게 보존됨.

## 디버깅 명령

```bash
gh workflow run "<name>" --repo giftbott/fetch-workflow [-f input=value]
gh run list   --repo giftbott/fetch-workflow --workflow "<name>" --limit 5
gh run view   <run-id> --repo giftbott/fetch-workflow --log         # 전체 로그
gh run view   <run-id> --repo giftbott/fetch-workflow --log-failed  # 실패 step 만
gh run watch  <run-id> --repo giftbott/fetch-workflow --exit-status # 완료까지 대기
```

## 매년 12월 말 할 일

`fetch-area-cost.yml` 의 `HOLIDAYS_KR` 인라인 리스트를 다음 해 한국 공휴일 `YYYY-MM-DD` 형식으로 갱신. **두 곳에 같은 값**:

1. `jobs.fetch.steps[Fetch missing months].run` 안 — chunk 단위 silent 판정용 (현재 약 line 270)
2. `jobs.commit.steps[Decide notification policy].run` 안 — 텔레그램 알림 silent 판정용 (현재 약 line 357)

둘이 어긋나면 chunk 는 silent 인데 알림은 가거나 그 반대가 됨. 갱신 시 한쪽 빠뜨리지 말 것.

## 트러블슈팅

| 증상 | 의심 |
|---|---|
| "Working tree clean. Nothing to commit" 인데 데이터는 수집된 듯 | upload-artifact path 가 단일 path (multi-line 필요) |
| 텔레그램 알림 안 옴 | silent 윈도우(KST 일자) 또는 `TELEGRAM_BOT_TOKEN` 누락 |
| chunk 다수 실패 | `--log-failed` 로 `[DIALOG:alert]` → k-apt 미공개 / `combo=false` → rate-limit |
| main push 안 됨 | `GH_PAT` 만료/권한 부족 |
