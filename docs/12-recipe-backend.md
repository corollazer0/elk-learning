# 12. 백엔드 / 플랫폼 레시피 (M-P1 ~ M-P12)

> **선수**: [11. SRE 레시피](11-recipe-sre.md) — 가용성/지연 기본
>
> 백엔드 플랫폼 엔지니어 관점의 12 메트릭. **MSA 매트릭스, 컨테이너, In/Out 짝짓기, Tier latency** 가 핵심.

> ⏱️ **시간 범위 안내**: 메트릭 제목 옆은 **권장 default**. 실습 데이터셋 (2026-04-01~30) 에선 우상단 시간 픽커 → "Absolute" → `2026-04-01 00:00` ~ `2026-04-30 23:59` 로 입력.

---

## 학습 순서 추천

```
1. M-P1 11 MSA Health Matrix     ← 인시던트 첫 화면
2. M-P10 Pipeline Lag             ← 데이터 신뢰성 (P0!)
3. M-P2 Tier Health Matrix
4. M-P5 Inter-Tier Latency
5. M-P3 In/Out Imbalance
6. M-P4 Stuck Requests
7. M-P8 Container Error Rate
8. M-P7 Top svc_id by Error
9. M-P6 Top svc_id by Traffic
10. M-P9 Container Throughput 분포
11. M-P11 Inter-Service Call Success
12. M-P12 호출 Hop 분포
```

---

## M-P1. 11 MSA Health Matrix — ⏱️ Last 24 hours

**한 줄 정의**: 11 svc_c × {availability, p95, tps, error count} 의 단일 표

**왜 보는가?**
- 인시던트 시 *어느 서비스인지* 를 1초 진단
- D-RT2 dashboard 의 1번 패널

**Kibana 만들기**:

```
1. Lens → 차트 타입: Table
2. Rows: svc_c (Top values, 11)
3. Metrics (4개 컬럼 추가):
   ① 함수: Formula
      수식: count(kql='sts_c:"OK"') / count(kql='log_div : (PT_OUT or BT_OUT)')
      Display: "Avail %"
      Format: Percent
   ② 함수: Percentile (95) of proc_tm
      KQL: log_div : *_OUT
      Display: "p95 ms"
   ③ 함수: Count
      KQL: log_div : (PT_OUT or BT_OUT)
      Display: "TPS (전체)"
   ④ 함수: Count
      KQL: log_div : *_OUT and sts_c : "ERROR"
      Display: "Errors"
4. Title: "M-P1 11 MSA Health Matrix"
5. Save
```

**해석 가이드**:
- 행 별로 비교 — 1줄만 빨강 / 길면 *그 서비스* 가 문제
- Conditional formatting: error count > 100 cell 을 빨강으로 (Lens 우측 패널 → Color)

**우리 데이터셋에서**:
- Day 22 에 PY 행만 Avail 96~98%, error count spike

**업무 개선 활용**:
- 인시던트 첫 화면 — 5초 진단
- "어느 팀에 ping 할까?" 의 정량 근거

---

## M-P2. Tier Health Matrix (3 tier × KPI) — ⏱️ Last 24 hours

**한 줄 정의**: PT/BT/MCI 각각의 같은 KPI

**왜 보는가?**
- 어느 layer 에서 문제? (M-P1 보다 위 추상화)
- 사내 책임 vs 외부 책임

**Kibana 만들기**:

```
M-P1 와 동일하되 Rows: tier_c (PT/BT/MCI)
Title: "M-P2 Tier Health Matrix"
```

**해석 가이드**:
- MCI 행 만 빨강 = 외부 의존성 문제 (우리 책임 아닐 수 있음)
- BT 만 = 사내 비즈니스 로직 회귀
- PT 만 = 프론트 / 라우팅 문제

---

## M-P3. In/Out Imbalance — ⏱️ Last 24 hours

