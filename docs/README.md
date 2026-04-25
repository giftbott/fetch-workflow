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
- `data-commit` — fetch-news / fetch-real-estate / fetch-management-fee 공유 (push 충돌 방지)
- `area-cost` — fetch-area-cost 만 별도 (실행 시간 길어 큐 분리)

**알림**: 실패 시에만 텔레그램 1건. 성공은 침묵. fetch-area-cost 만 별도 silent 윈도우 (개별 문서 참조).

**artifact path 트릭**: `upload-artifact@v4` 의 `path:` 가 단일 디렉토리이면 prefix 가 trim 되어 commit job 의 `file_pattern` 매칭 실패. multi-line 으로 두 개 이상 path 를 적어 공통 prefix 를 repo root 로 유도해야 한다 (fetch-management-fee, fetch-area-cost).

## 디버깅 명령

```bash
gh workflow run "<name>" --repo giftbott/fetch-workflow [-f input=value]
gh run list   --repo giftbott/fetch-workflow --workflow "<name>" --limit 5
gh run view   <run-id> --repo giftbott/fetch-workflow --log         # 전체 로그
gh run view   <run-id> --repo giftbott/fetch-workflow --log-failed  # 실패 step 만
gh run watch  <run-id> --repo giftbott/fetch-workflow --exit-status # 완료까지 대기
```

## 매년 12월 말 할 일

- `fetch-area-cost.yml` 의 `HOLIDAYS_KR` 인라인 리스트를 다음 해 한국 공휴일로 갱신 (2 곳).

## 트러블슈팅

| 증상 | 의심 |
|---|---|
| "Working tree clean. Nothing to commit" 인데 데이터는 수집된 듯 | upload-artifact path 가 단일 path (multi-line 필요) |
| 텔레그램 알림 안 옴 | silent 윈도우(KST 일자) 또는 `TELEGRAM_BOT_TOKEN` 누락 |
| chunk 다수 실패 | `--log-failed` 로 `[DIALOG:alert]` → k-apt 미공개 / `combo=false` → rate-limit |
| main push 안 됨 | `GH_PAT` 만료/권한 부족 |
