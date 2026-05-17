# 16. 4-Cadence 대시보드 구성 — 실시간 / 일간 / 주간 / 월간

> **선수**: [09c. 메트릭 우선순위](09c-metric-priority.md), [11~15. 메트릭 레시피](11-recipe-sre.md)
>
> 52 메트릭을 *시간 주기* 별로 4개 대시보드에 분배. 각 대시보드는 **명확한 사용자 + 명확한 의사결정** 을 담당. 동일 메트릭이 여러 대시보드에 들어갈 수 있으나 *시간 범위와 표현 방식이 다름*.

---

## 0. 한눈에 보기 — 4 Dashboard 매트릭스

| 대시보드 | 사용자 | Refresh | 시간 범위 | 메트릭 수 | 의사결정 |
|---|---|---|---|:--:|---|
| **🚨 D1. 실시간 (Real-time)** | SRE on-call | **30초** | Last 24 hours | 12 | 지금 정상인가? 사고 인지 |
| **📋 D2. 일간 점검 (Daily)** | 운영팀 리더 | 1시간 | Yesterday (절대) | 14 | 어제 어땠나? 오늘 무엇 봐야? |
| **📊 D3. 주간 점검 (Weekly)** | 매니저 / 개발팀 | 1일 | Last 7 days | 14 | 추세는? 회귀는? |
| **📈 D4. 월간 점검 (Monthly)** | 임원 / PM | 1주 | Last 30 days | 12 | 비즈니스 KPI? capacity? |

**핵심 원칙**:
- **D1** = SLO·페이저 — 즉각 행동
- **D2** = 매일 아침 standup — 어제 회고
- **D3** = 매주 월요일 보고 — 추세 분석
- **D4** = 매월 1일 보고 — 전략 의사결정

---

## 0-A. Lens 설정 가이드 — 본 문서 사용법

> 본 문서만 보고 Lens / Dashboard 모두 만들 수 있도록 각 메트릭에 **Lens 설정 블록**을 인라인으로 첨부했다.

### Lens 설정 블록 읽는 법

각 메트릭은 아래 형식으로 표기:

```
#### M-XXX 메트릭 이름

- **차트**: <Metric|Line|Bar|Heatmap|Pie|Table|Gauge>
- **Data view**: `msa-logs-*`   **시간**: <대시보드 픽커 따름 / 또는 강제 범위>
- **Filter (KQL)**: <KQL 식 또는 "없음">
- **차원 / 함수**:
  - <축 이름>: <필드 + 집계 + 옵션>
  - ...
- **Format**: <Number / Percent / Duration, 소수점 자리>
- **Color/Threshold**: <조건부 색칠 규칙>
- **Title**: `<라이브러리 저장 이름>`
```

### 공통 사전 조건

1. **Data view**: `msa-logs-*` (없으면 Stack Management → Data Views → Create. Time field = `@timestamp`)
2. **시간 픽커**: 상단 우측. "Last X hours" 또는 "Yesterday" (Absolute → Calendar → Yesterday)
3. **Filter 입력**: Lens 상단 KQL bar — 예: `sts_c : "ERROR"`
4. **Save**: 우상단 Save → **Add to library 토글 ON** → 위 `Title` 그대로 입력

### 자주 쓰는 Formula

| 메트릭 | Formula |
|---|---|
| **Availability %** | `(count(kql='sts_c : "OK"') / count()) * 100` |
| **Error Rate %** | `(count(kql='sts_c : "ERROR"') / count()) * 100` |
| **TPS (초당)** | `count() / time_range_in_seconds()` 또는 단순 `count() / 3600` (1h 고정 시) |
| **p95** | `percentile(proc_tm, percentile=95)` |
| **Filter ratio** | `count(kql='<조건>') / count()` |

### 필드 cheat sheet (06a 발췌)

| 필드 | 값 | 설명 |
|---|---|---|
| `sts_c` | `"OK"` / `"ERROR"` | 거래 상태 |
| `msg_c` | `"00000000"` 또는 에러코드 | 메시지 코드 |
| `svc_c` | BU/CD/FA/MB/PU/PC/PY/PP/WP/PA/AS | 11 MSA |
| `svc_id` | (string) | endpoint |
| `log_div` | `*_IN`/`*_OUT`/`MCI_*`/`GW_*` | 로그 구분 |
| `tier_c` | PT / BT / MCI | 계층 |
| `chan_c` | MA / MW / PW / DA | 채널 |
| `os_kdc` | I / A / W | OS |
| `ap_ver` | (string) | 앱 버전 |
| `device_model` | (string) | 기기 모델 |
| `proc_tm` | (number, ms) | 처리시간 |
| `guid` | (keyword) | 거래 root ID |
| `dgtl_cusno` | (keyword) | 디지털 고객번호 (사용자 식별) |
| `ctnr_nm` | (keyword) | 컨테이너 이름 |
| `fir_c` | (keyword) | 최초 에러 발생 코드 |
| `takes.total_ms` | (number) | Logstash 파이프라인 지연 |

> 자세한 필드 의미는 [09a. 실 운영 필드 매핑](09a-real-field-mapping.md) 참고.

---

## D1. 🚨 실시간 대시보드 (Real-time)

> **목적**: SRE on-call 이 사고 발생 시 5초 안에 *어디서 무엇이* 깨졌는지 진단
> **갱신**: 30초 자동 / **시간**: Last 24 hours / **인원**: SRE 1~2명 24/7

### 포함 메트릭 (12개)

| 영역 | 메트릭 | 차트 타입 | 시간 | 임계 |
|---|---|---|---|---|
| **SLO 4종** | M-S1 Availability | Metric | Last 1h | < 99.9% 빨강 |
| | M-S3 Error Rate | Metric | Last 1h | > 0.5% 빨강 |
| | M-S4 p95 Latency | Metric | Last 1h | > 500ms 노랑 |
| | M-S2 TPS | Metric (+sparkline) | Last 1h | (참조) |
| **Trend** | M-S2 TPS time series | Line | Last 24h | spike 자동 감지 |
| | M-S4 p50/p95/p99 | Line | Last 24h | tail 비율 |
| **외부 의존성** | M-S8 G/W p95 | Metric | Last 1h | > 2s |
| | M-S9 MCI core p95 | Metric | Last 1h | > 500ms |
| **Health Matrix** | M-P1 11 MSA Health | Table | Last 1h | conditional color |
| **Container** | M-P8 Container Error Rate | Bar horizontal | Last 1h | 단일 pod >5% |
| **메타 모니터** | M-P10 Pipeline Lag | Metric | Last 5m | > 5min P0 |
| **즉각 실패** | M-P4 Stuck Requests count | Metric | Last 30m | > 100 P1 |

### Lens 설정 상세 (D1)

> D1 의 모든 SLO Metric 은 **Lens 자체 시간 (Last 1 hour) 을 강제** 한다. 방법: Lens 편집 → 패널 우상단 ⚙️ → "Use time range from dashboard" 토글 OFF → "Custom" → Last 1 hour. (대시보드 픽커가 24h 여도 메트릭은 1h 로 계산.)

#### M-S1 Availability (1h)

- **차트**: Metric
- **시간**: Last 1 hour (강제)
- **Filter (KQL)**: 없음
- **Primary metric → Formula**:
  ```
  (count(kql='sts_c : "OK"') / count()) * 100
  ```
- **Format**: Percent, 소수점 2자리
- **Color (Static thresholds)**: `< 99.9` 빨강 / `99.9~99.95` 노랑 / `≥ 99.95` 녹색
- **Title**: `M-S1 Availability (1h)`

#### M-S3 Error Rate (1h)

- **차트**: Metric | **시간**: Last 1 hour (강제) | **Filter**: 없음
- **Primary metric → Formula**:
  ```
  (count(kql='sts_c : "ERROR"') / count()) * 100
  ```
