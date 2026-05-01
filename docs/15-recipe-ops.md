# 15. 운영 / Op-Excellence 레시피 (M-O1~O6 + M-X1~X4)

> **선수**: [14. 사용자 / UX 레시피](14-recipe-user-ux.md)
>
> *시간대 패턴* + *인덱스 헬스* — capacity 계획과 메타 모니터링. Op-Excellence (X 시리즈) 는 ES 운영팀 전용.

> ⏱️ **시간 범위 안내**: 메트릭 제목 옆은 **권장 default**. 운영 메트릭은 보통 *주~월 단위* 가 의미 있음. 실습 데이터셋은 2026-04-01~30 절대 시각.

---

## 학습 순서 추천

```
1. M-O5 DoD/WoW Traffic Change   ← Day 15 anomaly 학습
2. M-O1 Time-of-Day Pattern
3. M-O2 Day-of-Week Pattern
4. M-O3 Dead svc_id
5. M-O6 Peak RPS
6. M-O4 Shadow svc_id (외부 spring mapping 필요)
7. M-X1 Pipeline Stage Lag
8. M-X2 Index Storage Health
9. M-X3 doc_sz 분포
10. M-X4 Schema Stability
```

---

## M-O1. Time-of-Day Pattern (시간대 분포) — ⏱️ Last 30 days

**한 줄 정의**: 24시간 × 트래픽 heatmap

**왜 보는가?**
- peak / off-peak 시간 정량 파악
- capacity 계획의 baseline

**Kibana 만들기**:

```
1. Lens → Heatmap
2. Horizontal axis: @timestamp (Date histogram, 1h)
3. Vertical axis: @timestamp (Date histogram, 1d) — *같은 필드 다른 interval*
4. Cell value: Count
5. KQL: log_div : (PT_OUT or BT_OUT)
6. Title: "M-O1 Time-of-Day Heatmap"
7. Save
```

**더 직관적**: hour bucket only

```
1. Lens → Bar (vertical)
2. Horizontal axis: hour_of_day (Runtime field — 아래 §부록)
3. Vertical axis: Count
4. Title: "M-O1 시간대별 트래픽"
```

**해석 가이드**:
- 9~12시 / 13~17시 peak — 정상
- 새벽 spike = 비정상 (배치? 사고?)

**우리 데이터셋에서**:
- 의도된 hour weight (peak 9~12, 13~17) 가 명확히 보임

**업무 개선 활용**:
- HPA 정책 (시간대별 minReplicas)
- 배포 시간 결정 (off-peak)

---

## M-O2. Day-of-Week Pattern — ⏱️ Last 30 days

**Kibana 만들기**:

```
1. Lens → Bar (vertical)
2. Horizontal axis: day_of_week (Runtime field)
3. Vertical axis: Count
4. Title: "M-O2 요일별 트래픽"
5. Save
```

**우리 데이터셋**: 평일 100% / 토 70% / 일 50%

---

## M-O3. Dead svc_id — ⏱️ Last 7 days

**한 줄 정의**: Spring Mapping 선언 ∧ ES 호출 0인 svc_id (24h+)

**왜 보는가?**
- 코드는 있는데 호출 0 = 죽은 endpoint (정리 대상)

**Kibana 만들기** (cross-DB 외부 자료 필요):

```
1. 외부에서 Spring @RequestMapping list 추출 → CSV
2. Kibana: Lens → Table
3. Vertical axis: svc_id (Top 100, sort by count asc)
4. Metric: Count, KQL: log_div : *_OUT
5. → count = 0 인 svc_id list 발견 가능
   (단 ES 에 한 번도 안 찍힌 건 안 보임 — Discover index template 와 비교)

대안: ES 에 record 가 1건도 없는 svc_id 는 query 가 직접 못 잡음 →
  Spring mapping list 와 ES 에서 본 svc_id list 의 set diff
```

**업무 개선**:
- 분기 코드 cleanup
- mapping 폭발 방지

---

## M-O4. Shadow svc_id — ⏱️ Last 7 days

**한 줄 정의**: ES 호출 ∧ Spring mapping 미선언 svc_id

**왜 보는가?**
- 정의되지 않은 endpoint 가 호출되고 있음 = 유실된 코드 / 스캐닝

**Kibana 만들기**:
- Spring mapping list 와 svc_id terms aggregation 비교
- Set diff → Shadow list

---

