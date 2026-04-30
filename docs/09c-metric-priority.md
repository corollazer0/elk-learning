# 09c. 52 메트릭 우선순위 · 용량 · 플래티넘 영향 (리드 엔지니어 검토)

> **목적**: [09 의 52 메트릭](09-monitoring-strategy.md) 을 *모두 동등하게* 만들 필요는 없습니다. 리드 엔지니어 관점에서 (1) 실무 사용 빈도 / 비즈니스 임팩트 / 운영 의존도 로 **우선순위 4-Tier** 정렬하고, (2) 메트릭별 ES 저장 **용량 델타** 를 산정하며, (3) 플래티넘 라이선스에서만 가능 / 강하게 추천되는 메트릭을 명시합니다.
>
> 결론 1줄: **Tier S(7개) + Tier A(13개) = 20 메트릭이 80% 의 가치**. 나머지 32개는 drill-down·KPI 보고용. 추가 저장 비용은 raw 인덱스 대비 **+0.5%** 미만 (Transform 5개 합계 ≈ 30MB/일).

---

## 0. 사용 가이드 — 이 문서는 누구를 위한 건가?

| 독자 | 어떻게 읽나 | 결과물 |
|---|---|---|
| 처음 도입하는 팀 | §1 Tier S 7개만 먼저 만들고 운영 시작 | D-RT1 + 5개 P0 alert |
| 이미 Tier S 갖춘 팀 | §3 의 Tier A 13개를 4주에 걸쳐 추가 | D-RT2/RT3 + 일일 보고 |
| 매니저 / PM | §1 표만 + §6 도입 로드맵 | 분기별 KPI |
| 인프라 / DBA | §4 용량 산정 + §5 플래티넘 의사결정 | 라이선스 ROI |

---

## 1. Executive Summary — 한 페이지

```
┌──────────────────────────────────────────────────────────────────┐
│  Tier S (7) — 항상 화면, P0 알림                                  │
│   M-S1·S3·S4 (가용성/에러/지연) + M-S2(TPS) + M-P1(MSA matrix)    │
│   + M-P10(파이프라인 lag) + M-D3(critical svc 에러)               │
├──────────────────────────────────────────────────────────────────┤
│  Tier A (13) — 매일 점검, 인시던트 drill 의 시작점                 │
│   S5·S8·S9·S10 / P2·P3·P4·P5·P7·P8 / D1·D9 / U2                  │
├──────────────────────────────────────────────────────────────────┤
│  Tier B (18) — 주간 KPI / 도메인 분석 / 사용자 분석                │
│   S6·S7 / P6·P9·P11 / D2·D4·D5·D6·D8·D11 / U1·U6·U8·U11 / O3·O4·O5│
├──────────────────────────────────────────────────────────────────┤
│  Tier C (14) — 월/분기, niche, 용량 계획                           │
│   P12 / D7·D10·D12 / U3·U4·U5·U7·U9·U10·U12 / O1·O2·O6 / X1~X4    │
└──────────────────────────────────────────────────────────────────┘
                  ↓
       총 52 메트릭, 자체 저장 추가 비용 ≈ raw 의 0.5%
```

| Tier | 개수 | 평균 사용 빈도 | 알림 우선순위 | Platinum 권장 | 누적 가치 |
|:---:|:---:|---|---|---|---|
| **S** | 7 | 모든 화면, 24/7 | P0 (page) | — (Basic 충분) | **40%** |
| **A** | 13 | 매일 1~수십회 | P0~P1 | 일부 (M-D2/M-O5) | **80%** |
| **B** | 18 | 매주 / 인시던트 시 | P1 | M-O5(ML), M-U8(ML) | **95%** |
| **C** | 14 | 월/분기 | P2 / 보고 | M-X2(searchable snapshot) | **100%** |