- **Format**: Percent, 2 dec. | **Color**: `> 0.5` 빨강 / `> 0.2` 노랑
- **Title**: `M-S3 Error Rate (1h)`

#### M-S4 p95 Latency (1h)

- **차트**: Metric | **시간**: Last 1 hour (강제) | **Filter**: 없음
- **Primary metric**: `Percentile` of `proc_tm`, percentile = 95
- **Format**: Number, suffix `ms`
- **Color**: `> 500` 빨강 / `> 300` 노랑
- **Title**: `M-S4 p95 Latency (1h)`

#### M-S2 TPS (1h, sparkline)

- **차트**: Metric (with trend line) | **시간**: Last 1 hour (강제) | **Filter**: 없음
- **Primary metric → Formula**:
  ```
  count() / 3600
  ```
- **Trend line**: dimension 설정 → "Show trend line" ON (Date histogram Auto)
- **Format**: Number, suffix `/s`, 1 dec.
- **Title**: `M-S2 TPS (1h avg)`

#### M-S2 TPS Trend (24h Line)

- **차트**: Line | **시간**: 대시보드 픽커 (Last 24h)
- **Horizontal axis**: `@timestamp`, Date histogram, **Interval = Auto** (Lens가 5m bucket 자동 선택)
- **Vertical axis**: `Count of records`
- **Breakdown**: 없음
- **Title**: `M-S2 TPS Trend (24h)`

#### M-S4 p50/p95/p99 (24h Line)

- **차트**: Line | **시간**: Last 24 hours | **Filter**: 없음 (또는 `sts_c : "OK"` 로 성공만)
- **Horizontal axis**: `@timestamp`, Date histogram Auto
- **Vertical axis 3개 layer** (모두 `proc_tm` 의 Percentile):
  - p50 → 회색
  - p95 → 주황
  - p99 → 빨강
- **추가 방법**: 첫 metric 추가 후 dimension 박스에 같은 필드를 다시 드래그 → "Add" 선택 (3번 반복)
- **Title**: `M-S4 Latency p50/p95/p99 (24h)`

#### M-S8 G/W p95 (1h)

- **차트**: Metric | **시간**: Last 1 hour (강제)
- **Filter (KQL)**: `log_div : ("GW_SEND" or "GW_RECV")`
- **Primary metric**: Percentile of `proc_tm`, p95
- **Format**: Number, `ms` | **Color**: `> 2000` 빨강 / `> 1000` 노랑
- **Title**: `M-S8 G/W p95 (1h)`

#### M-S9 MCI Core p95 (1h)

- **차트**: Metric | **시간**: Last 1 hour (강제)
- **Filter (KQL)**: `log_div : ("MCI_SEND" or "MCI_RECV")`
- **Primary metric**: Percentile of `proc_tm`, p95
- **Format**: Number, `ms` | **Color**: `> 500` 빨강 / `> 300` 노랑
- **Title**: `M-S9 MCI Core p95 (1h)`

#### M-P1 11 MSA Health Matrix (Table)

- **차트**: **Table** | **시간**: Last 1 hour (강제) | **Filter**: 없음
- **Rows (그룹)**: `svc_c`, terms, **Top 11**, order by count desc
- **Metrics (열) — 4개**:
  1. **Availability** (Formula): `(count(kql='sts_c:"OK"') / count()) * 100` — Percent, 2 dec.
  2. **p95**: Percentile of `proc_tm`, p95 — Number `ms`
  3. **TPS** (Formula): `count() / 3600` — Number `/s`
  4. **Errors** (Formula): `count(kql='sts_c:"ERROR"')` — Number
- **Conditional color** (Availability 컬럼): dimension → Color → Add palette → `< 99.9` 빨강 / `< 99.95` 노랑
- **Title**: `M-P1 11 MSA Health (1h)`

#### M-P8 Container Error Rate (Bar horizontal)

- **차트**: Horizontal Bar | **시간**: Last 1 hour (강제) | **Filter**: 없음
- **Vertical axis (categorical)**: `ctnr_nm`, terms, **Top 30**, order by metric desc
- **Horizontal axis (Formula)**:
  ```
  (count(kql='sts_c:"ERROR"') / count()) * 100
  ```
- **Format**: Percent, 1 dec.
- **Color**: Static color > 5% 빨강 (Bar Color → "Add color stops")
- **Title**: `M-P8 Container Error Rate (1h)`

#### M-P10 Pipeline Lag (5m max)

- **차트**: Metric | **시간**: Last 5 minutes (강제)
- **Filter**: 없음
- **Primary metric**: Maximum of `takes.total_ms` (환경에 따라 `takes.logstash_ms` 등으로 대체)
- **Format**: Number, `ms` (또는 Formula `max(takes.total_ms) / 1000` 으로 `s` 표시)
- **Color**: `> 300000` (5분) 빨강
- **Title**: `M-P10 Pipeline Lag (5m max)`

#### M-P4 Stuck Requests (30m)

- **차트**: Metric | **시간**: Last 30 minutes (강제)
- **간이 정의** (Lens만으로): `proc_tm > 30000` 인 `*_IN` 로그 개수
- **Filter (KQL)**: `proc_tm > 30000 and log_div : *_IN`
- **Primary metric**: `Count of records`
- **Format**: Number | **Color**: `> 100` 빨강
- **Note**: 정확한 stuck 정의 (IN 있으나 OUT 없는 guid) 는 Transform 사용 — [12 §M-P4](12-recipe-backend.md) 참고.
- **Title**: `M-P4 Stuck Requests (30m)`

### 권장 레이아웃

```
┌─────────────────────────────────────────────────────────────────────┐
│ 🟢 SLO 4종 (Top — 가장 큰 글씨)                                       │
│ ┌──────────┬──────────┬──────────┬──────────┐                        │
│ │ Avail    │ Error %  │ p95 ms   │ TPS      │                        │
│ │ 99.94%   │ 0.32%    │ 412ms    │ 11.2K    │                        │
│ └──────────┴──────────┴──────────┴──────────┘                        │
├─────────────────────────────────────────────────────────────────────┤
│ 📈 TPS 24h Line (트래픽 패턴)                                          │
├─────────────────────────────────────────────────────────────────────┤
│ 📈 Latency p50/p95/p99 24h Line (3 색)                                │
├─────────────────────────┬───────────────────────────────────────────┤
│ 🟧 G/W p95 (M-S8)      │ 🟧 MCI core p95 (M-S9)                     │
├─────────────────────────┴───────────────────────────────────────────┤
│ 📊 11 MSA Health Matrix (M-P1, conditional color)                    │
│ ┌────┬────────┬─────┬──────┬────────┐                                │
│ │svc │ Avail  │p95  │ TPS  │ Errors │                                │
│ │PP  │ 99.95% │ 120 │ 3.2K │   2    │                                │
│ │PY  │ ⚠️96%  │ 180 │ 2.5K │  120 ⚠️ │ ← 빨강                         │
│ │... │        │     │      │        │                                │
│ └────┴────────┴─────┴──────┴────────┘                                │
├─────────────────────────────────────────────────────────────────────┤
│ 📊 Container Error Rate (M-P8) Top 30                                │
├──────────────────────────────────────┬──────────────────────────────┤
│ ⏱️ Pipeline Lag (M-P10)               │ 🟥 Stuck Count (M-P4)         │
│   3.2s ✅                             │   12 docs                     │
└──────────────────────────────────────┴──────────────────────────────┘
```

### 운영 절차

```
【0초~5초】SLO 4종 + Pipeline Lag 확인
  └─ 정상 → 종료
  └─ Avail/Error 빨강 → 다음 단계

【5초~30초】M-P1 매트릭스 → 어느 svc_c?
  └─ 1개 svc_c 만 → 그 팀에 ping
  └─ 모든 svc_c → 인프라/네트워크/MCI 점검

【30초~1분】M-P8 → 단일 pod 인가? 전체인가?
  └─ 단일 pod → k8s pod 재시작
  └─ 전체 → 코드 회귀 의심 → 최근 배포 rollback 검토
```