**한 줄 정의**: |IN 개수 - OUT 개수| / IN 개수 — 0 이면 완벽

**왜 보는가?**
- 모든 IN 에 OUT 짝이 있어야 함
- 짝 없으면 → 누락 / 행 / pipeline 문제 = *통계 왜곡*

**Kibana 만들기**:

```
1. Lens → 차트 타입: Bar horizontal
2. Vertical axis: svc_id (Top 30)
3. Horizontal axis:
   ─ 함수: Formula
   ─ 수식:
     abs( count(kql='log_div:*_IN') - count(kql='log_div:*_OUT') )
     / count(kql='log_div:*_IN')
   ─ 표시: Percent
4. Sort: Descending
5. Title: "M-P3 In/Out Imbalance by svc_id"
6. Save
```

**해석 가이드**:
- 0~0.5% = 정상 (network 지연으로 일부 OUT 은 다음 분 인덱스에)
- > 1% = 누락 의심 — 해당 svc_id drill
- > 5% = pipeline 장애

**우리 데이터셋에서**:
- 의도된 0.2% 누락 — Top 일부 svc_id 에서 보임

**업무 개선 활용**:
- 통계 신뢰도 검증의 1차 관문
- Logstash / Filebeat 헬스 체크

---

## M-P4. Stuck Requests (행 거래) — ⏱️ Last 24 hours

**한 줄 정의**: IN 만 있고 10분+ OUT 안 옴

**왜 보는가?**
- 사용자 입장: "버튼 눌렀는데 응답 안 옴"
- 데드락 / 타임아웃 / 백엔드 hang

**Kibana 만들기** (Lens 단독으로 어려움 — Discover + saved query):

```
1. Discover → msa-logs-*
2. KQL: log_div : *_IN and not _exists_ : "_no_match"  (전체 IN)
3. 시간 범위: Last 24 hours
4. Stack Management → Saved Objects → Lens 로 가능 표현:
   - 함수: Filter aggregation 사용
```

**더 실용적인 방법** — Transform 또는 Watcher:

```yaml
# Transform 'stuck-detection' (07-batch-transform.md 참고)
group_by: guid
aggregations:
  in_count:  filter log_div:*_IN, count
  out_count: filter log_div:*_OUT, count
  first_in:  filter log_div:*_IN, min(@timestamp)
filter (post): out_count == 0 AND first_in < now-10m
```

→ Transform 결과 인덱스 `stuck-requests-*` 를 Lens 로 시각화.

**해석 가이드**:
- 0 = 정상
- 100건 누적 = R-P1-2 alert
- 1000건+ = 장애 — 즉시 P0

**업무 개선 활용**:
- silent failure 발견 — 평균 응답 시간 보다 *훨씬* 늦은 실패
- 프론트 retry / 사용자 이탈과 매핑

---

## M-P5. Inter-Tier Latency (PT→BT, BT→MCI) — ⏱️ Last 24 hours

**한 줄 정의**: 같은 guid 의 두 tier 시간 차

**왜 보는가?**
- 어느 hop 이 bottleneck?
- proc_tm 만으론 *호출 사이 시간* 못 봄

**Kibana 만들기** (Transform 필수):

```yaml
# Transform 'tier-latency-5m' (99-tier-tracing.md §2.2 참고)
group_by:
  ts:   date_histogram(5m)
  guid: terms
aggregations:
  pt_out_ts: filter log_div:PT_OUT, max(@timestamp)
  bt_in_ts:  filter log_div:BT_IN,  min(@timestamp)
  pt_to_bt:  bt_in_ts - pt_out_ts (ms)
```

```
Transform 결과 인덱스 transform-tier-latency-5m 을:
  Lens → Line
  Vertical: percentile(95) of pt_to_bt
  Title: "M-P5 PT→BT p95 Latency"
```

**해석 가이드**:
- 평소 5~50ms (네트워크 + LB)
- 200ms+ 지속 = 네트워크 / 서비스 메시 문제

