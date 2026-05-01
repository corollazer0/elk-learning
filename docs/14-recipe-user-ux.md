# 14. 사용자 / UX 레시피 (M-U1 ~ M-U12)

> **선수**: [13. 도메인 레시피](13-recipe-domain.md)
>
> 사용자 segment / 채널 / OS / 기기 / 앱 버전 / 화면 / 코호트 — *PM 과 운영팀이 가장 좋아하는* 12 메트릭. **신규 카테고리** (09 정정 후 추가).

> ⏱️ **시간 범위 안내**: 메트릭 제목 옆은 **권장 default**. 사용자/UX 메트릭은 보통 *주~월 단위* 가 의미 있음. 실습 데이터셋은 2026-04-01~30 절대 시각.

---

## 학습 순서 추천

```
1. M-U1 채널별 Traffic Share        ← 가장 직관적
2. M-U2 채널별 Error Rate
3. M-U6 기기 모델별 painPoint        ← 우리 데이터셋의 의도된 시그널
4. M-U8 신 앱 버전 회귀              ← Day 18+ 시그널
5. M-U11 DAU / WAU / MAU            ← 임원 보고
6. M-U7 앱 버전 분포
7. M-U10 사용자 retry 패턴
8. M-U4 OS 별 분포
9. M-U5 OS 버전별 Error Rate
10. M-U3 채널별 Latency 비교
11. M-U9 화면별 Funnel
12. M-U12 IP 대역 분포
```

---

## M-U1. 채널별 Traffic Share — ⏱️ Last 7 days

**한 줄 정의**: count by chan_c

**왜 보는가?**
- 모바일 vs 웹 사용자 비중 — 투자 우선순위
- 채널별 고유 패턴 발견

**Kibana 만들기**:

```
1. Lens → Donut
2. Slice by: chan_c (4 values: MA/MW/PW/DA)
3. Size by: Count
4. KQL: log_div : (PT_OUT or BT_OUT)
5. Title: "M-U1 채널별 Traffic Share"
6. Save
```

**해석 가이드**:
- 우리 환경 baseline: MA 55% / MW 25% / PW 15% / DA 5%
- 모바일웹 (MW) 이 갑자기 +10%p = 모바일앱 장애 가능성

**우리 데이터셋에서**:
- 의도된 분포 정확히 위와 같음

**업무 개선 활용**:
- 매니저 보고: "이번 분기 모바일앱 사용자 +5%p"
- 채널별 마케팅 ROI

---

## M-U2. 채널별 Error Rate — ⏱️ Last 7 days

**Kibana 만들기**:

```
1. Lens → Bar (vertical)
2. Horizontal axis: chan_c (4 values)
3. Vertical axis:
   ─ 함수: Formula
   ─ 수식: count(kql='sts_c:"ERROR"') / count()
   ─ 표시: Percent
4. Filter: log_div : (PT_OUT or BT_OUT)
5. Title: "M-U2 채널별 Error Rate"
6. Save
```

**해석 가이드**:
- 모든 채널 비슷 = 정상
- 1 채널만 +50% = 그 채널 한정 painPoint

**업무 개선 활용**:
- Alert R-P1-9 source
- 채널 한정 회귀 발견

---

## M-U3. 채널별 Latency 비교 — ⏱️ Last 7 days

```
M-U2 와 동일 구조
Vertical axis: Percentile(95) of proc_tm
Filter: log_div : *_OUT
Title: "M-U3 채널별 p95 Latency"
```

**해석 가이드**:
- DA (PC) 가 빠른 게 자연스러움 (네트워크/디바이스 우위)
- MA 가 가장 느리면 모바일 최적화 필요

---

## M-U4. OS 별 분포 — ⏱️ Last 30 days

```
1. Lens → Donut
2. Slice by: os_kdc (A/I/W)
3. Size by: Count
4. Title: "M-U4 OS 분포"
```

**우리 데이터셋**: A 50% / I 45% / W 5%

---

## M-U5. OS 버전별 Error Rate — ⏱️ Last 30 days

**왜 보는가?**
- 호환성 이슈 발견 — 오래된 OS 버전이 깨지는 신호

**Kibana 만들기**:

```
1. Lens → Bar horizontal
2. Vertical axis: os_ver (Top 10)
3. Horizontal axis: count(kql='sts_c:"ERROR"') / count()
4. Breakdown: os_kdc
5. Title: "M-U5 OS 버전별 Error Rate"
6. Save
```

