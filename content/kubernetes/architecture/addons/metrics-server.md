# Metrics Server

## 1. Metrics Server란?

Metrics Server는 Kubernetes 클러스터에서 실행 중인 Node와 Pod의 CPU 및 메모리 사용량을 수집하고, Kubernetes API를 통해 제공하는 경량 메트릭 수집 컴포넌트입니다.

Kubernetes는 애플리케이션의 리소스 사용량을 기반으로 워크로드를 자동으로 확장하거나 컨테이너의 리소스 요구량을 조정하는 기능을 제공합니다. 이러한 기능이 동작하려면 현재 Node와 Pod가 사용하는 CPU 및 메모리 정보가 필요합니다.

Metrics Server는 각 Node의 kubelet으로부터 리소스 사용량 정보를 수집하고, 이를 Metrics API를 통해 제공함으로써 Kubernetes의 리소스 기반 자동 확장 기능을 지원합니다.

Metrics Server는 Kubernetes의 핵심 Control Plane 구성 요소가 아닌 Addon에 해당하며, 일반적으로 클러스터 내부에 Deployment 형태로 배포됩니다.

주요 사용 대상은 다음과 같습니다.

- HorizontalPodAutoscaler(HPA): CPU 및 메모리 사용량을 기반으로 Pod 복제본 수를 조정합니다.
- VerticalPodAutoscaler(VPA): 컨테이너의 리소스 사용량을 분석하여 CPU 및 메모리 요청량을 조정하거나 권장합니다.
- kubectl top: Node 및 Pod의 현재 리소스 사용량을 조회합니다.

## 2. 주요 역할

### 2.1. Node 및 Pod 리소스 사용량 수집

Metrics Server는 클러스터에 등록된 Node를 대상으로 kubelet에 접근하여 리소스 사용량 정보를 수집합니다.

주요 수집 대상은 다음과 같습니다.

| 대상 | 수집 정보 |
|---|---|
| Node | CPU 및 메모리 사용량 |
| Pod | Pod에 포함된 컨테이너들의 CPU 및 메모리 사용량 |
| Container | 개별 컨테이너의 CPU 및 메모리 사용량 |

Metrics Server는 일반적으로 15초 간격으로 메트릭을 수집합니다. 수집 주기는 `--metric-resolution` 옵션으로 조정할 수 있으며, kubelet에서 제공하는 메트릭의 해상도 등을 고려해야 합니다.

수집된 데이터는 Metrics Server의 메모리 내에 저장되며, 장기간의 시계열 데이터 저장을 목적으로 하지 않습니다.

### 2.2. Metrics API 제공

Metrics Server는 수집한 리소스 사용량 정보를 Kubernetes Metrics API를 통해 제공합니다.

Metrics API는 Kubernetes API의 확장 기능인 API Aggregation Layer를 통해 노출됩니다.

주요 API 리소스는 다음과 같습니다.

| API 리소스 | 설명 |
|---|---|
| `NodeMetrics` | 개별 Node의 CPU 및 메모리 사용량을 제공합니다. |
| `PodMetrics` | 개별 Pod 및 해당 Pod에 포함된 컨테이너의 CPU 및 메모리 사용량을 제공합니다. |

Metrics API의 일반적인 API 경로는 다음과 같습니다.

```text
/apis/metrics.k8s.io/v1beta1/nodes
/apis/metrics.k8s.io/v1beta1/nodes/{node}
/apis/metrics.k8s.io/v1beta1/pods
/apis/metrics.k8s.io/v1beta1/namespaces/{namespace}/pods
```

Metrics Server가 제공하는 API는 Kubernetes API Server를 통해 접근할 수 있으므로, 별도의 Metrics Server 전용 클라이언트를 구현하지 않고도 Kubernetes API를 사용하는 방식으로 리소스 사용량을 조회할 수 있습니다.

### 2.3. 자동 확장 기능 지원

Metrics Server는 Kubernetes의 자동 확장 기능이 CPU 및 메모리 사용량을 확인할 수 있도록 지원합니다.

대표적으로 HPA는 Metrics API에서 제공하는 Pod 리소스 사용량을 기반으로 현재 워크로드의 리소스 사용률을 계산하고, 설정된 목표 사용률과 비교하여 Pod 복제본 수를 조정합니다.

예를 들어 다음과 같이 HPA를 설정할 수 있습니다.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