**우리 데이터셋에서**:
- 의도된 평균 ~5ms (PT 처리 후 BT 인입)

---

## M-P6. Top svc_id by Traffic — ⏱️ Last 7 days

**한 줄 정의**: count(*_OUT) by svc_id, Top 20

**왜 보는가?**
- capacity 우선순위 (가장 부하 큰 endpoint)
- 의외의 hot endpoint 발견

**Kibana 만들기**:

```
1. Lens → Bar horizontal
2. Vertical axis: svc_id (Top 20)
3. Horizontal axis: Count, KQL: log_div : (PT_OUT or BT_OUT)
4. Sort: Descending
5. Title: "M-P6 Top 20 svc_id by Traffic"
6. Save
```

**해석 가이드**:
- 80/20 룰 — Top 5 svc_id 가 80% 트래픽이면 정상
- 갑자기 등장한 hot svc_id = 신 배포 여파 / 특정 캠페인

**업무 개선 활용**:
- 성능 최적화 우선순위 ("이거부터 fix")
- API gateway 캐시 정책 결정

---

## M-P7. Top svc_id by Error — ⏱️ Last 24 hours

**한 줄 정의**: count(ERROR) by svc_id, Top 10

**Kibana 만들기**:

```
M-P6 와 동일하되 KQL: sts_c : "ERROR"
Title: "M-P7 Top 10 svc_id by Error"
```

**해석 가이드**:
- 평소 분포 vs 오늘 분포 *비교* 가 핵심
- 신규 등장한 svc_id 가 가장 위험

**우리 데이터셋에서**:
- Day 22 에 `pay/confirm` (PY) 가 dominant

---

## M-P8. Container Error Rate — ⏱️ Last 24 hours

**한 줄 정의**: error rate by ctnr_nm

**왜 보는가?**
- 단일 pod 장애 vs 전체 서비스 문제 구분
- *다른 pod 은 멀쩡한데 1개만* — 즉시 재시작 의사결정

**Kibana 만들기**:

```
1. Lens → Bar horizontal
2. Vertical axis: ctnr_nm (Top 50)
3. Horizontal axis:
   ─ 함수: Formula
   ─ 수식:
     count(kql='sts_c:"ERROR"') / count()
   ─ 표시: Percent
4. Filter (Lens KQL bar): log_div : *_OUT
5. Sort: Descending
6. Title: "M-P8 Container Error Rate"
7. Save
```

**해석 가이드**:
- 같은 svc 의 pod 들이 비슷한 % 면 정상
- 1 pod 만 5%+ 면 *그 pod* 만 재시작
- 모든 pod 동일 spike = 코드 회귀

**우리 데이터셋에서**:
- **Day 22**: `py-pod-2` 만 에러율 ~3% (다른 py-pod 는 0.3%) — 핵심 시그널
- 시간 범위를 Day 22 로 좁히고 Top 50 → py-pod-2 가 최상단

**업무 개선 활용**:
- pod 재시작 / 격리 의사결정의 정량 근거
- HPA / k8s readiness probe 검증
- R-P1-12 alert source

---

## M-P9. Container Throughput 분포 — ⏱️ Last 24 hours

**한 줄 정의**: count by ctnr_nm

**왜 보는가?**
- 로드밸런서가 균등 분산하는가?

**Kibana 만들기**:

```
M-S6 와 동일하되 svc 별 색상 분리:
  Vertical axis: ctnr_nm
  Horizontal axis: Count
  Breakdown by: svc_c (자동 색상)
Title: "M-P9 Container Throughput by MSA"
```

**해석 가이드**:
- 같은 svc 내 pod 4개가 ±10% 이내 = 정상

---

## M-P10. Pipeline Lag (P0 — 데이터 신뢰) — ⏱️ Last 1 hour

