# ol-proto 중간보고: 진행 상황, 래핑 필요 이유, 단언 정리

**목적**: 중간보고 및 기록.  
**대상**: ol-proto 테스트 스위트( CAT 기본 2종 시나리오 )와 2번 시나리오 단언을 위한 Chainsaw 래핑.

---

## 1. 진행 상황 요약

| 항목 | 내용 |
|------|------|
| 테스트 스위트 | `clusterloader2/ol-proto.yaml` — 시나리오 2개 등록 |
| 시나리오 1 | API 가용성 SLO. config: `testing/proto/proto-01-api-availability-slo.yaml`. **CL2 자체 단언 있음** (threshold 0.99). |
| 시나리오 2 | 파드 간 TCP 네트워크 성능. config: `testing/proto/proto-02-network-performance.yaml`. **CL2 자체 단언 없음** (측정만). |
| 래핑 | Chainsaw 테스트 `chainsaw/ol-proto-02-with-assert.yaml` — CL2 실행 후 report JSON 파싱으로 throughput 하한 단언. |
| 문서 | `chainsaw/README.md` (실행 방법·필요 도구·2번 단언 가능 여부), `docs/MEASUREMENTS.md` (measurement 설명). `ol-proto.md`는 별도로 맨 마지막에 작성 예정. |

### 1.1 네이밍 규칙

- **의미**: `조건_대상_예상결과` (언더바는 3구간 구분만, 구간 내부는 카멜).
- **CL2 제한**: scenario **identifier**에는 언더바 불가(template_provider.go 검증). 따라서 `ol-proto.yaml`에서는 **하이픈** 사용.
  - 1번: `pollWindow-readyz-availabilityAbove99`
  - 2번: `podsReady-podToPodTcp-throughputRecorded`
- **config 내부 `name`**: 언더바 사용 가능. `pollWindow_readyz_availabilityAbove99`, `podsReady_podToPodTcp_throughputRecorded`.
- **GWT**: step **name**에 넣으면 CL2에서 에러 가능 → **주석으로만** 표기.

### 1.2 주요 경로

| 구분 | 경로 |
|------|------|
| 스위트 정의 | `clusterloader2/ol-proto.yaml` |
| 시나리오 config | `clusterloader2/testing/proto/proto-01-api-availability-slo.yaml`, `proto-02-network-performance.yaml` |
| Chainsaw 테스트 | `clusterloader2/chainsaw/ol-proto-02-with-assert.yaml` |
| Chainsaw 설명 | `clusterloader2/chainsaw/README.md` |
| CL2 report (2번) | `report/*NetworkPerformance*.json` — Throughput 값은 `dataItems[].labels.Metric == "Throughput"` 항목의 **`.data.Value`** (대문자 V). |

---

## 2. Chainsaw(또는 Ginkgo 등)로 래핑해야 하는 이유

### 2.1 CL2 measurement별 단언 지원

- **APIAvailability**: config `params`에 `threshold`(float, 예: 0.99) 지원. gather 시 가용률이 threshold 미만이면 **CL2가 실패 반환** (단언).
- **NetworkPerformanceMetrics**: **threshold 미지원**. TCP/UDP/HTTP 측정 후 결과만 수집·report로 출력하고, **pass/fail 판정을 하지 않음**.

따라서 **ol-proto만** 실행하면:

- **1번 시나리오**: 단언됨 (가용률 ≥ 0.99).
- **2번 시나리오**: 단언되지 않음 (throughput 수치만 기록).

### 2.2 2번 시나리오에 단언을 넣는 두 가지 방법

| 방법 | 설명 |
|------|------|
| **A. CL2 코드 수정** | `pkg/measurement/common/network/network_performance_measurement.go`에 `threshold` 파라미터를 추가하고, gather 단계에서 throughput이 threshold 미만이면 `errors.NewMetricViolationError` 반환. (SchedulingThroughput / APIAvailability와 동일 패턴.) |
| **B. 코드 수정 없이 래핑** | CL2는 그대로 두고, **실행 후** report JSON을 읽어 throughput 하한을 검사하는 단계를 **외부 테스트 러너**로 수행. |

**코드 수정을 하지 않기로 한 경우** → **B**만 가능. 이때 테스트 러너로 Chainsaw(또는 Ginkgo 등)를 사용해 “CL2 실행 → report 파싱 → 단언” 순서를 한 번에 실행하도록 **래핑**하는 구조가 필요하다.

### 2.3 정리

- **Chainsaw(또는 Ginkgo 등)로 래핑해야 하는 이유**: CL2의 NetworkPerformanceMetrics는 자체 단언이 없으므로, **CL2 코드를 수정하지 않는 전제** 아래에서는 **실행 결과(report)를 외부에서 검사**해야 2번 시나리오에 단언을 둘 수 있음. 그 “실행 + 검사”를 하나의 테스트로 묶는 역할이 Chainsaw(또는 Ginkgo) 래퍼이다.

