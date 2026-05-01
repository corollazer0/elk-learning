# 13. 도메인 / 비즈니스 레시피 (M-D1 ~ M-D12)

> **선수**: [12. 백엔드 레시피](12-recipe-backend.md)
>
> *비즈니스 도메인* 관점의 12 메트릭. msg_c (에러 코드) / 결제 funnel / fir_c (root cause) / 핵심 거래 KPI.

> ⏱️ **시간 범위 안내**: 메트릭 제목 옆은 **권장 default**. 실습 데이터셋 (2026-04-01~30) 에선 우상단 시간 픽커 → "Absolute" 입력.

---

## 학습 순서 추천

```
1. M-D1 Top msg_c                  ← 매일 점검 1번
2. M-D9 fir_c 분포 (root cause)
3. M-D3 Critical svc_id 에러
4. M-D2 신규 msg_c
5. M-D4 MSA × msg_c heatmap
6. M-D5 결제 Funnel (PY)
7. M-D6 인증 성공률 (MB)
8. M-D8 이체 성공률 (FA)
9. M-D7 카드 거래 추세 (CD)
10. M-D10 svc_c 별 거래량 추세
11. M-D11 동시 활성 거래
12. M-D12 biz_c 별 분포
```

---

## M-D1. Top msg_c (Top 에러 코드) — ⏱️ Last 7 days

**한 줄 정의**: count(*) by msg_c, Top 20 — sts_c:ERROR 만

**왜 보는가?**
- 매일 가장 많이 발생한 에러 코드 → 우선 fix 대상
- 낙수 (long tail) vs 집중 — 패턴 파악

**Kibana 만들기**:

```
1. Lens → Bar horizontal
2. Vertical axis: msg_c (Top 20)
3. Horizontal axis:
   ─ 함수: Count
   ─ KQL: sts_c : "ERROR"
4. Sort: Descending
5. Title: "M-D1 Top 20 msg_c (errors)"
6. Save
```

**해석 가이드**:
- E003 같은 dominant code 가 80% 차지 = *그것 1개* 만 잡아도 큰 효과
- 균등 분포 = 다양한 원인 — drill 어려움

**우리 데이터셋에서**:
- E003 가 dominant (의도된 weight 25%)
- Day 14+ 에 E099 등장 (M-D2 와 짝)

**업무 개선 활용**:
- 매일 아침 standup 의 backlog 1순위
- "이번 주 fix 우선순위" 결정

---

## M-D2. 신규 msg_c (set diff) — ⏱️ Last 30 days

**한 줄 정의**: 오늘 처음 본 msg_c (지난 7일 base 외)

**왜 보는가?**
- 새 결함 / 새 외부 시스템 응답 코드 조기 발견
- 신 배포 직후 회귀 신호

**Kibana 만들기** (KQL + Discover saved query):

```
방법 1 (Discover saved query):
  KQL: sts_c : "ERROR" and msg_c : "E099"
  → 시간 범위 Last 30 days 으로 보면 day 14 부터 등장

방법 2 (Lens 비교):
  Date histogram: 1d
  Vertical axis: cardinality(msg_c)
  필터: sts_c:"ERROR"
  → 어제 unique msg_c 수가 +1 이면 신규
```

**우리 데이터셋에서**:
- Day 14 부터 E099 등장 — M-D2 가 잡아내야 정상
- Discover 에서 `msg_c:"E099"` 검색 → day 14 가 첫 점

**업무 개선 활용**:
- Alert R-P1-1 source
- 💎 Platinum: ML rare event detection 으로 자동화 가능

---

## M-D3. Critical svc_id Error Rate — ⏱️ Last 24 hours

**한 줄 정의**: 결제/인증/이체 svc_id 의 error rate

**왜 보는가?**
- 비즈니스 임팩트 큰 endpoint 만 분리 모니터링
- 일반 에러와 구분 — *손실 = 매출 직격*

**Kibana 만들기**:

```
1. Lens → Line
2. Horizontal axis: @timestamp
3. Vertical axis:
   ─ 함수: Formula
   ─ 수식: count(kql='sts_c:"ERROR"') / count()
   ─ 표시: Percent
4. Filter (Lens KQL bar):
   svc_c : "PY" or
   (svc_c : "MB" and svc_id : "auth/*") or
   (svc_c : "FA" and biz_c : "transfer*")
5. Breakdown: svc_id (Top 10)
6. Title: "M-D3 Critical svc_id Error Rate"
7. Save
```

**해석 가이드**:
- 평소 < 0.5%
- > 1% = 매출 손실 시작 — P0

**우리 데이터셋에서**:
- Day 22 PY 의 pay/* 가 spike

**업무 개선 활용**:
- 매니저 / 임원 보고용 "비즈니스 영향" 직접 표현
- 결제 / 이체 SLA 의 직접 입력

---

## M-D4. MSA × msg_c Heatmap — ⏱️ Last 7 days

**한 줄 정의**: svc_c × msg_c grid

**왜 보는가?**
- 어느 *서비스 × 코드* 조합이 가장 많은가?
- 책임 영역 명확화 — "E102 는 FA 만, E003 는 모든 svc"

**Kibana 만들기**:

```
1. Lens → Heatmap
2. Horizontal axis: svc_c (Top 11)
3. Vertical axis: msg_c (Top 20)
4. Cell value: Count, KQL: sts_c:"ERROR"
5. Title: "M-D4 MSA × msg_c Heatmap"
6. Save
```

**해석 가이드**:
- 짙은 cell 1개 = 그 svc/code 가 dominant
- 가로 줄 (한 svc 에 다수 code) = 그 서비스가 다양한 결함
- 세로 줄 (한 code 가 여러 svc) = 공통 라이브러리 / 외부 의존성

**업무 개선 활용**:
- 에러 코드 책임 split 협의 자료
- 공통 fix 대상 발견 (가로/세로 패턴)

---

## M-D5. 결제 Funnel (PY scrn_uuid 단위) — ⏱️ Last 7 days

**한 줄 정의**: 결제 5단계 reach rate

**왜 보는가?**
- 결제 UX painPoint 의 정량화
- "어디서 사용자가 빠지는가?" — 가장 비즈니스 임팩트 큰 메트릭

**Kibana 만들기** (조금 복잡 — *순서 있는* funnel):

```
1. Lens → Bar (vertical)
2. Horizontal axis: svc_id (Filters로 5개 단계 매핑)
3. Vertical axis: Cardinality of scrn_uuid
4. Filters (좌측 Filters 영역):
   ① svc_id : "pay/init"     → "1. 진입"
   ② svc_id : "pay/auth"     → "2. 인증"
   ③ svc_id : "pay/confirm"  → "3. 확인"
   ④ ... 등등
5. KQL bar: svc_c : "PY" and log_div : *_OUT
6. Title: "M-D5 결제 Funnel"
7. Save
```

**해석 가이드**:
- 단계별 reach: 진입 100% → 인증 80% → 확인 60% → 완료 50%
- 단계 간 drop > 30% = UX 점검 대상

**우리 데이터셋에서**:
- 의도된 단계별 분포 50~80% reach

**업무 개선 활용**:
- PM / UX 팀과 정량 협업
- A/B 테스트 결과 측정

---

## M-D6. 인증 성공률 (MB) — ⏱️ Last 7 days

**한 줄 정의**: MB 의 sts_c:OK / total

**Kibana 만들기**:

```
1. Lens → Metric (또는 Line)
2. Primary metric:
   ─ 함수: Formula
   ─ 수식: count(kql='sts_c:"OK"') / count()
3. Filter: svc_c : "MB" and log_div : BT_OUT
4. Title: "M-D6 인증 성공률 (MB)"
5. Save
```

**해석 가이드**:
- 평소 > 99% (인증은 보통 매우 안정)
- < 98% = 사고

**업무 개선 활용**:
- 보안 SLA
- 로그인 실패 = UX painPoint

---

## M-D7. 카드 거래 추세 (CD) — ⏱️ Last 30 days

**Kibana 만들기**:

```
M-D6 와 동일 구조
Filter: svc_c : "CD"
시간 범위: Last 30 days
차트: Line
Title: "M-D7 카드 거래 추세"
```

---

## M-D8. 이체 성공률 (FA) — ⏱️ Last 7 days

```
Filter: svc_c : "FA" and biz_c : "transfer*"
함수: Formula — count(kql='sts_c:"OK"') / count()
Title: "M-D8 이체 성공률"
```

---

## M-D9. fir_c 분포 (Root Cause) — ⏱️ Last 24 hours

**한 줄 정의**: 첫 에러가 어느 tier 에서 발생했나?

**왜 보는가?**
- 인시던트 retro 의 1번 차트
- "사내 (PT/BT) vs 외부 (MCI)" 책임 분기점

**Kibana 만들기**:

```
1. Lens → Pie / Donut
2. Slice by: fir_c (3 values: PT/BT/MCI)
3. Size by: Count
4. Filter: fir_err : "Y"
5. Title: "M-D9 fir_c 분포 (root cause)"
6. Save
```

**해석 가이드**:
- 정상 분포: PT 5% / BT 35% / MCI 60% (외부 의존성이 가장 자주)
- BT > 50% = *우리* 코드 회귀 의심

**우리 데이터셋에서**:
- 의도된 분포 PT 5%, BT 35%, MCI 60% 와 일치

**업무 개선 활용**:
- 인시던트 retrospective 의 첫 차트
- 외부 의존성 책임 협상 근거
- 분기 KPI: BT % 를 줄이는 방향

---

## M-D10. svc_c 별 거래량 추세 — ⏱️ Last 30 days

**Kibana 만들기**:

```
1. Lens → Line
2. Horizontal axis: @timestamp (Date histogram, 1d)
3. Vertical axis: Count
4. Breakdown: svc_c (Top 11)
5. KQL: log_div : (PT_OUT or BT_OUT)
6. Title: "M-D10 MSA 별 일별 거래량"
7. Save
```

**해석 가이드**:
- 평일 vs 주말 차이 일정 = 정상
- 특정 svc 가 갑자기 +50% = 캠페인 / 이슈

---

## M-D11. 동시 활성 거래 — ⏱️ Last 24 hours

**한 줄 정의**: 5분 윈도우 안의 unique guid

**Kibana 만들기**:

```
1. Lens → Line
2. Horizontal axis: @timestamp (5m bucket)
3. Vertical axis: Cardinality of guid
4. Title: "M-D11 동시 활성 거래 (5분 unique guid)"
5. Save
```

---

## M-D12. biz_c 별 분포 (도메인 painPoint) — ⏱️ Last 7 days

```
Filter: svc_c : "<특정>"
Pie slice: biz_c
Title: "M-D12 PY 의 biz_c 분포"
```

→ 어느 업무가 가장 빈번 / 가장 에러 많은지.

---

## D-D1: 일일 점검 Dashboard

```
┌──────────────────────────────────────────────────────────────┐
│ 어제 SLO (M-S1) │ 어제 에러 수 (M-S3) │ 신규 msg_c (M-D2)    │
├─────────────────┴─────────────────────┴─────────────────────┤
│ M-D1 Top 20 msg_c (Bar horizontal)                           │
├──────────────────────────────────────────────────────────────┤
│ M-D9 fir_c 분포 (Pie)        │ M-D4 MSA × msg_c (Heatmap)    │
├──────────────────────────────┴───────────────────────────────┤
│ M-D5 결제 Funnel (어제)                                       │
├──────────────────────────────────────────────────────────────┤
│ M-D3 Critical svc 에러 trend (Line)                          │
└──────────────────────────────────────────────────────────────┘

Save: "D-D1 일일 점검"
시간 범위: Yesterday (또는 Last 24h)
```

---

## 다음 문서

- [14. 사용자 / UX 레시피](14-recipe-user-ux.md) — M-U1~U12
