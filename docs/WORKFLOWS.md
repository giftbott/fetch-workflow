# Workflows

이 레포(`giftbott/fetch-workflow`)는 우리단지(`giftbott/wooridanji`) 의 데이터 수집을 GitHub Actions 로 위임하는 곳이다. 여기엔 **워크플로우 정의(.yml) 만** 두고, 실제 fetcher 스크립트(`scripts/*`) 는 wooridanji 레포에 산다. 모든 워크플로우가 wooridanji 를 체크아웃해서 거기 안의 스크립트를 실행하고, 결과 JSON 을 wooridanji main 에 직접 push 한다.

## 목차

1. [공통 설계](#공통-설계)
2. [워크플로우 일람](#워크플로우-일람)
3. [fetch-news (Fetch News)](#fetch-news-fetch-news)
4. [fetch-real-estate (Fetch Real Estate Data)](#fetch-real-estate-fetch-real-estate-data)
5. [fetch-management-fee (Fetch Management Fee Snapshot)](#fetch-management-fee-fetch-management-fee-snapshot)
6. [fetch-area-cost (Fetch Management Fee by Area)](#fetch-area-cost-fetch-management-fee-by-area)
7. [공통 운영 노트](#공통-운영-노트)

---

## 공통 설계

### 레포 분리

- **이 레포 (`giftbott/fetch-workflow`)**: 워크플로우 정의만. 빌링 분리 + wooridanji 의 actions 사용량 절약.
- **`giftbott/wooridanji`**: fetcher 스크립트(`scripts/*`), 데이터 저장소, 운영 코드.
- 모든 fetch job 은 `actions/checkout@v5` 로 wooridanji 를 가져와 `npm ci` 후 `npx tsx scripts/*.ts` 실행.

### 시크릿

| 이름                  | 용도                                                                |
| --------------------- | ------------------------------------------------------------------- |
| `GH_PAT`              | wooridanji 레포에 push 하기 위한 classic PAT (repo 권한)            |
| `OPEN_DATA_API_KEY`   | data.go.kr 의 부동산 실거래가 / K-apt Open API 키                   |
| `TELEGRAM_BOT_TOKEN`  | 실패 알림 봇 토큰. 미설정 시 알림 step 이 자동 skip                 |
| `TELEGRAM_CHAT_ID`    | 알림 수신 채팅 ID                                                   |

k-apt 웹 크롤링(area-cost) 은 인증 불요.

### Concurrency

- `concurrency.group: data-commit` — fetch-news / fetch-real-estate / fetch-management-fee 가 공유. wooridanji main 에 동시 push 충돌 방지.
- `concurrency.group: area-cost` — fetch-area-cost 만 별도. 실행 시간이 길어 다른 워크플로우 큐를 막지 않게 분리.
- 모두 `cancel-in-progress: false` — 중간에 새 트리거가 와도 진행 중인 run 을 죽이지 않음.

### artifact 흐름 (matrix → 단일 commit)

대부분 워크플로우는 다음 패턴:

```
setup (단지 목록 emit) → fetch (matrix, 단지별/chunk별 수집 + upload-artifact)
                          → commit (download-artifact + git-auto-commit-action)
```

`actions/upload-artifact@v4` 의 `path:` 가 **단일 디렉토리** 이면 디렉토리 prefix 가 trim 되어 artifact 안엔 파일만 들어간다(아카이브 root 가 그 디렉토리). `multi-line path` 로 두 항목 이상을 적으면 두 경로의 **공통 prefix 가 root** 가 되어 디렉토리 구조가 보존된다. fetch-management-fee / fetch-area-cost 둘 다 이 동작을 활용한다 (실제 파일 없는 dummy 두 번째 path 추가).

### 알림 정책

- 실패 시에만 텔레그램 1건. 성공은 침묵 (project memory 정책).
- 워크플로우별 세부 silent 윈도우는 각 섹션 참조.

### 트리거 / 디버깅 명령

```bash
# 수동 dispatch
gh workflow run "<name>" --repo giftbott/fetch-workflow [-f input=value]

# 최근 run 조회
gh run list --repo giftbott/fetch-workflow --workflow "<name>" --limit 5

# run 상세 + 로그
gh run view <run-id> --repo giftbott/fetch-workflow                 # 요약
gh run view <run-id> --repo giftbott/fetch-workflow --log           # 전체 로그
gh run view <run-id> --repo giftbott/fetch-workflow --log-failed    # 실패 step 만

# run 완료까지 대기 (CI 등)
gh run watch <run-id> --repo giftbott/fetch-workflow --exit-status
```

---

## 워크플로우 일람

| 이름                            | 파일                            | 주기                              | 데이터 소스                           | 출력 경로 (wooridanji)                                          |
| ------------------------------- | ------------------------------- | --------------------------------- | ------------------------------------- | --------------------------------------------------------------- |
| Fetch News                      | `fetch-news.yml`                | 매일 07:50 KST (1회)              | Google News RSS                       | `src/data/shared/news/{키워드}.json`                            |
| Fetch Real Estate Data          | `fetch-real-estate.yml`         | 매일 07:00 KST                    | 국토부 실거래가 OpenAPI (매매+전월세) | `public/data/real-estate/{아파트}.json`                         |
| Fetch Management Fee Snapshot   | `fetch-management-fee.yml`      | 매달 23~30일 09:00 KST            | 공공데이터 K-apt Open API (26 EP)     | `src/data/complexes/{slug}/management-fee/{YYYYMM}.json`        |
| Fetch Management Fee by Area    | `fetch-area-cost.yml`           | 매달 24~28일 08:00 KST            | k-apt.go.kr (Playwright + Excel)      | `src/data/real-estate/area-cost/{kaptCode}/{YYYYMM}.json`       |

---

## fetch-news (Fetch News)

- **파일**: `.github/workflows/fetch-news.yml`
- **목적**: 단지별 검색어 뉴스를 한 번에 수집해 뉴스 페이지·홈 위젯에 제공.
- **트리거**:
  - `schedule: '50 22 * * *'` → KST 07:50 (UTC 22:50 전날)
  - `workflow_dispatch` (수동)
- **데이터 소스**: Google News RSS — 각 단지의 `complexes.json` 안의 검색 키워드. 모든 단지의 키워드를 union(중복 제거) 한 다음 한 번에 fetch (단지마다 별도 job 안 만듬).
- **출력**: wooridanji 의 `src/data/shared/news/{키워드}.json`
- **commit file_pattern**: `src/data/shared/news/*.json`
- **단일 job** (matrix 없음, setup/commit 분리도 없음). `npx tsx scripts/fetch-news.ts` 한 번 실행 → 직접 commit.
- **특이사항**:
  - 이전엔 09:00 + 17:00 하루 2회였으나 트래픽 분석 후 **출근 직전 1회로 축소**. 새벽에 쌓인 새 기사를 출근 시간 직전에 한 번만 갱신.
  - Google News RSS 는 인증·rate limit 거의 없음.
  - UI 측 `news.astro` 의 안내 문구도 "매일 07:50 업데이트" 로 동기화돼 있음.

---

## fetch-real-estate (Fetch Real Estate Data)

- **파일**: `.github/workflows/fetch-real-estate.yml`
- **목적**: 단지별 매매·전월세 실거래가를 매일 갱신.
- **트리거**:
  - `schedule: '0 22 * * *'` → KST 07:00 (UTC 22:00 전날)
  - `workflow_dispatch`
- **데이터 소스**: 국토교통부 실거래가 공공 API (매매 + 전월세). API 키 = `OPEN_DATA_API_KEY`.
- **출력**: wooridanji 의 `public/data/real-estate/{아파트}.json`
- **jobs**:
  - `setup` — `complexes.json` sparse-checkout → matrix 출력
  - `fetch` — matrix 단지별로 `scripts/fetch-real-estate.ts` 실행
  - `commit` — artifact download → git-auto-commit
- **특이사항**:
  - 스크립트가 `scripts/fetch-real-estate.ts` 내부에서 **신규 거래 감지 시 텔레그램 알림** (env 로 토큰 전달). 워크플로우 자체엔 별도 알림 step 없음.
  - `public/` 경로에 떨어지므로 빌드 산출물에 그대로 포함 (정적 fetch 가능).

---

## fetch-management-fee (Fetch Management Fee Snapshot)

- **파일**: `.github/workflows/fetch-management-fee.yml`
- **목적**: 메인 단지의 **상세 관리비 스냅샷** (장충금/에너지/개별사용료/공용관리비 26 endpoint) 을 월간 수집. **"단지 공통" 탭 전용 데이터**.
- **트리거**:
  - `schedule: '0 0 23-30 * *'` → KST 09:00, 매달 23~30일
  - `workflow_dispatch`
- **데이터 소스**: 공공데이터포털 K-apt Open API. 인증 = `OPEN_DATA_API_KEY`.
- **대상 월**: `date -d '1 month ago' +'%Y%m'` → **전월** 1개. 미공개면 skip.
- **대상 단지**: `complexes.json` 의 메인 단지 (현재 포레나송파 / 송파레이크힐).
- **출력**: `src/data/complexes/{slug}/management-fee/{YYYYMM}.json`
- **commit file_pattern**: `src/data/complexes/*/management-fee/*.json`
- **idempotency**: fetch step 이 시작 시 `TARGET` 파일 존재 확인 → 있으면 즉시 `exit 0` (스냅샷은 한 번 만들면 변하지 않음).
- **특이사항**:
  - K-apt Open API 가 단지/월 데이터를 아직 안 올렸으면 스크립트가 `[WARN] 데이터 없음 — 파일 저장 스킵 (26/26 엔드포인트 응답)` 출력하고 정상 종료. commit job 은 "Working tree clean" 으로 push 없이 끝남. 다음 cron 이 자동 재시도.
  - **artifact path 는 multi-line** (`src/data/complexes/{slug}/management-fee/` + `failures-fee/{slug}.txt`). 단일 path 면 v4 가 prefix trim 해서 commit job 의 `file_pattern` 매칭 실패 → "Nothing to commit" 으로 끝나는 버그가 있었음 (기존 동작 분석 후 multi-line 으로 수정 완료).

---

## fetch-area-cost (Fetch Management Fee by Area)

- **파일**: `.github/workflows/fetch-area-cost.yml`
- **목적**: 메인+비교 **42 단지** 면적별 관리비 (단위면적당 breakdown) 를 k-apt 에서 엑셀로 받아 JSON 변환.
- **트리거**:
  - `schedule: '0 23 23-27 * *'` → KST 08:00, 매달 24~28일
  - `workflow_dispatch` (입력: `exclude`=쉼표로 kaptCode 제외, `force`=이미 수집된 월도 재수집)
- **데이터 소스**: k-apt.go.kr (인증 불요). Playwright + HTTP hybrid:
  1. `goto main` → `_csrf` 토큰 추출
  2. `request.post('/cmmn/selectKapt.do', {kapt_code, _csrf})` 로 세션에 단지 고정 (302)
  3. `page.goto('/apiinfo/goApiInfoSearch.do?searchDate=YYYYMM&kaptCode=')` 로 검색 결과 페이지 직진입 (UI 클릭 우회)
  4. 콤보 선택 + 엑셀 다운로드 → `scripts/lib/xls-parser.ts` 가 JSON 변환
- **대상 월 계산** (setup job, KST 기준):
  - 1~23일 cron: 전전월까지 (예: 4/23 → 202602)
  - 24일~말일 cron: 전월까지 (예: 4/24 → 202603)
  - 단지별로 `apartments.json` 의 `startMonth`, `occuFirstDate` (입주 시점), 옵션 `endMonth` 고려해 실제 가능한 월 범위 계산
  - 기존 `area-cost/{kaptCode}/{YYYYMM}.json` 존재하는 월은 제외 (`force=true` 면 모두 재시도)
- **chunk 구성**: 단지 × 2년 단위 (1년 단위면 ~336 chunk → matrix 256 한도 초과, 2년이면 ~180 선)
- **출력**: `src/data/real-estate/area-cost/{kaptCode}/{YYYYMM}.json`
- **commit file_pattern**: `src/data/real-estate/area-cost/**/*.json`
- **jobs**:
  - `setup` — apartments.json + 기존 area-cost 디렉터리 sparse-checkout → 누락 월 chunks 계산
  - `fetch` — chunk matrix, **`max-parallel: 5`** (k-apt rate-limit 완화). 각 chunk 가 자기 chunk 의 월들을 순차 시도.
  - `commit` — failures/*.txt 집계 → git-auto-commit + (조건부) 텔레그램 요약
- **rate-limit 처리**:
  - chunk 단위로 60초 쿨다운, 연속 3회 rate-limit 시 chunk 조기 종료
  - 실패 월은 `failures/{kaptCode}-{yearLabel}.txt` 에 기록 → commit job 이 집계
- **silent 윈도우** (cron 만 적용, dispatch 는 무시):
  | KST 일자 | 실패 시 동작 |
  |---|---|
  | 24일 | 무조건 silent (chunk green, 알림 skip) — k-apt 롤링 공개 첫날 |
  | 25일 | 무조건 silent — 롤링 공개 진행 중 |
  | 26일 | 토·일 또는 한국 공휴일이면 silent / 평일이고 공휴일 아니면 알림 |
  | 27·28일 | 항상 알림 (실제 이슈 가능성) |
  | 수동 dispatch | 항상 알림 |
  공휴일은 **인라인 `HOLIDAYS_KR`** (yaml 안에 2026 리스트). 매년 12월 말 갱신 필요. chunk fail 분기와 알림 분기 양쪽에 동일 로직 중복 작성.
- **특이사항**:
  - 새 코드(scripts/fetch-area-cost.ts) 와 데이터(src/data/real-estate/area-cost/) 가 모두 wooridanji 의 main 에 있음. 과거에는 feat 브랜치 ref 가 yml 에 있었으나 머지 후 제거.
  - Playwright 브라우저 cache 활용 (cache miss 시 `playwright install --with-deps` 추가 시간).
  - k-apt 가 단지/월 데이터를 안 올렸으면 페이지 alert 로 `관리비 데이터가 존재해야 엑셀 다운로드가 가능합니다` 가 뜬다 → 스크립트가 fail 처리. 25일 새벽까지도 17/42 단지가 미공개인 경우는 흔하다 (silent 윈도우 25일 추가 이유).

---

## 공통 운영 노트

### wooridanji 의 어떤 파일이 변경되는가

push 발생하는 파일 패턴 한눈에:

| 워크플로우            | file_pattern                                          |
| --------------------- | ----------------------------------------------------- |
| fetch-news            | `src/data/shared/news/*.json`                         |
| fetch-real-estate     | `public/data/real-estate/*.json`                      |
| fetch-management-fee  | `src/data/complexes/*/management-fee/*.json`          |
| fetch-area-cost       | `src/data/real-estate/area-cost/**/*.json`            |

운영 서버(`~/wooridanji`) 는 cron 으로 main pull 하고 있어 자동 반영. 즉시 빌드 반영하려면 사용자가 `make publish` 또는 `make deploy`.

### 실패가 났을 때 흐름

1. **chunk red** → workflow run 전체가 빨강 표시
2. **commit job 의 Decide notification policy** → silent 윈도우 아니면 `alert=true`
3. **Telegram failure summary** step → 실패 단지/월 요약을 1건의 메시지로 발송
4. (텔레그램 토큰 미설정 시 step 자체 skip — 안전)

### 매년 12월 말 해야 할 것

- `fetch-area-cost.yml` 의 `HOLIDAYS_KR` 인라인 리스트를 다음 해 한국 공휴일로 갱신 (chunk silent 분기 + 알림 silent 분기 양쪽).
- 잊으면: 새해 1월에 26일 공휴일이 평일로 잘못 분류되어 알림이 가는 정도의 영향 (매월 26일 평일+공휴일 케이스가 적어 큰 사고는 아님).

### 트러블슈팅 체크리스트

- "Nothing to commit" 인데 데이터는 수집된 듯 → upload-artifact path 가 단일 path 인지 확인 (multi-line 필요)
- 텔레그램 알림 안 옴 → silent 윈도우인지 KST 일자/요일 확인, 또는 토큰 시크릿 누락
- chunk 다수 실패 → k-apt 미공개 vs rate-limit vs 우리 스크립트 버그 구분. 실패 chunk 의 `--log-failed` 로 `[DIALOG:alert]` 메시지 보면 미공개, `combo=false` 보면 rate-limit
- main 에 push 안 됨 → `GH_PAT` 만료 또는 권한 부족 의심
