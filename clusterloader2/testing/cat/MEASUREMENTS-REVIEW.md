# ClusterLoader2 Measurement 검토 보고서

## 1. 결론 요약

- **ClusterLoader2에서 사용 가능한 measurement는 14개가 아니다.**  
  코드에 등록된 measurement는 **49개**이다.
- **ol-cat.yaml의 14개 시나리오**는 “measurement별 1개”가 아니라, **최소 클러스터·선택 옵션 없이** 돌릴 수 있는 measurement만 골라서 **그중 하나씩** 대표 시나리오를 둔 것이다.
- 즉, **14개 = CAT 프로토타입에서 쓰는 measurement 수**이고, **49개 = CL2 전체에서 사용 가능한 measurement 수**이다.

---

## 2. 코드 기준 등록 measurement 전체 목록 (49개)

`pkg/measurement/` 아래 `measurement.Register(이름, ...)` 호출을 기준으로 정리했다.  
YAML의 `Method` 필드에는 아래 **이름**을 그대로 쓴다.

| № | Method 이름 (등록명) | 소스 파일 |
|---|---------------------|-----------|
| 1 | Sleep | sleep.go |
| 2 | Timer | timer.go |
| 3 | Exec | exec.go |
| 4 | WaitForRunningPods | wait_for_pods.go |
| 5 | WaitForNodes | wait_for_nodes.go |
| 6 | WaitForFinishedJobs | wait_for_jobs.go |
| 7 | WaitForGenericK8sObjects | wait_for_generic_k8s_object.go |
| 8 | WaitForControlledPodsRunning | wait_for_controlled_pods.go |
| 9 | WaitForAvailablePVs | wait_for_pvs.go |
| 10 | WaitForBoundPVCs | wait_for_pvcs.go |
| 11 | SystemPodMetrics | system_pod_metrics.go |
| 12 | SchedulingMetrics | scheduler_latency.go |
| 13 | SchedulingThroughput | scheduling_throughput.go |
| 14 | SchedulingThroughputPrometheus | scheduling_throughput_prometheus.go |
| 15 | PrometheusSchedulingMetrics | prometheus_scheduler_latency.go |
| 16 | PodStartupLatency | slos/pod_startup_latency.go |
| 17 | ServiceCreationLatency | service_creation_latency.go |
| 18 | JobLifecycleLatency | job_lifecycle_latency.go |
| 19 | APIAvailability | api_availability_measurement.go |
| 20 | MetricsForE2E | metrics_for_e2e.go |
| 21 | NetworkPerformanceMetrics | network/network_performance_measurement.go |
| 22 | APIResponsivenessPrometheus | slos/api_responsiveness_prometheus.go |
| 23 | ResourceUsageSummary | resource_usage.go |
| 24 | EtcdMetrics | etcd_metrics.go |
| 25 | CPUProfile | profile.go |
| 26 | MemoryProfile | profile.go |
| 27 | BlockProfile | profile.go |
| 28 | PodPeriodicCommand | pod_command.go |
| 29 | ClusterOOMsTracker | ooms_tracker.go |
| 30 | NodeLocalDNSLatencyPrometheus | nodelocaldns_latency_prometheus.go |
| 31 | NetworkPolicyEnforcement | network-policy/network-policy-enforcement-latency.go |
| 32 | NegLatency | neg_latency_measurement.go |
| 33 | MetricsServerPrometheus | metrics_server_prometheus.go |
| 34 | LoadBalancerNodeSyncLatency | loadbalancer_nodesync_latency.go |
| 35 | KubeStateMetricsLatency | kube_state_metrics_measurement.go |
| 36 | GenericPrometheusQuery | generic_query_measurement.go |
| 37 | DNSPerformanceK8sHostnames | dns/dns_performance-k8s-hostnames.go |
| 38 | ContainerRestarts | container_restarts.go |
| 39 | CiliumEndpointPropagationDelay | cilium_endpoint_propagation_delay.go |
| 40 | ChaosMonkey | chaos_monkey_measurement.go |
| 41 | TestMetrics | bundle/test_metrics.go |
| 42 | SLOMeasurement | slos/slo_measurement.go |
| 43 | ResourceClaimAllocationLatency | slos/resourceclaim_allocation_latency.go |
| 44 | NetworkProgrammingLatency | slos/network_programming.go |
| 45 | WindowsResourceUsagePrometheus | slos/windows_node_resource_usage.go |
| 46 | InClusterNetworkLatency | probes/probes.go |
| 47 | DnsLookupLatency | probes/probes.go |
| 48 | InClusterAPIServerRequestLatency | probes/probes.go |
| 49 | DnsPropagation | probes/probes.go |

(위 표 기준 **49개**가 코드에 등록된 measurement 전부이다.)

---

## 3. README vs 코드