### Save 정보

- **Title**: `D1 실시간 (Real-time SRE)`
- **Refresh**: 30 seconds
- **시간 범위**: Last 24 hours (Live)
- **권한**: SRE / 플랫폼팀

---

## D2. 📋 일간 점검 대시보드 (Daily)

> **목적**: 매일 아침 standup 전 *어제* 어땠는지 회고 + 오늘 봐야 할 것
> **갱신**: 1시간 / **시간**: **Yesterday (절대)** / **인원**: 운영팀 리더 + 백엔드 team lead

### 포함 메트릭 (14개)

| 영역 | 메트릭 | 차트 타입 | 시간 | 학습 포인트 |
|---|---|---|---|---|
| **어제 결과** | M-S1 어제 Avail | Metric | Yesterday | SLO 위반 여부 |
| | M-S3 어제 Error % | Metric | Yesterday | budget burn |
| | M-S4 어제 p95 | Metric | Yesterday | latency 회귀 |
| | M-S2 어제 peak TPS | Metric | Yesterday | capacity |
| **에러 분석** | M-D1 Top 20 msg_c | Bar horizontal | Yesterday | E003 dominant 등 |
| | M-D2 신규 msg_c | List/Metric | Yesterday vs 7d | E099 발견 |
| | M-D9 fir_c 분포 | Pie | Yesterday | MCI vs 사내 |
| | M-D4 MSA × msg_c | Heatmap | Yesterday | 책임 영역 |
| **거래 무결성** | M-P3 In/Out Imbalance Top 10 | Bar | Yesterday | 누락 svc_id |
| | M-P5 Inter-Tier Latency | Line | Yesterday | bottleneck |
| | M-P7 Top 10 svc_id by Error | Bar | Yesterday | 우선 fix |
| **사용자** | M-U6 기기 painPoint Top 10 | Bar | Yesterday | SM-G960N 등 |
| | M-U10 retry 패턴 count | Metric | Yesterday | 이탈 위험 |
| **인프라** | M-X1 Pipeline lag 평균/최대 | Metric | Yesterday | takes.* |

### Lens 설정 상세 (D2)

> D2 는 모든 차트가 **대시보드 픽커의 "Yesterday" (Absolute)** 를 따른다. Lens 편집 시 시간을 굳이 강제하지 말고 픽커 따라가게 둔다. ("Use time range from dashboard" 토글 ON 유지.)

#### M-S1 Avail Yesterday (Metric)

- **차트**: Metric | **시간**: 대시보드 픽커 (Yesterday)
- **Primary metric → Formula**: `(count(kql='sts_c:"OK"') / count()) * 100`
- **Format**: Percent, 2 dec. | **Color**: `< 99.9` 빨강 / `< 99.95` 노랑
- **Title**: `M-S1 Avail (Yesterday)`

#### M-S3 Error % Yesterday (Metric)

- **차트**: Metric | **시간**: 대시보드 픽커
- **Primary metric → Formula**: `(count(kql='sts_c:"ERROR"') / count()) * 100`
- **Format**: Percent, 2 dec. | **Color**: `> 0.5` 빨강
- **Title**: `M-S3 Error % (Yesterday)`

#### M-S4 p95 Yesterday (Metric)

- **차트**: Metric | **시간**: 대시보드 픽커
- **Primary metric**: Percentile of `proc_tm`, 95
- **Format**: Number `ms` | **Color**: `> 500` 빨강
- **Title**: `M-S4 p95 (Yesterday)`

#### M-S2 Peak TPS Yesterday (Bar)

- Lens 의 Metric chart 만으로는 "1분 bucket 의 max" 를 단일 숫자로 못 뽑음. **Bar chart 로 시각 표시** 권장.
- **차트**: Vertical Bar | **시간**: 대시보드 픽커
- **Horizontal axis**: `@timestamp`, Date histogram, **Interval = 1 minute**
- **Vertical axis (Formula)**: `count() / 60` (분당 호출 → RPS)
- **Color/Reference line**: Reference line — Static value (예: 평소 peak) 표시 가능
- **Title**: `M-S2 Peak TPS Trend (Yesterday, 1m)`
- **대안**: 단일 숫자가 꼭 필요하면 TSVB 또는 Transform 사용.

#### M-D1 Top 20 Error msg_c (Bar horizontal)

- **차트**: Horizontal Bar | **시간**: 대시보드 픽커
- **Filter (KQL)**: `sts_c : "ERROR"`
- **Vertical axis (categorical)**: `msg_c`, terms, **Top 20**, order by count desc
- **Horizontal axis**: `Count of records`
- **Title**: `M-D1 Top 20 Error msg_c (Yesterday)`

#### M-D2 New msg_c (Table)

- 정확한 "신규" 판정은 Transform/Saved query 필요. Lens 만으로는 **첫 등장일 (min @timestamp)** 으로 근사.
- **차트**: Table | **시간**: **Last 8 days** (Lens 강제 — 픽커 무시)
- **Filter (KQL)**: `sts_c : "ERROR"`
- **Rows**: `msg_c`, terms, Top 50, order by **Min `@timestamp` desc**
- **Metrics**: ① Min `@timestamp` ② Count of records
- **사용법**: Min @timestamp 가 어제 날짜인 행만 신규로 판단.
- **권장 (정밀)**: ES Transform → [13 §M-D2](13-recipe-domain.md)
- **Title**: `M-D2 New msg_c (8d)`

#### M-D9 fir_c Distribution (Pie)

- **차트**: Pie / Donut | **시간**: 대시보드 픽커
- **Filter (KQL)**: `sts_c : "ERROR" and fir_c : *`
- **Slice by**: `fir_c`, terms, Top 10
- **Size by**: `Count of records`
- **Title**: `M-D9 fir_c Distribution (Yesterday)`

#### M-D4 MSA × msg_c Heatmap

- **차트**: Heatmap | **시간**: 대시보드 픽커
- **Filter (KQL)**: `sts_c : "ERROR"`
- **Horizontal axis**: `msg_c`, terms, Top 20
- **Vertical axis**: `svc_c`, terms, Top 11
- **Cell value**: `Count of records`
- **Color palette**: Sequential (white → red), Min auto, Max auto
- **Title**: `M-D4 MSA × msg_c (Yesterday)`

#### M-P3 In/Out Imbalance Top 10 (Bar)

- **차트**: Vertical Bar (또는 Horizontal Bar) | **시간**: 대시보드 픽커
- **Horizontal axis**: `svc_id`, terms, Top 10, order by formula desc
- **Vertical axis (Formula)**:
  ```
  count(kql='log_div : *_IN') - count(kql='log_div : *_OUT')
  ```
- **Format**: Number (양수 = OUT 누락)
- **Title**: `M-P3 In/Out Imbalance Top 10 (Yesterday)`

#### M-P5 Inter-Tier p95 Latency (Line)

- **차트**: Line | **시간**: 대시보드 픽커
- **Filter (KQL)**: `tier_c : ("PT" or "BT" or "MCI")`
- **Horizontal axis**: `@timestamp`, Date histogram, Interval = Hour
- **Vertical axis**: Percentile of `proc_tm`, 95
- **Breakdown**: `tier_c`, terms, Top 3
- **Title**: `M-P5 Inter-Tier p95 (Yesterday)`

#### M-P7 Top 10 svc_id by Error (Bar horizontal)

- **차트**: Horizontal Bar | **시간**: 대시보드 픽커
- **Filter (KQL)**: `sts_c : "ERROR"`
- **Vertical axis**: `svc_id`, terms, **Top 10**, order by count desc
- **Horizontal axis**: `Count of records`
- **Title**: `M-P7 Top 10 svc_id by Error (Yesterday)`