위 설정은 대상 Deployment의 Pod 평균 CPU 사용률을 70%로 유지하도록 복제본 수를 조정하는 예시입니다.

HPA는 Metrics Server가 제공하는 CPU 사용량을 조회하고, 컨테이너의 CPU requests를 기준으로 사용률을 계산합니다.

따라서 CPU 기반 HPA가 정상적으로 동작하려면 대상 컨테이너에 적절한 CPU requests가 설정되어 있어야 합니다.

## 3. 아키텍처 및 동작 방식

### 3.1. 전체 구성

Metrics Server는 kubelet에서 리소스 사용량을 수집하고 Kubernetes API Server를 통해 이를 제공하는 구조로 동작합니다.

주요 구성 요소는 다음과 같습니다.

| 구성 요소 | 역할 |
|---|---|
| Container Runtime | 컨테이너의 실제 리소스 사용량 정보를 제공합니다. |
| cAdvisor | 컨테이너의 CPU 및 메모리 사용량을 수집하고 집계하는 기능을 제공합니다. |
| kubelet | Node에서 실행 중인 Pod 및 Container의 리소스 사용량을 제공하는 Node Agent입니다. |
| Metrics Server | 각 Node의 kubelet에서 리소스 사용량을 수집하고 메모리에 저장합니다. |
| kube-apiserver | API Aggregation Layer를 통해 Metrics API 요청을 Metrics Server로 전달합니다. |
| HPA / VPA | Metrics API에서 제공하는 리소스 정보를 활용하여 자동 확장 또는 리소스 조정을 수행합니다. |
| kubectl top | Metrics API를 통해 Node 및 Pod의 리소스 사용량을 조회합니다. |

전체적인 데이터 흐름은 다음과 같습니다.

```text
+--------------------------------------+
|             Kubernetes               |
|                                      |
|  +-------------+                     |
|  |     HPA     |                     |
|  +------+------+                     |
|         |                            |
|         | Metrics API                |
|         v                            |
|  +-------------+                     |
|  | kube-apiserver                    |
|  | (Aggregation Layer)                |
|  +------+------+                     |
|         |                            |
|         v                            |
|  +-------------+                     |
|  | Metrics     |                     |
|  | Server      |                     |
|  +------+------+                     |
|         ^                            |
|         | Resource Metrics           |
|         |                            |
|  +------+------+                     |
|  |    Node     |                     |
|  |             |                     |
|  |  +--------+ |                     |
|  |  | kubelet| |                     |
|  |  +---+----+ |                     |
|  |      ^      |                     |
|  |  +---+----+ |                     |
|  |  |cAdvisor| |                     |
|  |  +---+----+ |                     |
|  |      ^      |                     |
|  |  +---+-----------+                 |
|  |  | Container     |                 |
|  |  | Runtime       |                 |
|  |  +---------------+                 |
|  +-------------+                     |
+--------------------------------------+
```

### 3.2. 리소스 메트릭 수집 과정

Metrics Server가 Node의 리소스 사용량을 수집하는 과정은 다음과 같습니다.

1. Metrics Server는 Kubernetes API를 통해 클러스터의 Node 정보를 확인합니다.
2. Node 객체에 등록된 IP 주소와 kubelet 포트 정보를 확인합니다.
3. Metrics Server는 각 Node의 kubelet에 HTTPS 요청을 전송합니다.
4. kubelet은 컨테이너 런타임 또는 cAdvisor를 통해 수집한 리소스 사용량 정보를 제공합니다.
5. Metrics Server는 응답받은 Node 및 Pod의 CPU와 메모리 사용량을 메모리에 저장합니다.
6. kube-apiserver를 통해 Metrics API 요청이 들어오면 Metrics Server는 저장된 데이터를 반환합니다.

Metrics Server는 일반적으로 kubelet의 `/metrics/resource` 엔드포인트를 사용하여 리소스 사용량 정보를 수집합니다.

```text
Metrics Server
      |
      | 1. Node 정보 조회
      v
kube-apiserver
      |
      | 2. Node IP 및 kubelet 정보
      v
Metrics Server
      |
      | 3. HTTPS GET /metrics/resource
      v
kubelet
      |
      | 4. Container Resource Metrics
      v
Metrics Server
      |
      | 5. In-memory Metrics Cache
      v
Metrics API
```

### 3.3. Metrics API 요청 처리 과정

