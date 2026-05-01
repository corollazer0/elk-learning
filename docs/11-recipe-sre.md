# 11. SRE Golden Signals 레시피 (M-S1 ~ M-S10)

> **선수 학습**: [10b. Kibana Lens 첫걸음](10b-kibana-lens-basics.md)
>
> SRE 4대 시그널 (Latency / Traffic / Errors / Saturation) 을 우리 환경에 맞게 10개 메트릭으로 분리. 이 카테고리에서 [Tier S](09c-metric-priority.md) 5개 (M-S1, S2, S3, S4, S10) 가 나옴 — *가장 먼저 만들 것*.

> ⏱️ **시간 범위 안내**:
> - 각 메트릭 제목 옆 시간 범위는 **권장 default** (실 운영 기준)
> - 우리 실습 데이터셋은 **2026-04-01 ~ 04-30 (30일 고정)** — Kibana 우상단 시간 픽커에서 *절대 시각* 으로 입력
>   ```
>   Start: 2026-04-01 00:00:00     End: 2026-04-30 23:59:59
>   ```
>   또는 "Last 30 days" 가 실 데이터를 모두 덮으면 그대로 사용 가능

---

## 학습 순서 추천

```
1. M-S1 Availability   ← 가장 먼저
2. M-S3 Error Rate
3. M-S4 Latency p95/p99
4. M-S2 TPS / RPS
5. M-S5 Slow Request
6. M-S8 G/W p95
7. M-S9 MCI core p95
8. M-S7 Tier 별 Availability
9. M-S6 Saturation
10. M-S10 Error Budget
```

---

## M-S1. Availability (가용성) — ⏱️ Last 24 hours

**한 줄 정의**: OK 응답 / 전체 응답 ratio

**왜 보는가?**
- 서비스가 살아있는지를 가장 직관적으로 보여주는 지표
- SLO (예: 99.9%) 의 numerator
- 사용자가 "안 돼" 라고 느끼는 임계점이 99.5% 정도

**Kibana 만들기** (Lens, 5분):

```
1. Analytics → Visualize Library → Create visualization → Lens
2. Data view: msa-logs-*
3. 시간 범위: Last 30 days
4. 차트 타입: Metric (우상단)
5. Primary metric:
   ─ 함수: Formula
   ─ 수식:
     count(kql='log_div : (PT_OUT or BT_OUT or MCI_RECV or GW_RECV) and sts_c : "OK"')
     /
     count(kql='log_div : (PT_OUT or BT_OUT or MCI_RECV or GW_RECV)')
   ─ 표시: Percent, 소수점 3자리
6. Title: "M-S1 Availability (전체)"
7. Save
```

**해석 가이드**:

| 값 | 의미 | 행동 |
|---|---|---|
| ≥ 99.9% | SLO 충족 | 정상 |
| 99.5~99.9% | budget 소진 중 | 모니터링 강화 |
| 99.0~99.5% | 이상 신호 | 인시던트 prep |
| < 99.0% | 비상 | 즉시 대응 |

**우리 데이터셋에서**:
- 전체 30일 평균 ~99.5% 정도
- `svc_c:"PY"` 필터를 추가하면 day 22 spike 발견 가능 (96~98%)
- 원인: `py-pod-2` 컨테이너 (M-P8 에서 drill)

**업무 개선 활용**:
- Dashboard D-RT1 의 첫 패널 — 매일 5초 진단
- SLO 보고서의 numerator
- 인시던트 회고 시 영향 범위 정량화 ("0.3%p 하락 = 30만 거래 영향")
- M-S10 Error Budget 의 입력값

---

## M-S2. TPS / RPS (Throughput) — ⏱️ Last 24 hours

**한 줄 정의**: 초당 응답 수

**왜 보는가?**
- 트래픽 baseline 의 척도
- spike (이벤트성) 또는 dropout (장애) 즉시 감지
- capacity 증설 의사결정

**Kibana 만들기**:

```
1. Lens → 차트 타입: Line (자동)
2. Horizontal axis: @timestamp (Date histogram, Auto)
3. Vertical axis:
   ─ 함수: Count of records
   ─ KQL filter: log_div : (PT_OUT or BT_OUT or MCI_RECV or GW_RECV)
   ─ Display name: "TPS (응답 기준)"
4. Title: "M-S2 TPS / RPS"
5. Save
```