#### M-U6 Device painPoint Top 10 (Bar horizontal)

- **차트**: Horizontal Bar | **시간**: 대시보드 픽커
- **Filter (KQL)**: `device_model : *` (값 있는 것만)
- **Vertical axis**: `device_model`, terms, Top 10, order by formula desc
- **Horizontal axis (Formula)**:
  ```
  (count(kql='sts_c:"ERROR"') / count()) * 100
  ```
- **Secondary metric (옵션)**: `Count of records` (호출수 함께 보기)
- **Format**: Percent, 1 dec.
- **Tip**: 저빈도 device 노이즈 → "Group by" → `Other` bucket 끄거나 호출수 100+ 만 보려면 별도 saved query
- **Title**: `M-U6 Device painPoint Top 10 (Yesterday)`

#### M-U10 Retry Pattern Count (Metric)

- 정확한 retry 정의 (동일 guid+svc_id N+회) 는 Lens 단독으론 어려움. **근사**: 동일 guid 가 같은 시간대 여러 ERROR.
- **간이 차트**: Metric | **시간**: 대시보드 픽커
- **Filter (KQL)**: `sts_c : "ERROR"`
- **Primary metric**: Unique count of `guid`
- **Format**: Number (대략적 retry 사용자 수)
- **권장 (정밀)**: Transform → 14 §M-U10
- **Title**: `M-U10 Retry Users approx (Yesterday)`

#### M-X1 Pipeline Lag avg/max (Metric)

- **차트**: Metric (Multi-metric) | **시간**: 대시보드 픽커
- **Primary metric**: Average of `takes.total_ms`
- **Secondary metric**: Maximum of `takes.total_ms`
- **Format**: Number, `ms`
- **Color (Secondary)**: `> 5000` 빨강
- **Title**: `M-X1 Pipeline Lag avg/max (Yesterday)`

### 권장 레이아웃

```
┌─────────────────────────────────────────────────────────────────────┐
│ 어제 SLO 4종 + 신규 msg_c                                              │
│ ┌──────┬──────┬───────┬───────┬──────────┐                           │
│ │Avail │Error │p95 ms │peak T │신규 msg_c│                           │
│ │99.91%│0.4%  │ 425   │ 14K   │  1 (E099)│                           │
│ └──────┴──────┴───────┴───────┴──────────┘                           │
├─────────────────────────────────────────────────────────────────────┤
│ 📊 M-D1 Top 20 msg_c (Bar horizontal, 어제만)                         │
├─────────────────────────┬───────────────────────────────────────────┤
│ 🍩 M-D9 fir_c 분포 (Pie) │ 📊 M-D4 MSA × msg_c heatmap                │
├─────────────────────────┴───────────────────────────────────────────┤
│ 📊 M-P5 PT→BT, BT→MCI Latency (Line, 어제 24h)                       │
├──────────────────────────────────────┬──────────────────────────────┤
│ 📊 M-P7 Top 10 svc_id by Error       │ 📊 M-P3 In/Out 누락 Top 10    │
├──────────────────────────────────────┴──────────────────────────────┤
│ 📊 M-U6 기기 painPoint Top 10 (Bar)                                   │
├──────────────────────────────────────┬──────────────────────────────┤
│ M-U10 retry 사용자 수: 248명          │ M-X1 Pipeline lag avg: 162ms  │
│  → CX 팀에 명단 전달                  │       max: 4.1s               │
└──────────────────────────────────────┴──────────────────────────────┘
```

### 운영 절차 — 매일 아침 standup 체크리스트

```
□ SLO 4종 — 어제 SLO 위반 여부 → 위반 시 root cause 검색
□ 신규 msg_c 등장 — 있으면 1순위 backlog 추가
□ Top msg_c — 1위가 어제와 동일한지? (만성 결함)
□ fir_c 분포 — MCI 비율 60% 이상 정상, BT 가 50%+ 면 코드 회귀 의심
□ M-P3 In/Out 누락 — 1% 이상 svc_id 있나? 있으면 pipeline 점검
□ M-U6 painPoint — 새 기기 모델 등장? QA 매트릭스 추가
□ retry 사용자 명단 → CX 팀 전달
```

### Save 정보

- **Title**: `D2 일간 점검 (Daily Ops Review)`
- **Refresh**: 1 hour
- **시간 범위**: Yesterday (Absolute, 매일 자정 갱신)
- **권한**: 운영팀 + 백엔드 lead

---

## D3. 📊 주간 점검 대시보드 (Weekly)

> **목적**: 매주 월요일 *지난주 vs 그 전주* 추세 비교 → 매니저 보고
> **갱신**: 1일 / **시간**: **Last 7 days** + timeshift 1주 / **인원**: 매니저 + 개발팀 leads

### 포함 메트릭 (14개)

| 영역 | 메트릭 | 차트 타입 | 시간 | 비교 |
|---|---|---|---|---|
| **SLO 트렌드** | M-S10 Error Budget Burn | Metric (gauge) | Rolling 30d | 잔여 budget |
| | M-S4 p95 (week over week) | Line + timeshift | 7d, shift 1w | 회귀? |
| | M-S1 7d avg Avail | Metric | Last 7d | SLO |
| **트래픽** | M-S2 7d TPS pattern | Line | Last 7d | DoW 패턴 |
| | M-O5 DoD/WoW Traffic | Line + timeshift | 7d, shift 1w | spike 감지 |
| | M-D11 동시 활성 거래 peak | Metric | Last 7d | concurrency |
| **비즈니스 KPI** | M-D5 결제 Funnel (PY) | Funnel/Bar | Last 7d | drop rate |
| | M-D6 인증 성공률 (MB) | Metric | Last 7d | 보안 |
| | M-D8 이체 성공률 (FA) | Metric | Last 7d | 핵심 거래 |
| **사용자** | M-U1 채널별 share trend | Line | Last 7d | MA vs MW |
| | M-U7 앱 버전 분포 | Donut | Last 7d | 강제 업데이트 검토 |
| | M-U8 신 앱 ver 회귀 | Bar (timeshift) | Last 14d | 4.5.1 회귀? |
| | M-U11 WAU | Metric | Last 7d | 활성 사용자 |
| **신 결함** | M-D2 신규 msg_c (이번 주) | List | Last 7d | base 외 |

### Lens 설정 상세 (D3)

> D3 의 핵심은 **time shift** (WoW 비교). dimension 박스 → "Add advanced options" → "Time shift" → `1w`.

#### M-S10 Error Budget Burn (Gauge)

- **차트**: Gauge | **시간**: Last 30 days (강제 — 픽커 7d 무시)
- **Metric → Formula**:
  ```
  (count(kql='sts_c:"ERROR"') / count()) / 0.001
  ```
  (SLO 99.9% 가정 → 분모 0.001 = 허용 에러율. 결과 = burn 비율 0~1+)
- **Gauge max**: `1.0`
- **Threshold**: `0.8` 노랑 / `1.0` 빨강
- **Format**: Percent (× 100)
- **Title**: `M-S10 Error Budget Burn (30d rolling)`

#### M-S4 p95 WoW (Line + timeshift)

- **차트**: Line | **시간**: 대시보드 픽커 (Last 7 days)
- **Horizontal axis**: `@timestamp`, Date histogram, Interval = Auto
- **Vertical axis 2 layer**:
  1. p95 of `proc_tm` (이번 주) — 진한 파랑
  2. p95 of `proc_tm`, **Time shift = `1w`** (지난주) — 회색 점선
- **Time shift 설정 위치**: 두 번째 metric dimension → 클릭 → "Add advanced options" → Time shift → `1w`
- **Title**: `M-S4 p95 WoW (7d)`

#### M-S1 Avg Avail (7d Metric)