## M-O5. DoD / WoW Traffic Change ⭐ — ⏱️ Last 14 days

**한 줄 정의**: 어제 / 지난주 같은 요일 대비 % 변화

**왜 보는가?**
- 트래픽 anomaly 자동 감지
- 이벤트 vs 사고 구분

**Kibana 만들기** (timeshift):

```
1. Lens → Line
2. Horizontal axis: @timestamp (1h bucket)
3. Vertical axis (2개 라인):
   ① Count, Time shift: None, Display: "Today"
   ② Count, Time shift: 1d, Display: "Yesterday"
   ③ (선택) Count, Time shift: 1w, Display: "Last week same day"
4. Title: "M-O5 DoD/WoW Traffic"
5. Save
```

**더 정밀**: 비율로 표현

```
함수: Formula
수식:
  count() / count(shift='1d')
표시: Number (1.0 = 동일, > 1.4 = anomaly)
```

**해석 가이드**:
- ±10% = 자연 변동
- > 1.3x = anomaly (이벤트 또는 사고)
- < 0.7x = drop (사용자 못 들어옴)

**우리 데이터셋에서**:
- **Day 15 (4월 15일)** 에 1.4배 spike — 가장 명확한 학습 포인트
- 시간 범위 4월 13일 ~ 17일 으로 좁히면 드러남

**업무 개선 활용**:
- Alert R-P0 후보
- 💎 **Platinum ML Anomaly Detection 의 1순위 적용 대상** — 시간대×요일 자연 패턴 학습 → FP ↓

---

## M-O6. Peak RPS + 시간대 — ⏱️ Last 30 days

**한 줄 정의**: 일/주 단위 max(count) + 그 시각

**Kibana 만들기**:

```
TSVB (Time Series Visual Builder) 사용:
1. Visualize Library → Create → TSVB
2. Aggregation: Max of count() over time (1m bucket)
3. Time range: This week
4. Title: "M-O6 이번주 peak RPS"
```

**또는 Lens metric**:

```
함수: Max of count() — interval 1 minute
시간 범위: This week
Display: "이번주 peak RPS"
```

---

## M-X1. Pipeline Stage Lag — ⏱️ Last 24 hours

**한 줄 정의**: takes.* 필드 — 각 단계 처리 시간

**왜 보는가?**
- 어느 stage 가 병목? (앱 → Logstash → ES 인덱싱)

**Kibana 만들기**:

```
1. Lens → Line (또는 Area stacked)
2. Horizontal: @timestamp
3. Vertical (3 라인):
   ① Average of takes.ap_logstash_takes
   ② Average of takes.logstash_process_takes
   ③ Average of takes.es_index_takes
4. Title: "M-X1 Pipeline Stage Lag"
5. Save
```

**해석 가이드**:
- 3 stage 시간 합 = 전체 lag
- 1 stage 만 spike = 그 단계 점검

**우리 데이터셋**:
- 의도된 평균: ap_logstash 50ms, logstash_process 80ms, es_index 40ms

---

## M-X2. Index Storage Health — ⏱️ Last 30 days

**한 줄 정의**: 일자별 인덱스 size, shard 수, segment 수

**Kibana 만들기** (Stack Management 또는 _cat API):

```
방법 1: Kibana Stack Monitoring
  좌측 사이드바 → Management → Stack Monitoring → Indices

방법 2: Dev Tools 에서 직접
  GET _cat/indices/msa-logs-*?v&h=index,store.size,docs.count,pri,segments.count

방법 3: Lens (메타 인덱스)
  특수 — index size 는 doc 안에 없음. 외부 input plugin 또는 별도 _cat 풀링 필요
```

**해석 가이드**:
- 일자별 size 가 일정 = 정상
- segment count > 50 = force_merge 필요

**우리 데이터셋**:
- 일자별 약 1GB ~ 1.4GB (peak day 15 큼)

**업무 개선 활용**:
- ILM 정책 결정
- 💎 Platinum: Searchable Snapshots cold/frozen 전환 시기

---

## M-X3. doc_sz 분포 — ⏱️ Last 7 days

**Kibana 만들기**:

```
1. Lens → Histogram
2. Horizontal axis: doc_sz (numeric histogram)
3. Vertical axis: Count
4. Title: "M-X3 doc_sz 분포"
5. Save
```

**해석**:
- 평균 1.5KB / p95 2KB 정상
- 갑자기 큰 doc 등장 = data 필드 폭발

---

