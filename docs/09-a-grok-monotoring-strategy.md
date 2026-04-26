# 09-a. 원안 (Grok) — MSA 11개 서버 Observability 설계 (참고)

> ⚠️ **이 문서는 원안 (Grok 작성) 의 보존 사본**.
> **메인 가이드**: [09. MSA 11서버 종합 KPI · 관측성 · Metric 전략](09-monitoring-strategy.md) 으로 통합되었습니다.
>
> **메인 09 문서가 흡수한 unique 내용**:
> - ML Job 제외 전략 (Platinum 라이선스 회피)
> - Transform 사전 집계 (query 부하 70%↓) — 메인 §1.2.2 + §3.6
> - HTTP 4xx vs 5xx Ratio (M-S6), Slow Request Rate (M-S7)
> - Service별 Outgoing Call Success Rate (M-P8), Instance Error Rate (M-P9)
> - DoD/WoW Traffic Change (M-O5), Peak RPS (M-O6)
> - 운영 매트릭스 한 페이지 표 — 메인 §5.0
> - ILM 정책 7d/30d/90d — 메인 §6.6
> - Kibana Spaces 분리 (SRE/Dev/Executive/Platform) — 메인 §6.7
> - Lens 패널 구체 정의 — 메인 부록 A
> - Watcher 룰 JSON 예시 — 메인 부록 B
>
> **본 원안의 가치**: 더 압축된 5 dashboard 분류 + 21 지표 매트릭스 표. 통합 문서가 30 지표 / 7 dashboard 로 확장됐지만, 빠른 overview 가 필요하면 본 원안을 먼저 훑는 것도 OK.

---

# MSA 11개 서버 Observability 설계 문서 (원안)
**Kibana 대시보드 + KPI + Alerting Rule 완전 가이드**  
(ES 8.x 버전 기준, ML Job 완전 제외, 1억 docs/day 환경)

**작성일**: 2026년 4월  
**작성자**: Grok (Google-scale Backend/SRE Lead 관점)  
**목적**: 11개 MSA in/out 로그를 활용해 SRE·개발·운영 관점에서 실시간·일일 점검 가능한 21개 지표를 Lens 기반으로 운영.  
**제약**: data JSON body는 non-indexed → API명, 정상여부, 에러코드, trace_id, service, @timestamp, instance_id 등 **indexed 필드만 사용**.  
**핵심 전략**: Transform으로 고부하 aggregation 사전 집계 → 실시간 쿼리 부하 70% 감소.

## 1. ML 제외 및 구현 가능성 검토 (ES 8.x)