- **차트**: Metric | **시간**: 대시보드 픽커 (7d)
- **Formula**: `(count(kql='sts_c:"OK"') / count()) * 100`
- **Format**: Percent, 2 dec.
- **Title**: `M-S1 Avg Avail (7d)`

#### M-S2 TPS Pattern (7d Line)

- **차트**: Line | **시간**: 대시보드 픽커
- **Horizontal axis**: `@timestamp`, Date histogram, Interval = Hour
- **Vertical axis**: `Count of records`
- **Breakdown**: 없음 (DoW 패턴은 line 자체로 visible)
- **Title**: `M-S2 TPS Pattern (7d)`

#### M-O5 DoD vs WoW Traffic (Line + timeshift)

- **차트**: Line | **시간**: 대시보드 픽커
- **Horizontal axis**: `@timestamp`, Date histogram, Interval = Day
- **Vertical axis 2 layer**:
  1. `Count of records` (이번 주) — 파랑
  2. `Count of records`, **Time shift = `1w`** (지난주) — 회색
- **Title**: `M-O5 Daily Traffic WoW (7d)`

#### M-D11 Concurrent Active Peak (Metric)

- 정확한 "동시 활성 guid (active session)" 는 Lens 단독으로 어려움. 근사:
- **차트**: Metric | **시간**: 대시보드 픽커
- **Primary metric**: Unique count of `guid` (7일 전체 unique → "활성 사용자수")
- **Title**: `M-D11 Active guid (7d)`
- **권장 (정밀)**: 1m bucket cardinality 의 max → Transform 으로 별도 인덱스화

#### M-D5 Payment Funnel — PY (Bar)

- **차트**: Vertical Bar | **시간**: 대시보드 픽커
- **Filter (KQL)**: `svc_c : "PY"`
- **Horizontal axis (Filters bucket — 4단계)**:
  - `svc_id : *PRODUCT*VIEW* and log_div : *_OUT` (조회)
  - `svc_id : *CARD*SELECT* and log_div : *_OUT` (카드 선택)
  - `svc_id : *PAY*REQ* and log_div : *_OUT` (결제 요청)
  - `svc_id : *PAY*OK* and log_div : *_OUT and sts_c : "OK"` (결제 완료)
  > 실제 svc_id 패턴은 환경 따라 조정. [13 §M-D5](13-recipe-domain.md) 참고.
- **Vertical axis**: Unique count of `guid`
- **Title**: `M-D5 Payment Funnel (7d)`
- **Note**: Kibana 정식 "Funnel" 차트는 없음. Bar 단계별 시각화 또는 Canvas 활용.

#### M-D6 Auth Success Rate — MB (Metric)

- **차트**: Metric | **시간**: 대시보드 픽커
- **Filter (KQL)**: `svc_c : "MB" and svc_id : *AUTH*` (환경 조정)
- **Formula**: `(count(kql='sts_c:"OK"') / count()) * 100`
- **Format**: Percent, 2 dec. | **Color**: `< 99.5` 노랑 / `< 99` 빨강
- **Title**: `M-D6 Auth Success Rate (7d)`

#### M-D8 Transfer Success Rate — FA (Metric)

- **차트**: Metric | **시간**: 대시보드 픽커
- **Filter (KQL)**: `svc_c : "FA" and svc_id : *TRANSFER*` (환경 조정)
- **Formula**: 동일
- **Format**: Percent, 2 dec. | **Color**: `< 99.95` 빨강
- **Title**: `M-D8 Transfer Success Rate (7d)`

#### M-U1 Channel Share (Line, 100% stacked)

- **차트**: Area (Percentage / 100% stacked) | **시간**: 대시보드 픽커
- **Horizontal axis**: `@timestamp`, Date histogram, Interval = Day
- **Vertical axis**: `Count of records`
- **Breakdown**: `chan_c`, terms, Top 4 (MA, MW, PW, DA)
- **Display**: 우상단 Y axis 형식 → "Stacked **Percentage**" (= 100% normalize)
- **Title**: `M-U1 Channel Share (7d)`

#### M-U7 App Version Share (Donut)

- **차트**: Donut | **시간**: 대시보드 픽커
- **Filter (KQL)**: `chan_c : "MA"` (앱만)
- **Slice by**: `ap_ver`, terms, Top 10
- **Size by**: Unique count of `guid`
- **Title**: `M-U7 App Version Share (7d)`

#### M-U8 App Version Regression (Bar timeshift)

- **차트**: Vertical Bar | **시간**: **Last 14 days** (Lens 강제)
- **Filter (KQL)**: `chan_c : "MA"`
- **Horizontal axis**: `ap_ver`, terms, Top 5
- **Vertical axis 2 layer**:
  1. `(count(kql='sts_c:"ERROR"') / count()) * 100` (이번 주)
  2. 같은 Formula, **Time shift = `1w`** (지난주)
- **Format**: Percent, 2 dec.
- **Title**: `M-U8 ap_ver Regression WoW (14d)`

#### M-U11 WAU (Metric)

- **차트**: Metric | **시간**: 대시보드 픽커 (7d)
- **Primary metric**: Unique count of `dgtl_cusno`
- **Format**: Number
- **Title**: `M-U11 WAU (7d)`

#### M-D2 New msg_c — 7d (Table)

- D2 의 동일 구조, **시간만 Last 8 days → 첫 등장이 이번 주에 시작된 것**으로 판정
- **Title**: `M-D2 New msg_c (7d window)`

### 권장 레이아웃

```
┌─────────────────────────────────────────────────────────────────────┐
│ 주간 KPI 4종 + Error Budget                                            │
│ ┌──────┬─────┬───────┬───────┬─────────────┐                         │
│ │Avail │Err% │p95 ms │ WAU   │Budget burn  │                         │
│ │99.92%│0.38%│ 405   │ 165K  │ 0.62 (62%)  │                         │
│ └──────┴─────┴───────┴───────┴─────────────┘                         │
├─────────────────────────────────────────────────────────────────────┤
│ 📈 M-S2 7d TPS Pattern (Line, DoW 패턴 시각화)                        │
├─────────────────────────────────────────────────────────────────────┤
│ 📈 M-S4 p95 (이번 주 vs 지난주, timeshift)                            │
├─────────────────────────────────────────────────────────────────────┤
│ 📊 M-O5 DoD/WoW Traffic (anomaly day 자동 발견)                       │
├─────────────────────────────────────────────────────────────────────┤
│ 비즈니스 KPI 3종 (가로 배치)                                            │
│ ┌──────────────┬──────────────┬──────────────┐                       │
│ │ M-D5 결제    │ M-D6 인증    │ M-D8 이체    │                       │
│ │ Funnel       │ 99.85% →99.82│ 99.97%       │                       │
│ │ 100→80→60→50 │              │              │                       │
│ └──────────────┴──────────────┴──────────────┘                       │
├──────────────────────────────────────┬──────────────────────────────┤
│ 🍩 M-U1 채널별 share (이번 주)        │ 🍩 M-U7 앱 버전 분포          │
├──────────────────────────────────────┴──────────────────────────────┤
│ 📊 M-U8 신 앱 ver 회귀 (4.5.1 vs others, 14d)                         │
├──────────────────────────────────────┬──────────────────────────────┤
│ 신규 msg_c 이번 주: E099 (12건)       │ 동시 활성 peak: 3.2K guid    │
└──────────────────────────────────────┴──────────────────────────────┘
```

### 운영 절차 — 매주 월요일 매니저 보고

```
1. Error Budget burn 비율 → 80%+ 면 다음 주 신 배포 자제 권고
2. p95 timeshift → 이번 주 +20% 회귀 시 retro
3. M-O5 anomaly → 이벤트성인가 사고였나 분류
4. 비즈니스 KPI 3종 → 매니저 ppt 자동 import
5. 신 앱 ver 회귀 → 배포 후 24h 내 발견됐는지 검증
6. 신규 msg_c → 다음 주 backlog 추가
7. 채널 share 변화 → 마케팅 / 모바일팀 협의
```

