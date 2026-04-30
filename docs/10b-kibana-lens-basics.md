# 10b. Kibana Lens 첫걸음 — 처음 사용자를 위한 5분 가이드

> **목적**: [11~15 의 메트릭 레시피](11-recipe-sre.md) 를 따라하기 전에 Lens 의 *기본 조작* 을 5분에 익히는 가이드. **ES/Kibana 처음 사용자도 그대로 따라하면 동작** 하는 단계로 작성.

---

## 1. Lens 가 뭔가?

Kibana 의 **시각화 도구**. SQL/KQL 을 직접 작성하지 않고 *드래그 & 드롭* 으로 차트를 만든다.

```
다른 BI 도구 비유:
  Tableau / Power BI / Excel 피벗테이블 ≈ Kibana Lens
```

핵심 개념 4개만 알면 된다:

| 개념 | 한 줄 설명 |
|---|---|
| **Data view** | 어느 인덱스를 다룰지 (예: `msa-logs-*`) |
| **시간 범위** | 우상단의 시간 픽커 ("Last 7 days" 등) |
| **차원 (Dimension)** | X축 / Break down by — *그룹화 기준* |
| **함수 (Function)** | count, sum, average, percentile, formula 등 |

---

## 2. Lens 에 도달하는 법

```
Kibana 좌측 사이드바
└─ Analytics
   └─ Visualize Library     ← 차트 모음
      └─ [Create visualization]
         └─ Lens             ← 우리가 쓸 도구
```

또는 **Dashboard → Edit → Create visualization → Lens**.

---

## 3. Lens 화면 구조

```
┌─────────────────────────────────────────────────────────────┐
│ [Data view: msa-logs-*]      [Last 30 days ▼]   [Save] [...]│ ← 상단
├──────────┬──────────────────────────────────┬────────────────┤
│          │                                  │                │
│ 필드 목록 │      차트 미리보기                 │  설정 패널     │
│          │                                  │                │
│ svc_c    │                                  │ Horizontal axis│
│ msg_c    │   (드래그하면 여기에 차트가)       │ Breakdown      │
│ proc_tm  │                                  │ Vertical axis  │
│ ...      │                                  │ Filters        │
│          │                                  │                │
└──────────┴──────────────────────────────────┴────────────────┘
```

| 영역 | 역할 |
|---|---|
| **좌측 (필드)** | 우리 데이터의 모든 필드 — 검색/드래그 가능 |
| **중앙** | 차트 미리보기 — 실시간 갱신 |
| **우측 (설정)** | 축, 필터, 함수 설정 |

---

## 4. 차트 타입 결정

차트 종류는 *데이터의 모양* 에 따라 자동 추천. 핵심 4종:

| 타입 | 언제 쓰나 | 예시 메트릭 |
|---|---|---|
| **Metric** | 숫자 1개 | M-S1 Availability "99.5%" |
| **Line** | 시간 추세 | M-S2 TPS, M-S4 latency |
| **Bar (수직)** | 카테고리 비교 | M-D1 Top msg_c |
| **Bar (수평) / Table** | 많은 항목 비교 | M-P1 11 MSA Health |
| **Heatmap** | 두 차원 격자 | M-D4 MSA × msg_c |
| **Pie / Donut** | 비율 | M-U1 채널 분포 |

> 우상단 차트 타입 버튼에서 변경 가능. Lens 가 데이터를 보고 *추천* 도 해줌.

---

## 5. 가장 자주 쓰는 함수 5가지

설정 패널에서 차원을 추가할 때 함수를 고른다.

### ① Count

```
함수: Count of records
의미: 행(doc) 개수
예시: 전체 트래픽
```

### ② Average / Median / Percentile

```
함수: Average / Percentile
필드: proc_tm
의미: 처리시간의 평균 / 95% / 99%
예시: M-S4 Latency p95
```

### ③ Unique count (Cardinality)

```
함수: Unique count
필드: dgtl_cusno
의미: 고유 사용자 수
예시: M-U11 DAU
```

### ④ Filter

```
함수: Count
KQL filter: sts_c : "ERROR"
의미: 에러 개수만
예시: M-S3 Error Rate
```

### ⑤ Formula (가장 강력)

```
함수: Formula
수식: count(kql='sts_c:"OK"') / count()
의미: 임의 비율 계산
예시: M-S1 Availability
```

