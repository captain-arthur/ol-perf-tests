# CAT(Cluster Acceptance Test) 프로토타입 시나리오

CAT 스위트(`ol-cat.yaml`)에서 사용하는 최소 구성의 ClusterLoader2(CL2) 시나리오 모음입니다. **ol-cat에서는 measurement별로 시나리오 1개씩**만 넣어 프로토타입을 구성합니다. 각 시나리오는 해당 measurement(및 보조 measurement)만 사용하고 부하는 최소로 둡니다(예: 파드 1개). 프로토타입이므로 SLO(서비스 수준 목표) 검증은 생략하거나 완화되어 있습니다.

---

## 1. 사용 가능한 measurement 찾기

`clusterloader2/README.md`에는 모든 measurement 목록이 없습니다. **실제 등록처는 코드**입니다.

- **등록 위치 검색**  
  `grep -r "measurement.Register(" clusterloader2/pkg/measurement/`
- **measurement 이름**  
  `Register(...)`의 첫 번째 인자(예: `"Sleep"`, `"Timer"`). 테스트 설정 YAML의 `Method` 필드에 이 문자열을 씁니다.
- **상수 이름**  
  각 파일에서 등록 이름은 보통 const로 정의됩니다(예: `sleepName = "Sleep"`, `timerMeasurementName = "Timer"`).

많은 measurement는 Prometheus, exec service, 특정 클러스터 크기, GCP NEG 등 **추가 구성**이 필요합니다. 이 폴더의 시나리오는 **최소 클러스터만으로** 돌아가는 measurement만 사용합니다.

- **49개 measurement 상세 문서(파라미터·예시):** [docs/MEASUREMENTS.md](../docs/MEASUREMENTS.md)

---

## 2. 시나리오 목록

| 설정 파일 | 사용 measurement | 설명 |
|-----------|-------------------|------|
| `sleep.yaml` | Sleep | 5초 대기 한 스텝. |
| `timer.yaml` | **Timer**, Sleep | 타이머 시작 → 2초 대기 → 타이머 종료 → gather. (start/stop 사이 지연용 Sleep.) |
| `wait-for-running-pods.yaml` | WaitForRunningPods | 파드 1개 생성(phase) 후, 1개 Running 될 때까지 대기. |
| `wait-for-controlled-pods.yaml` | WaitForControlledPodsRunning | 레플리카 1개 Deployment 생성 후, controlled pod가 Running 될 때까지 대기. |
| `wait-for-nodes.yaml` | WaitForNodes | 노드 1~4개 Ready 될 때까지 대기 (timeout 2m). |
| `wait-for-finished-job.yaml` | WaitForFinishedJobs | Job 1개 생성 후, 해당 Job이 완료될 때까지 대기 (timeout 2m). |
| `job-lifecycle-latency.yaml` | **JobLifecycleLatency** | Job 생성 라이프사이클 레이턴시 측정: start → Job 1개 생성 → gather. |
| `service-creation-latency.yaml` | **ServiceCreationLatency** | Deployment + ClusterIP Service 생성 후 서비스 생성/도달 레이턴시 측정. **실행 시 `--enable-exec-service` 필요.** |
| `wait-for-generic-deployment.yaml` | WaitForGenericK8sObjects | Deployment 1개 생성 후, `Available=True` 조건 만족할 때까지 대기. |
| `scheduling-throughput-minimal.yaml` | **PodStartupLatency**, **SchedulingThroughput**, WaitForRunningPods | 스루풋/시작 지연 측정 시작 → 파드 1개 생성 → Running 대기 → gather → 파드 삭제. SLO 임계값 없음. |
| `api-availability.yaml` | **APIAvailability** | API 서버 `/readyz` 폴링 후 가용성 수집. `useHostInternalIPs: false`로 최소 클러스터에서 실행. |
| `metrics-for-e2e.yaml` | **MetricsForE2E** | apiserver/scheduler/controller-manager 메트릭 1회 수집. |

**네트워크(TCP 1:1)** 는 `testing/network/config.yaml`과 override로 다룹니다. **NetworkPerformanceMetrics** 만 사용하며, `ol-cat.yaml`에서 `configPath: testing/network/config.yaml`과 TCP·1:1 ratio용 `overridePaths`로 참조됩니다.

### ol-cat.yaml: measurement별 1개 (14개 시나리오)

ol-cat 스위트에는 **동일 measurement의 파라미터 변형은 넣지 않고**, measurement당 대표 시나리오 1개만 등록합니다.

| ol-cat identifier | 사용 measurement |
|-------------------|------------------|
| `sleep` | Sleep |
| `timer` | Timer |
| `exec-echo` | Exec |
| `wait-for-running-pods` | WaitForRunningPods |
| `wait-for-controlled-pods` | WaitForControlledPodsRunning |
| `wait-for-nodes` | WaitForNodes |
| `network-tcp-1-1` | NetworkPerformanceMetrics |
| `scheduling-throughput-minimal` | PodStartupLatency, SchedulingThroughput |
| `wait-for-finished-job` | WaitForFinishedJobs |
| `job-lifecycle-latency` | JobLifecycleLatency |
| `service-creation-latency` | ServiceCreationLatency |
| `wait-for-generic-deployment` | WaitForGenericK8sObjects |
| `api-availability` | APIAvailability |
| `metrics-for-e2e` | MetricsForE2E |

이 폴더에는 위 14개 외에 **변형용** 시나리오 파일(sleep-1s, timer-5s, wait-for-nodes-1-2, create-configmap 등)도 있으나, ol-cat에는 포함하지 않습니다. 필요 시 해당 파일을 직접 참조해 실행하면 됩니다.

---

## 3. CAT 스위트 실행 방법

스위트 경로는 **clusterloader2/** 기준 상대 경로**입니다.

**저장소 루트에서:**

```bash
./run-e2e.sh cluster-loader2 --testsuite=ol-cat.yaml --provider=skeleton --kubeconfig=$HOME/.kube/config
```

**clusterloader2/ 디렉터리에서:**

```bash
./run-e2e.sh --testsuite=ol-cat.yaml --provider=skeleton --kubeconfig=$HOME/.kube/config
```

결과를 파일로 남기려면 `--report-dir=/path/to/report` 를 추가하세요. 자세한 실행 옵션과 ol-cat 스위트 설명은 **clusterloader2/README-ol-cat.md** 를 참고하세요.

---

## 4. 특수 요구사항

- **service-creation-latency**  
  ServiceCreationLatency measurement가 exec-service 파드에서 ClusterIP로 curl을 보내 도달 가능 여부를 판단하므로, **반드시 `--enable-exec-service`** 를 넣어 실행해야 합니다. 없으면 `action: start` 단계에서 `"enable-exec-service flag not enabled"` 로 실패합니다.
- **이미지**  
  `service-creation-latency` 의 Deployment는 `registry.k8s.io/e2e-test-images/agnhost:2.45` (serve-hostname, 포트 9376)를 사용합니다. `docker.io` 풀 제한이 있는 환경에서도 registry.k8s.io 기준으로 동작하도록 한 구성입니다.