> **핵심 권고**: 처음 1개월은 **Tier S 7개만** 만들고 운영을 안정화. Tier A 는 인시던트 후 retro 에서 필요성 검증된 것부터 추가. Tier B/C 는 분기 단위 도입.

---

## 2. 우선순위 결정 기준

리드 엔지니어로서 메트릭에 점수를 줄 때 따져보는 5가지:

| 기준 | 가중치 | 평가 질문 |
|---|:---:|---|
| **인시던트 첫 응답 (MTTR)** | 30% | 사고 났을 때 처음 5분에 보는가? |
| **자동 알림 (page)** | 25% | P0 으로 oncall 깨워도 되는가 (FP 가 적은가)? |
| **비즈니스 임팩트** | 20% | 결제·인증 등 핵심 거래 KPI 와 직결되는가? |
| **drill-down 출발점** | 15% | 다른 지표로 내려가는 anchor 인가? |
| **데이터 무결성 / 신뢰** | 10% | 이 지표가 깨지면 다른 지표를 못 믿는가? |

→ 각 메트릭에 5점 척도로 점수 → 가중 평균 → Tier 결정.

```
점수 ≥ 4.0  → Tier S
3.0 ~ 3.9  → Tier A
2.0 ~ 2.9  → Tier B
< 2.0      → Tier C
```

---

## 3. Tier 별 메트릭 정렬 — 사용 빈도 순

### 3.1 Tier S — 항상 화면, P0 알림 (7개)

| 순위 | 메트릭 | 점수 | 사용 빈도 | 왜 Tier S? |
|:--:|---|:--:|---|---|
| 1 | **M-S1 Availability** | 4.9 | 모든 화면 | SLO 의 numerator. 이거 깨지면 모든 정의가 흔들림 |
| 2 | **M-S3 Error Rate** | 4.8 | 모든 화면 | 이상 감지 1순위. P0 alert anchor |
| 3 | **M-S4 Latency p95/p99** | 4.7 | 모든 화면 | UX = latency. SLO 와 짝 |
| 4 | **M-P10 Pipeline Lag** | 4.5 | 항상 (메타) | 깨지면 위 3개를 못 믿음 — *meta-monitoring* |
| 5 | **M-S2 TPS / RPS** | 4.4 | 모든 화면 | capacity baseline. spike·dropout 즉시 감지 |
| 6 | **M-P1 11 MSA Health Matrix** | 4.3 | 인시던트 첫 화면 | "어느 서비스?" 1초 진단 |
| 7 | **M-D3 Critical svc_id Error Rate** | 4.1 | 매일 + 알림 | 결제/인증 직격 = 비즈니스 손실 |

> 💡 **이 7개 = D-RT1 (SRE Golden Signals 대시보드) + R-P0-1 ~ R-P0-5 alert**. 이걸로 운영의 80% 가 가능.

---

### 3.2 Tier A — 매일 점검 + drill 출발점 (13개)

| 순위 | 메트릭 | 점수 | 언제 보나 |
|:--:|---|:--:|---|
| 8 | **M-S10 Error Budget Burn Rate** | 3.9 | 주간 SLO 보고. 30일 rolling — 신뢰 핵심 |
| 9 | **M-P5 Inter-Tier Latency** | 3.8 | 인시던트 drill: "PT? BT? MCI?" |
| 10 | **M-S8 G/W p95** | 3.8 | 외부 의존성 감시. 회귀 직격 |
| 11 | **M-S9 MCI core p95** | 3.8 | 코어 의존성. 결제 실패 첫 신호 |
| 12 | **M-P2 Tier Health Matrix** | 3.7 | drill: "PT 인가 BT 인가?" |
| 13 | **M-P4 Stuck Requests** | 3.6 | silent failure detection. 누락 알림 |
| 14 | **M-P3 In/Out Imbalance** | 3.5 | 로그 무결성. 깨지면 통계 무효 |
| 15 | **M-D9 fir_c 분포 (root cause)** | 3.4 | 인시던트 retrospective 의 1번 차트 |
| 16 | **M-D1 Top msg_c** | 3.3 | 일일 점검. backlog 청소 |
| 17 | **M-P8 Container Error Rate** | 3.3 | "단일 pod 인가 전체인가?" |
| 18 | **M-P7 Top svc_id by Error** | 3.2 | 어느 endpoint 가 nuisance 인가 |
| 19 | **M-S5 Slow Request Rate** | 3.1 | p99 보완. ratio 가 직관적 |
| 20 | **M-U2 채널별 Error Rate** | 3.0 | 단일 채널 painPoint 즉시 감지 |