### Save 정보

- **Title**: `D3 주간 점검 (Weekly Review)`
- **Refresh**: 1 day
- **시간 범위**: Last 7 days
- **권한**: 매니저 + lead

---

## D4. 📈 월간 점검 대시보드 (Monthly)

> **목적**: 매월 1일 *전월 vs 그 전월* 비즈니스 KPI + capacity 예측 → 임원 보고
> **갱신**: 1주 / **시간**: **Last 30 days** + 가능하면 timeshift 1m / **인원**: 임원 + PM + 인프라 lead

### 포함 메트릭 (12개)

| 영역 | 메트릭 | 차트 타입 | 시간 | 보고 가치 |
|---|---|---|---|---|
| **SLA 보고** | M-S1 월간 Avail | Metric | Last 30d | SLA 99.9% 달성? |
| | M-S10 Error Budget 잔여 | Metric (gauge) | Last 30d | 다음 달 위험 예측 |
| | M-S4 월간 p95 분포 | Line | Last 30d | UX 추세 |
| **비즈니스 KPI** | M-D5 결제 Funnel 월 | Funnel | Last 30d | UX 효과 |
| | M-D7 카드 거래 추세 | Line | Last 30d | 성장 |
| | M-D10 svc_c 거래량 월 | Bar | Last 30d | 서비스 성장 |
| **사용자** | M-U11 MAU + cohort | Line | Last 30d | 임원 KPI 1번 |
| | M-U4 OS 분포 변화 | Bar (month over month) | Last 60d | 모바일 전략 |
| | M-U7 앱 버전 분포 (월말) | Donut | Last 30d | 강제 업데이트 |
| | M-U6 기기 painPoint Top 20 | Bar | Last 30d | QA 매트릭스 |
| **Capacity** | M-O6 월 peak RPS + 시간 | Metric + Line | Last 30d | 증설 의사결정 |
| | M-X2 인덱스 증가율 | Line | Last 30d | 클러스터 확장 시기 |

### Lens 설정 상세 (D4)

> D4 는 대시보드 픽커 `Last 30 days` 를 기본으로 따른다. 일부 cohort/추세 차트는 Lens 자체에서 더 긴 범위 (60d, 12m) 를 강제.

#### M-S1 SLA Avail (30d Metric)

- **차트**: Metric | **시간**: 대시보드 픽커 (30d)
- **Formula**: `(count(kql='sts_c:"OK"') / count()) * 100`
- **Format**: Percent, 3 dec. (SLA 공식 보고)
- **Color**: `< 99.9` 빨강 (SLA 미달)
- **Title**: `M-S1 SLA Avail (30d)`

#### M-S10 Error Budget Remaining (Gauge)

- **차트**: Gauge | **시간**: 대시보드 픽커 (30d)
- **Metric → Formula**:
  ```
  1 - ((count(kql='sts_c:"ERROR"') / count()) / 0.001)
  ```
  (= 1.0 → 100% 잔여, 0 → 소진)
- **Gauge max**: `1.0` | **Min**: `0`
- **Threshold**: `< 0.2` 빨강 / `< 0.5` 노랑
- **Format**: Percent
- **Title**: `M-S10 Error Budget Remaining (30d)`

#### M-S4 Monthly p95 (30d Line)

- **차트**: Line | **시간**: 대시보드 픽커
- **Horizontal axis**: `@timestamp`, Date histogram, Interval = Day
- **Vertical axis**: Percentile of `proc_tm`, 95
- **Breakdown (선택)**: `tier_c` (PT/BT/MCI 별 추세)
- **Title**: `M-S4 Monthly p95 (30d)`

#### M-D5 Payment Funnel (30d Bar)

- D3 와 동일 구조, **시간만 대시보드 픽커 (30d)**
- **Title**: `M-D5 Payment Funnel (30d)`

#### M-D7 Card Transactions Trend (Line)

- **차트**: Line | **시간**: 대시보드 픽커
- **Filter (KQL)**: `svc_c : "CD" and log_div : *_OUT and sts_c : "OK"` (환경 조정 — [13 §M-D7](13-recipe-domain.md))
- **Horizontal axis**: `@timestamp`, Date histogram, Interval = Day
- **Vertical axis**: Unique count of `guid` (= 거래 건수)
- **Title**: `M-D7 Card Transactions (30d)`

#### M-D10 svc_c Transaction Volume (Bar)

- **차트**: Vertical Bar | **시간**: 대시보드 픽커
- **Filter (KQL)**: `log_div : *_OUT`
- **Horizontal axis**: `svc_c`, terms, Top 11
- **Vertical axis**: `Count of records`
- **Title**: `M-D10 svc_c Volume (30d)`

#### M-U11 MAU (Line / Metric)

- **차트**: Line (월별 추세) 또는 Metric (단일 숫자)
- **시간**: **Last 12 months** 권장 (Lens 강제)
- **Horizontal axis**: `@timestamp`, Date histogram, Interval = Month
- **Vertical axis**: Unique count of `dgtl_cusno`
- **Cohort (옵션)**: 사용자 첫등장월 — Transform 필요. 단순 버전은 cohort 생략.
- **Title**: `M-U11 MAU Trend (12mo)`

#### M-U4 OS Distribution (Bar, MoM)

- **차트**: Vertical Bar (Percentage stacked) | **시간**: **Last 60 days** (Lens 강제)
- **Horizontal axis**: `@timestamp`, Date histogram, Interval = Month
- **Vertical axis**: `Count of records`
- **Breakdown**: `os_kdc`, terms (I, A, W)
- **Display**: 100% stacked
- **Title**: `M-U4 OS Distribution MoM (60d)`

#### M-U7 App Version (Month-end Donut)

- **차트**: Donut | **시간**: 대시보드 픽커
- **Filter (KQL)**: `chan_c : "MA"`
- **Slice by**: `ap_ver`, terms, Top 10
- **Size by**: Unique count of `guid`
- **Title**: `M-U7 App Version (30d)`

#### M-U6 Device painPoint Top 20 (Bar horizontal)

- D2 와 동일 구조, Top **20**, 시간 30d
- **Title**: `M-U6 Device painPoint Top 20 (30d)`

#### M-O6 Peak RPS (Line)

- **차트**: Line | **시간**: 대시보드 픽커
- **Horizontal axis**: `@timestamp`, Date histogram, **Interval = 1 minute**
- **Vertical axis (Formula)**: `count() / 60` (분당 호출 → RPS)
- **Reference line (옵션)**: 기존 capacity 한계선 표시
- **Tip**: peak 시각은 호버로 확인. 단일 숫자 필요시 Metric chart 별도 + Max of formula는 Lens 비지원 → TSVB 사용.
- **Title**: `M-O6 Peak RPS Trend (30d)`

#### M-X2 Document Volume (Line, capacity)

- **차트**: Line | **시간**: 대시보드 픽커
- **Horizontal axis**: `@timestamp`, Date histogram, Interval = Day
- **Vertical axis**: `Count of records` (= 일별 도큐먼트 수, 데이터량 proxy)
- **Trendline**: dimension → "Trend line" → Linear (회귀선으로 증가율 시각화)
- **Title**: `M-X2 Document Volume (30d)`

### 권장 레이아웃