## M-X4. Schema Stability — ⏱️ Last 30 days

**한 줄 정의**: 신규 필드 자동 매핑 빈도

**Kibana 만들기**:

```
방법: 정기 _mapping API 스냅샷 → 비교

GET msa-logs-*/_mapping
  → fields list 추출
  → 어제 vs 오늘 set diff
```

**업무 개선**:
- mapping 폭발 방지 (10000+ 필드 = ES 메모리 폭발)
- dynamic:strict 정책 (우리는 이미 적용)

---

## D-O1: 인프라 헬스 Dashboard

```
┌──────────────────────────────────────────────────────────────┐
│ M-X1 Pipeline Stage Lag (Line, 3 stages)                     │
├──────────────────────────────────────────────────────────────┤
│ M-X2 일자별 인덱스 size (Bar)                                 │
├──────────────────────────────────────────────────────────────┤
│ M-X3 doc_sz 분포 (Histogram)                                  │
├──────────────────────────────────────────────────────────────┤
│ M-P10 Pipeline Lag (Metric, 단일)                            │
├──────────────────────────────────────────────────────────────┤
│ M-O1 Time-of-Day Heatmap                                     │
└──────────────────────────────────────────────────────────────┘

Save: "D-O1 인프라 헬스"
```

---

## 부록 A. Runtime Field — hour_of_day, day_of_week

```
Stack Management → Data Views → msa-logs-* → Add field

Name: hour_of_day
Type: long
Format: 0~23
Painless script:
  emit(doc['@timestamp'].value.toInstant()
    .atZone(ZoneId.of('Asia/Seoul'))
    .getHour())

Name: day_of_week
Type: keyword
Painless script:
  emit(doc['@timestamp'].value.toInstant()
    .atZone(ZoneId.of('Asia/Seoul'))
    .getDayOfWeek().toString())
```

→ 이후 Lens 에서 일반 필드처럼 사용 가능.

---

## 부록 B. 전체 메트릭 Dashboard 매핑

| Dashboard | 포함 메트릭 | 용도 |
|---|---|---|
| **D-RT1** SRE Golden Signals | M-S1, S2, S3, S4, S5, S6, S7, S8, S9, S10 | SRE 24/7 |
| **D-RT2** 11 MSA Health Matrix | M-P1, P2, P6, P7, P8, P9, P3, P4 | 인시던트 첫 화면 |
| **D-RT3** 3-Tier Health (drill) | M-P2, P5, M-S7 | drill from RT1/RT2 |
| **D-RT4** 거래 추적 | guid 단위 모든 docs | 단건 디버깅 |
| **D-D1** 일일 점검 | M-D1, D2, D9, M-O3, P5 | 매일 아침 |
| **D-D2** 도메인 에러 분석 | M-D1, D2, D4, D9 | 인시던트 retro |
| **D-D3** 사용자 / UX | M-U1~U12 | PM / 운영 |
| **D-K1** 주간 KPI | M-S1, S10, D5, D6, U11 | 매주 월요일 |
| **D-K2** 월간 비즈니스 KPI | M-D5/6/7/8, U11 | 임원 |
| **D-O1** 인프라 헬스 | M-X1, X2, X3, P10 | DBA/플랫폼 |

---

## 부록 C. 다음 단계 — 메트릭을 *고를* 시간

[09c §6 도입 로드맵](09c-metric-priority.md#6-도입-로드맵--tier-기준) 권장:

```
Week 1-2  : Tier S 7개만 — D-RT1 + R-P0 alert
Week 3-4  : Tier A 13개 추가 — D-RT2/3 + D-D1
Week 5-8  : Tier B 18개 — D-D2/D3 + D-K1
Quarter+  : Tier C 14개 + Platinum ML 검토
```

> 만든 후 *3주 동안 안 본 차트* 는 archive. 메트릭은 *유지비용* 이 있다 — 알림 FP, 차트 응답 시간, 운영 인지 부담.

---

## 함께 보기

- [09. 종합 KPI 전략](09-monitoring-strategy.md) — 52 메트릭 정의 원본
- [09c. 메트릭 우선순위](09c-metric-priority.md) — Tier S/A/B/C
- [10. 실습 데이터셋](10-practice-dataset.md) — 30M / 30일 / 의도된 시그널
- [10b. Kibana Lens 첫걸음](10b-kibana-lens-basics.md)
- [11~14. 카테고리별 레시피](11-recipe-sre.md)
