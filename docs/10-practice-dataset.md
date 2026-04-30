# 10. 실습 데이터셋 가이드 — 30M docs / 30일 / 11 MSA

> **목적**: [09 의 52 메트릭](09-monitoring-strategy.md) · [09a 실 필드 매핑](09a-real-field-mapping.md) 을 *직접 Lens 로 만들어보면서* 어떤 지표가 우리 팀에 진짜 필요한지 가려내기 위한 **의도된 학습 데이터셋**.
>
> 30일 × 1M docs/일 = **30M docs** 적재. 모든 메트릭이 가시화되도록 의도된 시그널을 특정 날짜에 주입.

---

## 1. 데이터셋 개요

| 항목 | 값 |
|---|---|
| 인덱스 패턴 | `msa-logs-YYYY.MM.DD` |
| 기간 | 2026-04-01 ~ 2026-04-30 (30일) |
| 총 docs | ~30,000,000 |
| 총 transaction | ~5M (평균 6 docs/tx) |
| 총 unique 사용자 | ~500K (`dgtl_cusno`) |
| ES 노드 | 단일 (4GB heap) |
| 디스크 사용량 | ~30GB |

### 1.1 11 MSA 매핑 (정정 반영: AS=BT)

| svc_c | tier | 한국어 | 트래픽 가중치 |
|:-:|:-:|---|:-:|
| **PP** | PT | App PT | 70% |
| **WP** | PT | Web PT | 30% |
| **PY** | BT | 간편결제 | 25% |
| **CD** | BT | 카드 | 15% |
| **FA** | BT | 금융 | 12% |
| **MB** | BT | 회원 | 12% |
| **PU** | BT | 이용내역 | 10% |
| **BU** | BT | 혜택 | 8% |
| **PC** | BT | 공통 | 8% |
| **AS** | BT | 결제 코어 | 8% |
| **PA** | BT | 관리자 | 2% |

> MCI 계층은 *외부 계정계 / 대외 G/W* — 사내 MSA 가 아닌 외부 의존성. 호출 발생 시 (`MCI_SEND/RECV`, `GW_SEND/RECV`) docs 생성.

---

## 2. 시간 패턴

### 2.1 시간대 (HOUR)

```
0  1  2  3  4  5   6  7  8  9 10 11   12 13 14 15 16 17   18 19 20 21 22 23
─  ─  ─  ─  ─  ─   ▁  ▂  ▄  █ █ █    ▄  █ █ █ ▆ ▅    ▃  ▃  ▂  ▁  ─  ─
                  9~12시 peak       13~17시 peak (점심 후)
```

### 2.2 요일 (DOW)

| 일 | 월 | 화 | 수 | 목 | 금 | 토 |
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| 50% | 100% | 100% | 100% | 100% | 100% | 70% |

### 2.3 채널 (chan_c)

| MA | MW | PW | DA |
|:-:|:-:|:-:|:-:|
| 55% | 25% | 15% | 5% |

### 2.4 OS (os_kdc) + ap_ver

```
OS:   A 50%    I 45%    W 5%
ap_ver baseline: 4.4.0(40%) / 4.4.5(35%) / 4.5.0(25%)
```

---

## 3. 의도된 시그널 — *어느 날 무엇이 일어나나?*

| 일자 | 시그널 | 어느 메트릭 학습? |
|---|---|---|
| **Day 1~14** | 평소 (baseline) | 기본 분포 학습 |
| **Day 14** | 신규 msg_c `E099` 등장 | **M-D2 신규 msg_c**, R-P1-1 alert |
| **Day 15** | 트래픽 ×1.4 spike (이벤트성) | **M-O5 DoD/WoW**, M-S2 TPS anomaly |
| **Day 18** | 신 앱 `ap_ver=4.5.1` 출시 + 회귀 | **M-U7 분포**, **M-U8 신앱 회귀** (R-P1-8) |
| **Day 22** | `py-pod-2` 컨테이너 에러 spike (×6) | **M-P8 Container error**, R-P1-12 |
| **Day 1~30 상시** | `SM-G960N` 모델 에러율 ×3 | **M-U6 기기 painPoint** |
| **Day 1~30 상시** | 결제 ERROR 시 0.5% 사용자 retry 2회 | **M-U10 retry pattern** (R-P1-11) |
| **Day 1~30 상시** | 0.1~0.2% transaction 에서 *_OUT 누락 | **M-P3 In/Out Imbalance**, **M-P4 Stuck** |
| **Day 1~30 상시** | 1% slow tail (proc_tm > 1500ms) | **M-S5 Slow Request Rate** |
| **Day 1~30 상시** | G/W p95 ~800ms vs MCI p95 ~250ms | **M-S8 / M-S9 의존성 비교** |