**한 줄 정의**: now() - max(@timestamp) — 마지막 데이터가 얼마나 오래됐나?

**왜 보는가?**
- 이게 깨지면 *모든 다른 메트릭을 못 믿음* (메타 모니터)
- log pipeline (Filebeat → Logstash → ES) 장애 1차 신호

**Kibana 만들기** (TSVB 또는 Lens):

```
1. Lens → Metric
2. Primary metric:
   ─ 함수: Last value of @timestamp
   ─ Display: "마지막 doc 시각"
3. Secondary metric:
   ─ 함수: Formula
   ─ 수식: now() - last_value(@timestamp)
4. Title: "M-P10 Pipeline Lag"
5. Save
```

**해석 가이드**:
- < 10초 = 정상 (5초 refresh 기준)
- 30초+ = 경고
- 5분+ = pipeline 장애 (P0)

**업무 개선 활용**:
- Alert R-P0-5 — *최우선* alert
- 인시던트 회의 시작 전: "데이터는 신뢰 가능한가?" 확인

---

## M-P11. Inter-Service Call Success — ⏱️ Last 7 days

**한 줄 정의**: caller_uuid 기반 callee 의 success rate

**왜 보는가?**
- "PY 가 AS 를 호출할 때 몇 % 성공?" — 의존성 건강도

**Kibana 만들기** (좀 복잡 — Transform 권장):

```yaml
# Transform 'inter-service-1h'
group_by:
  caller_svc: terms (caller's svc_c — runtime field 필요)
  callee_svc: terms (svc_c)
  ts: date_histogram(1h)
aggregations:
  total: count
  ok: filter sts_c:OK
metrics: ok / total
```

**대안**: KQL 만으로 caller-callee 페어 표현이 어려움 — Lens 표현은 *runtime field* 또는 *Transform 결과* 필요.

---

## M-P12. 호출 Hop 분포 — ⏱️ Last 7 days

**한 줄 정의**: 한 guid 의 PT/BT/MCI/GW doc 개수 분포

**왜 보는가?**
- N+1 문제 발견 — 한 거래에서 BT 가 3번 호출되면 비효율
- 비정상 패턴 (예: PT만 있고 BT 없음 = 라우팅 실패)

**Kibana 만들기** (Transform 권장):

```yaml
# Transform 'guid-hop-count'
group_by: guid
aggregations:
  pt_count:  filter tier_c:PT
  bt_count:  filter tier_c:BT
  mci_count: filter tier_c:MCI
  total_hops: pt_count + bt_count + mci_count
```

→ Transform 결과를 Lens histogram 으로 visualize.

**해석 가이드**:
- 정상 분포: 4 hops (PT in/out + BT in/out) — 60%
- 6 hops (+ MCI/GW) — 30%
- 8 hops + (N+1 의심) — 10% 이하

---

## D-RT2: 11 MSA Health Matrix Dashboard

```
┌──────────────────────────────────────────────────────────────┐
│ M-P1 11 MSA Health Matrix (Table, 11 행)                     │
├──────────────────────────────────────────────────────────────┤
│ M-P2 Tier Health (Bar, 3 rows)                               │
├────────────────────────────────┬─────────────────────────────┤
│ M-P6 Top 20 svc_id by Traffic  │ M-P7 Top 10 by Error        │
├────────────────────────────────┴─────────────────────────────┤
│ M-P8 Container Error Rate (Bar horizontal, Top 50)           │
├──────────────────────────────────────────────────────────────┤
│ M-P9 Container Throughput by MSA (grouped)                   │
├──────────────────────────────────────────────────────────────┤
│ M-P3 In/Out Imbalance Top 10  │ M-P4 Stuck Requests count    │
└──────────────────────────────────────────────────────────────┘

Save: "D-RT2 11 MSA Health Matrix"
```

---

## 다음 문서

- [13. 도메인 / 비즈니스 레시피](13-recipe-domain.md) — M-D1~D12
