# CAT 프로토타입 트러블슈팅 가이드

ol-cat 스위트 및 testing/cat 시나리오 실행·개발 중 자주 겪은 문제와 해결 방법을 정리했습니다. 동일 증상이 나오면 아래에서 해당 항목을 찾아 참고하세요.

---

## 1. 실행이 끊긴 뒤 파드·네임스페이스가 클러스터에 남음

### 증상
- 타임아웃이나 Ctrl+C로 CL2를 중단했는데, `load-test-*`(또는 cluster-loader prefix) 네임스페이스와 그 안의 파드가 삭제되지 않고 남아 있음.

### 원인
- ClusterLoader2는 **각 시나리오의 `ExecuteTest()`가 정상 return될 때** `defer cleanupTestResourcesAndDependencies()`로 automanaged 네임스페이스를 삭제함.
- 프로세스가 **타임아웃·강제 종료**로 죽으면 해당 시나리오의 `ExecuteTest()`가 return되지 않아 defer가 실행되지 않고, 그 시나리오에서 만든 리소스가 그대로 남음.

### 해결
- 남은 네임스페이스 확인 후 수동 삭제.

```bash
kubectl get ns | grep load-test
kubectl delete ns <namespace-name>
```

- 여러 개면 한 번에: `kubectl get ns -o name | grep load-test | xargs kubectl delete`

---

## 2. 어떤 시나리오가 성공/실패했는지, 결과를 어떻게 보는지 모름

### 증상
- 스위트를 돌렸는데 어디서 통과했고 어디서 실패했는지, 또는 수치 결과를 어디서 보는지 헷갈림.

### 확인 방법

| 목적 | 방법 |
|------|------|
| 시나리오별 성공/실패 | **실행 로그(stdout)**: 시나리오마다 `Test Finished` / `Test: ...` / `Status: Success` 또는 `Status: Fail` / 실패 시 `Errors: ...` 출력됨. |
| 결과 파일로 보기 | 실행 시 **`--report-dir=/path`** 지정. 해당 디렉터리에 `junit.xml`, `generatedConfig_<이름>.yaml`, measurement 요약 파일 등이 생성됨. |
| JUnit으로 보기 | `--report-dir` 아래 **junit.xml**. 각 시나리오(또는 스텝)가 `<testcase>` 하나, 소요 시간·실패 메시지 포함. |

### 주의
- **junit.xml은 스위트가 정상 종료될 때** 한 번에 기록됨. 타임아웃 등으로 중간에 끊기면 junit이 없거나 불완전할 수 있음.
- 그럴 때는 **끊기기 직전까지 출력된 stdout의 Success/Fail** 이 유일한 결과일 수 있음. 실행 시 `2>&1 | tee run.log` 로 로그를 남겨 두면 나중에 확인하기 좋음.

---

## 3. service-creation-latency: 테스트가 끝나지 않거나, perc50/90/99 가 0s만 반복 출력

### 증상
- `Running service-creation-latency(...)` 다음에 `ServiceCreationLatency: perc50: 0s, perc90: 0s, perc99: 0s` 가 여러 줄 반복되고, 테스트가 끝나지 않는 것처럼 보이거나, 결과가 전부 0s로 나옴.

### 원인 1: 백엔드가 Service 포트에서 수신하지 않음
- ServiceCreationLatency measurement는 Service가 생성된 뒤 **exec-service 파드에서 ClusterIP:포트로 curl**을 보내, **연속 3번 성공**해야 “도달 가능(reachability)”으로 기록함.
- Deployment에서 **pause** 같은 “아무 것도 안 듣는” 이미지를 쓰고 있으면 curl이 계속 실패하고, measurement 워커가 **무한 재시도**에 빠짐. reachability가 세팅되지 않아 gather에서도 0s 또는 빈 데이터만 나옴.

### 해결 1
- Deployment 이미지를 **해당 Service 포트에서 실제로 HTTP 등을 응답하는 이미지**로 변경.
- 예: `registry.k8s.io/e2e-test-images/agnhost:2.45` + `args: ["serve-hostname"]` (기본 포트 9376). Service의 port/targetPort를 9376으로 맞춤.
- 참고: [testing/cat/deployment-for-svc.yaml](deployment-for-svc.yaml), [service-clusterip.yaml](service-clusterip.yaml).