**해석 가이드**:
- 평일 주간 (9~18시) peak — off-peak 의 3~5배가 정상
- 피크 시간 spike 가 평소의 1.4배 이상이면 anomaly
- 갑자기 drop 하면 "사용자가 못 들어온다" → 인입 단 문제

**우리 데이터셋에서**:
- 평소 10K~12K req/min peak
- **Day 15 (4월 15일)** 에 ×1.4 spike — anomaly 학습 포인트
- 토/일 30~50% drop 정상

**업무 개선 활용**:
- Capacity 계획의 1차 지표 (peak 의 80% 도달 시 증설)
- ML Anomaly Detection (Platinum) 의 1순위 적용 대상
- 매니저 보고: "지난주 대비 +12%"

---

## M-S3. Error Rate (에러율) — ⏱️ Last 24 hours

**한 줄 정의**: ERROR 응답 / 전체 응답

**왜 보는가?**
- 사고 첫 신호. 보통 가용성보다 *민감하게* 반응
- 비즈니스 임팩트와 직결 (실패한 요청 = 손실)

**Kibana 만들기**:

```
1. Lens → 차트 타입: Line
2. Horizontal axis: @timestamp
3. Vertical axis:
   ─ 함수: Formula
   ─ 수식:
     count(kql='log_div : (PT_OUT or BT_OUT or MCI_RECV or GW_RECV) and sts_c : "ERROR"')
     /
     count(kql='log_div : (PT_OUT or BT_OUT or MCI_RECV or GW_RECV)')
   ─ 표시: Percent (0.00%)
4. Breakdown: svc_c (Top 5) — 어느 MSA 가 에러 주범인지
5. Title: "M-S3 Error Rate"
6. Save
```

**해석 가이드**:
- 평소 < 0.5% — 알림 임계 0.5%
- > 1% = P1 alert
- > 2% = P0 alert (oncall page)
- spike 모양 (5분 이내 회귀) vs sustained (장애 지속)

**우리 데이터셋에서**:
- baseline 0.5%
- **Day 22**: PY 만 에러율 ~4% — `svc_c:"PY"` 필터로 보면 명확
- breakdown 으로 보면 PY 만 튀는 것을 한눈에

**업무 개선 활용**:
- Alert R-P0-2 의 source
- 인시던트 P1 vs P0 분류의 정량 기준
- 신 배포 후 30분 모니터링 1순위 지표

---

## M-S4. Latency p50/p95/p99 — ⏱️ Last 24 hours

**한 줄 정의**: 처리시간의 백분위수

**왜 보는가?**
- 평균은 outlier 에 흔들림 — percentile 이 더 진실
- p50 = 중앙값 (대부분 사용자 경험)
- p95 = "안 좋은 5% 가 얼마나 안 좋은가"
- p99 = tail risk (1% outlier — 1만 명 중 100명이 이렇게 느림)

**Kibana 만들기**:

```
1. Lens → 차트 타입: Line
2. Horizontal axis: @timestamp
3. Vertical axis (3개 추가 — 같은 패널에 3 라인):
   ① 함수: Percentile, 필드: proc_tm, Percentile: 50, KQL: log_div : *_OUT, Display: "p50"
   ② 함수: Percentile, 필드: proc_tm, Percentile: 95, KQL: log_div : *_OUT, Display: "p95"
   ③ 함수: Percentile, 필드: proc_tm, Percentile: 99, KQL: log_div : *_OUT, Display: "p99"
4. Y축 단위: ms
5. Title: "M-S4 Latency p50/p95/p99"
6. Save
```

**해석 가이드**:
- p99/p50 비율이 5 이하 = 정상 분포
- p99/p50 > 10 = tail 이 비대칭 — 일부만 매우 느림
- 시간대에 따라 변동 — peak 시 p95 +30% 정상

**우리 데이터셋에서**:
- p50 ≈ 80~100ms / p95 ≈ 400ms / p99 ≈ 1500ms
- ap_ver:"4.5.1" 필터를 더하면 (day 18+) p95 가 +50% — 신앱 회귀 확인

**업무 개선 활용**:
- SLO 의 latency target ("p95 < 500ms")
- 신 배포 후 24시간 회귀 감지
- 사용자 불만 이슈와 매핑 ("로딩 느려요" = p95 에서 보이는가?)

---

