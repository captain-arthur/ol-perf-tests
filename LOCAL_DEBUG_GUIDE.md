# 네트워크 성능 측정 워커 로컬 디버깅 가이드

이 가이드는 `clusterloader2`를 통하지 않고, Docker를 사용하여 네트워크 측정 도구(`siege`, `iperf`)의 동작을 로컬에서 직접 확인하는 방법을 설명합니다.

## 0. 베이스 이미지 정보
모든 테스트는 프로젝트에서 사용하는 공식 이미지를 사용합니다.
- **Image**: `gcr.io/k8s-testimages/netperfbenchmark:0.3`

---

## 1. HTTP 테스트 (`siege`) 디버깅
HTTP 측정값이 `0`으로 나오거나 파싱 에러(`lack of Transactions: line`)가 발생할 때 사용합니다.

### 서버 (Mock) 실행
컨테이너 내부에 5301 포트로 응답하는 임시 서버를 띄웁니다.
```bash
docker run --name cl2-http-server -d alpine sh -c "while true; do echo -e 'HTTP/1.1 200 OK\nContent-Length: 5\n\nHello' | nc -l -p 5301; done"
```

### 클라이언트 (Siege) 실행 및 결과 확인
서버의 IP를 확인한 후, 워커 이미지의 `siege`를 직접 호출합니다.
```bash
# 서버 IP 확인
SERVER_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' cl2-http-server)

# Siege 직접 실행 (우리가 설정한 최적의 인자 적용)
docker run --rm gcr.io/k8s-testimages/netperfbenchmark:0.3 \
  siege http://$SERVER_IP:5301/test -c10 -t5S -b -q
```
> **Tip**: `-q`를 빼고 실행하면 개별 요청 로그를 볼 수 있고, `-v`를 넣으면 더 상세한 정보를 볼 수 있습니다.

---

## 2. TCP/UDP 테스트 (`iperf`) 디버깅
TCP 처리량(Throughput) 측정이 실패하거나 연결 오류가 의심될 때 사용합니다.

### 서버 실행
```bash
docker run --name cl2-iperf-server -d gcr.io/k8s-testimages/netperfbenchmark:0.3 iperf -s -f K
```

### 클라이언트 실행
```bash
# 서버 IP 확인
SERVER_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' cl2-iperf-server)

# TCP 테스트 (10초 동안 측정)
docker run --rm gcr.io/k8s-testimages/netperfbenchmark:0.3 \
  iperf -c $SERVER_IP -t 10 -f K
```

---

## 3. 워커 바이너리 자체 실행 (K8s 연동)
워커가 쿠버네티스의 `NetworkTestRequest` 이벤트를 제대로 수신하고 처리하는지 확인합니다.

```bash
docker run -it --rm \
  -v ~/.kube/config:/root/.kube/config \
  -e KUBECONFIG=/root/.kube/config \
  gcr.io/k8s-testimages/netperfbenchmark:0.3 \
  --v=4 --httpClientExtraArguments="-c10 -b -q"
```
*   이 명령어를 실행하면 컨테이너가 로컬 `Kubeconfig`를 사용하여 클러스터에 접속합니다.
*   이 상태에서 `clusterloader2`를 실행하면, 클러스터 내부의 포드가 아닌 **내 로컬의 Docker 컨테이너가** 일을 가져와서 수행하게 됩니다. 로컬 로그 출력을 실시간으로 모니터링하기에 매우 좋습니다.

---

## 4. 유용한 디버깅 팁
1.  **이미지 내부 구경하기**: `docker run -it --rm --entrypoint sh gcr.io/k8s-testimages/netperfbenchmark:0.3`를 실행하여 내부 경로(`/usr/local/bin/siege` 등)를 확인할 수 있습니다.
2.  **로그 파일 가로채기**: 워커는 로그를 `/var/log/*.log`에 남깁니다. `docker run` 시 `-v $(pwd)/logs:/var/log`를 주면 컨테이너 밖에서 로그 파일을 직접 실시간으로 볼 수 있습니다.