> **Tier A 추가 시 dashboard**: D-RT2 (MSA 매트릭스) + D-RT3 (Tier matrix) + D-D1 (일일 점검).

---

### 3.3 Tier B — 주간 KPI · 도메인 분석 (18개)

| 메트릭 | 사용 시나리오 | 빈도 |
|---|---|---|
| **M-S6** Saturation (ctnr_nm 부하 분포) | 주간 capacity review | 주 |
| **M-S7** Tier 별 Availability | SLA 보고 | 주 |
| **M-P6** Top svc_id by Traffic | capacity 우선순위 | 주 |
| **M-P9** Container Throughput 분포 | 부하 분산 검증 | 주 |
| **M-P11** Inter-Service Call Success | 의존성 건강도 | 주 |
| **M-D2** 신규 msg_c | 새 결함 조기 발견 (alert 가능) | 주 (P1) |
| **M-D4** MSA × msg_c heatmap | 책임 영역 명확화 | 주 |
| **M-D5** 결제 Funnel (PY) | 비즈니스 핵심 KPI | 주 |
| **M-D6** 인증 성공률 (MB) | 보안/UX KPI | 주 |
| **M-D8** 이체 성공률 (FA) | 핵심 거래 KPI | 주 |
| **M-D11** 동시 활성 거래 | concurrency 추세 | 주 |
| **M-U1** 채널별 Traffic Share | 모바일 vs 웹 추세 | 주 |
| **M-U6** 기기 모델 painPoint | 특정 기기 한정 버그 | 일/주 |
| **M-U8** 신 앱 버전 회귀 | release 안정성 (P1 alert) | 주 |
| **M-U11** DAU/WAU/MAU | 경영/PM 핵심 KPI | 일/주/월 |
| **M-O3** Dead svc_id | endpoint 청소 (분기) | 분기 |
| **M-O4** Shadow svc_id | 미선언 endpoint 발견 | 월 |
| **M-O5** DoD/WoW Traffic Change | 추세 비교 (Platinum ML 추천) | 일/주 |

---

### 3.4 Tier C — 월/분기, niche (14개)

| 메트릭 | 사용 시나리오 |
|---|---|
| **M-P12** 호출 hop 분포 | 분기. N+1 패턴 발견 |
| **M-D7** 카드 거래 추세 (CD) | 월/분기. 비즈니스 보고 |
| **M-D10** svc_c 별 거래량 추세 | 월. capacity 예측 |
| **M-D12** biz_c 별 분포 | 분기. 도메인 painPoint |
| **M-U3** 채널별 Latency 비교 | 분기. UX 정밀 분석 |
| **M-U4** OS 별 분포 | 분기. iOS vs Android |
| **M-U5** OS 버전별 Error Rate | 호환성 이슈 발생 시 |
| **M-U7** 앱 버전 분포 | 강제 업데이트 의사결정 |
| **M-U9** 화면별 Funnel (scrn_id) | UX 개선 분기 리뷰 |
| **M-U10** 사용자 retry 패턴 | 분기. 이탈 위험 신호 |
| **M-U12** IP 대역 분포 | 보안 / 비정상 트래픽 |
| **M-O1** Time-of-Day Pattern | 분기 / capacity 계획 |
| **M-O2** Day-of-Week Pattern | 분기 |
| **M-O6** Peak RPS + 시간대 | 분기 |
| **M-X1** Pipeline Stage Lag | 운영팀 항상. 별도 dashboard (D-O1) |
| **M-X2** Index Storage Health | 월. capacity 예측 |
| **M-X3** doc_sz 분포 | 월. 비용 |
| **M-X4** Schema Stability | 월. mapping 폭발 감시 |

