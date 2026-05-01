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

## 자주 막히는 함정

| 함정 | 해결 |
|---|---|
| D2 의 "Yesterday" 가 *어제* 가 아닌 *24시간 전* 으로 인식 | 시간 픽커 → "Yesterday" 절대 옵션 사용 |
| 동일 차트를 4 dashboard 에 복제 | Save to Library + 각 dashboard 에서 *다른 시간 범위* 로 add |
| D3/D4 의 timeshift 가 안 보임 | Lens 의 dimension 박스 → "Add advanced options" → Time shift |
| D4 의 가독성이 떨어짐 | Markdown 패널로 *한 줄 요약 narrative* 추가 — 임원이 읽기 좋음 |

---

## 함께 보기

- [09. 종합 KPI 전략](09-monitoring-strategy.md)
- [09c. 메트릭 우선순위](09c-metric-priority.md)
- [10. 데이터셋 가이드](10-practice-dataset.md)
- [11~15. 메트릭 레시피](11-recipe-sre.md)