## M-S5. Slow Request Rate — ⏱️ Last 24 hours

**한 줄 정의**: proc_tm > 1000ms 인 요청의 비율

**왜 보는가?**
- p95/p99 보다 *직관적* — "1초 이상 걸린 사용자 몇 %?"
- SLO 정의가 단순 (예: "Slow rate < 1%")

**Kibana 만들기**:

```
1. Lens → 차트 타입: Metric (단일) 또는 Line (시계열)
2. Primary metric:
   ─ 함수: Formula
   ─ 수식:
     count(kql='log_div : *_OUT and proc_tm > 1000')
     /
     count(kql='log_div : *_OUT')
   ─ 표시: Percent
3. Title: "M-S5 Slow Request Rate (>1s)"
4. Save
```

**해석 가이드**:
- 평소 < 1%
- > 2% = 사용자 체감 저하 시작
- > 5% = SLA 위반 위험

**우리 데이터셋에서**:
- 의도된 1% slow tail — 약 1% 가 1.5~6초

**업무 개선 활용**:
- 매니저 보고에 가장 직관적 ("99% 사용자는 1초 안에 응답")
- 회사의 페이지 로딩 SLA 와 직결

---

## M-S6. Saturation (인스턴스 부하) — ⏱️ Last 24 hours

**한 줄 정의**: 컨테이너 별 트래픽 분포의 max/avg

**왜 보는가?**
- 로드밸런서가 *제대로 분산* 하고 있는가?
- 단일 pod 에 트래픽 집중 = SPOF 위험

**Kibana 만들기**:

```
1. Lens → 차트 타입: Bar horizontal
2. Vertical axis: ctnr_nm (Top 30)
3. Horizontal axis:
   ─ 함수: Count
   ─ KQL: log_div : *_OUT
4. Title: "M-S6 컨테이너 부하 분포"
5. Save
```

**해석 가이드**:
- 같은 svc 의 pod 들이 *비슷한 길이* 면 정상 (예: py-pod-0 ~ py-pod-3 모두 ~25%)
- 1 pod 만 길거나 짧으면 LB 문제 / pod 다운

**우리 데이터셋에서**:
- 각 MSA 당 4개 pod (`{svc}-pod-0~3`) 로 균등 분포
- Day 22 의 py-pod-2 도 traffic 은 정상 (에러만 spike)

**업무 개선 활용**:
- 배포 후 LB 정상 분산 확인
- HPA (오토스케일) 동작 검증

---

## M-S7. Tier 별 Availability — ⏱️ Last 7 days

**한 줄 정의**: PT/BT/MCI 각각의 가용성 분리

**왜 보는가?**
- 어느 layer 에서 떨어지는가? (사내 코드 vs 외부 의존성)
- MCI/GW 가 떨어지면 우리 책임이 아닐 수 있음

**Kibana 만들기**:

```
1. Lens → 차트 타입: Bar (수직, grouped)
2. Horizontal axis: tier_c (PT/BT/MCI)
3. Vertical axis:
   ─ 함수: Formula
   ─ 수식:
     count(kql='sts_c:"OK"') / count()
   ─ 표시: Percent
4. Filter (좌측 Filters 또는 Lens KQL bar):
   log_div : (PT_OUT or BT_OUT or MCI_RECV or GW_RECV)
5. Title: "M-S7 Tier 별 Availability"
6. Save
```

**해석 가이드**:
- PT > BT > MCI 순으로 일반적
- MCI 가 99.5% 이하면 *외부 책임*
- BT/PT 가 떨어지면 우리 책임

**업무 개선 활용**:
- 외부 의존성 SLA 협상 근거
- 인시던트 책임 split

---

## M-S8. 외부 의존성 (G/W) Latency — ⏱️ Last 24 hours

**한 줄 정의**: GW_SEND/RECV 의 p95

**왜 보는가?**
- 대외 시스템은 통제 불가 → 별도 모니터링 필요
- 사내 코어 (MCI) 와 분리해야 패턴 보임

**Kibana 만들기**:

```
1. Lens → 차트 타입: Line
2. Horizontal axis: @timestamp
3. Vertical axis:
   ─ 함수: Percentile (95)
   ─ 필드: proc_tm
   ─ KQL: log_div : (GW_SEND or GW_RECV)
4. Breakdown: svc_id (Top 10)
5. Title: "M-S8 G/W p95 by svc_id"
6. Save
```