Metrics Server는 Kubernetes API Aggregation Layer를 통해 Metrics API를 제공합니다.

API Aggregation Layer는 Kubernetes API Server가 기본 제공 API 외에 별도의 확장 API 서버가 제공하는 API도 동일한 Kubernetes API 인터페이스를 통해 노출할 수 있도록 지원하는 구조입니다.

Metrics Server는 `APIService` 리소스를 통해 `metrics.k8s.io` API 그룹의 구현체로 등록됩니다.

일반적인 요청 처리 과정은 다음과 같습니다.

1. 사용자가 `kubectl top pods` 명령을 실행합니다.
2. kubectl은 kube-apiserver에 Pod Metrics API 요청을 전송합니다.
3. kube-apiserver는 APIService에 등록된 Metrics Server로 요청을 전달합니다.
4. Metrics Server는 메모리에 저장된 Pod 및 Container 메트릭을 조회합니다.
5. Metrics Server는 조회 결과를 `PodMetricsList` 형태로 반환합니다.
6. kube-apiserver는 해당 응답을 클라이언트에 전달합니다.
7. kubectl은 반환된 리소스 사용량을 출력합니다.

이러한 구조를 통해 Metrics Server는 Kubernetes API의 인증 및 인가 체계와 API 접근 방식을 활용할 수 있습니다.

## 4. Metrics API

Metrics API는 Kubernetes에서 Node 및 Pod의 리소스 사용량을 조회하기 위한 API입니다.

일반적인 Kubernetes API와 달리 Metrics API는 Kubernetes의 기본 API Server에 내장되어 있지 않으며, Metrics Server와 같은 별도의 API 구현체가 필요합니다.

현재 널리 사용되는 Metrics API 그룹과 버전은 다음과 같습니다.

```text
metrics.k8s.io/v1beta1
```

### 4.1. NodeMetrics

`NodeMetrics`는 개별 Node의 CPU 및 메모리 사용량을 나타내는 리소스입니다.

예를 들어 다음과 같은 명령으로 Node의 리소스 사용량을 조회할 수 있습니다.

```bash
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
```

조회 결과에는 Node의 이름과 측정 시각, 측정 구간 및 리소스 사용량 정보가 포함됩니다.

### 4.2. PodMetrics

`PodMetrics`는 Pod와 해당 Pod에 포함된 개별 컨테이너의 CPU 및 메모리 사용량을 나타내는 리소스입니다.

다음과 같이 특정 Namespace의 Pod 리소스 사용량을 조회할 수 있습니다.

```bash
kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/default/pods
```

PodMetrics는 Pod 전체의 리소스 사용량뿐 아니라 각 컨테이너의 리소스 사용량도 제공합니다.

## 5. 주요 명령어

Metrics Server가 정상적으로 설치되면 `kubectl top` 명령을 통해 리소스 사용량을 확인할 수 있습니다.

### 5.1. Node 리소스 사용량 조회

```bash
kubectl top nodes
```

각 Node의 CPU 및 메모리 사용량을 확인할 수 있습니다.

### 5.2. Pod 리소스 사용량 조회

```bash
kubectl top pods
```

현재 Namespace에 존재하는 Pod들의 CPU 및 메모리 사용량을 확인할 수 있습니다.

### 5.3. 특정 Namespace의 Pod 조회

```bash
kubectl top pods -n production
```

특정 Namespace에 존재하는 Pod들의 리소스 사용량을 조회합니다.

### 5.4. 컨테이너별 리소스 사용량 조회

```bash
kubectl top pods --containers
```

Pod 전체의 사용량뿐 아니라 각 컨테이너의 CPU 및 메모리 사용량을 개별적으로 확인할 수 있습니다.

### 5.5. Label Selector를 활용한 조회

```bash
kubectl top pods -l app=backend
```

Label Selector에 일치하는 Pod들의 리소스 사용량을 조회합니다.

## 6. 주요 특징 및 제한 사항

### 6.1. 단기 리소스 메트릭 제공

Metrics Server는 자동 확장 및 기본적인 리소스 사용량 확인을 목적으로 설계되었습니다.

수집된 메트릭은 메모리에 저장되며, 장기간의 시계열 데이터를 보관하지 않습니다.

따라서 과거 특정 시점의 CPU 및 메모리 사용량을 조회하거나 장기적인 추세를 분석하는 용도로는 적합하지 않습니다.

