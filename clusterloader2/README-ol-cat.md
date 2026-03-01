# ol-cat.yaml 스위트 설명 (CAT 프로토타입)

**ol-cat.yaml** 은 ClusterLoader2(CL2)로 돌리는 **CAT(Cluster Acceptance Test) 프로토타입 스위트**입니다. 여러 시나리오를 순서대로 실행해 클러스터의 기본 동작과 measurement가 정상 동작하는지 검증합니다.

---

## 1. 스위트 개요

| 항목 | 내용 |
|------|------|
| **파일** | `clusterloader2/ol-cat.yaml` |
| **역할** | CAT 프로토타입: **measurement별 시나리오 1개**로 각 measurement 실행 가능 여부 확인 |
| **특징** | 시나리오별 파드/노드 수 최소(예: 파드 1개), SLO 검증 생략 또는 완화. 동일 measurement의 파라미터 변형(sleep-1s/5s/10s 등)은 제외. |
| **시나리오 수** | **14개** (measurement당 1개) |

---

## 2. 스위트에 포함된 시나리오 (14개, measurement 대응)

| 순서 | identifier | 대응 measurement |
|------|------------|------------------|
| 1 | `sleep` | Sleep |
| 2 | `timer` | Timer |
| 3 | `exec-echo` | Exec |
| 4 | `wait-for-running-pods` | WaitForRunningPods |
| 5 | `wait-for-controlled-pods` | WaitForControlledPodsRunning |
| 6 | `wait-for-nodes` | WaitForNodes |
| 7 | `network-tcp-1-1` | NetworkPerformanceMetrics |
| 8 | `scheduling-throughput-minimal` | PodStartupLatency, SchedulingThroughput |
| 9 | `wait-for-finished-job` | WaitForFinishedJobs |
| 10 | `job-lifecycle-latency` | JobLifecycleLatency |
| 11 | `service-creation-latency` | ServiceCreationLatency (**`--enable-exec-service` 필요**) |
| 12 | `wait-for-generic-deployment` | WaitForGenericK8sObjects |
| 13 | `api-availability` | APIAvailability |
| 14 | `metrics-for-e2e` | MetricsForE2E |

각 시나리오의 설정 경로·설명은 **testing/cat/README.md** 를 참고하세요.

---

## 3. 실행 방법

### 3.1 기본 실행

**clusterloader2/** 디렉터리에서:**

```bash
./run-e2e.sh --testsuite=ol-cat.yaml --provider=skeleton --kubeconfig=$HOME/.kube/config
```

**저장소 루트에서:**

```bash
./run-e2e.sh cluster-loader2 --testsuite=ol-cat.yaml --provider=skeleton --kubeconfig=$HOME/.kube/config
```

### 3.2 결과 저장(리포트 디렉터리)

성공/실패·measurement 요약·JUnit XML을 파일로 남기려면 `--report-dir` 을 지정하세요.

```bash
./run-e2e.sh --testsuite=ol-cat.yaml --provider=skeleton --kubeconfig=$HOME/.kube/config --report-dir=/tmp/cl2-cat-report
```

- **junit.xml**  
  시나리오별 성공/실패·소요 시간. 스위트가 **정상 종료**될 때 기록됩니다.
- **generatedConfig_<이름>.yaml**  
  시나리오별 컴파일된 최종 설정.
- **measurement 요약 파일**  
  파일명 예: `{Measurement이름}_{config이름}_{타임스탬프}.json` 등.

실행이 타임아웃 등으로 중간에 끊기면 junit.xml이 없거나 불완전할 수 있으므로, 그 경우 터미널 stdout의 `Status: Success` / `Status: Fail` 로 성공·실패를 확인하는 것이 좋습니다.

### 3.3 service-creation-latency 실행을 위한 필수 플래그

`service-creation-latency` 시나리오는 **exec-service**가 필요합니다. 반드시 다음 플래그를 추가하세요.

```bash
./run-e2e.sh --testsuite=ol-cat.yaml --provider=skeleton --kubeconfig=$HOME/.kube/config --enable-exec-service
```

`--enable-exec-service` 없이 실행하면 해당 시나리오의 "Start service creation latency measurement" 단계에서 `enable-exec-service flag not enabled` 로 실패합니다.

---

## 4. 성공/실패 확인

- **실행 중**  
  터미널에 시나리오마다 `Running <identifier>(<configPath>)` 와 `Test Finished` / `Status: Success` 또는 `Status: Fail` 이 출력됩니다.
- **실행 후**  
  `--report-dir` 을 지정했다면 해당 디렉터리의 **junit.xml** 과 measurement 요약 파일로 상세 결과를 확인할 수 있습니다.

---

## 5. ol-cat.yaml 파일 구조

스위트 파일은 **테스트 시나리오 배열** 하나로 이루어져 있습니다. 각 항목은 다음을 가집니다.

- **identifier**  
  시나리오를 구분하는 이름. 로그와 리포트에 사용됩니다.
- **configPath**  
  시나리오 설정 YAML 경로(clusterloader2/ 기준).
- **overridePaths**  
  (선택) configPath에 적용할 override 파일 목록. `network-tcp-1-1` 만 `testing/network/` 의 override를 사용합니다.

예시:

```yaml
- identifier: service-creation-latency
  configPath: testing/cat/service-creation-latency.yaml
  overridePaths: []
```

---

## 6. 관련 문서

- **시나리오별 설명·measurement 목록**  
  [testing/cat/README.md](testing/cat/README.md)
- **CL2 전반 사용법·플래그·API**  
  [clusterloader2/README.md](README.md)