> **Formula 의 핵심**: `count(kql='...')` 으로 *조건부 카운트* 가능. 이게 ES 의 `terms` 와 `filter` agg 를 한 줄에 표현하는 방법.

---

## 6. 시간축 (Date Histogram)

대부분의 차트는 X축에 시간을 둔다.

```
Horizontal axis (X축):
  필드: @timestamp
  함수: Date histogram
  Interval: 자동 / 1 hour / 1 day
```

> `@timestamp` 를 X축에 드롭하면 자동으로 Date histogram. interval 은 시간 범위에 맞춰 자동 (Last 30 days → 1 day).

---

## 7. 차원 분리 (Breakdown)

같은 차트에 여러 그룹을 색상으로 구분.

```
Breakdown by:
  필드: svc_c
  함수: Top values
  Number of values: 11 (전체 표시)
```

→ 결과: 11 MSA 별로 색상 다른 시계열.

---

## 8. KQL 필터 — 한 페이지

KQL = Kibana Query Language. SQL 의 WHERE 와 비슷.

| 패턴 | 예시 |
|---|---|
| 필드 = 값 | `svc_c : "PY"` |
| 다중 값 (OR) | `svc_c : ("PY" or "AS")` |
| 와일드카드 | `log_div : *_OUT` |
| AND | `svc_c : "PY" and sts_c : "ERROR"` |
| NOT | `not svc_c : "PA"` |
| 비교 | `proc_tm > 1000` |
| 존재 | `gw_uuid : *` |
| 부재 | `not gw_uuid : *` |

> 자세한 cheatsheet → [99-kql-cheatsheet.md](99-kql-cheatsheet.md).

---

## 9. 차트 저장 → Dashboard 추가

```
1. 차트 만들기 완료 → 우상단 [Save] 클릭
2. Title 입력: "M-S1 Availability"
3. Description: 한 줄 설명 (선택)
4. Add to Library
5. Dashboard 화면 → [Add from library] → 검색해서 추가
```

---

## 10. 5분 실습 — 첫 차트 만들기

따라하면 *전체 시계열 트래픽* 차트가 만들어진다.

```
① Analytics → Visualize Library → Create visualization → Lens
② Data view: msa-logs-*
③ 시간 범위 (우상단): "Last 30 days"
④ 차트 타입: Line (자동 추천될 것)
⑤ Horizontal axis: @timestamp 드래그 → Date histogram (자동)
⑥ Vertical axis: 좌측 검색창에 "Records" 검색 → 빈 곳에 드래그
   → 함수: Count of records
⑦ Breakdown: svc_c 드래그 → Top values, Number of values: 11
⑧ 우상단 Save → Title: "전체 트래픽 by MSA" → Save and return
```

→ Day 15 의 anomaly spike 가 보인다면 환경이 정상이다.

---

## 11. 자주 막히는 함정 7가지

| 증상 | 원인 / 해결 |
|---|---|
| "No results found" | 시간 범위가 데이터 외부. 우상단 시간 픽커를 데이터 범위로 |
| 차트 빈 막대 | KQL 오타 (필드명 case-sensitive) |
| 카운트가 너무 적다 | Filter 누락 — 좌측 Filters 영역 확인 |
| Formula 가 빨강 | 따옴표 짝 안 맞음. `kql='...'` 또는 `kql="..."` |
| Top values 가 5개만 | Number of values 기본값 5 → 늘리기 |
| 시간이 한국시간이 아님 | 우상단 톱니 → Advanced → `dateFormat:tz` 를 `Asia/Seoul` 로 |
| Dashboard 에 같은 차트 2번 | Save 후 한번 더 [Add from library] — Library 에 있는지 먼저 확인 |

---

## 12. 다음 단계

| 다음 문서 | 내용 |
|---|---|
| [11. SRE 레시피 (M-S1~S10)](11-recipe-sre.md) | Tier S 의 가용성/에러/지연 — *가장 먼저 만들 것* |
| [12. 백엔드 레시피 (M-P1~P12)](12-recipe-backend.md) | MSA 매트릭스 / 컨테이너 / Tier latency |
| [13. 도메인 레시피 (M-D1~D12)](13-recipe-domain.md) | msg_c / 결제 funnel / fir_c |
| [14. 사용자 레시피 (M-U1~U12)](14-recipe-user-ux.md) | 채널 / OS / 기기 / 신앱 회귀 |
| [15. 운영 레시피 (M-O1~M-X4)](15-recipe-ops.md) | 시간대 패턴 / 인덱스 헬스 |