**해석 가이드**:
- 옛 버전 (예: iOS 15.7, Android 11) 만 에러율 높으면 호환성 이슈
- 최신 버전이 +30% 면 최신 OS 적응 미흡

---

## M-U6. 기기 모델별 painPoint ⭐ — ⏱️ Last 30 days

**한 줄 정의**: count(ERROR) by device_model, Top 20

**왜 보는가?**
- 특정 기기에서만 깨지는 버그 발견
- 회사 입장에서 *발견하기 어려운* 결함

**Kibana 만들기**:

```
1. Lens → Bar horizontal
2. Vertical axis: device_model (Top 20)
3. Horizontal axis:
   ─ 함수: Count
   ─ KQL: sts_c : "ERROR"
4. Sort: Descending
5. Title: "M-U6 Top 20 기기 모델 painPoint"
6. Save
```

**더 정확한 버전 (error rate)**:

```
2. Vertical axis: device_model
3. Horizontal axis:
   ─ 함수: Formula
   ─ 수식: count(kql='sts_c:"ERROR"') / count()
   ─ 표시: Percent
   ─ Filter (advanced filter): count() > 100  (적은 표본 제외)
```

**우리 데이터셋에서** (의도된 시그널):
- **`SM-G960N`** 모델만 에러율 ~1.5% (전체 0.5% 의 3배) — *상시*
- 전체 30일 동안 일관되게 보임

**업무 개선 활용**:
- 회사가 *몰랐던 결함* 발견의 1순위
- QA 테스트 매트릭스에 추가
- CS 티켓 매핑 ("S22 사용자 컴플레인" → 정량 확인)

---

## M-U7. 앱 버전 분포 — ⏱️ Last 30 days

```
Lens → Donut (또는 Bar)
Slice: ap_ver
Size: Count
Title: "M-U7 앱 버전 분포"
```

**우리 데이터셋**:
- Day 1~17: 4.4.0 / 4.4.5 / 4.5.0 만
- Day 18+: 4.5.1 등장 (의도된 신앱 출시)

**업무 개선**:
- 강제 업데이트 의사결정 ("구 버전 5% 이하 → 강제 적용")

---

## M-U8. 신 앱 버전 회귀 ⭐⭐ — ⏱️ Last 14 days

**한 줄 정의**: 신 ap_ver 의 error rate vs 이전 ver

**왜 보는가?**
- 배포 후 회귀 *조기* 발견 — 사용자 본격 영향 전에
- A/B 테스트 결과 측정

**Kibana 만들기**:

```
1. Lens → Bar (vertical)
2. Horizontal axis: ap_ver (Top 5)
3. Vertical axis:
   ─ 함수: Formula
   ─ 수식: count(kql='sts_c:"ERROR"') / count()
   ─ 표시: Percent
4. Filter: log_div : *_OUT
5. Title: "M-U8 ap_ver 별 Error Rate"
6. Save
```

**더 정밀**: timeshift 비교

```
함수: Formula
수식:
  count(kql='ap_ver:"4.5.1" and sts_c:"ERROR"') / count(kql='ap_ver:"4.5.1"')
  /
  count(kql='not ap_ver:"4.5.1" and sts_c:"ERROR"') / count(kql='not ap_ver:"4.5.1"')
표시: Number (1.0 = 동일, > 1.0 = 신앱 회귀)
```

**우리 데이터셋에서**:
- **Day 18+ 의 ap_ver:"4.5.1" 가 다른 ver 의 약 2.5배 에러율** — 의도된 회귀 시그널
- 시간 범위 Last 14 days 으로 좁혀서 보면 ratio ~2.5

**업무 개선 활용**:
- Alert R-P1-8 source
- 배포 후 24시간 자동 모니터링
- 💎 Platinum: ML population analysis 로 다변량 회귀 자동 감지

---

## M-U9. 화면별 Funnel (scrn_id) — ⏱️ Last 7 days

**한 줄 정의**: 한 scrn_uuid 안의 svc_id 시퀀스 분석

**Kibana 만들기** (Discover 탐색):