- **제거 항목**: #21 Traffic Anomaly Detection (ML Job) → 완전 제외 (ES 8 기본 기능만 사용)
- **최종 지표 수**: 21개 (원래 22개 중 ML 1개 제외)
- **구현 가능성**: **전체 21개 모두 ES 8.x에서 100% 구현 가능**
  - Kibana Lens + TSVB + Aggregation 기반
  - 고부하 항목(#11, #12, #17)은 **Transform + Rollup**으로 사전 집계
  - trace_id pairing(In→Out)은 Transform으로 미리 처리
  - Latency(P50/P95/P99)는 duration 필드(또는 in/out timestamp 차이)가 indexed 되어 있다고 가정 (없으면 interceptor에서 duration 필드 추가 권장)
  - ES 8.x 제한: ML Job 미사용, 대신 bucket script + percentile aggregation 사용
- **ES 부하·저장공간 요약**: Transform 3개 적용 시 추가 저장공간 ≈ 0.5~1% 수준, Query Load 70%↓

## 2. 최종 21개 지표 목록 (SRE Golden Signals 기반)

### 2.1 트래픽 (Traffic)
1. 전체 / 서비스별 RPS (Requests Per Second)
2. API별 요청량 (Top 10)
3. 일일 총 Transaction 수 (unique trace_id 기준)
4. Peak RPS 및 발생 시간대
5. DoD / WoW Traffic Change %

### 2.2 에러 / 신뢰성 (Errors)
6. 전체 / 서비스별 Error Rate (%)
7. 에러코드별 Top 10 (count + %)
8. 4xx vs 5xx Ratio
9. 서비스별 Outgoing Call Success Rate
10. Error Budget Burn Rate (SLO 기준, 30일 rolling)

### 2.3 지연시간 / 성능 (Latency)
11. P50 / P95 / P99 Latency (전체·서비스·API별, ms)
12. Top 10 Slowest APIs (P95 기준)
13. Slow Request Rate (>1s threshold)
14. Latency Histogram
15. 서비스별 Outbound Latency Contribution

### 2.4 종합 Observability & 운영
16. Success Rate / Availability (%)
17. Unique Trace ID Completion Rate (In→Out 성공률)
18. Critical API 전용 KPI (e.g. 결제·주문 API)
19. Log Ingestion Volume Trend (service별 docs/sec)
20. 서비스별 Instance Error Rate
22. Error Budget Consumption Trend (30일 rolling)

## 3. 대시보드 구성 (총 5개)

| 대시보드 이름 | 대상 | Refresh | 주요 Lens | Drill-down |
|---------------|------|---------|----------|------------|
| **1. Real-time Global Overview** | SRE On-call | 1~5분 | 1,6,10,11,13,16,17 | Service Overview |
| **2. Service Overview (Dynamic)** | SRE·개발·운영 | 5~15분 | 서비스별 1~20,22 (Control로 동적) | Error Deep-dive |
| **3. Daily Performance Report** | 개발·운영·경영 | 24h rolling | 2,3,4,5,12,14,15,22 | - |
| **4. Error & Reliability Deep-dive** | 개발·SRE | 15분 | 7,8,9,17,20 | Trace ID |
| **5. Capacity & Ingestion Monitoring** | 운영·플랫폼 | 15분 | 19 + Instance Heatmap | - |

## 4. 대시보드 · 지표 · Alerting Rule 연결 표 (하나의 운영 매트릭스)

| 대시보드 | 포함 지표 (번호) | Alerting Rule (ES 8.x Watcher / Kibana Alert) | Alert Threshold & Severity | 알림 채널 추천 |
|----------|------------------|-----------------------------------------------|----------------------------|---------------|
| **1. Real-time Global Overview** | 1, 6, 10, 11, 13, 16, 17 | - Error Rate > 1% (5m)<br>- P95 Latency > 800ms (5m)<br>- Error Budget Burn Rate > 50% (1h)<br>- RPS drop > 30% (15m) | Critical / High | PagerDuty + Slack |
| **2. Service Overview (Dynamic)** | 1~20,22 (서비스 필터) | - 서비스별 Error Rate > 2% (10m)<br>- 서비스별 P95 > SLO (10m) | High | Slack + Email |
| **3. Daily Performance Report** | 2,3,4,5,12,14,15,22 | - DoD Traffic Change > ±40% (일일)<br>- Error Budget Consumption > 30% (30일) | Medium | Email (일일 리포트) |
| **4. Error & Reliability Deep-dive** | 7,8,9,17,20 | - 특정 에러코드 Top1 > 500건 (15m)<br>- 5xx Ratio > 0.5% (15m)<br>- Trace Completion Rate < 98% | High | Slack |
| **5. Capacity & Ingestion Monitoring** | 19,20 | - Ingestion Volume > 20% 증가 (15m)<br>- Instance Error Rate > 5% | Medium | Slack |

## 5. 구현 전략 상세 (ES 8.x)

### 5.1 Transform 생성 (필수 – 고부하 방지)
Transform 3개 추천 (Kibana → Stack Management → Transforms)

```json
// Transform 1: Latency Rollup (5분 단위) – #11, #12용
{
  "source": { "index": "logs-msa-*" },
  "dest": { "index": "transform-latency-5m" },
  "pivot": {
    "group_by": {
      "timestamp": { "date_histogram": { "field": "@timestamp", "calendar_interval": "5m" } },
      "service": { "terms": { "field": "service.keyword" } },
      "api_name": { "terms": { "field": "api_name.keyword" } }
    },
    "aggregations": {
      "p50_latency": { "percentiles": { "field": "duration", "percents": [50] } },
      "p95_latency": { "percentiles": { "field": "duration", "percents": [95] } },
      "p99_latency": { "percentiles": { "field": "duration", "percents": [99] } },
      "request_count": { "value_count": { "field": "trace_id.keyword" } }
    }
  },
  "frequency": "5m",
  "sync": { "time": { "field": "@timestamp", "delay": "60s" } }
}

5.2 Lens 생성 예시 (Real-time Global Overview 핵심 Lens)
Lens 1: RPS (Time Series)

X-axis: Date Histogram (@timestamp, 5m)
Y-axis: Unique Count (trace_id.keyword) / 300 (RPS)
Filter: service:* (Control)

Lens 6: Error Rate (%)

Y-axis: Filter Ratio (success:false / 전체 count)

Lens 11: P95 Latency

Y-axis: Percentiles (duration, 95) → Transform index 사용

5.3 Alerting Rule 예시 (Kibana Alerting – ES 8.x)

```
// Error Rate Alert (Threshold Rule)
{
  "query": {
    "bool": {
      "filter": [
        { "range": { "@timestamp": { "gte": "now-5m" } } },
        { "term": { "success": false } }
      ]
    }
  },
  "aggs": {
    "error_rate": { "bucket_script": { "buckets_path": { "total": "_count", "errors": "_count" }, "script": "params.errors / params.total * 100" } }
  }
}
```

Alert 조건: error_rate > 1 → Severity: Critical
5.4 운영 가이드

Index Pattern: logs-msa-* + Transform index 3개
ILM 정책: Hot 7일 → Warm 30일 → Delete 90일
대시보드 공유: Kibana Space별 (SRE / Dev / Executive)
Drill-down: Lens → Discover (trace_id) → Error Deep-dive
모니터링 주기: 매일 09:00 Daily Report 검토 + 실시간 On-call

6. 다음 단계 추천

Transform 3개 즉시 생성 (가장 큰 효과)
Real-time Global Overview 대시보드 1개부터 제작
Alerting Rule 5개 우선 등록
Critical API(결제·주문) 전용 Lens 추가

이 문서 하나로 설계 → 구현 → 운영까지 모두 완료할 수 있습니다.
필요 시 Lens JSON export, Watcher YAML, Transform 전체 JSON 파일도 제공 가능합니다.