> Tier C 의 상당수는 **on-demand** — 평소엔 안 보다가 분기 review / 이슈 발생 시 만든다.

---

## 4. 용량 산정

### 4.1 기본 가정 (회사 환경)

| 가정 | 값 | 근거 |
|---|---|---|
| 일 raw doc 수 | **1억 docs/day** | 11 MSA × 평균 약 1만 TPS |
| 평균 doc 크기 | **1.5 KB** | data 필드 ~500B + 인덱싱 필드 1KB |
| Replica | **× 2** (1 primary + 1 replica) | HA |
| Raw 일 저장량 | **150GB × 2 = 300GB/일** | 1억 × 1.5KB × 2 |
| Hot retention | 30일 | ILM hot |
| Hot 합계 | **9TB** | 300GB × 30 |

> 실제 운영 환경 수치로 대체 필요. 회사 환경에서 [_cat/indices?bytes=gb](http://localhost:5601/app/dev_tools#/console) 로 검증.

### 4.2 메트릭의 3가지 저장 유형

```
┌─────────────────────────────────────────────────────────────────┐
│ Type 1. Raw Query (저장 추가 = 0)                                │
│   Lens 가 raw 인덱스에서 직접 query.                              │
│   → 추가 저장 없음. 단점: query 부하.                             │
├─────────────────────────────────────────────────────────────────┤
│ Type 2. Transform 사전 집계 (소량)                                │
│   5분/1시간 단위 aggregated index.                               │
│   → 인덱스 1개당 ~수~수십 MB/일.                                  │
├─────────────────────────────────────────────────────────────────┤
│ Type 3. ML / Watcher 결과 인덱스                                  │
│   ML anomaly score, watcher history.                            │
│   → 인덱스 1개당 ~1~수 GB/월. Platinum.                           │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 5 Transform 의 용량 산정

| Transform | 출력 docs/일 | 평균 doc 크기 | 일 저장 (×2 replica) | 30일 |
|---|---:|---:|---:|---:|
| `latency-5m` (S4 by svc_c) | 11 svc × 288 buckets = **3.2K** | ~500B | **3.2MB** | **96MB** |
| `errors-container-5m` (P8 by ctnr_nm) | ~50 ctnr × 288 = **14K** | ~400B | **11MB** | **330MB** |
| `tier-latency-5m` (P5 by tier) | 3 tier × 288 = **0.9K** | ~600B | **1MB** | **30MB** |
| `svc-biz-1h` (D5/D6/D8 funnel) | 11 svc × 24 × ~10 biz_c = **2.6K** | ~700B | **3.6MB** | **108MB** |
| `user-channel-1h` (U1/U2/U3) | 4 chan × 3 os × 24 = **288** | ~400B | **0.2MB** | **6MB** |
| **합계** | | | **~19MB/일** | **~570MB/30일** |

> **결론**: Transform 5개의 30일 저장량 ≈ **570MB**. raw 9TB 대비 **0.006%**. 거의 무시 가능.
> 단, query 부하는 **70%↓** ([Q-05](99-qna.md#q-05)) — Transform 의 진짜 가치는 용량이 아닌 *query 응답시간*.

### 4.4 메트릭별 용량 델타 — 한 표

> 추가 저장 = `0` 이면 raw query 직접. Transform 사용 메트릭만 비용 발생.

#### Tier S (7개)

| 메트릭 | 저장 유형 | 추가 저장/일 | Transform | Platinum 영향 |
|---|---|---:|---|---|
| M-S1 Availability | Raw query | 0 | — | 동일 |
| M-S3 Error Rate | Raw query | 0 | — | 동일 |
| M-S4 Latency p95/p99 | Transform | ~3.2MB | latency-5m | 동일 (Basic 충분) |
| M-P10 Pipeline Lag | Raw query | 0 | — | 동일 |
| M-S2 TPS / RPS | Raw query | 0 | — | 💎 ML anomaly 권장 (M-O5 통합) |
| M-P1 11 MSA Health Matrix | Transform | ~3.2MB | latency-5m 재활용 | 동일 |
| M-D3 Critical svc_id Error Rate | Raw query | 0 | — | 동일 |
| **소계** | | **~6.5MB** | | |

#### Tier A (13개)

| 메트릭 | 저장 유형 | 추가 저장/일 | Transform | Platinum 영향 |
|---|---|---:|---|---|
| M-S10 Error Budget | Raw query | 0 | — | 동일 |
| M-P5 Inter-Tier Latency | Transform | ~1MB | tier-latency-5m | 동일 |
| M-S8 G/W p95 | Transform | (latency-5m 일부) | — | 동일 |
| M-S9 MCI p95 | Transform | (latency-5m 일부) | — | 동일 |
| M-P2 Tier Health | Transform | (tier-latency-5m 재활용) | — | 동일 |
| M-P4 Stuck Requests | Raw query (scripted) | 0 | — | 동일 |
| M-P3 In/Out Imbalance | Raw query | 0 | — | 동일 |
| M-D9 fir_c 분포 | Raw query | 0 | — | 동일 |
| M-D1 Top msg_c | Raw query | 0 | — | 동일 |
| M-P8 Container Error Rate | Transform | ~11MB | errors-container-5m | 동일 |
| M-P7 Top svc_id by Error | Raw query | 0 | — | 동일 |
| M-S5 Slow Request Rate | Raw query | 0 | — | 동일 |
| M-U2 채널별 Error Rate | Transform | (user-channel-1h 일부) | — | 동일 |
| **소계** | | **~12MB** | | |

#### Tier B (18개)

| 메트릭 | 저장 유형 | 추가 저장/일 | Platinum 영향 |
|---|---|---:|---|
| M-S6, M-S7 | Raw query / Transform | 0~1MB | 동일 |
| M-P6, M-P9, M-P11 | Raw query | 0 | 동일 |
| M-D2 신규 msg_c | Raw query (set diff) | 0 | 💎 **ML rare event detection** 정확도 ↑ |
| M-D4, M-D5, M-D6, M-D8, M-D11 | Raw / Transform | ~3.6MB (svc-biz-1h) | 동일 |
| M-U1, M-U6, M-U11 | Raw / Transform | ~0.2MB (user-channel-1h) | 동일 |
| M-U8 신 앱 회귀 | Raw query (timeshift) | 0 | 💎 **ML population analysis** 권장 |
| M-O3, M-O4 (Dead/Shadow) | Cross-DB query (외부 + ES) | 0 | 동일 |
| M-O5 DoD/WoW | Raw query (timeshift) | 0 | 💎 **ML Anomaly Detection** 강력 권장 |
| **소계** | | **~5MB** | |

#### Tier C (14개)

| 메트릭 | 저장 유형 | 추가 저장/일 | Platinum 영향 |
|---|---|---:|---|
| M-P12, M-D7, M-D10, M-D12 | Raw query | 0 | 동일 |
| M-U3, M-U4, M-U5, M-U7, M-U9, M-U10, M-U12 | Raw query | 0 | 동일 |
| M-O1, M-O2, M-O6 | Raw query | 0 | 동일 |
| M-X1 Pipeline Stage Lag | Raw query (takes.*) | 0 | 동일 |
| M-X2 Index Storage Health | _cat API | 0 | 💎 **Searchable Snapshots** = 90% 비용 ↓ |
| M-X3 doc_sz 분포 | Raw query | 0 | 동일 |
| M-X4 Schema Stability | _mapping snapshot | <1MB (스냅샷) | 동일 |
| **소계** | | **~1MB** | |

### 4.5 총 합계

| 항목 | 일 저장 | 30일 | raw 대비 |
|---|---:|---:|---:|
| Raw 인덱스 (1억 docs) | 300GB | **9TB** | 100% |
| Transform 5개 합계 | 19MB | **570MB** | **0.006%** |
| ML 결과 (Platinum, ML job 5개 가정) | ~50MB | **1.5GB** | **0.017%** |
| Watcher history | ~10MB | **300MB** | **0.003%** |
| Kibana dashboard saved objects | 무시 | <100MB | <0.001% |
| **52 메트릭 추가 저장 합계** | **~80MB** | **~2.5GB** | **~0.03%** |

> **결론**: 메트릭 산출에 필요한 **추가 저장 비용은 사실상 무시 가능**. 진짜 비용은 raw 인덱스 자체. 따라서 의사결정은 *어느 메트릭이 가치 있나* 에 집중하면 됨.

---

## 5. Platinum 라이선스 영향 — 메트릭별

### 5.1 결론 박스

```
┌────────────────────────────────────────────────────────────────┐
│ ❌ Platinum 이 *반드시 필요한* 메트릭 = 0개                      │
│ ✅ Basic 으로 52 메트릭 전부 구현 가능                            │
├────────────────────────────────────────────────────────────────┤
│ 💎 Platinum 으로 *정확도/자동화 향상* 메트릭 = 5개                │
│   M-S2, M-D2, M-O5, M-U8 (ML Anomaly)                          │
│   M-X2 (Searchable Snapshots)                                  │
└────────────────────────────────────────────────────────────────┘
```

### 5.2 Platinum 활용 5가지 + 메트릭 매핑

| Platinum 기능 | 적용 메트릭 | Basic 대안 | 가치 |
|---|---|---|---|
| **ML Anomaly Detection** | M-O5 DoD/WoW · M-S2 TPS · M-D2 신규 msg_c | threshold alert / set-diff query | 시간대×요일×채널 *자연 패턴* 학습 → FP ↓ |
| **ML Population Analysis** | M-U8 신 앱 버전 회귀 · M-U6 기기 painPoint | timeshift % 비교 | 다변량 정상범위 자동 산출 |
| **Searchable Snapshots** | M-X2 index storage · ILM cold/frozen | s3 backup + restore | object storage 90% 비용 ↓ |
| **Field-Level Security (FLS)** | (메트릭 산출 자체엔 영향 X) | role 별 인덱스 분리 | data 필드 PII role-based 차폐 |
| **Cross-Cluster Replication (CCR)** | DR Active-Active | 별도 backup 파이프라인 | 다중 데이터센터 |

> 자세한 라이선스 활용 시나리오 → [Q-03](99-qna.md#q-03), 인프라 부담 → [Q-04](99-qna.md#q-04).

### 5.3 Platinum ML 우선 적용 권장 (사내 운영)

가장 ROI 가 좋은 ML 적용 4가지:

#### ML #1. M-O5 DoD/WoW Anomaly (Multi-metric job)

```yaml
job:
  name: msa-traffic-anomaly
  detectors:
    - high_count by svc_c          # MSA 별 traffic spike
    - high_mean(proc_tm) by svc_c  # latency 회귀
  influencers: [chan_c, os_kdc, ap_ver]
  bucket_span: 5m
```

**가치**: "어제 오후 3시 BU 가 평소 대비 +40% 트래픽" 을 **수동 threshold 없이** 자동 감지. 시간대×요일×채널 자연 패턴 학습.

**Basic 대안**: timeshift query 로 "어제 같은 시간 vs 오늘" % 비교 + 정적 threshold (±20%).
- 단점: 광주 행사 같은 정상 spike 도 alert. FP 많음.

#### ML #2. M-D2 신규 msg_c Rare Event

```yaml
job:
  name: new-msg-code-detection
  detectors:
    - rare by msg_c
  bucket_span: 1h
```

**가치**: "오늘 처음 본 msg_c E1234" 를 자동 alert.

**Basic 대안**: "지난 7일 unique(msg_c)" set 과 today set diff. 동작은 하지만 ML 이 더 정밀.

#### ML #3. M-U8 신 앱 버전 회귀 (Population Analysis)

```yaml
job:
  name: app-version-regression
  detectors:
    - high_mean(proc_tm) by ap_ver over chan_c
  bucket_span: 1h
```

**가치**: "ap_ver 4.5.1 만 p95 +50%" 자동 발견.

**Basic 대안**: 신 ap_ver 배포 후 24h timeshift 비교 + threshold alert.

#### ML #4. M-S2 TPS Pattern

```yaml
job:
  name: tps-pattern
  detectors:
    - low_count          # dropout 감지
    - high_count         # spike 감지
  bucket_span: 1m
```

**가치**: 새벽 2시 "트래픽 갑자기 0" 같은 *느린 장애* 를 정적 threshold 보다 빨리 감지.

**Basic 대안**: 정적 RPS 하한선 alert. 시간대 변동 못 따라감.

### 5.4 Platinum 도입 의사결정 (재무 관점)

```
┌────────────────────────────────────────────────────────────────┐
│ Platinum = Basic 가격의 약 2배 (Elastic 공식)                    │
├────────────────────────────────────────────────────────────────┤
│ 추가 비용 정당화 조건:                                            │
│  ✅ 인시던트 FP 가 oncall 피로의 30%+                             │
│  ✅ Cold tier 저장 비용이 hot 의 2배 이상 차지                    │
│  ✅ DR (CCR) 가 컴플라이언스 요구사항                             │
│  ✅ data 필드 PII role-based 차폐 (FLS) 가 보안 요구              │
└────────────────────────────────────────────────────────────────┘
```

> 본 09 문서의 모든 메트릭은 **Basic 으로 100% 구현** — Platinum 결정은 별도 ROI 분석 후 분기 review 에서.

---

## 6. 도입 로드맵 — Tier 기준

```
Week 1-2  ━━━━━━━━━━━━━━━━━━ Tier S 7개 + D-RT1 + R-P0-1~5
Week 3-4  ━━━━━━━━━━━━━━━━━━ Tier A 13개 + D-RT2/3 + D-D1 + R-P1-1~6
Week 5-8  ━━━━━━━━━━━━━━━━━━ Tier B 18개 + D-D2 + D-D3 + D-K1
Quarter+  ━━━━━━━━━━━━━━━━━━ Tier C 14개 + D-K2 + D-O1
Quarter+  ━━━━━━━━━━━━━━━━━━ 💎 Platinum 도입 검토 (ML 4 jobs)
```

### 6.1 Week 1-2 — Tier S 만들기

```yaml
deliverables:
  - dashboard: D-RT1 (Golden Signals)
  - transform: latency-5m
  - alerts: R-P0-1, R-P0-2, R-P0-3, R-P0-4, R-P0-5
  - oncall_runbook: R-P0 5개 대응 절차
acceptance:
  - oncall 5분 내 D-RT1 에서 사고 1차 진단 가능
  - 모든 P0 alert 의 FP 비율 < 10%
```

### 6.2 Week 3-4 — Tier A 추가

```yaml
deliverables:
  - dashboard: D-RT2 (MSA Matrix), D-RT3 (Tier Matrix), D-D1 (일일 점검)
  - transform: errors-container-5m, tier-latency-5m
  - alerts: R-P1-1 ~ R-P1-6
acceptance:
  - 인시던트 후 retro 에 D-D1 의 Top msg_c / fir_c 활용
  - drill: D-RT1 → D-RT2 → D-RT3 → D-D2 자연 흐름
```

### 6.3 Week 5-8 — Tier B (도메인/사용자)

```yaml
deliverables:
  - dashboard: D-D2 (도메인 에러), D-D3 (사용자/UX), D-K1 (주간 KPI)
  - transform: svc-biz-1h, user-channel-1h
  - alerts: R-P1-7 ~ R-P1-12
acceptance:
  - 매주 월요일 D-K1 으로 매니저 보고
  - PM 이 D-D3 에서 채널/기기/버전 분석 self-service
```

### 6.4 Quarter+ — Tier C + Platinum 검토

```yaml
deliverables:
  - dashboard: D-K2 (월간 비즈니스), D-O1 (인프라 헬스)
  - capacity_review: M-X2 + M-X3
  - platinum_decision: 💎 ML 4 jobs ROI 분석
acceptance:
  - 월간 capacity report 자동 생성
  - Platinum 도입 가/부 분기 review 에서 결정
```

---

## 7. 자가 점검 — 우선순위가 맞나?

다음 질문에 모두 "네" 면 우선순위가 팀 현실에 맞는 것:

- [ ] Tier S 7개로 어제 인시던트 1차 진단이 가능했나?
- [ ] Tier A 의 어느 것을 만들지 결정할 때 *이전 인시던트 retro* 가 근거였나?
- [ ] Tier B 진입 *전에* Tier A 가 6주 이상 안정 운영됐나?
- [ ] Tier C 의 어느 것이 "만들었지만 1분기 안 봤다" 인가? (있으면 archive 검토)
- [ ] Platinum 도입 의사결정이 *메트릭 정확도 향상* 이 아닌 *FP 감소 / 비용 절감* 으로 정당화되나?

---

## 8. 부록 — 가정과 계산식

### 8.1 doc 평균 크기 1.5KB 산정

```
인덱싱 필드 ~30개 × 평균 30B (string/keyword)   ≈  900B
@timestamp + meta (id, _source pointer)          ≈  100B
data 필드 (free-form, store_only)                ≈  500B (지금 운영 평균)
                                                 ─────────
                                                 ≈ 1500B = 1.5KB
```

회사 환경에서 검증:
```bash
GET _cat/indices/logs-msa-*?v&h=index,store.size,docs.count
# size_in_bytes / docs.count = 평균 doc 크기
```

### 8.2 Transform 출력 doc 크기 산정

Transform output schema (예: `latency-5m`):
```json
{
  "@timestamp": "2026-04-30T12:00:00Z",   // 25B
  "svc_c": "BU",                          //  8B
  "count": 12450,                         // 10B
  "p50": 84.3, "p95": 421.7, "p99": 1230, // 30B
  "error_count": 12,                      // 10B
  ...
}
                                       총 ≈ 300~600B
```

### 8.3 Cardinality 영향

`user-channel-1h` 출력량 = `chan_c × os_kdc × hour buckets`:
```
4 chan_c × 3 os_kdc × 24 buckets  = 288 docs/일
288 × 400B × 2 replica           = 230KB/일 ≈ 7MB/30일
```

→ **dimension 추가 시 곱셈 효과**. `device_model` 추가하면 (1000 모델 × …) 폭발. 그래서 Tier C 의 M-U6 는 raw query 또는 Top-N 만.

---

## 9. 함께 보기

- [09. MSA 종합 KPI 전략](09-monitoring-strategy.md) — 52 메트릭 원본 정의
- [09a. 실 운영 필드 매핑](09a-real-field-mapping.md) — 필드 cheatsheet
- [Q-03 Platinum 기능](99-qna.md#q-03)
- [Q-04 ML 인프라 부담](99-qna.md#q-04)
- [Q-05 Transform = 사전집계](99-qna.md#q-05)
- [99-tier-tracing.md](99-tier-tracing.md) — Inter-Tier Latency 기반
