# Chainsaw 래퍼 (ol-proto 2번 단언)

[Chainsaw](https://kyverno.github.io/chainsaw/main/)로 CL2 ol-proto를 실행한 뒤, **2번 시나리오(파드 간 TCP throughput)** 에 대해 report JSON 기반 단언을 추가한다.

## 필요 도구

- **chainsaw**: `brew install kyverno/chainsaw/chainsaw` 또는 [설치 가이드](https://kyverno.github.io/chainsaw/main/quick-start/install/)
- **jq**: JSON 파싱 (throughput 값 추출)
- **clusterloader**: `clusterloader2/` 에서 빌드된 바이너리

## 실행 방법

테스트 파일 기준 **상위 디렉터리가 clusterloader2** 이어야 한다. (Chainsaw 실행 시 작업 디렉터리가 테스트 디렉터리로 설정됨.)  
CL2 실행 시간이 길어 **--exec-timeout** 을 넉넉히 주고, 테스트 파일 이름을 **--test-file** 로 지정한다.

```bash
# clusterloader2/ 에서
cd clusterloader2
chainsaw test ./chainsaw --test-file ol-proto-02-with-assert.yaml --exec-timeout 15m
```

또는 repo 루트(ol-perf-tests)에서:

```bash
chainsaw test ./clusterloader2/chainsaw --test-file ol-proto-02-with-assert.yaml --exec-timeout 15m
```

(스크립트 내 `CL2_DIR=..` 는 테스트 디렉터리 기준으로 `clusterloader2` 를 가리킨다. CL2의 scenario identifier에는 언더바를 쓸 수 없어 `ol-proto.yaml` 에서는 하이픈을 사용한다.)

## 테스트 내용

1. **Step 1**: `./clusterloader --testsuite=ol-proto.yaml --provider=skeleton --report-dir=./report` 실행 → 1번·2번 시나리오 모두 실행되고 `report/` 에 JSON 저장.
2. **Step 2**: `report/*NetworkPerformance*` JSON에서 `dataItems[].labels.Metric == "Throughput"` 인 항목의 `data.value` 를 읽어, **throughput >= MIN_THROUGHPUT** 인지 단언. 기본 MIN_THROUGHPUT=1000000.

## 환경 변수 (선택)

- `MIN_THROUGHPUT`: throughput 하한 (기본 1000000). Chainsaw 테스트 YAML 내 env로 덮어쓸 수 있음.
- `CL2_DIR` / `REPORT_DIR`: 스크립트 내 기본값(.., ../report)을 바꿀 때 사용.

## 2번 단언 가능 여부

- CL2는 NetworkPerformanceMetrics에 threshold를 두지 않아 **자체 단언이 없음**.
- Chainsaw로 **실행 후 report JSON을 파싱해 throughput 하한을 검사**하면, 2번 시나리오에도 단언을 둘 수 있음.
- 이 테스트로 “Chainsaw 래핑으로 2번도 단언 구현 가능한지”를 확인할 수 있으며, 실행 결과 통과 확인됨 (Step 2: report 내 Throughput >= MIN_THROUGHPUT 단언).