- **clusterloader2/README.md** 의 “Measurement” 섹션에는 **일부만** 나열되어 있다.  
  (APIAvailability, APIResponsivenessPrometheus, CPUProfile, EtcdMetrics, MemoryProfile, MetricsForE2E, PodPeriodicCommand, PodStartupLatency, ResourceUsageSummary, SchedulingMetrics, SchedulingThroughput, Timer, WaitForControlledPodsRunning, WaitForRunningPods, Sleep, WaitForGenericK8sObjects 등.)
- **전체 목록은 코드가 정확한 근거**이다.  
  `grep -r "measurement.Register(" clusterloader2/pkg/measurement/` 로 확인 가능하다.

---

## 4. ol-cat에 포함된 14개 vs 나머지 (35개)

### 4.1 ol-cat에 포함된 14개 (프로토타입에서 사용)

| Method | ol-cat 시나리오 |
|--------|------------------|
| Sleep | sleep |
| Timer | timer |
| Exec | exec-echo |
| WaitForRunningPods | wait-for-running-pods |
| WaitForControlledPodsRunning | wait-for-controlled-pods |
| WaitForNodes | wait-for-nodes |
| NetworkPerformanceMetrics | network-tcp-1-1 |
| PodStartupLatency | scheduling-throughput-minimal |
| SchedulingThroughput | scheduling-throughput-minimal |
| WaitForFinishedJobs | wait-for-finished-job |
| JobLifecycleLatency | job-lifecycle-latency |
| ServiceCreationLatency | service-creation-latency |
| WaitForGenericK8sObjects | wait-for-generic-deployment |
| APIAvailability | api-availability |
| MetricsForE2E | metrics-for-e2e |

(실제 시나리오 수는 14개; PodStartupLatency와 SchedulingThroughput은 scheduling-throughput-minimal 하나에서 함께 사용.)

### 4.2 ol-cat에 포함하지 않은 measurement (요약)

- **Prometheus 필수**  
  APIResponsivenessPrometheus, SchedulingThroughputPrometheus, PrometheusSchedulingMetrics, ResourceUsageSummary, ContainerRestarts, NodeLocalDNSLatencyPrometheus, MetricsServerPrometheus, GenericPrometheusQuery, NetworkProgrammingLatency  
  → 프로토타입은 Prometheus 없이 돌리므로 제외.
- **인프라/플랫폼 특화**  
  NegLatency(GCP NEG), LoadBalancerNodeSyncLatency(L4 LB), EtcdMetrics(etcd 접근), KubeStateMetricsLatency(KSM), CiliumEndpointPropagationDelay(Cilium)  
  → 최소 범용 클러스터 가정과 맞지 않아 제외.
- **Windows**  
  WindowsResourceUsagePrometheus  
  → Windows 노드 전제이므로 제외.
- **스토리지/리소스**  
  WaitForAvailablePVs, WaitForBoundPVCs, ResourceClaimAllocationLatency(DRA)  
  → PV/PVC/DRA 설정이 필요해 프로토타입에서 제외.
- **기타**  
  SystemPodMetrics(플래그/설정), PodPeriodicCommand(파드 내 주기 명령), ClusterOOMsTracker, NetworkPolicyEnforcement(네트워크 정책 테스트), ChaosMonkey, SLOMeasurement(복합), TestMetrics(bundle), CPUProfile/MemoryProfile/BlockProfile(pprof), Probes 4종(InClusterNetworkLatency, DnsLookupLatency, InClusterAPIServerRequestLatency, DnsPropagation)  
  → 각각 추가 구성·목적이 달라 프로토타입 범위에서 제외.

---

## 5. 정리

| 구분 | 개수 | 설명 |
|------|------|------|
| **CL2 코드에 등록된 measurement** | **49개** | `measurement.Register` 기준, 사용 가능한 전체 목록 |
| **README에 문서화된 measurement** | 일부 | README는 일부만 나열, 코드가 전체 목록의 근거 |
| **ol-cat 시나리오 수** | **14개** | measurement별 1개가 아니라, “최소 클러스터에서 돌릴 수 있는 것”만 골라서 그중 하나씩 대표 시나리오를 둔 것 |
| **ol-cat에서 사용하는 measurement** | **14종** | Sleep, Timer, Exec, WaitFor* 5종, NetworkPerformanceMetrics, PodStartupLatency, SchedulingThroughput, JobLifecycleLatency, ServiceCreationLatency, APIAvailability, MetricsForE2E |

**결론:**  
ClusterLoader2에서 사용 가능한 measurement는 **14개가 아니라 49개**이다.  
ol-cat의 14개 시나리오는 그 49개 중 **프로토타입 조건(최소 클러스터, 추가 옵션 없음)에 맞는 14종**만 골라, measurement당 대표 시나리오 1개씩 둔 것이다.