---

## 3. 단언 정리

### 3.1 시나리오별 단언 주체

| 시나리오 | 측정 대상 | 단언 내용 | 단언 주체 |
|----------|-----------|-----------|-----------|
| 1번 | API 서버 /readyz 가용률 | 가용률 ≥ threshold (0.99) | **CL2** (APIAvailability measurement) |
| 2번 | 파드 간 TCP throughput | throughput ≥ MIN_THROUGHPUT (기본 1,000,000) | **Chainsaw** (Step 2: report JSON에서 `.data.Value` 추출 후 비교) |

### 3.2 Chainsaw 테스트 단언 상세

- **Step 1**: `./clusterloader --testsuite=ol-proto.yaml --provider=skeleton --report-dir=./report` 실행. 1번·2번 모두 실행되고 `report/`에 JSON 저장.
- **Step 2**:
  - `report/*NetworkPerformance*` JSON에서 `dataItems[].labels.Metric == "Throughput"` 인 항목의 **`.data.Value`** 를 jq로 추출.
  - **단언**: `throughput >= MIN_THROUGHPUT`. 기본값 `MIN_THROUGHPUT=1000000`. 미달 시 exit 1로 실패.

실행 결과 통과 확인됨 (Step 2: report 내 Throughput ≥ MIN_THROUGHPUT 단언까지 성공).

### 3.3 실행 명령 요약

**CL2만 실행 (1번만 단언):**

```bash
cd clusterloader2
./clusterloader --testsuite=ol-proto.yaml --provider=skeleton --report-dir=./report
```

**Chainsaw로 2번까지 단언:**

```bash
cd clusterloader2
chainsaw test ./chainsaw --test-file ol-proto-02-with-assert.yaml --exec-timeout 15m
```

---

## 4. Prometheus 활용 (참고)

### 4.1 정해진 메트릭 vs 직접 정의

CL2에서 Prometheus를 쓰는 방식은 두 가지다.

| 구분 | 설명 | 예시 |
|------|------|------|
| **정해진(고정) 메트릭** | measurement별로 쿼리/메트릭이 코드에 고정됨 | APIResponsivenessPrometheus, SchedulingThroughputPrometheus, NodeLocalDNSLatencyPrometheus, NetworkProgrammingLatency, ContainerRestarts, MetricsServerPrometheus 등 |
| **직접 메트릭 정의** | config에서 PromQL·threshold 지정 | **GenericPrometheusQuery** — `params`에 `metricName`, `metricVersion`, `unit`, `queries[]`(각각 `name`, `query`, `threshold`, `lowerBound`)로 사용자 정의 쿼리 실행 및 단언 가능 |

GenericPrometheusQuery: `queries[].threshold` 설정 시 gather에서 쿼리 결과와 비교해, `lowerBound: true`면 값 < threshold 시 실패, 아니면 값 > threshold 시 실패. (`pkg/measurement/common/generic_query_measurement.go`)

### 4.2 네트워크(2번) 처리량을 Prometheus로 CL2만 단언 가능한가?

- **2번이 쓰는 수치**: NetworkPerformanceMetrics는 netperf/iperf로 파드 간 트래픽을 측정하고, 결과를 CR status → report JSON으로만 출력한다. 이 throughput 값은 **Prometheus에 노출되지 않음** (CL2 기본 구조에 해당 스크래핑 없음).
- 따라서 **동일한 “파드 간 TCP throughput”** 을 GenericPrometheusQuery로 조회·단언하려면, 그 수치를 Prometheus에 넣는 별도 구성(exporter 등)이 필요하다. **ol-proto/CL2 기본만으로는 불가**.
- **이미 Prometheus에 있는 메트릭**(예: node_network_*, CNI 메트릭)에 대한 PromQL·threshold를 GenericPrometheusQuery에 넣으면 **CL2만으로 단언 가능**하지만, 그건 NetworkPerformanceMetrics가 측정하는 throughput과는 **다른 지표**이다.

정리: Prometheus 측정으로 단언 추가는 “정해진 measurement” + “GenericPrometheusQuery로 직접 정의” 둘 다 가능. 2번 시나리오의 **그 throughput 수치**를 CL2만으로 단언하려면 현재 구조상 불가하며, Chainsaw(또는 report 파싱) 래핑이 필요하다.

---

## 5. 참고 사항

- **CL2 framework 코드 수정**: 하지 않기로 함. 2번 단언은 래핑으로만 구현.
- **ol-proto 관련 문서**: 본 보고서와 `chainsaw/README.md`, `MEASUREMENTS.md`에 정리. 최종 통합 설명은 `ol-proto.md`를 맨 마지막에 작성 예정.