> 모든 시그널은 [generate-msa-logs.ts](https://github.com/corollazer0/elk-learning/blob/main/) 에 코드로 명시됨.

---

## 4. 메트릭별 학습 KQL — 바로 복사해서 Lens

### 4.1 SRE Golden Signals

```kql
# M-S1 Availability
log_div : (PT_OUT or BT_OUT or MCI_RECV or GW_RECV) and sts_c : "OK"

# M-S3 Error Rate
log_div : (PT_OUT or BT_OUT or MCI_RECV or GW_RECV) and sts_c : "ERROR"

# M-S4 Latency p95/p99 (Lens: percentile of proc_tm, filter)
log_div : *_OUT

# M-S5 Slow Request
log_div : *_OUT and proc_tm > 1000

# M-S8 G/W p95
log_div : (GW_SEND or GW_RECV)

# M-S9 MCI core p95
log_div : (MCI_SEND or MCI_RECV)
```

### 4.2 백엔드 / 플랫폼

```kql
# M-P1 11 MSA Health Matrix → svc_c × KPI
log_div : *_OUT
# 그룹: svc_c
# 메트릭: count, error count, p95(proc_tm), error rate

# M-P3 In/Out Imbalance — sum(*_IN) vs sum(*_OUT) by svc_id

# M-P4 Stuck Requests — bucket by guid, filter (out_count = 0 AND first_in < now-10m)

# M-P8 Container Error Rate by ctnr_nm
log_div : *_OUT
# 그룹: ctnr_nm  (day 22 에서 py-pod-2 spike 발견 가능)
```

### 4.3 도메인 / 비즈니스

```kql
# M-D1 Top msg_c
sts_c : "ERROR"
# Top 20 by msg_c (E003 dominant)

# M-D2 신규 msg_c
sts_c : "ERROR" and msg_c : "E099"   # day 14+ 만 등장

# M-D3 Critical svc 결제
svc_c : ("PY" or "AS" or "FA")
# Lens: error rate timeline → day 22 spike

# M-D5 결제 Funnel
svc_c : "PY" and log_div : *_OUT
# 그룹: svc_id (pay/init → pay/auth → pay/confirm 순서)

# M-D9 fir_c 분포 (root cause)
fir_err : "Y"
# 그룹: fir_c (대부분 MCI 60%)
```

### 4.4 사용자 / UX

```kql
# M-U1 채널별 Traffic
log_div : *_OUT
# 그룹: chan_c

# M-U6 기기 모델 painPoint
sts_c : "ERROR"
# 그룹: device_model, Top 20 → SM-G960N 가 dominant 발견 가능

# M-U7 ap_ver 분포
# 그룹: ap_ver → day 18+ 4.5.1 등장

# M-U8 신 앱 회귀 (timeshift 필요)
ap_ver : "4.5.1"
# 비교: ap_ver != "4.5.1"

# M-U10 retry pattern
sts_c : "ERROR"
# 그룹: dgtl_cusno + scrn_uuid → 같은 조합이 ≥3 인 row
# (Lens 의 split rows + filter aggregation)

# M-U11 DAU
log_div : *_OUT
# Cardinality(dgtl_cusno) by day
```

### 4.5 운영 / Capacity

```kql
# M-O1 Time-of-Day Heatmap
*
# X: hour, Y: day-of-week, value: count

# M-O5 DoD / WoW (timeshift)
*
# Lens: count, with timeshift "1d" → day 15 anomaly 확인
```

---

## 5. 실습 권장 순서

### Phase 1. 데이터 친숙화 (1시간)

```
1. Discover → msa-logs-* 선택, time range "Last 30 days"
2. 한 doc 펼쳐서 모든 필드 한번씩 읽기
3. 다음 KQL 시도:
   svc_c : "PY" and sts_c : "ERROR"
   guid : "<특정 거래>" → 6 docs 한번에 보기
4. 한 거래의 PT_IN → PT_OUT → BT_IN → BT_OUT → MCI_SEND/RECV 시계열 확인
```

### Phase 2. 메트릭 단건 만들기 (각 5~10분)

```
Tier S 7개 (09c §3.1) 부터 시작:
  M-S1 → M-S3 → M-S4 → M-P10 → M-S2 → M-P1 → M-D3
```

### Phase 3. Dashboard 조립 (각 30분)

```
D-RT1 SRE Golden Signals
D-RT2 11 MSA Health Matrix
D-D1 일일 점검 (어제 KPI)
```

### Phase 4. 의도된 시그널 발견 (1~2시간)

```
1. day 15 트래픽 anomaly 를 M-O5 timeshift 로 발견
2. day 22 컨테이너 spike 를 M-P8 로 발견
3. day 18 신 앱 회귀를 M-U8 로 발견
4. day 14 신규 msg_c E099 를 M-D2 로 발견
5. SM-G960N 기기 painPoint 를 M-U6 로 상시 발견
6. retry pattern 을 M-U10 으로 상시 발견
```

### Phase 5. 도구화 (2~4시간)

```
1. Transform 5개 등록 (09 §3.7)
2. Alert 룰 5개 (P0) 등록
3. 자기 팀에 필요한 Tier 만 결정 (불필요한 메트릭은 archive)
```

---

## 6. 기대 분포 — *대시보드가 이렇게 보여야 정상*

| 메트릭 | 평소 값 | 의도된 변동 |
|---|---|---|
| Availability (전체) | ~99.5% | day 22 PY 만 96~98% |
| Error Rate (전체) | ~0.5% | day 22 PY 만 ~4% |
| p95 latency PT | 150ms | 일정 |
| p95 latency BT | 280ms | 일정 |
| p95 latency MCI | 250ms | 일정 |
| p95 latency GW | 1100ms | 큰 변동 |
| TPS 평균 | 12K/min | day 15 ×1.4 spike |
| chan_c MA 비중 | 55% | 일정 |
| ap_ver 4.5.1 비중 | 0% (day 1~17) → 15% (day 18+) | 회귀 감지 가능 |
| 기기 SM-G960N 에러율 | 1.5% (전체 0.5% 의 3배) | 상시 |

---

## 7. 데이터셋 제거 / 재생성

```bash
# 모든 인덱스 삭제
curl -u elastic:$EP -XGET "http://localhost:9200/_cat/indices/msa-logs-*?h=index" \
  | xargs -I {} curl -u elastic:$EP -XDELETE "http://localhost:9200/{}"

# 재생성 (40~60분 소요)
ES_PASSWORD=$EP tsx scripts/generate-msa-logs.ts \
  --days 30 --start 2026-04-01 --per-day 1000000 --bulk 5000
```

### 시드 변경으로 다른 시그널

```bash
# --seed 99 로 다른 사용자 ID, 다른 noise 패턴
ES_PASSWORD=$EP tsx scripts/generate-msa-logs.ts \
  --days 30 --start 2026-04-01 --per-day 1000000 --seed 99
```

---

## 8. 함께 보기

- [09. 종합 KPI 전략](09-monitoring-strategy.md) — 52 메트릭 정의
- [09a. 실 필드 매핑](09a-real-field-mapping.md) — 필드 cheatsheet
- [09c. 메트릭 우선순위](09c-metric-priority.md) — Tier S/A/B/C
- [99-tier-tracing.md](99-tier-tracing.md) — 분산 추적 시나리오
- generator: `elastic/scripts/generate-msa-logs.ts`
- index template: `elastic/seed/msa-logs/index-template.json`