```
┌─────────────────────────────────────────────────────────────────────┐
│ 월간 SLA + MAU + Budget                                                │
│ ┌──────────┬──────────┬──────────┬──────────┬──────────────┐         │
│ │SLA Avail │ MAU      │Budget    │peak RPS  │인덱스 증가율  │         │
│ │99.92% ✅ │ 421K     │잔여 38%  │ 18K/min  │   +12% / mo  │         │
│ └──────────┴──────────┴──────────┴──────────┴──────────────┘         │
├─────────────────────────────────────────────────────────────────────┤
│ 📈 M-U11 MAU 월별 추세 (Line, 12개월 cohort)                          │
├─────────────────────────────────────────────────────────────────────┤
│ 📊 M-D5 결제 Funnel 월별 (단계별 reach % 추세)                        │
├──────────────────────────────────────┬──────────────────────────────┤
│ 📊 M-D7 카드 거래 (Line)              │ 📊 M-D10 svc_c 별 월 거래량    │
├──────────────────────────────────────┴──────────────────────────────┤
│ 📊 M-S4 월간 p95 추세 (sub-bucket: PT/BT/MCI)                        │
├──────────────────────────────────────┬──────────────────────────────┤
│ 🍩 M-U4 OS 분포 (월 vs 전월)          │ 🍩 M-U7 ap_ver 분포 (월말)     │
├──────────────────────────────────────┴──────────────────────────────┤
│ 📊 M-U6 기기 painPoint Top 20 (월별)                                  │
├──────────────────────────────────────┬──────────────────────────────┤
│ 📈 M-O6 peak RPS / 시간대 (Line)      │ 📈 M-X2 인덱스 증가율          │
└──────────────────────────────────────┴──────────────────────────────┘
```

### 운영 절차 — 매월 1일 임원 보고

```
1. SLA 99.9% 달성 여부 → 미달 시 보상 / 사과
2. MAU 전월 대비 % → 마케팅 ROI
3. Error Budget 잔여 → 다음 달 신 기능 출시 가능 여부
4. 결제 Funnel drop rate 변화 → UX 개선 ROI
5. peak RPS 증가율 + 인덱스 증가율 → capacity 예측
   (예: +12%/월 = 6개월 후 클러스터 확장 필요)
6. ap_ver 분포 → 구 버전 5%+ 면 강제 업데이트 검토
7. M-U6 painPoint Top 20 → QA 테스트 매트릭스 갱신
```

### Save 정보

- **Title**: `D4 월간 점검 (Monthly Business Review)`
- **Refresh**: 1 week
- **시간 범위**: Last 30 days
- **권한**: 임원 / PM / 인프라 lead

---

## 메트릭 사용 매트릭스 — 어느 대시보드에 들어가나?

> 동일 메트릭이 여러 대시보드에 등장. 단, **시간 범위와 표현이 다름**.

| # | 메트릭 | D1 실시간 | D2 일간 | D3 주간 | D4 월간 |
|:--:|---|:--:|:--:|:--:|:--:|
| 1 | M-S1 Availability | ✅ Last 1h | ✅ Yesterday | ✅ 7d avg | ✅ 30d SLA |
| 2 | M-S2 TPS | ✅ Last 1h + 24h | ✅ Yesterday peak | ✅ 7d pattern | — |
| 3 | M-S3 Error Rate | ✅ Last 1h | ✅ Yesterday | — | — |
| 4 | M-S4 Latency p50/p95/p99 | ✅ Last 24h | ✅ Yesterday p95 | ✅ WoW timeshift | ✅ 월 추세 |
| 5 | M-S5 Slow Rate | △ | — | — | — |
| 6 | M-S6 Saturation | △ | — | — | — |
| 7 | M-S7 Tier Avail | △ | — | — | — |
| 8 | M-S8 G/W p95 | ✅ | — | — | — |
| 9 | M-S9 MCI core p95 | ✅ | — | — | — |
| 10 | M-S10 Error Budget | △ | — | ✅ rolling | ✅ 잔여 |
| 11 | M-P1 11 MSA Health | ✅ | — | — | — |
| 12 | M-P2 Tier Health | △ | — | — | — |
| 13 | M-P3 In/Out Imbalance | △ | ✅ Top 10 | — | — |
| 14 | M-P4 Stuck Requests | ✅ | △ | — | — |
| 15 | M-P5 Inter-Tier Latency | — | ✅ | — | — |
| 16 | M-P6 Top svc by Traffic | — | △ | — | △ |
| 17 | M-P7 Top svc by Error | — | ✅ | — | — |
| 18 | M-P8 Container Error | ✅ | △ | — | — |
| 19 | M-P9 Container Throughput | — | △ | — | — |
| 20 | M-P10 Pipeline Lag | ✅ | △ | — | — |
| 21 | M-P11 Inter-Service | — | △ | — | — |
| 22 | M-P12 호출 hop | — | — | — | △ |
| 23 | M-D1 Top msg_c | — | ✅ | — | — |
| 24 | M-D2 신규 msg_c | — | ✅ | ✅ | — |
| 25 | M-D3 Critical svc Error | ✅ | △ | — | — |
| 26 | M-D4 MSA × msg_c | — | ✅ | — | — |
| 27 | M-D5 결제 Funnel | — | — | ✅ | ✅ |
| 28 | M-D6 인증 성공률 | — | — | ✅ | △ |
| 29 | M-D7 카드 거래 | — | — | — | ✅ |
| 30 | M-D8 이체 성공률 | — | — | ✅ | △ |
| 31 | M-D9 fir_c 분포 | △ | ✅ | — | — |
| 32 | M-D10 svc_c 거래량 | — | — | △ | ✅ |
| 33 | M-D11 동시 활성 거래 | — | — | ✅ peak | — |
| 34 | M-D12 biz_c 분포 | — | — | △ | △ |
| 35 | M-U1 채널 Traffic | — | — | ✅ | △ |
| 36 | M-U2 채널 Error | — | — | △ | — |
| 37 | M-U3 채널 Latency | — | — | — | △ |
| 38 | M-U4 OS 분포 | — | — | — | ✅ |
| 39 | M-U5 OS 버전 Error | — | — | — | △ |
| 40 | M-U6 기기 painPoint | — | ✅ | — | ✅ |
| 41 | M-U7 ap_ver 분포 | — | — | ✅ | ✅ |
| 42 | M-U8 신앱 회귀 | — | — | ✅ | — |
| 43 | M-U9 화면 funnel | — | — | △ | — |
| 44 | M-U10 retry 패턴 | — | ✅ | — | — |
| 45 | M-U11 DAU/WAU/MAU | — | — | ✅ WAU | ✅ MAU |
| 46 | M-U12 IP 대역 | — | — | — | △ |
| 47 | M-O1 Time-of-Day | — | — | — | △ |
| 48 | M-O2 Day-of-Week | — | — | △ | △ |
| 49 | M-O3 Dead svc_id | — | △ | △ | — |
| 50 | M-O4 Shadow svc_id | — | — | — | △ |
| 51 | M-O5 DoD/WoW | — | — | ✅ | △ |
| 52 | M-O6 Peak RPS | — | — | — | ✅ |
| 53 | M-X1 Pipeline stage lag | — | ✅ | — | — |
| 54 | M-X2 Index storage | — | — | — | ✅ |
| 55 | M-X3 doc_sz | — | — | — | △ |
| 56 | M-X4 Schema stability | — | — | — | △ |

> ✅ = 핵심 패널, △ = 보조 / 선택, — = 해당 없음.

---

## 대시보드 우선 도입 순서 (4주 로드맵)

### Week 1: D1 실시간 만들기 (8시간)

```
Day 1 (2h): M-S1, S2, S3, S4 (SLO 4종) — 11번 §M-S1~S4
Day 2 (2h): M-S8, S9 + M-P10, P4 — 11번 §M-S8/9 + 12번 §M-P10/P4
Day 3 (2h): M-P1 11 MSA Matrix + M-P8 Container — 12번 §M-P1/P8
Day 4 (2h): D1 dashboard 조립 + 30s refresh + alert R-P0-1~5
```

### Week 2: D2 일간 점검 (6시간)