### 6.2. 제한적인 메트릭 제공

Metrics Server는 기본적으로 CPU 및 메모리 사용량을 제공합니다.

다음과 같은 상세한 모니터링 지표를 제공하기 위한 용도로 설계되지 않았습니다.

- HTTP 요청 수 및 응답 시간
- 애플리케이션별 비즈니스 지표
- 네트워크 트래픽의 상세 분석
- 디스크 I/O 및 파일시스템 사용량에 대한 종합적인 모니터링
- 장기간의 리소스 사용량 추세 분석

이러한 정보가 필요한 경우 Prometheus와 같은 별도의 모니터링 시스템을 사용하는 것이 적절합니다.

### 6.3. 정확한 성능 분석을 위한 용도가 아님

Metrics Server는 자동 확장에 필요한 간소화된 리소스 메트릭을 제공합니다.

따라서 정밀한 성능 분석이나 장기간의 리소스 사용량 분석을 위한 데이터 소스로 사용하는 것은 적합하지 않습니다.

또한 `kubectl top`에서 표시하는 CPU 및 메모리 사용률은 Metrics Server가 직접 계산하는 값이 아니라 조회하는 클라이언트 측에서 리소스 사용량과 requests 또는 capacity 등을 활용하여 계산하는 값입니다.

### 6.4. Prometheus와의 차이점

Metrics Server와 Prometheus는 모두 메트릭을 다루지만 목적과 제공하는 기능이 다릅니다.

| 구분 | Metrics Server | Prometheus |
|---|---|---|
| 주요 목적 | Kubernetes 자동 확장 지원 | 시스템 및 애플리케이션 모니터링 |
| 주요 데이터 | CPU 및 메모리 사용량 | 다양한 시스템 및 애플리케이션 메트릭 |
| 데이터 저장 | 메모리 기반 단기 저장 | 시계열 데이터 저장 |
| 데이터 수집 | kubelet에서 리소스 메트릭 수집 | Exporter 및 HTTP Endpoint에서 메트릭 수집 |
| API 제공 | Kubernetes Metrics API | Prometheus HTTP API |
| 주요 사용 대상 | HPA, VPA, kubectl top | 대시보드, 알림, 성능 분석 |
| 장기 데이터 분석 | 지원하지 않음 | 지원 |

두 컴포넌트는 서로 대체하는 관계가 아니라 목적에 따라 함께 사용할 수 있습니다.

## 7. 다른 Kubernetes 컴포넌트와의 관계

Metrics Server는 Kubernetes의 여러 컴포넌트와 연동하여 리소스 사용량 정보를 수집하고 제공합니다.

| 구성 요소 | 관계 |
|---|---|
| kube-apiserver | API Aggregation Layer를 통해 Metrics API 요청을 Metrics Server로 전달합니다. |
| kubelet | Node 및 Pod의 CPU와 메모리 사용량 정보를 제공합니다. |
| Container Runtime | 컨테이너의 리소스 사용량 정보를 제공합니다. |
| cAdvisor | 컨테이너의 리소스 사용량을 수집하고 집계하는 기능을 제공합니다. |
| APIService | Metrics Server를 `metrics.k8s.io` API 그룹의 구현체로 등록합니다. |
| HPA | Metrics API에서 제공하는 리소스 사용량을 기반으로 Pod 복제본 수를 조정합니다. |
| VPA | 리소스 사용량 정보를 활용하여 컨테이너의 리소스 requests 및 limits 조정을 지원합니다. |
| kubectl | `kubectl top` 명령을 통해 Node 및 Pod의 리소스 사용량을 조회합니다. |
| Prometheus | Metrics Server와 별도로 상세한 모니터링 및 시계열 데이터 수집을 담당할 수 있습니다. |

## 8. 참고자료

- [Kubernetes Documentation - Resource Metrics Pipeline](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)
- [Kubernetes Documentation - Resource Usage Monitoring](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-usage-monitoring/)
- [Kubernetes Documentation - Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Kubernetes Documentation - API Aggregation Layer](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)
- [Kubernetes SIGs - Metrics Server](https://kubernetes-sigs.github.io/metrics-server/)
- [Metrics Server - GitHub Repository](https://github.com/kubernetes-sigs/metrics-server)
- [Metrics Server - FAQ](https://github.com/kubernetes-sigs/metrics-server/blob/master/FAQ.md)