### 원인 2: exec-service 미사용
- ServiceCreationLatency는 **`--enable-exec-service`** 가 켜져 있어야 동작함. 코드에서 꺼져 있으면 start/gather 모두 `"enable-exec-service flag not enabled"` 로 실패함.

### 해결 2
- 스위트 실행 시 **반드시 `--enable-exec-service`** 추가.

```bash
./run-e2e.sh --testsuite=ol-cat.yaml --provider=skeleton --kubeconfig=$HOME/.kube/config --enable-exec-service
```

---

## 4. service-creation-latency: Deployment 이미지 ImagePullBackOff

### 증상
- Deployment 파드가 `ImagePullBackOff` / `ErrImagePull` (예: nginx 이미지 풀 실패, “failed to read expected number of bytes: unexpected EOF”).

### 원인
- `docker.io/library/nginx` 등 외부 레지스트리 풀 제한·네트워크 문제·레지스트리 정책으로 노드에서 이미지를 당겨오지 못함.

### 해결
- **registry.k8s.io** 의 e2e 이미지로 바꿔서 사용. 같은 클러스터에서 CL2 exec-service가 쓰는 레지스트리와 맞추면 풀 성공 가능성이 높음.
- 예: `registry.k8s.io/e2e-test-images/agnhost:2.45` (또는 2.32). agnhost는 `serve-hostname`으로 포트 9376에서 HTTP 응답하므로, Service를 9376으로 맞추면 ServiceCreationLatency의 curl 검사가 동작함.
- **이미지 풀 가능 여부**는 사용 환경에서 한 번 확인하는 것이 좋음: `docker pull registry.k8s.io/e2e-test-images/agnhost:2.45`

---

## 5. CAT와 무관한 변경이 섞여 있음 (커밋/PR 정리)

### 증상
- ol-cat 프로토타입 작업 중 네트워크·프로메테우스·adhoc·docs 등 다른 용도의 수정/추가가 함께 포함되어 있음.

### 해결
- **ol-cat 관련만 남기고 나머지 제거**:
  - **수정된 파일 되돌리기**: `git checkout -- <path>` (예: network 매니페스트, prometheus 매니페스트, testing/network/config.yaml).
  - **삭제된 파일 복원**: `git checkout -- clusterloader2/pkg/prometheus/manifests/master-ip/` 등.
  - **추적 안 하는 디렉터리/파일 삭제**: `adhoc/`, `docs/`, `clusterloader2/generatedConfig_*.yaml` 등 불필요한 untracked 항목 `rm -rf` 또는 수동 삭제.
- 남겨 둘 것: `clusterloader2/ol-cat.yaml`, `clusterloader2/testing/cat/` 아래 시나리오·템플릿.

---

## 6. 참고: 관련 코드·파일 위치

| 항목 | 위치 |
|------|------|
| 시나리오 성공/실패 출력 | `clusterloader2/cmd/clusterloader.go`: `printTestResult`, `runSingleTest` |
| 네임스페이스 정리 | `clusterloader2/pkg/test/simple_test_executor.go`: `cleanupTestResourcesAndDependencies`, `DeleteAutomanagedNamespaces` |
| ServiceCreationLatency (exec-service 필수, pingChecker) | `clusterloader2/pkg/measurement/common/service_creation_latency.go` |
| perc50/90/99 로그 출력 | `clusterloader2/pkg/measurement/util/phase_latency.go`: `printLatencies` |
| JUnit·report-dir | `clusterloader2/cmd/clusterloader.go`: `CreateSimpleReporter`, `report-dir` 플래그 |

---

이 가이드는 CAT 프로토타입(ol-cat.yaml, testing/cat/*) 기준으로 작성되었습니다. 다른 스위트나 measurement에서 비슷한 증상이 나오면 원인·해결 흐름을 참고해 적용하면 됩니다.