```
Day 1 (2h): M-D1, D2, D9, D4 (msg_c 분석) — 13번 §M-D1, D2, D9, D4
Day 2 (2h): M-P3, P5, P7 + M-X1 — 12번 + 15번
Day 3 (2h): M-U6, U10 — 14번
Day 4 (1h): D2 dashboard 조립 + 1h refresh + 절대 시각 (Yesterday)
```

### Week 3: D3 주간 점검 (6시간)

```
Day 1 (2h): M-S10, M-O5 timeshift — 11번/15번
Day 2 (2h): M-D5/D6/D8 비즈니스 KPI Funnel — 13번
Day 3 (2h): M-U1, U7, U8, U11 사용자 메트릭 — 14번
Day 4 (1h): D3 dashboard 조립
```

### Week 4: D4 월간 점검 (4시간)

```
Day 1 (2h): M-U11 MAU cohort + M-D7/D10 비즈니스 추세
Day 2 (2h): M-X2 capacity 예측 + M-O6 peak + M-U4/U6 사용자 분포
Day 3 (1h): D4 dashboard 조립 + 임원 보고용 PDF 자동화 (Reporting, Gold+)
```

---

## 대시보드 조립 단계별 가이드

> 위 §D1~D4 에서 만든 Lens 차트들을 어떻게 묶어 대시보드로 완성하는지.

### Step 1 — Data view 확인

```
Stack Management → Data Views → "msa-logs-*" 존재 확인
없으면 [Create data view] → Name: msa-logs-* / Time field: @timestamp
```

### Step 2 — Lens 차트 만들기 (메트릭별 반복)

각 메트릭에 대해:

1. **Analytics → Visualize Library → Create visualization → Lens** 클릭
2. 좌상단 **Data view** 가 `msa-logs-*` 인지 확인
3. **시간 픽커** 설정 (우상단) — 위 §X.X 의 "시간" 항목대로
4. **차트 타입** 선택 (우상단 드롭다운) — Metric / Line / Bar / Table / Heatmap / Pie / Gauge
5. **KQL Filter** 입력 (상단 검색 bar) — 위 §X.X "Filter (KQL)"
6. **차원 박스** (우측 패널) 에 필드 드래그
   - **Formula** 가 필요한 경우: dimension 박스 → "Formula" 탭 → 식 입력
   - **Time shift**: dimension 박스 → "Add advanced options" → "Time shift" → `1w` / `1d`
7. **Format** 설정: dimension 박스 → "Value format" → Percent / Number + 단위
8. **Color/Threshold**: 좌상단 visualization 설정 → Color → "Add color stops" 또는 "Static palette"
9. 우상단 **Save** → **Add to library** 토글 ON → Title 입력 (위 §X.X 의 `Title` 그대로)

> 💡 동일 메트릭이 여러 대시보드에 등장하면 (예: M-S1) **시간 범위만 다른 별도 Lens 로 각각 저장** 권장. 예: `M-S1 Availability (1h)`, `M-S1 Avail (Yesterday)`, `M-S1 Avg Avail (7d)`, `M-S1 SLA Avail (30d)`.

### Step 3 — Dashboard 생성

각 D1/D2/D3/D4 에 대해:

1. **Analytics → Dashboards → Create dashboard**
2. 편집 모드 (자동 진입) 에서 → **Add from library** 클릭
3. §X.X 의 Lens 차트들을 선택해 추가
4. 각 패널 드래그/리사이즈 — §X.X 의 "권장 레이아웃" 참고
5. (선택) **Add panel → Text** → Markdown 으로 운영 절차/안내문구 추가

### Step 4 — 시간 범위 + Refresh 설정

| Dashboard | 시간 픽커 | Refresh |
|---|---|---|
| D1 | `Last 24 hours` (Live, 우상단 시간 픽커 → "Quick selects" → Last 24h) | **30 seconds** |
| D2 | `Yesterday` (Calendar → Yesterday — Absolute) | **1 hour** |
| D3 | `Last 7 days` | **1 day** |
| D4 | `Last 30 days` | **1 week** |

Refresh 설정 위치: 시간 픽커 옆 🔄 아이콘 → "Set refresh interval".

> ⚠️ D2 의 "Yesterday" 는 **상대 (Last 24h) 가 아니라 절대** 를 선택해야 자정 기준 정렬. 시간 픽커 → Quick select → **Yesterday**. ("Last 24 hours" 와 다름.)

### Step 5 — Dashboard 저장

1. 우상단 **Save** 클릭
2. **Title** (§X.X Save 정보 참고):
   - D1: `D1 실시간 (Real-time SRE)`
   - D2: `D2 일간 점검 (Daily Ops Review)`
   - D3: `D3 주간 점검 (Weekly Review)`
   - D4: `D4 월간 점검 (Monthly Business Review)`
3. **"Store time with dashboard"** 체크 → 위 시간/refresh 함께 저장
4. **Tags** (선택): `SRE`, `Daily`, `Weekly`, `Monthly` 등

### Step 6 — 패널별 시간 override (D1 의 SLO Metric 등)

D1 의 "Last 1 hour 강제" 차트들은:

1. 해당 패널 → 우상단 ⚙️ → **Customize time range** 토글 ON
2. 시간: Last 1 hour 직접 지정

이러면 대시보드 픽커가 `Last 24h` 여도 그 패널만 1h 로 계산.

### Step 7 — 권한/공유

1. 대시보드 우상단 **Share** → URL 복사 → 친구에게 전달
2. 권한 분리는 [Spaces](https://www.elastic.co/guide/en/kibana/current/xpack-spaces.html) 사용 — 예: `sre-space` (D1) / `ops-space` (D2~D4)
3. Read-only 뷰는 [Reporting](https://www.elastic.co/guide/en/kibana/current/reporting-getting-started.html) (Platinum) 으로 PDF 자동 송부 가능

---

## 자주 막히는 함정

| 함정 | 해결 |
|---|---|
| D2 의 "Yesterday" 가 *어제* 가 아닌 *24시간 전* 으로 인식 | 시간 픽커 → "Yesterday" 절대 옵션 사용 |
| 동일 차트를 4 dashboard 에 복제 | Save to Library + 각 dashboard 에서 *다른 시간 범위* 로 add |
| D3/D4 의 timeshift 가 안 보임 | Lens 의 dimension 박스 → "Add advanced options" → Time shift |
| D4 의 가독성이 떨어짐 | Markdown 패널로 *한 줄 요약 narrative* 추가 — 임원이 읽기 좋음 |
| Formula 에 `count(kql=...)` 식 입력 후 Apply 안 됨 | KQL 문자열은 작은따옴표(`'...'`) 로 감싸야 함. 큰따옴표는 식 자체 종료로 오인. |
| Percentile 입력 시 small bucket warning | 시간 범위가 좁아 표본 부족. 시간 늘리거나 percentile 을 90/85 로 완화. |
| Lens Metric 의 "Last 1h 강제" 가 적용 안 됨 | 패널 우상단 ⚙️ → "Customize time range" 토글 ON 필요 (Lens 편집창의 시간 픽커가 아니라 대시보드 패널 레벨 설정). |
| Pie chart 의 "Other" bucket 이 너무 큼 | dimension → terms → "Group remaining as Other" 토글 OFF 또는 Top N 늘림. |
| Heatmap cell 이 거의 비어 보임 | Color palette → "Stop type" → "Percent" 로 변경 (절대값 → 분위수). |
| Time shift `1w` 가 7d 범위에서 데이터 없다고 뜸 | Lens 의 시간 범위를 **14d** 로 늘려야 timeshift 대상 영역이 포함됨. |

---

## 함께 보기

- [09. 종합 KPI 전략](09-monitoring-strategy.md)
- [09c. 메트릭 우선순위](09c-metric-priority.md)
- [10. 데이터셋 가이드](10-practice-dataset.md)
- [11~15. 메트릭 레시피](11-recipe-sre.md)