**해석 가이드**:
- 평소 700~1000ms (외부는 느림)
- > 2000ms 지속 = 외부 시스템 점검 요청

**우리 데이터셋에서**:
- 평소 G/W p95 ≈ 1100ms
- MCI 와 비교하면 약 4배

**업무 개선 활용**:
- 외부 SLA 위반 추적 (월별 보고)
- 외부 도구 변경 의사결정 (대안 G/W 검토)

---

## M-S9. 코어 의존성 (MCI) Latency — ⏱️ Last 24 hours

**한 줄 정의**: MCI_SEND/RECV 의 p95

**왜 보는가?**
- 외부 계정계 / 레거시 호스트 의존성
- 결제 critical path — 느려지면 즉시 사용자 영향

**Kibana 만들기**:

```
M-S8 와 동일하되 KQL 만:
  KQL: log_div : (MCI_SEND or MCI_RECV)
Title: "M-S9 MCI core p95"
```

**해석 가이드**:
- 평소 200~300ms (코어니까 빠름)
- > 500ms 지속 = 결제 영향

**우리 데이터셋에서**:
- 평소 MCI p95 ≈ 250ms

**업무 개선 활용**:
- 결제 SLA 의 직접 입력
- 코어 인프라팀과의 협업 baseline

---

## M-S10. Error Budget Burn Rate — ⏱️ Last 30 days (rolling)

**한 줄 정의**: 30일 rolling SLO 위반 누적

**왜 보는가?**
- 한 달 중 *얼마나 SLO 를 어겼나* 를 0~100% 로 표현
- 80% 넘으면 *남은 기간 동안 더 위험 감수 X*

**Kibana 만들기**:

```
SLO = 99.9% (= 0.001 budget)
Burn rate = (1 - Availability) / 0.001

1. Lens → 차트 타입: Metric
2. Primary metric:
   ─ 함수: Formula
   ─ 수식:
     ( 1 -
       count(kql='log_div : (PT_OUT or BT_OUT or MCI_RECV or GW_RECV) and sts_c : "OK"')
       /
       count(kql='log_div : (PT_OUT or BT_OUT or MCI_RECV or GW_RECV)')
     ) / 0.001
3. 시간 범위: Last 30 days (rolling)
4. Title: "M-S10 Error Budget Burn"
5. Save
```

**해석 가이드**:
- 1.0 = budget 정확히 다 씀
- 0.5 = budget 50% 사용
- > 1.5 = 다음 달까지 *추가 사고 없어야* 회복

**우리 데이터셋에서**:
- 평소 0.5 정도 (= budget 의 50% 사용)

**업무 개선 활용**:
- 분기 SLA 보고
- "이번 달은 신 배포 자제" 같은 의사결정 근거
- 매니저 / 임원 보고

---

## D-RT1: SRE Golden Signals Dashboard 조립

위 10개를 모두 만든 후:

```
1. Dashboards → Create dashboard
2. [Add from library] → 위에서 저장한 10개 추가
3. 권장 배치 (4×3 그리드):

  ┌──────────────┬──────────────┬──────────────┬──────────────┐
  │ M-S1 Avail   │ M-S3 Error % │ M-S4 p95     │ M-S2 TPS     │
  │  (Metric)    │  (Metric)    │  (Metric)    │  (Metric)    │
  ├──────────────┼──────────────┴──────────────┴──────────────┤
  │ M-S10 Budget │ M-S2 TPS time series (Line)                 │
  ├──────────────┼─────────────────────────────────────────────┤
  │ M-S5 Slow %  │ M-S4 p50/p95/p99 time series (Line)         │
  ├──────────────┴──────────────┬──────────────────────────────┤
  │ M-S8 G/W p95                │ M-S9 MCI core p95            │
  ├─────────────────────────────┴──────────────────────────────┤
  │ M-S6 Container 부하 (Bar horizontal)                       │
  ├────────────────────────────────────────────────────────────┤
  │ M-S7 Tier 별 Availability (Bar grouped)                    │
  └────────────────────────────────────────────────────────────┘

4. Save: "D-RT1 SRE Golden Signals"
5. 시간 범위: Last 24 hours / Refresh: 30 seconds
```

---

## 다음 문서

- [12. 백엔드 / 플랫폼 레시피](12-recipe-backend.md) — M-P1~P12
