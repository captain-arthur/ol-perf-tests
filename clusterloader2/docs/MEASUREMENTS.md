# ClusterLoader2 Measurement 전용 문서 (49개)

테스트 설정 YAML의 `measurements`에서 `Method` 필드에 넣는 이름과, 각 measurement의 파라미터·동작·예시를 정리한 문서입니다. 코드(`pkg/measurement/`) 기준으로 작성했습니다.

각 measurement에는 다음을 포함합니다.
- **Params:** 파라미터 목록(필수/선택, 설명)
- **Threshold 설정 가능 여부:** SLO/성공·실패 판단용 임계값(threshold, perc50/90/99, 가용률 등)을 설정할 수 있는지, 가능하면 어떤 파라미터로 설정하는지
- **사용처:** 해당 측정이 쓰이는 대표 시나리오(CAT, 성능 테스트, SLO 검증, 모니터링 등)
- **요구사항:** Prometheus, exec-service, 특정 클라우드/플랫폼 등
- **예시:** 가능한 경우 YAML 스니펫

---

## 목차 (49개 Method)

| № | Method | № | Method | № | Method |
|---|--------|---|--------|---|--------|
| 1 | [Sleep](#1-sleep) | 18 | [JobLifecycleLatency](#18-joblifecyclelatency) | 35 | [KubeStateMetricsLatency](#35-kubestatemetricslatency) |
| 2 | [Timer](#2-timer) | 19 | [APIAvailability](#19-apiavailability) | 36 | [GenericPrometheusQuery](#36-genericprometheusquery) |
| 3 | [Exec](#3-exec) | 20 | [MetricsForE2E](#20-metricsfore2e) | 37 | [DNSPerformanceK8sHostnames](#37-dnsperformancek8shostnames) |
| 4 | [WaitForRunningPods](#4-waitforrunningpods) | 21 | [NetworkPerformanceMetrics](#21-networkperformancemetrics) | 38 | [ContainerRestarts](#38-containerrestarts) |
| 5 | [WaitForNodes](#5-waitfornodes) | 22 | [APIResponsivenessPrometheus](#22-apiresponsivenessprometheus) | 39 | [CiliumEndpointPropagationDelay](#39-ciliumendpointpropagationdelay) |
| 6 | [WaitForFinishedJobs](#6-waitforfinishedjobs) | 23 | [ResourceUsageSummary](#23-resourceusagesummary) | 40 | [ChaosMonkey](#40-chaosmonkey) |
| 7 | [WaitForGenericK8sObjects](#7-waitforgenerick8sobjects) | 24 | [EtcdMetrics](#24-etcdmetrics) | 41 | [TestMetrics](#41-testmetrics) |
| 8 | [WaitForControlledPodsRunning](#8-waitforcontrolledpodsrunning) | 25 | [CPUProfile](#25-cpuprofile) | 42 | [SLOMeasurement](#42-slomeasurement) |
| 9 | [WaitForAvailablePVs](#9-waitforavailablepvs) | 26 | [MemoryProfile](#26-memoryprofile) | 43 | [ResourceClaimAllocationLatency](#43-resourceclaimallocationlatency) |
| 10 | [WaitForBoundPVCs](#10-waitforboundpvcs) | 27 | [BlockProfile](#27-blockprofile) | 44 | [NetworkProgrammingLatency](#44-networkprogramminglatency) |
| 11 | [SystemPodMetrics](#11-systempodmetrics) | 28 | [PodPeriodicCommand](#28-podperiodiccommand) | 45 | [WindowsResourceUsagePrometheus](#45-windowsresourceusageprometheus) |
| 12 | [SchedulingMetrics](#12-schedulingmetrics) | 29 | [ClusterOOMsTracker](#29-clusteroomstracker) | 46 | [InClusterNetworkLatency](#46-inclusternetworklatency) |
| 13 | [SchedulingThroughput](#13-schedulingthroughput) | 30 | [NodeLocalDNSLatencyPrometheus](#30-nodelocaldnslatencyprometheus) | 47 | [DnsLookupLatency](#47-dnslookuplatency) |
| 14 | [SchedulingThroughputPrometheus](#14-schedulingthroughputprometheus) | 31 | [NetworkPolicyEnforcement](#31-networkpolicyenforcement) | 48 | [InClusterAPIServerRequestLatency](#48-inclusterapiserverrequestlatency) |
| 15 | [PrometheusSchedulingMetrics](#15-prometheusschedulingmetrics) | 32 | [NegLatency](#32-neglatency) | 49 | [DnsPropagation](#49-dnspropagation) |
| 16 | [PodStartupLatency](#16-podstartuplatency) | 33 | [MetricsServerPrometheus](#33-metricsserverprometheus) | | |
| 17 | [ServiceCreationLatency](#17-servicecreationlatency) | 34 | [LoadBalancerNodeSyncLatency](#34-loadbalancernodesynclatency) | | |

---

## 공통 파라미터 (여러 measurement에서 사용)

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `labelSelector` | string | 리소스 필터용 라벨 셀렉터 (예: `group=cat-pods`) |
| `namespace` | string | 네임스페이스. 생략 시 `""`(all namespaces) |
| `fieldSelector` | string | 필드 셀렉터 (고급) |
| `timeout` | duration | 대기 최대 시간 (예: `2m`, `5m`) |
| `action` | string | 동작: `start` / `stop` / `gather` 등 (measurement별로 상이) |

---

## 1. Sleep

**설명:** 지정한 시간만큼 실행을 멈춥니다. 스텝 사이 지연이나 “최소 유지 시간”을 두는 데 씁니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `duration` | 예 | 대기 시간 (예: `5s`, `1m`) |

**Threshold 설정:** 없음 (지연 시간만 지정).

**사용처:** 스텝 간 대기, 측정 구간 유지, 다른 measurement와의 타이밍 조합(예: Timer start 후 일정 시간 유지 후 stop).

**예시:**

```yaml
- name: Sleep 5s
  measurements:
  - Identifier: Sleep5s
    Method: Sleep
    Params:
      duration: 5s
```

---

## 2. Timer

**설명:** `start`와 `stop` 사이 구간의 소요 시간을 측정합니다. `label`로 여러 구간을 구분할 수 있습니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `action` | 예 | `start` \| `stop` \| `gather` |
| `label` | start/stop 시 | 타이머 구간 식별자 (동일 label로 start/stop 쌍) |

**Threshold 설정:** 없음 (구간 시간을 “측정”만 하고, 성공/실패 임계값은 없음).

**사용처:** Phase/스텝 구간 소요 시간 측정, 성능 테스트 구간 구분, 리포트용 경과 시간 수집.

**예시:**

```yaml
- name: Start timer
  measurements:
  - Identifier: MyTimer
    Method: Timer
    Params:
      action: start
      label: phase1
- name: Do work
  measurements:
  - Identifier: Sleep
    Method: Sleep
    Params:
      duration: 2s
- name: Stop timer and gather
  measurements:
  - Identifier: MyTimer
    Method: Timer
    Params:
      action: stop
      label: phase1
  - Identifier: MyTimer
    Method: Timer
    Params:
      action: gather
```

---

## 3. Exec

**설명:** **CL2가 돌아가는 로컬 환경**에서 지정한 명령을 실행합니다. 클러스터 내부가 아니라 테스트 러너 머신에서 실행됩니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `command` | 예 | 실행할 명령 배열 (예: `["sh", "-c", "echo ok"]`) |
| `timeout` | 아니오 | 명령 타임아웃 (기본 1h) |
| `retries` | 아니오 | 실패 시 재시도 횟수 (기본 1) |
| `backoffDelay` | 아니오 | 재시도 전 대기 (기본 1s) |

**Threshold 설정:** 없음 (명령 성공/실패만 구분, 수치 임계값 없음).

**사용처:** 테스트 러너에서 사전/사후 스크립트 실행, 로컬 헬스체크, 외부 도구 호출.

**예시:**

```yaml
- name: Run local command
  measurements:
  - Identifier: ExecEcho
    Method: Exec
    Params:
      command: ["sh", "-c", "echo ok"]
      timeout: 10s
```

---

## 4. WaitForRunningPods

**설명:** 지정한 조건의 Pod가 원하는 개수만큼 Running 상태가 될 때까지 대기합니다. 타임아웃 시 테스트는 계속 진행되며, `isFatal: true`면 실패로 처리할 수 있습니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `desiredPodCount` | 예 | 기다릴 Pod 개수 |
| `labelSelector` | 아니오 | Pod 라벨 셀렉터 |
| `namespace` | 아니오 | 네임스페이스 (생략 시 all) |
| `timeout` | 아니오 | 대기 시간 (기본 60s) |
| `refreshInterval` | 아니오 | 확인 주기 (기본 5s) |
| `isFatal` | 아니오 | 실패 시 테스트 중단 여부 (기본 false) |

**Threshold 설정:** 없음. 대기 조건은 `desiredPodCount`·`timeout`·`isFatal`로 제어.

**사용처:** Pod 생성 후 Running 전환 대기, 스케줄링/시작 레이턴시 측정 전 준비, CAT·부하 테스트의 동기화 포인트.

**예시:**

```yaml
- name: Wait for 1 pod running
  measurements:
  - Identifier: WaitPods
    Method: WaitForRunningPods
    Params:
      desiredPodCount: 1
      labelSelector: group=cat-wait-pods
      timeout: 2m
```

---

## 5. WaitForNodes

**설명:** Ready 상태 노드 수가 `minDesiredNodeCount` 이상 `maxDesiredNodeCount` 이하가 될 때까지 대기합니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `minDesiredNodeCount` | 예 | 최소 노드 수 |
| `maxDesiredNodeCount` | 예 | 최대 노드 수 |
| `timeout` | 아니오 | 대기 시간 (기본 30m) |
| `refreshInterval` | 아니오 | 확인 주기 (기본 30s) |
| `labelSelector` / `fieldSelector` | 아니오 | 노드 필터 |

**Threshold 설정:** 없음. 노드 수 범위(`minDesiredNodeCount`~`maxDesiredNodeCount`)와 `timeout`으로만 제어.

**사용처:** 클러스터 확장/축소 후 Ready 대기, 노드 풀 준비 확인, 스케일 테스트 전 조건 충족 대기.

**예시:**

```yaml
- name: Wait for nodes ready
  measurements:
  - Identifier: WaitNodes
    Method: WaitForNodes
    Params:
      minDesiredNodeCount: 1
      maxDesiredNodeCount: 4
      timeout: 2m
```

---

## 6. WaitForFinishedJobs

**설명:** 지정한 label의 Job들이 완료(Finished)될 때까지 대기합니다. `start`로 관찰을 시작하고 `gather`로 대기·결과 수집을 합니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `action` | 예 | `start` \| `gather` |
| `labelSelector` | start 시 | 관찰할 Job 라벨 |
| `timeout` | 아니오 | gather 대기 시간 (기본 **10m**. 코드: `defaultWaitForFinishedJobsTimeout`) |

**Threshold 설정:** 없음. 완료 대기만 하며, 레이턴시/성공률 임계값은 없음.

**사용처:** Job 배치 실행 후 완료 대기, JobLifecycleLatency 등과 조합해 Job 기반 워크로드 검증.

**예시:**

```yaml
- name: Start wait for jobs
  measurements:
  - Identifier: WaitJobs
    Method: WaitForFinishedJobs
    Params:
      action: start
      labelSelector: group=cat-job
- name: Create Job
  phases:
  - namespaceRange: { min: 1, max: 1 }
    replicasPerNamespace: 1
    tuningSet: default
    objectBundle:
    - basename: cat-job
      objectTemplatePath: job.yaml
- name: Wait for job finish
  measurements:
  - Identifier: WaitJobs
    Method: WaitForFinishedJobs
    Params:
      action: gather
      timeout: 2m
```

---

## 7. WaitForGenericK8sObjects

**설명:** 지정한 GVK(GroupVersionResource)의 오브젝트가 `successfulConditions`를 만족할 때까지 대기합니다. Deployment `Available=True` 등 status.conditions 기반입니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `objectGroup` | 예 | API 그룹 (예: `apps`) |
| `objectVersion` | 예 | API 버전 (예: `v1`) |
| `objectResource` | 예 | 리소스명 (예: `deployments`) |
| `namespaceRange` | 아니오 | 네임스페이스 범위 (min/max) |
| `successfulConditions` | 예 | 조건 배열 (예: `["Available=True"]`) |
| `failedConditions` | 예 | 실패로 볼 조건 (빈 배열 가능) |
| `minDesiredObjectCount` | 예 | 최소 오브젝트 수 |
| `maxFailedObjectCount` | 예 | 허용 실패 개수 |
| `timeout` | 아니오 | 대기 시간 (기본 30m) |
| `refreshInterval` | 아니오 | 확인 주기 (기본 30s) |

**Threshold 설정:** 없음. 조건(`successfulConditions`/`failedConditions`)과 개수·`timeout`으로만 제어.

**사용처:** Deployment/StatefulSet 등이 Available 될 때까지 대기, 롤아웃 검증, 제네릭 오브젝트 상태 동기화.

**예시:**

```yaml
- name: Wait for deployment Available
  measurements:
  - Identifier: WaitDep
    Method: WaitForGenericK8sObjects
    Params:
      objectGroup: apps
      objectVersion: v1
      objectResource: deployments
      namespaceRange: { min: 1, max: 1 }
      timeout: 2m
      successfulConditions: ["Available=True"]
      failedConditions: []
      minDesiredObjectCount: 1
      maxFailedObjectCount: 0
```

---

## 8. WaitForControlledPodsRunning

**설명:** Deployment/ReplicaSet/Job 등 컨트롤러가 만든 Pod가 모두 Running이 될 때까지 대기합니다. `start`로 관찰 시작, `gather`로 대기합니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `action` | 예 | `start` \| `gather` |
| `apiVersion` | start 시 | 예: `apps/v1` |
| `kind` | start 시 | 예: `Deployment` |
| `labelSelector` | start 시 | 컨트롤러 라벨 |
| `operationTimeout` | 아니오 | start 동기화 타임아웃 (기본 5m) |

**Threshold 설정:** 없음. 컨트롤러 Pod Running 대기만 하며, 레이턴시 임계값 없음.

**사용처:** Deployment/ReplicaSet 기반 부하 생성 후 Pod 준비 대기, PodStartupLatency·스케줄링 스루풋 측정 전 동기화.

**예시:**

```yaml
- name: Start wait
  measurements:
  - Identifier: WaitDeploy
    Method: WaitForControlledPodsRunning
    Params:
      action: start
      apiVersion: apps/v1
      kind: Deployment
      labelSelector: group=cat-controlled
- name: Create Deployment
  phases:
  - namespaceRange: { min: 1, max: 1 }
    replicasPerNamespace: 1
    tuningSet: default
    objectBundle:
    - basename: cat-deploy
      objectTemplatePath: deployment.yaml
- name: Wait for controlled pods running
  measurements:
  - Identifier: WaitDeploy
    Method: WaitForControlledPodsRunning
    Params:
      action: gather
```

---

## 9. WaitForAvailablePVs

**설명:** 지정한 provisioner의 Available PV가 `desiredPVCount`개 될 때까지 대기합니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `desiredPVCount` | 예 | 기다릴 PV 개수 |
| `provisioner` | 예 | StorageClass provisioner (예: `kubernetes.io/gce-pd`) |
| `timeout` | 아니오 | 대기 시간 |

**Threshold 설정:** 없음. `desiredPVCount`·`timeout`으로만 제어.

**사용처:** 동적 프로비저닝 테스트 전 PV 풀 준비, 스토리지 부하 테스트 전 조건 충족.

**요구사항:** PV/StorageClass 환경.

**예시:**

```yaml
- name: Wait for PVs
  measurements:
  - Identifier: WaitPVs
    Method: WaitForAvailablePVs
    Params:
      desiredPVCount: 5
      provisioner: kubernetes.io/gce-pd
      timeout: 10m
```

---

## 10. WaitForBoundPVCs

**설명:** PVC가 Bound 상태가 될 때까지 대기합니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `desiredPVCCount` | 예 | 기다릴 PVC 개수 |
| `labelSelector` / `namespace` | 아니오 | PVC 필터 |
| `timeout` | 아니오 | 대기 시간 |

**Threshold 설정:** 없음. `desiredPVCCount`·`timeout`으로만 제어.

**사용처:** PVC 바인딩 대기, 스토리지 워크로드 테스트 전 준비.

**요구사항:** PVC/StorageClass 환경.

**예시:**

```yaml
- name: Wait for PVCs bound
  measurements:
  - Identifier: WaitPVCs
    Method: WaitForBoundPVCs
    Params:
      desiredPVCCount: 1
      timeout: 5m
```

---

## 11. SystemPodMetrics

**설명:** 시스템 파드(kube-system 등)의 메트릭을 start 시점과 gather 시점에 수집·비교합니다. 재시작 횟수 등 확인에 사용됩니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `action` | 예 | `start` \| `gather` |
| `systemPodMetricsEnabled` | 아니오 | true여야 수집 수행 (기본 false. false면 스킵) |
| `enableRestartCountCheck` | 아니오 | true여야 재시작 횟수 검사 수행 (기본 false) |
| `restartCountThresholdOverrides` | 아니오 | YAML 직렬화 맵. 컨테이너별 또는 `default` 키로 허용 재시작 횟수. 미지정 시 0(재시작 불허) |

**Threshold 설정:** 가능. `restartCountThresholdOverrides`(Params, YAML 맵)로 컨테이너별(또는 `default`) 재시작 허용 상한 설정. `enableRestartCountCheck=true`일 때만 검사하며, 초과 시 gather에서 실패.

**사용처:** kube-system 등 시스템 파드의 재시작·메트릭 변화 모니터링, 클러스터 안정성 검증.

**요구사항:** Params에서 `systemPodMetricsEnabled=true`로 수집 활성화, 재시작 검사 시 `enableRestartCountCheck=true` 및 `restartCountThresholdOverrides` 설정.

**예시:**

```yaml
- name: Start system pod metrics
  measurements:
  - Identifier: SysPods
    Method: SystemPodMetrics
    Params:
      action: start
- name: Gather system pod metrics
  measurements:
  - Identifier: SysPods
    Method: SystemPodMetrics
    Params:
      action: gather
```

---

## 12. SchedulingMetrics

**설명:** 스케줄러 메트릭(스케줄 시도/성공 등)을 수집합니다. `start`로 관찰 시작, `gather`로 결과 수집.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `action` | 예 | `start` \| `gather` |
| `labelSelector` | 아니오 | 관찰할 Pod 라벨 |

**Threshold 설정:** 없음. 메트릭 수집·리포트만 수행.

**사용처:** 스케줄러 메트릭 수집, 스케줄링 성능 분석·디버깅.

**요구사항:** 스케줄러 메트릭 노출 필요.

**예시:**

```yaml
- name: Start scheduler metrics
  measurements:
  - Identifier: SchedMetrics
    Method: SchedulingMetrics
    Params:
      action: start
      labelSelector: group=my-pods
- name: Gather
  measurements:
  - Identifier: SchedMetrics
    Method: SchedulingMetrics
    Params:
      action: gather
```

---

## 13. SchedulingThroughput

**설명:** 스케줄링 처리량(초당 스케줄된 Pod 수)을 측정합니다. PodStartupLatency와 함께 자주 사용됩니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `action` | 예 | `start` \| `gather` |
| `labelSelector` | start 시 | 관찰할 Pod 라벨 |
| `measurmentInterval` | 아니오 | 집계 구간 (기본 1s). 주의: 오타(measurment) 유지 |
| `threshold` | gather 시 | 최소 스루풋 (미달 시 실패, 선택. 기본 0이면 검사 안 함) |
| `enableViolations` | 아니오 | true면 threshold 미달 시 MetricViolationError 반환 (기본 true). false면 위반 시에도 err 무시 |

**Threshold 설정:** 가능. `threshold`(float64): gather 시 최소 스루풋(초당 Pod 수). `threshold > 0`이고 Max < threshold이면 MetricViolationError. `enableViolations=false`면 해당 에러를 무시하고 성공 처리.

**사용처:** 스케줄링 처리량 SLO 검증, 스케일/부하 테스트에서 스케줄러 성능 측정.

**요구사항:** 없음.

**예시:**

```yaml
- name: Start scheduling throughput
  measurements:
  - Identifier: SchedThroughput
    Method: SchedulingThroughput
    Params:
      action: start
      labelSelector: group=cat-scheduling
      measurmentInterval: 1s
- name: Create pods
  phases: [ ... ]
- name: Gather
  measurements:
  - Identifier: SchedThroughput
    Method: SchedulingThroughput
    Params:
      action: gather
      threshold: 0.5
```

---

## 14. SchedulingThroughputPrometheus

**설명:** Prometheus에서 스케줄링 스루풋 관련 메트릭(예: `apiserver_request_total` binding 201)을 쿼리해 최대 스루풋을 계산합니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `action` | 예 | `start` \| `gather` (Prometheus measurement 공통) |
| `threshold` | 아니오 | 최소 스루풋. 미달 시 MetricViolationError (기본 0, 검사 안 함) |

**Threshold 설정:** 가능. `threshold`(float64): gather 시 최소 스루풋. 미달 시 MetricViolationError.

**사용처:** Prometheus 기반 스케줄링 스루풋 SLO 검증, 프로덕션 메트릭과 동일 소스로 검사.

**요구사항:** Prometheus 서버. apiserver 메트릭(binding 201) 스크래핑 필요.

**예시:**

```yaml
- name: Start
  measurements:
  - Identifier: SchedThroughputProm
    Method: SchedulingThroughputPrometheus
    Params:
      action: start
- name: Create pods
  phases: [ ... ]
- name: Gather
  measurements:
  - Identifier: SchedThroughputProm
    Method: SchedulingThroughputPrometheus
    Params:
      action: gather
      threshold: 10
```

---

## 15. PrometheusSchedulingMetrics

**설명:** Prometheus에서 스케줄러 레이턴시 히스토그램 메트릭을 쿼리해 50/90/99 백분위를 계산합니다. `scheduler_latency.go`와 동일한 JSON 요약을 Prometheus 소스로 생성합니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `action` | 예 | `start` \| `gather` |

**Threshold 설정:** 없음. 50/90/99 백분위 계산·리포트만 하며, 실패 임계값은 코드에 없음.

**사용처:** Prometheus 기반 스케줄러 레이턴시 수집, 스케줄링 지연 분석.

**요구사항:** Prometheus 서버. kube-scheduler 메트릭 엔드포인트 스크래핑 필요.

---

## 16. PodStartupLatency

**설명:** Pod가 스케줄부터 Running까지 걸리는 시간을 측정합니다. perc50/90/99 임계값(threshold)으로 SLO 검증 가능합니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `action` | 예 | `start` \| `gather` |
| `labelSelector` | start 시 | 관찰할 Pod 라벨 |
| `threshold` | 아니오 | 기본 SLO 임계값 (예: `30s`) |
| `perc50Threshold` / `perc90Threshold` / `perc99Threshold` | 아니오 | 백분위별 임계값 |

**Threshold 설정:** 가능. `threshold`(기본 5s): 공통 임계값. `perc50Threshold`/`perc90Threshold`/`perc99Threshold`로 백분위별 override. gather 시 VerifyThresholdByPercentile로 검증, 초과 시 실패.

**사용처:** Pod 시작 SLO 검증(스케줄→Running), CAT·성능 테스트의 핵심 지표, Kubernetes SLO 문서와 연동.

**요구사항:** 없음.

**예시:**

```yaml
- name: Start pod startup latency
  measurements:
  - Identifier: PodStartup
    Method: PodStartupLatency
    Params:
      action: start
      labelSelector: group=cat-scheduling
      threshold: 30s
- name: Create pod
  phases: [ ... ]
- name: Wait for running
  measurements:
  - Identifier: WaitPods
    Method: WaitForRunningPods
    Params:
      desiredPodCount: 1
      labelSelector: group=cat-scheduling
      timeout: 2m
- name: Gather pod startup latency
  measurements:
  - Identifier: PodStartup
    Method: PodStartupLatency
    Params:
      action: gather
```

---

## 17. ServiceCreationLatency

**설명:** Service 생성부터 “도달 가능(reachability)”까지의 시간을 측정합니다. ClusterIP/NodePort/LoadBalancer에 대해 exec-service 파드에서 curl로 도달 여부를 확인합니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `action` | 예 | `start` \| `waitForReady` \| `waitForDeletion` \| `gather` |
| `labelSelector` | start 시 | 관찰할 Service 라벨 |
| `waitTimeout` | 아니오 | waitForReady 대기 시간 (기본 10m) |
| `checkIngress` | 아니오 | Ingress 포함 여부 (기본 false) |

**Threshold 설정:** 없음. 레이턴시 측정·리포트만 하며, 코드에 임계값 검증 없음.

**사용처:** Service 생성 후 도달 가능까지 시간 측정, 네트워크/서비스 레이턴시 분석, kube-proxy·CNI 검증.

**요구사항:** **`--enable-exec-service`** 필수. Service의 백엔드 Pod가 해당 포트에서 수신 중이어야 함.

**예시:**

```yaml
- name: Start service creation latency
  measurements:
  - Identifier: SvcLatency
    Method: ServiceCreationLatency
    Params:
      action: start
      labelSelector: group=cat-svc
      waitTimeout: 5m
- name: Create Deployment and Service
  phases:
  - namespaceRange: { min: 1, max: 1 }
    replicasPerNamespace: 1
    objectBundle:
    - basename: cat-svc-dep
      objectTemplatePath: deployment-for-svc.yaml
    - basename: cat-svc
      objectTemplatePath: service-clusterip.yaml
- name: Gather service creation latency
  measurements:
  - Identifier: SvcLatency
    Method: ServiceCreationLatency
    Params:
      action: gather
```

---

## 18. JobLifecycleLatency

**설명:** Job 생성부터 완료까지의 라이프사이클 구간별 레이턴시를 측정합니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `action` | 예 | `start` \| `gather` |
| `labelSelector` | start 시 | 관찰할 Job 라벨 |
| `timeout` | gather 시 | 대기 시간 (기본 **10m**. 코드: `defaultWaitForFinishedJobsTimeout`) |

**Threshold 설정:** 없음. 구간별 레이턴시 수집·리포트만 하며, SLO 임계값 검증은 코드에 없음.

**사용처:** Job 생성→완료 구간 레이턴시 분석, 배치 워크로드 성능·SLO 검토.

**요구사항:** 없음.

**예시:**

```yaml
- name: Start job lifecycle latency
  measurements:
  - Identifier: JobLifecycle
    Method: JobLifecycleLatency
    Params:
      action: start
      labelSelector: group=cat-job
- name: Create Job
  phases: [ ... ]
- name: Gather job lifecycle latency
  measurements:
  - Identifier: JobLifecycle
    Method: JobLifecycleLatency
    Params:
      action: gather
      timeout: 2m
```

---

## 19. APIAvailability

**설명:** 컨트롤 플레인 가용성을 측정합니다. `/readyz`를 주기적으로 호출해 가용률을 수집합니다. host 단위 폴링은 exec-service가 필요합니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `action` | 예 | `start` \| `pause` \| `unpause` \| `gather` |
| `pollFrequency` | start 시 | 폴링 주기 (예: `5s`) — 필수 |
| `threshold` | 아니오 | 목표 가용률 (예: `0.99`, 기본 0) |
| `useHostInternalIPs` | 아니오 | true 시 각 호스트 /readyz 폴링 (exec-service 필요). 기본은 exec-service 사용 시 true |
| `useHostPublicIPs` | 아니오 | true 시 퍼블릭 IP로 폴링 (기본 false) |
| `hostPollTimeoutSeconds` | 아니오 | 호스트별 curl connect 타임아웃(초). 기본 5 (host 폴링 시) |
| `hostPollExecTimeoutSeconds` | 아니오 | 호스트 폴링 exec 전체 타임아웃(초). 기본 10 (host 폴링 시) |

**Threshold 설정:** 가능. `threshold`(float64, 기본 0): 목표 가용률. gather 시 이 비율 미달이면 실패(코드에서 가용률 계산 후 비교).

**사용처:** 컨트롤 플레인 가용성 SLO, /readyz 기반 가용률 모니터링, 장애 시나리오 후 복구 검증.

**요구사항:** host 폴링 사용 시 `--enable-exec-service`. 클러스터 수준만 쓰면 불필요.

**예시:**

```yaml
- name: Start API availability
  measurements:
  - Identifier: APIAvail
    Method: APIAvailability
    Params:
      action: start
      pollFrequency: 5s
      threshold: 0.99
      useHostInternalIPs: false
      useHostPublicIPs: false
- name: Sleep 10s
  measurements:
  - Identifier: Sleep
    Method: Sleep
    Params:
      duration: 10s
- name: Gather API availability
  measurements:
  - Identifier: APIAvail
    Method: APIAvailability
    Params:
      action: gather
```

---

## 20. MetricsForE2E

**설명:** kube-apiserver, scheduler, controller-manager(및 선택 시 kubelet) 메트릭을 한 번 수집합니다. e2e 결과 해석용입니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `gatherKubeletsMetrics` | 아니오 | true 시 모든 kubelet 메트릭 수집 (기본 false, provider 지원 시에만 동작) |

**Threshold 설정:** 없음. 메트릭 수집·출력만 수행.

**사용처:** e2e 테스트 후 apiserver/scheduler/controller-manager(및 선택 시 kubelet) 메트릭 수집, 결과 해석·버그 분석.

**요구사항:** 없음. (kubelet 수집은 provider 기능에 따름)

**예시:**

```yaml
- name: Gather e2e metrics
  measurements:
  - Identifier: E2EMetrics
    Method: MetricsForE2E
    Params:
      gatherKubeletsMetrics: false
```

---

## 21. NetworkPerformanceMetrics

**설명:** 클러스터 내 네트워크 성능(throughput, latency 등)을 측정합니다. netperf 등 프로토콜·서버/클라이언트 수를 설정할 수 있습니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `action` | 예 | `start` \| `gather` |
| `duration` | start 시 | 측정 지속 시간 (예: `10s`) |
| `protocol` | 예 | `TCP` \| `UDP` \| `HTTP` (템플릿 변수로 override 가능) |
| `numberOfServers` | 예 | 서버 수 (템플릿 변수) |
| `numberOfClients` | 예 | 클라이언트 수 (템플릿 변수) |

**Threshold 설정:** 없음(코드에 임계값 검증 없음). throughput/latency 결과 수집·리포트만.

**사용처:** 클러스터 내 TCP/UDP/HTTP 네트워크 성능 측정, CNI·노드 간 대역폭/지연 분석.

**요구사항:** 네트워크 테스트용 이미지/매니페스트(netperf 등). `testing/network/config.yaml` 참고.

**예시:** `testing/network/config.yaml` + override (예: `tcp_protocol_override.yaml`, `1_1_ratio_override.yaml`).

---

## 22. APIResponsivenessPrometheus

**설명:** Prometheus에 수집된 API 호출 레이턴시를 기반으로 리소스/verb/scope별 백분위를 계산하고, SLO 위반 여부를 검사합니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `action` | 예 | `start` \| `gather` |
| `summaryName` | 아니오 | 요약 이름 (기본: measurement 이름) |
| `allowedSlowCalls` | 아니오 | 허용 slow call 개수 (기본 0) |
| `useSimpleLatencyQuery` | 아니오 | 단순 레이턴시 쿼리 사용 (기본 false) |
| `customThresholds` | 아니오 | 리소스별 커스텀 임계값 문자열 |

**Threshold 설정:** 가능. `allowedSlowCalls`: 허용 slow call 개수. `customThresholds`: API 호출별 임계값(기본은 single 1s, list 30s). perc50/90/99 검증, 초과 시 실패.

**사용처:** Kubernetes API 레이턴시 SLO 검증, 공식 SLO 문서와 연동, 프로덕션 API 응답성 모니터링.

**요구사항:** Prometheus 서버. apiserver 요청 레이턴시 메트릭 필요.

---

## 23. ResourceUsageSummary

**설명:** 컴포넌트(apiserver, scheduler 등)별 리소스 사용량을 수집해 90/99/100 백분위 요약을 냅니다. 제약 파일로 SLO 검증 가능합니다.

**Params:** action: start/gather. `resourceConstraints`: 제약 YAML 파일 경로(컨테이너별 CPU/Memory 상한).

**Threshold 설정:** 가능. `resourceConstraints` 파일로 컨테이너별 CPUConstraint/MemoryConstraint 지정. 초과 시 MetricViolationError.

**사용처:** 컨트롤 플레인 리소스 사용 SLO, 용량 계획, 리소스 회귀 검증.

**요구사항:** 메트릭 수집 경로(일반적으로 Prometheus 또는 메트릭 grabber).

---

## 24. EtcdMetrics

**설명:** etcd 메트릭 및 DB 크기를 수집합니다.

**Params:** action: start/gather, waitTime 등. etcd 접근 경로 설정 필요.

**Threshold 설정:** 없음(코드에 임계값 검증 없음). 메트릭·DB 크기 수집·리포트만.

**사용처:** etcd 성능·용량 모니터링, 스토리지 부하 테스트 결과 해석.

**요구사항:** etcd 접근 가능(보통 테스트 환경에서 etcd 포트/인증 설정).

---

## 25. CPUProfile

**설명:** 지정 컴포넌트의 CPU pprof 프로파일을 수집합니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `action` | 예 | `start` \| `gather` |
| `componentName` | 예 | 대상 컴포넌트 (예: `kube-apiserver`) |

**Threshold 설정:** 없음. 프로파일 수집·저장만 수행.

**사용처:** apiserver/scheduler 등 CPU 병목 분석, 프로파일 기반 최적화.

**요구사항:** 해당 컴포넌트에 pprof 노출 및 접근 경로.

**예시:**

```yaml
- name: Start CPU profile
  measurements:
  - Identifier: CPUProf
    Method: CPUProfile
    Params:
      action: start
      componentName: kube-apiserver
- name: Gather CPU profile
  measurements:
  - Identifier: CPUProf
    Method: CPUProfile
    Params:
      action: gather
```

---

## 26. MemoryProfile

**설명:** 지정 컴포넌트의 heap(메모리) pprof 프로파일을 수집합니다.

**Params:** CPUProfile과 동일 (`action`, `componentName`).

**Threshold 설정:** 없음. 프로파일 수집만 수행.

**사용처:** 메모리 사용·누수 분석, heap 프로파일 기반 디버깅.

**요구사항:** 해당 컴포넌트에 pprof 노출.

---

## 27. BlockProfile

**설명:** 지정 컴포넌트의 block pprof 프로파일을 수집합니다.

**Params:** CPUProfile과 동일 (`action`, `componentName`).

**Threshold 설정:** 없음. 프로파일 수집만 수행.

**사용처:** 블로킹/동기화 병목 분석, goroutine 대기 프로파일.

**요구사항:** 해당 컴포넌트에 pprof 노출.

---

## 28. PodPeriodicCommand

**설명:** 지정 라벨의 Pod 안에서 주기적으로 명령을 실행하고 출력을 수집합니다. CPU/메모리 프로파일 폴링 등에 사용됩니다.

**Params:**

| 파라미터 | 필수 | 설명 |
|----------|------|------|
| `action` | 예 | `start` \| `gather` |
| `interval` | 예 | 명령 실행 주기 |
| `command` / `timeout` 등 | 예 | Pod 내 실행할 명령 정의 (배열) |

**Threshold 설정:** 없음. 주기적 명령 출력 수집만 수행.

**사용처:** Pod 내 주기적 명령 실행(프로파일 폴링, 헬스체크 등), e2e 메트릭 수집.

**요구사항:** 대상 Pod 존재 및 명령 실행 가능.

---

## 29. ClusterOOMsTracker

**설명:** 클러스터에서 OOM(Out of Memory) 발생 이벤트를 추적합니다.

**Params:** action: start/gather. 설정에 따라 무시할 프로세스 목록 등.

**Threshold 설정:** 없음(또는 코드 내 허용 OOM 횟수). OOM 발생 여부·횟수 리포팅.

**사용처:** 메모리 부하 테스트 중 OOM 감지, 클러스터 안정성·리소스 한도 검증.

**요구사항:** OOM 이벤트 수집 경로(예: Prometheus 또는 노드 로그).

---

## 30. NodeLocalDNSLatencyPrometheus

**설명:** NodeLocal DNS 레이턴시를 Prometheus 메트릭으로 쿼리해 임계값(threshold)과 비교합니다.

**Params:** `threshold`(duration, 기본 5s). action: start/gather. Prometheus 필요.

**Threshold 설정:** 가능. `threshold`: perc99 등 레이턴시 상한. 초과 시 실패.

**사용처:** NodeLocal DNS SLO 검증, CoreDNS/NodeLocal DNS 성능 모니터링.

**요구사항:** Prometheus, NodeLocal DNS 메트릭 노출.

---

## 31. NetworkPolicyEnforcement

**설명:** 네트워크 정책 적용 지연(정책 생성부터 적용 반영까지)을 측정합니다.

**Params:** action: start/gather, targetLabelValue, testClientNodeSelectorValue, testType 등.

**Threshold 설정:** 코드에 따라 가능할 수 있음(임계값 검증 여부는 소스 참고).

**사용처:** NetworkPolicy 적용 레이턴시 SLO, CNI·네트워크 정책 성능 검증.

---

## 32. NegLatency

**설명:** GCP NEG(Network Endpoint Group) 레이턴시를 측정합니다.

**Params:** GCP NEG 관련 설정.

**Threshold 설정:** 코드 확인 필요. 레이턴시 측정·리포트 위주.

**사용처:** GCP NEG 기반 서비스 레이턴시, 인그레스/엔드포인트 그룹 성능 검증.

---

## 33. MetricsServerPrometheus

**설명:** metrics-server 관련 Prometheus 메트릭을 쿼리합니다.

**Threshold 설정:** 없음(코드에 임계값 검증 없음). 메트릭 수집·리포트.

**사용처:** metrics-server 동작·성능 모니터링, HPA/리소스 메트릭 파이프라인 검증.

---

## 34. LoadBalancerNodeSyncLatency

**설명:** L4 LoadBalancer 서비스의 노드 동기화 레이턴시를 측정합니다.

**Params:** action: start/gather, labelSelector, waitTimeout 등.

**Threshold 설정:** 코드 확인 필요. 레이턴시 측정 위주.

**사용처:** L4 LB 백엔드 동기화 SLO, 클라우드 LB 컨트롤러 성능 검증.

---

## 35. KubeStateMetricsLatency

**설명:** Kube State Metrics(KSM) 수집/반영 지연을 측정합니다.

**Params:** action: start/gather.

**Threshold 설정:** 코드 확인 필요. 지연 측정·리포트 위주.

**사용처:** KSM 메트릭 신선도 검증, 모니터링 파이프라인 지연 분석.

---

## 36. GenericPrometheusQuery

**설명:** 사용자 정의 PromQL 쿼리를 실행하고 결과를 검증/리포트합니다.

**Params:** action: start/gather. `queries[].name`, `queries[].query`, `queries[].threshold`(선택), `queries[].lowerBound` 등.

**Threshold 설정:** 가능. 각 쿼리별 `threshold`(float64), `lowerBound`(true면 값 < threshold 시 실패). 초과 시 에러.

**사용처:** 커스텀 SLO/메트릭 검증, Prometheus 기반 통합 검사.

---

## 37. DNSPerformanceK8sHostnames

**설명:** Kubernetes 내부 호스트명에 대한 DNS 조회 성능을 측정합니다.

**Params:** 테스트용 DNS 클라이언트/쿼리 설정. ServiceAccount 등 권한 리소스 생성.

**Threshold 설정:** 코드 확인 필요. 레이턴시 측정 위주.

**사용처:** 클러스터 내 DNS 성능·SLO, CoreDNS/NodeLocal DNS 비교.

---

## 38. ContainerRestarts

**설명:** Prometheus 메트릭으로 컨테이너 재시작 횟수를 집계하고, 허용 재시작 수(override)를 초과하면 실패 처리합니다.

**Params:** `defaultAllowedRestarts`(기본 0), override로 컨테이너별 허용 횟수 등.

**Threshold 설정:** 가능. `defaultAllowedRestarts` 및 override로 컨테이너별 최대 허용 재시작 횟수. 초과 시 실패.

**사용처:** 재시작 SLO 검증, 워크로드 안정성·크래시 검증.

---

## 39. CiliumEndpointPropagationDelay

**설명:** Cilium CNI 환경에서 Endpoint 전파 지연을 측정합니다.

**Threshold 설정:** 코드 확인 필요. 전파 지연 측정 위주.

**사용처:** Cilium 엔드포인트 전파 SLO, CNI 성능 검증.

---

## 40. ChaosMonkey

**설명:** 테스트 중 컴포넌트 장애(예: Pod kill)를 시뮬레이션합니다.

**Params:** action: start/stop 등, chaos 설정.

**Threshold 설정:** 해당 없음(장애 유발용). 성공/실패 임계값 없음.

**사용처:** 장애 주입 테스트, 복구·가용성 시나리오와 APIAvailability 등과 조합.

---

## 41. TestMetrics

**설명:** 번들 테스트용 메트릭 수집/리포트입니다.

**Threshold 설정:** 없음(번들 내부 용도).

**사용처:** 특정 테스트 번들 내 메트릭 집계·리포트.

---

## 42. SLOMeasurement

**설명:** 여러 SLO를 묶어서 실행·검증하는 복합 measurement입니다.

**Params:** 내부 SLO 정의에 따름. 각 하위 measurement의 threshold 등 전달 가능.

**Threshold 설정:** 가능. 내부 SLO별 threshold/params로 제어(예: SchedulingThroughput threshold, PodStartupLatency perc*Threshold).

**사용처:** 통합 SLO 스위트, Kubernetes 공식 SLO 문서 기반 일괄 검증.

---

## 43. ResourceClaimAllocationLatency

**설명:** DRA(ResourceClaim) 할당 레이턴시를 측정하고 perc50/90/99 임계값으로 검증합니다.

**Params:** `threshold`(기본 5m), `perc50Threshold`/`perc90Threshold`/`perc99Threshold`, labelSelector.

**Threshold 설정:** 가능. `threshold` 및 백분위별 threshold. gather 시 VerifyThresholdByPercentile, 초과 시 실패.

**사용처:** DRA 할당 SLO, 리소스 클레임 성능 검증.

---

## 44. NetworkProgrammingLatency

**설명:** kube-proxy 네트워크 프로그래밍 지연을 Prometheus 메트릭으로 쿼리합니다.

**Params:** `threshold`(duration, 필수). action: start/gather.

**Threshold 설정:** 가능. `threshold`: perc99 등 상한. VerifyThreshold로 검증, 초과 시 실패.

**사용처:** kube-proxy 프로그래밍 SLO, 서비스/엔드포인트 반영 지연 검증.

---

## 45. WindowsResourceUsagePrometheus

**설명:** Windows 노드의 리소스 사용량을 Prometheus로 수집합니다.

**Threshold 설정:** 코드 확인 필요. 리소스 수집·리포트 위주.

**사용처:** Windows 노드 리소스 모니터링, 하이브리드 클러스터 검증.

---

## 46. InClusterNetworkLatency

**설명:** 클러스터 내 네트워크 레이턴시(핑 등)를 프로브로 측정합니다. Probes 매니페스트와 Prometheus가 필요합니다.

**Params:** action: start/gather, replicasPerProbe, checkProbesReadyTimeout, `threshold`(duration, 선택) 등.

**Threshold 설정:** 가능. `threshold` 지정 시 perc99 검증, 초과 시 실패.

**사용처:** 클러스터 내 네트워크 레이턴시 SLO, CNI·노드 간 지연 검증.

---

## 47. DnsLookupLatency

**설명:** DNS 조회 레이턴시를 프로브로 측정합니다.

**Params:** Probes 공통 (action, `threshold` 등).

**Threshold 설정:** 가능. `threshold` 지정 시 레이턴시 검증.

**사용처:** DNS 조회 SLO, CoreDNS/NodeLocal DNS 성능 검증.

---

## 48. InClusterAPIServerRequestLatency

**설명:** 클러스터 내에서 apiserver 요청 레이턴시를 프로브로 측정합니다.

**Params:** Probes 공통 (action, threshold 등).

**Threshold 설정:** 가능. Probes 공통 threshold 검증.

**사용처:** 클러스터 내 API 레이턴시 SLO, 네트워크/인증 경로 검증.

---

## 49. DnsPropagation

**설명:** DNS 전파 시간을 프로브로 측정합니다.

**Params:** Probes 공통 (action, threshold 등).

**Threshold 설정:** 코드 확인 필요. 전파 시간 측정·리포트 위주.

**사용처:** DNS 전파 SLO, 서비스/파드 DNS 반영 시간 검증.

---

## 요약 표: Threshold 설정 가능 여부 · ol-cat 포함 · 요구사항

| Method | Threshold 설정 | ol-cat 시나리오 | Prometheus | 기타 요구사항 |
|--------|----------------|-----------------|------------|----------------|
| Sleep | 없음 | sleep | 아니오 | 없음 |
| Timer | 없음 | timer | 아니오 | 없음 |
| Exec | 없음 | exec-echo | 아니오 | 없음 |
| WaitForRunningPods | 없음 | wait-for-running-pods | 아니오 | 없음 |
| WaitForNodes | 없음 | wait-for-nodes | 아니오 | 없음 |
| WaitForFinishedJobs | 없음 | wait-for-finished-job | 아니오 | 없음 |
| WaitForGenericK8sObjects | 없음 | wait-for-generic-deployment | 아니오 | 없음 |
| WaitForControlledPodsRunning | 없음 | wait-for-controlled-pods | 아니오 | 없음 |
| PodStartupLatency | 가능 (threshold, perc50/90/99) | scheduling-throughput-minimal | 아니오 | 없음 |
| SchedulingThroughput | 가능 (threshold) | scheduling-throughput-minimal | 아니오 | 없음 |
| ServiceCreationLatency | 없음 | service-creation-latency | 아니오 | **--enable-exec-service**, 백엔드 수신 |
| JobLifecycleLatency | 없음 | job-lifecycle-latency | 아니오 | 없음 |
| APIAvailability | 가능 (threshold 가용률) | api-availability | 아니오 | host 폴링 시 exec-service |
| MetricsForE2E | 없음 | metrics-for-e2e | 아니오 | 없음 |
| NetworkPerformanceMetrics | 없음 | network-tcp-1-1 | 아니오 | 네트워크 테스트 이미지/설정 |
| SystemPodMetrics | 가능 (restartCountThresholdOverrides, enableRestartCountCheck) | (미포함) | 아니오 | Params systemPodMetricsEnabled 등 |
| SchedulingThroughputPrometheus | 가능 (threshold) | (미포함) | 예 | apiserver binding 메트릭 |
| APIResponsivenessPrometheus | 가능 (customThresholds, allowedSlowCalls) | (미포함) | 예 | apiserver 레이턴시 메트릭 |
| ResourceUsageSummary | 가능 (resourceConstraints) | (미포함) | 선택 | 메트릭 grabber |
| NodeLocalDNSLatencyPrometheus | 가능 (threshold) | (미포함) | 예 | NodeLocal DNS 메트릭 |
| ContainerRestarts | 가능 (defaultAllowedRestarts 등) | (미포함) | 예 | container 메트릭 |
| GenericPrometheusQuery | 가능 (쿼리별 threshold) | (미포함) | 예 | - |
| ResourceClaimAllocationLatency | 가능 (threshold, perc*) | (미포함) | 아니오 | DRA |
| NetworkProgrammingLatency | 가능 (threshold) | (미포함) | 예 | kubeproxy 메트릭 |
| Probes(46–49) | 가능 (threshold 등) | (미포함) | 예 | Probes 매니페스트 |
| 그 외 | 코드 참고 | (미포함) | 대부분 필요 또는 특수 | GCP/Windows/Cilium 등 |

정확한 파라미터·기본값은 각 measurement 소스 파일(`pkg/measurement/common/...`)을 참고하세요.

---

## 검토 및 팩트체크 (코드·가이드 기준)

이 문서는 코드(`pkg/measurement/`) 및 README/가이드와 대조해 검토·수정했습니다.

### 검증한 항목 (코드 근거)

| 항목 | 근거 |
|------|------|
| Sleep | `duration` 필수, `GetString(config.Params, "duration")` (sleep.go) |
| Timer | action `start`/`stop`/`gather`, start/stop 시 `label` 필수 (timer.go) |
| Exec | `command` 배열 필수, `timeout` 기본 1h, `retries` 기본 1, `backoffDelay` 기본 1s (exec.go) |
| WaitForRunningPods | `desiredPodCount` 필수, `timeout` 60s, `refreshInterval` 5s, `isFatal` false, ObjectSelector (wait_for_pods.go, util/selector.go) |
| WaitForNodes | `minDesiredNodeCount`/`maxDesiredNodeCount` 필수, timeout 30m, refreshInterval 30s (wait_for_nodes.go) |
| WaitForFinishedJobs | timeout 기본 **10m** (`defaultWaitForFinishedJobsTimeout`, wait_for_jobs.go) |
| WaitForGenericK8sObjects | `successfulConditions`/`failedConditions`/`minDesiredObjectCount`/`maxFailedObjectCount` 필수, timeout 30m (wait_for_generic_k8s_object.go) |
| PodStartupLatency | threshold 기본 5s, perc50/90/99 기본값=threshold, VerifyThresholdByPercentile (slos/pod_startup_latency.go) |
| SchedulingThroughput | threshold 기본 0, **enableViolations** 기본 true (scheduling_throughput.go) |
| APIAvailability | threshold 기본 0, gather 시 `sli < a.threshold`이면 MetricViolationError (api_availability_measurement.go) |
| ServiceCreationLatency | action start/waitForReady/waitForDeletion/gather, waitTimeout 기본 10m, exec-service 사용 (service_creation_latency.go) |
| JobLifecycleLatency | timeout 기본 **10m** (defaultWaitForFinishedJobsTimeout 사용, job_lifecycle_latency.go) |
| SystemPodMetrics | **Params** `systemPodMetricsEnabled`(기본 false), `enableRestartCountCheck`(기본 false), `restartCountThresholdOverrides`(YAML 맵). 플래그가 아닌 Params로 활성화 (system_pod_metrics.go) |
| NodeLocalDNSLatencyPrometheus | threshold 기본 5s, **Perc99** 상한 검증 (nodelocaldns_latency_prometheus.go) |
| NetworkProgrammingLatency | **threshold 필수** (`GetDuration`), enableViolations 기본 false (slos/network_programming.go) |
| Probes(46–49) | threshold GetDurationOrDefault 0, threshold>0일 때 VerifyThreshold (probes/probes.go) |

### 수정 반영 사항

1. **WaitForFinishedJobs, JobLifecycleLatency**  
   timeout 기본값을 문서에서 15m으로 적었으나 코드는 **10m** (`defaultWaitForFinishedJobsTimeout = 10 * time.Minute`). → **10m**으로 수정.
2. **SystemPodMetrics**  
   "플래그 --enable-system-pod-metrics"라고 했으나, 코드에는 해당 플래그 없음. 수집·검사 활성화는 **Params** `systemPodMetricsEnabled`, `enableRestartCountCheck`, `restartCountThresholdOverrides`로 함. → Params 기준으로 문단·요약표 수정 및 파라미터 표 보강.
3. **SchedulingThroughput**  
   `enableViolations` 파라미터(기본 true, false면 threshold 위반 시 에러 무시)가 코드에 있으나 문서에 없었음. → Params 및 Threshold 설명에 추가.

### 참고

- 공통 셀렉터(namespace, labelSelector, fieldSelector)는 `pkg/util/selector.go`의 `ObjectSelector.Parse(params)`로 파싱됩니다.
- Prometheus 기반 measurement는 `CreatePrometheusMeasurement` 래퍼를 쓰며, action `start`/`gather`와 시간 구간 내 쿼리로 동작합니다.
- 일부 measurement(예: NetworkPolicyEnforcement, NegLatency, Cilium 등)는 코드만 간단히 확인했으며, 상세 파라미터는 해당 소스 파일을 직접 참고해야 합니다.

---

## 관련 문서

- **목록·분류·ol-cat 포함 여부:** [testing/cat/MEASUREMENTS-REVIEW.md](../testing/cat/MEASUREMENTS-REVIEW.md)
- **CAT 시나리오 개요:** [testing/cat/README.md](../testing/cat/README.md)
- **소스 코드:** `clusterloader2/pkg/measurement/common/` 및 하위(`slos/`, `probes/`, `network/`, `dns/` 등)