```
1. Discover → 시간 범위 Last 24h
2. KQL: scrn_uuid : "<특정 화면 인스턴스>"
3. Sort: @timestamp asc
4. 컬럼 추가: svc_id, sts_c, proc_tm
→ 한 사용자 한 화면의 모든 호출 시퀀스 가시화
```

**Lens 로 집계**:

```
Bar horizontal
Vertical: svc_id (Top 20)
Horizontal: Cardinality of scrn_uuid
KQL: scrn_id : "<특정 화면>"
```

---

## M-U10. 사용자 retry 패턴 ⭐ — ⏱️ Last 24 hours

**한 줄 정의**: 같은 dgtl_cusno + scrn_uuid 에서 sts_c:ERROR 가 ≥3 회

**왜 보는가?**
- 사용자 이탈 위험 신호 — 3번 시도 후 포기
- 알림 fatigue 측정

**Kibana 만들기** (Lens 단독으로 어려움 — Transform 권장):

```yaml
# Transform 'user-retry-1h'
group_by:
  dgtl_cusno: terms
  scrn_uuid:  terms
  ts:         date_histogram(1h)
aggregations:
  err_count: filter sts_c:"ERROR", count
filter (post): err_count >= 3
```

**Discover 대안**:

```
1. Discover
2. KQL: sts_c : "ERROR"
3. Save → Aggregation: dgtl_cusno + scrn_uuid 그룹 count, having count >= 3
   (Lens 의 advanced 표현 또는 ES SQL 사용)
```

**우리 데이터셋**:
- 의도된 0.5% retry pattern — 약 ~250 사용자/일 발견 가능

**업무 개선 활용**:
- CX 팀에 *심각한 painPoint* 사용자 list 전달
- 자동 응대 / 보상 트리거

---

## M-U11. DAU / WAU / MAU ⭐⭐ — ⏱️ Last 1d / 7d / 30d (각각)

**한 줄 정의**: cardinality(dgtl_cusno)

**왜 보는가?**
- 매니저 / 임원 KPI 1순위
- 비즈니스 성장의 직접 척도

**Kibana 만들기**:

```
DAU:
1. Lens → Metric
2. Primary: Cardinality of dgtl_cusno
3. 시간 범위: Today (또는 Yesterday)
4. Title: "M-U11 DAU"

WAU: 시간 범위 Last 7 days
MAU: 시간 범위 Last 30 days
```

**시계열로**:

```
Lens → Line
Horizontal: @timestamp (1d)
Vertical: Cardinality of dgtl_cusno
Title: "M-U11 일별 DAU 추세"
```

**우리 데이터셋**:
- 일 unique 약 8만 (사용자 풀 50만 중)
- 30일 cumulative ≈ 30만~40만 (MAU)

**업무 개선 활용**:
- 임원 보고 1번
- 마케팅 캠페인 효과 측정 ("캠페인 후 DAU +12%")

---

## M-U12. IP 대역 분포 — ⏱️ Last 30 days

```
1. Lens → Bar horizontal
2. Vertical axis: usr_ip (Top 30, 또는 ip_range field)
3. Horizontal: Count
4. Title: "M-U12 IP 대역 분포"
```

**해석**:
- 평소 분산 = 정상
- 단일 IP 가 trafffic 1% 이상 = bot 의심

---

## D-D3: 사용자 / UX 분석 Dashboard

```
┌──────────────────────────────────────────────────────────────┐
│ M-U11 DAU/WAU/MAU (Metric ×3)                                │
├────────────────────────────┬─────────────────────────────────┤
│ M-U1 채널 Traffic Share    │ M-U4 OS 분포 (Donut)            │
├────────────────────────────┼─────────────────────────────────┤
│ M-U2 채널별 Error Rate     │ M-U5 OS 버전별 Error Rate       │
├────────────────────────────┴─────────────────────────────────┤
│ M-U6 Top 20 기기 painPoint                                   │
├──────────────────────────────────────────────────────────────┤
│ M-U7 앱 버전 분포      │ M-U8 신앱 ver 회귀 (timeshift)       │
├────────────────────────┴─────────────────────────────────────┤
│ M-U10 사용자 retry 패턴                                      │
└──────────────────────────────────────────────────────────────┘

Save: "D-D3 사용자 / UX 분석"
```

---

## 다음 문서

- [15. 운영 / Op-Excellence 레시피](15-recipe-ops.md) — M-O1~O6 + M-X1~X4
