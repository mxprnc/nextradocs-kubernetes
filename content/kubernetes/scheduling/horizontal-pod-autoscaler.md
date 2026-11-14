# HorizontalPodAutoscaler (HPA)

## 1. HorizontalPodAutoscaler란?

HorizontalPodAutoscaler(HPA)는 Kubernetes에서 애플리케이션의 부하에 따라 Pod의 개수를 자동으로 증가시키거나 감소시키는 리소스입니다.

일반적으로 Deployment 또는 StatefulSet과 같은 워크로드 리소스를 대상으로 설정하며, CPU 사용률, Memory 사용량 또는 사용자 정의 메트릭 등을 기준으로 적절한 Pod 개수를 유지합니다.

Horizontal Scaling은 기존 Pod에 할당된 CPU나 Memory를 증가시키는 대신, Pod의 복제본(Replica) 개수를 늘려 처리 용량을 확장하는 방식입니다.

예를 들어 트래픽이 증가하면 HPA가 Pod의 개수를 증가시켜 요청을 분산하고, 트래픽이 감소하면 불필요한 Pod를 줄여 컴퓨팅 자원의 낭비를 방지할 수 있습니다.

주요 특징은 다음과 같습니다.

- **자동 스케일링:** 메트릭을 지속적으로 관찰하고 설정된 기준에 따라 Pod의 개수를 자동으로 조정합니다.
- **수평 확장 및 축소:** 부하가 증가하면 Pod를 추가하고, 부하가 감소하면 Pod를 제거합니다.
- **다양한 메트릭 지원:** CPU, Memory뿐만 아니라 Custom Metrics 및 External Metrics를 활용할 수 있습니다.
- **워크로드 리소스 연동:** Deployment, StatefulSet 등 Scale Subresource를 지원하는 워크로드를 대상으로 동작합니다.
- **안정적인 스케일링:** Stabilization Window와 Scaling Policy를 통해 급격한 스케일링이나 반복적인 확장 및 축소를 제어할 수 있습니다.

HPA는 Kubernetes API 리소스이자 Controller로 구현되어 있으며, Control Plane의 Horizontal Pod Autoscaler Controller가 실제 스케일링 작업을 수행합니다.

## 2. Horizontal Scaling과 Vertical Scaling

Kubernetes의 자동 스케일링은 크게 Horizontal Scaling과 Vertical Scaling으로 구분할 수 있습니다.

### 2.1. Horizontal Scaling

Horizontal Scaling은 애플리케이션을 실행하는 Pod의 개수를 증가시키거나 감소시키는 방식입니다.

예를 들어 현재 3개의 Pod가 실행 중인 상태에서 트래픽이 증가하면 6개로 확장하고, 트래픽이 감소하면 다시 3개로 축소할 수 있습니다.

```text
Scale Out

Before:
┌────────┐
│ Pod 1  │
├────────┤
│ Pod 2  │
├────────┤
│ Pod 3  │
└────────┘

After:
┌────────┐
│ Pod 1  │
├────────┤
│ Pod 2  │
├────────┤
│ Pod 3  │
├────────┤
│ Pod 4  │
├────────┤
│ Pod 5  │
├────────┤
│ Pod 6  │
└────────┘
```

- Scale Out: Pod 개수를 증가시켜 처리 용량을 확장합니다.
- Scale In: Pod 개수를 감소시켜 사용하지 않는 자원을 회수합니다.

HPA는 이러한 Horizontal Scaling을 자동으로 수행하는 리소스입니다.

### 2.2. Vertical Scaling

Vertical Scaling은 기존 Pod의 개수를 유지하면서 각 Pod에 할당되는 CPU 및 Memory 등의 리소스를 조정하는 방식입니다.

예를 들어 Pod 3개를 유지하면서 각 Pod에 할당된 CPU와 Memory를 증가시키는 방식입니다.

Kubernetes에서는 VerticalPodAutoscaler(VPA)를 통해 이러한 작업을 자동화할 수 있습니다.

| 구분 | HPA | VPA |
|---|---|---|
| 스케일링 방식 | Horizontal Scaling | Vertical Scaling |
| 조정 대상 | Pod 개수 | Pod의 CPU 및 Memory Requests |
| 확장 방식 | Pod 추가 | 기존 Pod의 리소스 할당량 증가 |
| 주요 목적 | 처리량 및 동시 요청 처리 능력 확장 | 개별 Pod의 리소스 최적화 |

HPA와 VPA는 서로 다른 방식으로 워크로드를 최적화합니다. 두 리소스를 함께 사용할 수도 있지만, 동일한 CPU 또는 Memory 리소스를 기준으로 HPA와 VPA가 동시에 조정하도록 구성하면 서로 간섭할 수 있으므로 주의해야 합니다.

## 3. HPA의 아키텍처

HPA는 Kubernetes Control Plane의 Horizontal Pod Autoscaler Controller와 메트릭 수집 시스템을 통해 동작합니다.

일반적인 구성 요소는 다음과 같습니다.

| 구성 요소 | 역할 |
|---|---|
| HorizontalPodAutoscaler | 스케일링 대상, 최소 및 최대 Pod 개수, 메트릭 목표값 등을 정의하는 API 리소스입니다. |
| HPA Controller | 메트릭을 주기적으로 조회하고 현재 상태와 목표 상태를 비교하여 스케일링을 결정합니다. |
| Metrics Server | CPU 및 Memory 사용량을 수집하여 Resource Metrics API를 통해 제공합니다. |
| Metrics API | HPA Controller가 메트릭을 조회할 수 있도록 API를 제공합니다. |
| API Server | HPA 리소스와 워크로드의 상태를 관리하고 Controller 간 통신을 중개합니다. |
| Deployment / StatefulSet | HPA가 지정한 목표 복제본 개수에 따라 실제 Pod를 생성하거나 제거합니다. |
| ReplicaSet | Deployment가 관리하는 Pod의 복제본 개수를 유지합니다. |

### 3.1. HPA Controller

HPA Controller는 HPA의 핵심 동작을 수행하는 Control Plane 구성 요소입니다.

`kube-controller-manager`에 포함되어 있으며, 설정된 주기에 따라 각 HPA의 메트릭을 확인하고 목표 복제본 개수를 계산합니다.

주요 역할은 다음과 같습니다.

- HPA 리소스에 정의된 스케일링 대상과 메트릭 설정을 확인합니다.
- Metrics API를 통해 현재 메트릭을 조회합니다.
- 현재 메트릭과 목표 메트릭을 비교합니다.
- 스케일링 알고리즘을 사용하여 필요한 Pod 개수를 계산합니다.
- 대상 워크로드의 Scale Subresource를 업데이트합니다.
- 스케일링 기록과 상태 정보를 HPA 리소스에 반영합니다.

HPA Controller는 기본적으로 15초 간격으로 동작합니다. 이 주기는 `kube-controller-manager`의 `--horizontal-pod-autoscaler-sync-period` 옵션으로 조정할 수 있습니다.

### 3.2. Metrics Server

Metrics Server는 Kubernetes 클러스터의 CPU 및 Memory 사용량을 수집하는 구성 요소입니다.

각 Node의 kubelet을 통해 컨테이너 및 Pod의 리소스 사용량을 수집하고, 이를 Resource Metrics API인 `metrics.k8s.io`를 통해 제공합니다.

HPA Controller는 Metrics Server 자체에 직접 접근하는 대신 Metrics API를 통해 필요한 메트릭을 조회합니다.

일반적인 메트릭 조회 구조는 다음과 같습니다.

```text
┌──────────────────────────────┐
│      kube-controller-manager │
│                              │
│   Horizontal Pod Autoscaler  │
│          Controller          │
└──────────────┬───────────────┘
               │
               │ Resource Metrics API
               ▼
┌──────────────────────────────┐
│        API Server            │
│                              │
│     metrics.k8s.io            │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       Metrics Server         │
│                              │
│    CPU / Memory Metrics      │
└──────────────┬───────────────┘
               │
               │ kubelet Metrics
               ▼
┌──────────────────────────────┐
│            Nodes             │
│                              │
│     kubelet / Containers     │
└──────────────────────────────┘
```

Metrics Server는 Kubernetes에 기본으로 설치되는 구성 요소가 아니므로 별도로 설치해야 합니다.

또한 Metrics Server는 기본적인 CPU 및 Memory 메트릭을 제공하는 용도이며, 장기적인 모니터링이나 복잡한 사용자 정의 메트릭 수집 시스템을 대체하는 용도는 아닙니다.

## 4. HPA의 동작 원리

HPA는 현재 메트릭과 사용자가 설정한 목표 메트릭의 차이를 계산하여 필요한 Pod 개수를 결정합니다.

기본적으로 다음과 같은 Control Loop를 반복합니다.

1. HPA Controller가 HPA 리소스를 확인합니다.
2. `scaleTargetRef`에 지정된 워크로드를 조회합니다.
3. 해당 워크로드가 관리하는 Pod를 확인합니다.
4. Metrics API를 통해 현재 메트릭을 수집합니다.
5. 현재 메트릭과 목표 메트릭을 비교합니다.
6. 스케일링 알고리즘을 통해 필요한 Replica 개수를 계산합니다.
7. 최소 및 최대 Replica 제한과 스케일링 정책을 적용합니다.
8. 대상 워크로드의 Scale Subresource를 업데이트합니다.
9. Deployment 등의 Controller가 변경된 Replica 개수에 맞춰 Pod를 생성하거나 제거합니다.

이 과정은 일정한 간격으로 반복되므로 HPA는 이벤트가 발생하는 즉시 반응하는 방식이 아니라 주기적으로 상태를 확인하는 방식으로 동작합니다.

### 4.1. HPA 스케일링 계산 공식

HPA의 기본 스케일링 계산 공식은 다음과 같습니다.

\[
desiredReplicas =
\left\lceil
currentReplicas \times
\frac{currentMetricValue}{desiredMetricValue}
\right\rceil
\]

각 항목의 의미는 다음과 같습니다.

| 항목 | 설명 |
|---|---|
| desiredReplicas | 새롭게 설정할 목표 Pod 개수입니다. |
| currentReplicas | 현재 실행 중인 Pod 개수입니다. |
| currentMetricValue | 현재 관측된 메트릭 값입니다. |
| desiredMetricValue | 사용자가 설정한 목표 메트릭 값입니다. |

계산 결과는 소수점 이하를 올림하여 정수로 변환합니다.

예를 들어 현재 3개의 Pod가 실행 중이고, Pod당 평균 CPU 사용량이 목표값보다 2배 높은 상황이라면 다음과 같이 계산됩니다.

\[
desiredReplicas =
\left\lceil 3 \times \frac{200}{100} \right\rceil
= 6
\]

따라서 HPA는 목표 Pod 개수를 6개로 설정합니다.

반대로 현재 메트릭이 목표값의 절반이라면 다음과 같이 계산됩니다.

\[
desiredReplicas =
\left\lceil 3 \times \frac{50}{100} \right\rceil
= 2
\]

이 경우 목표 Pod 개수는 2개가 됩니다.

단, 실제 HPA 동작에서는 단순히 위 공식만 적용하는 것이 아니라 다음과 같은 추가 조건을 고려합니다.

- 최소 및 최대 Replica 제한
- 메트릭 오차를 무시하기 위한 Tolerance
- 메트릭이 누락된 Pod에 대한 보정
- 초기화 중이거나 Ready 상태가 아닌 Pod에 대한 처리
- Stabilization Window
- Scaling Policy

따라서 실제 스케일링 결과는 기본 공식의 계산값과 다를 수 있습니다.

### 4.2. Tolerance

Tolerance는 목표 메트릭과 현재 메트릭 사이의 작은 차이로 인해 불필요한 스케일링이 반복되는 것을 방지하는 허용 범위입니다.

기본값은 10%이며, 별도의 설정이 없다면 현재 메트릭과 목표 메트릭의 비율이 0.9~1.1 범위에 있을 때 스케일링을 수행하지 않습니다.

예를 들어 목표 CPU 사용률이 60%라면 다음과 같습니다.

| 현재 CPU 사용률 | 판단 |
|---|---|
| 55% | 스케일링하지 않습니다. |
| 60% | 스케일링하지 않습니다. |
| 63% | 스케일링하지 않습니다. |
| 67% | Scale Out을 검토합니다. |
| 45% | Scale In을 검토합니다. |

Tolerance는 메트릭의 작은 변동으로 인해 Pod 개수가 불필요하게 증가하거나 감소하는 현상을 방지합니다.

`autoscaling/v2`에서는 Kubernetes v1.33부터 도입된 Configurable Tolerance 기능을 사용할 수 있으며, Kubernetes v1.37부터 Stable 상태입니다. 이를 통해 Scale Up과 Scale Down의 허용 범위를 각각 설정할 수 있습니다.

### 4.3. 최소 및 최대 Replica 제한

HPA는 `minReplicas`와 `maxReplicas`를 통해 스케일링 가능한 Pod 개수의 범위를 제한합니다.

```yaml
spec:
  minReplicas: 2
  maxReplicas: 10
```

위 설정에서는 Pod 개수가 최소 2개에서 최대 10개 사이로 유지됩니다.

예를 들어 스케일링 공식에 따라 15개의 Pod가 필요하다고 계산되더라도 `maxReplicas`가 10이라면 실제 목표 Pod 개수는 10개로 제한됩니다.

반대로 부하가 감소하여 1개의 Pod만 필요하다고 계산되더라도 `minReplicas`가 2라면 최소 2개의 Pod를 유지합니다.

`minReplicas`를 생략하면 기본값은 1입니다.

단, 일반적인 HPA 동작에서는 `minReplicas`를 0으로 설정하여 임의로 Scale to Zero를 수행할 수 있는 것은 아닙니다. Scale to Zero는 별도의 조건과 기능 지원이 필요합니다.

## 5. HPA에서 사용하는 메트릭 유형

HPA는 CPU 및 Memory와 같은 기본 리소스 메트릭뿐만 아니라 다양한 사용자 정의 메트릭을 기준으로 스케일링할 수 있습니다.

`autoscaling/v2`에서 지원하는 대표적인 메트릭 유형은 다음과 같습니다.

| Metric Type | 설명 | 예시 |
|---|---|---|
| Resource | Pod의 CPU 및 Memory와 같은 리소스 사용량을 기준으로 스케일링합니다. | CPU 사용률 60% |
| ContainerResource | Pod 내 특정 컨테이너의 리소스 사용량을 기준으로 스케일링합니다. | 특정 컨테이너의 CPU 사용률 70% |
| Pods | Pod별 사용자 정의 메트릭을 기준으로 스케일링합니다. | Pod당 초당 요청 수 |
| Object | Kubernetes 클러스터 내 특정 객체와 관련된 메트릭을 기준으로 스케일링합니다. | Ingress의 초당 요청 수 |
| External | Kubernetes 클러스터 외부에서 제공하는 메트릭을 기준으로 스케일링합니다. | 외부 메시지 큐의 대기 메시지 수 |

### 5.1. Resource Metrics

Resource Metrics는 CPU와 Memory 사용량을 기준으로 스케일링하는 방식입니다.

일반적으로 Metrics Server가 제공하는 `metrics.k8s.io` API를 사용합니다.

CPU 사용률을 기준으로 설정하는 경우 Pod에 정의된 CPU Requests를 기준으로 실제 CPU 사용량의 비율을 계산합니다.

예를 들어 다음과 같은 Pod 설정이 있다고 가정하겠습니다.

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "1000m"
    memory: "1024Mi"
```

이 Pod의 CPU Request는 500m입니다.

실제 CPU 사용량이 300m라면 CPU 사용률은 다음과 같습니다.

\[
CPU\ Utilization =
\frac{300m}{500m} \times 100
= 60\%
\]

HPA의 목표 CPU 사용률이 70%라면 현재 사용률 60%는 목표보다 낮은 상태입니다.

여기서 중요한 점은 CPU 사용률의 기준이 CPU Limit이 아니라 CPU Request라는 것입니다.

또한 CPU Utilization을 기준으로 HPA를 사용하려면 대상 Pod의 모든 관련 컨테이너에 해당 리소스의 Requests가 정의되어 있어야 합니다. 필요한 Requests가 누락되면 해당 메트릭의 Pod 사용률을 계산할 수 없어 스케일링 판단이 제한될 수 있습니다.

Memory 역시 Requests를 기준으로 사용률을 계산할 수 있지만, CPU와 달리 Memory 사용량은 일반적으로 사용량이 감소하지 않는 특성이 있으므로 메모리 기반 Scale In을 설계할 때는 주의해야 합니다.

### 5.2. ContainerResource Metrics

ContainerResource는 Pod 전체의 리소스 사용량이 아니라 Pod 내부의 특정 컨테이너를 대상으로 CPU 또는 Memory 사용률을 계산하는 방식입니다.

예를 들어 하나의 Pod에 애플리케이션 컨테이너와 Sidecar 컨테이너가 함께 실행되는 경우, 애플리케이션 컨테이너의 CPU 사용량만 기준으로 스케일링할 수 있습니다.

```yaml
metrics:
  - type: ContainerResource
    containerResource:
      name: cpu
      container: application
      target:
        type: Utilization
        averageUtilization: 70
```

이 설정에서는 `application` 컨테이너의 CPU 사용률이 70%를 목표로 유지되도록 스케일링합니다.

Sidecar의 CPU 사용량이 높더라도 해당 컨테이너의 사용량은 이 메트릭의 직접적인 스케일링 기준에 포함되지 않습니다.

### 5.3. Pods Metrics

Pods Metrics는 각 Pod에서 측정되는 사용자 정의 메트릭을 기준으로 스케일링하는 방식입니다.

예를 들어 API 서버의 Pod당 초당 요청 수(Requests Per Second)를 기준으로 스케일링할 수 있습니다.

```yaml
metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
```

위 설정에서는 Pod당 평균 초당 요청 수가 100이 되도록 스케일링합니다.

이러한 방식은 CPU 사용량보다 애플리케이션의 실제 처리량이나 업무 부하를 기준으로 스케일링하는 것이 적절한 경우에 활용할 수 있습니다.

### 5.4. Object Metrics

Object Metrics는 Kubernetes 클러스터 내 특정 객체와 관련된 메트릭을 기준으로 스케일링하는 방식입니다.

예를 들어 Ingress 객체에서 발생하는 요청 수를 기준으로 애플리케이션 Pod의 개수를 조정할 수 있습니다.

```yaml
metrics:
  - type: Object
    object:
      describedObject:
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        name: application-ingress
      metric:
        name: requests_per_second
      target:
        type: Value
        value: "1000"
```

위 설정에서는 지정한 Ingress 객체와 관련된 메트릭의 목표값을 1000으로 설정합니다.

Object Metrics는 특정 Kubernetes 객체에 연결된 메트릭을 기준으로 스케일링해야 할 때 활용합니다.

### 5.5. External Metrics

External Metrics는 Kubernetes 클러스터 외부에서 제공하는 메트릭을 기준으로 스케일링하는 방식입니다.

예를 들어 외부 메시지 큐에 대기 중인 메시지 수를 기준으로 Consumer Pod의 개수를 조정할 수 있습니다.

```yaml
metrics:
  - type: External
    external:
      metric:
        name: queue_messages_ready
      target:
        type: AverageValue
        averageValue: "30"
```

위 설정에서는 메시지 큐에 대기 중인 메시지를 Pod당 평균 30개 수준으로 처리할 수 있도록 스케일링합니다.

External Metrics는 다음과 같은 상황에서 활용할 수 있습니다.

- Kafka Consumer의 처리 대기 메시지 수
- AWS SQS의 Queue Depth
- 외부 메시지 브로커의 Pending Message 수
- 외부 서비스에서 제공하는 요청량 및 처리 대기량

External Metrics를 사용하려면 `external.metrics.k8s.io` API를 제공하는 Metrics Adapter가 필요합니다.

### 5.6. Metrics API와 Adapter

HPA가 메트릭을 수집하는 방식은 메트릭 유형에 따라 달라집니다.

| API | 용도 | 일반적인 제공 구성 요소 |
|---|---|---|
| `metrics.k8s.io` | CPU 및 Memory 등의 Resource Metrics | Metrics Server |
| `custom.metrics.k8s.io` | Kubernetes 객체 및 Pod와 관련된 Custom Metrics | Custom Metrics Adapter |
| `external.metrics.k8s.io` | 클러스터 외부에서 제공하는 External Metrics | External Metrics Adapter |

Metrics Server는 기본적인 CPU 및 Memory 사용량을 제공하지만, 애플리케이션별 요청 수나 외부 메시지 큐의 대기 메시지 수 등을 직접 제공하지는 않습니다.

이러한 메트릭을 사용하려면 Prometheus Adapter 등의 별도 Adapter를 구성하거나 해당 메트릭을 제공하는 시스템과 연동해야 합니다.

## 6. HPA 설정 (autoscaling/v2)

HPA는 Kubernetes API의 `autoscaling` 그룹에 정의된 리소스입니다.

현재 일반적으로 사용하는 API 버전은 `autoscaling/v2`입니다.

다음은 Deployment의 CPU 사용률을 기준으로 Pod 개수를 자동 조정하는 HPA 예시입니다.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: application-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: application

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

각 필드의 의미는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| apiVersion | HPA 리소스의 API 버전입니다. |
| kind | 리소스의 종류를 나타내며 `HorizontalPodAutoscaler`를 사용합니다. |
| metadata.name | HPA 리소스의 이름입니다. |
| scaleTargetRef | 스케일링 대상 워크로드를 지정합니다. |
| scaleTargetRef.apiVersion | 대상 워크로드의 API 버전입니다. |
| scaleTargetRef.kind | 대상 워크로드의 종류입니다. |
| scaleTargetRef.name | 대상 워크로드의 이름입니다. |
| minReplicas | 유지할 최소 Pod 개수입니다. |
| maxReplicas | 허용할 최대 Pod 개수입니다. |
| metrics | 스케일링에 사용할 메트릭 목록입니다. |

위 예시에서는 `application`이라는 Deployment를 대상으로 설정합니다.

CPU 평균 사용률이 70%를 초과하거나 감소하는 상황에 따라 Pod 개수를 조정하며, 최소 2개에서 최대 10개까지 확장할 수 있습니다.

### 6.1. scaleTargetRef

`scaleTargetRef`는 HPA가 실제로 스케일링할 워크로드를 지정하는 필드입니다.

```yaml
scaleTargetRef:
  apiVersion: apps/v1
  kind: Deployment
  name: application
```

HPA는 지정된 워크로드의 Scale Subresource를 통해 현재 Replica 개수를 확인하고 새로운 목표 Replica 개수를 설정합니다.

Deployment를 대상으로 설정하면 HPA는 Deployment의 Replica 개수를 변경합니다.

이후 Deployment Controller와 ReplicaSet Controller가 변경된 목표 개수에 맞춰 실제 Pod를 생성하거나 제거합니다.

HPA가 직접 Pod를 생성하거나 삭제하는 것이 아니라, 대상 워크로드의 스케일을 변경한다는 점이 중요합니다.

### 6.2. metrics

`metrics`는 HPA가 스케일링 판단에 사용할 메트릭을 정의하는 필드입니다.

하나 이상의 메트릭을 지정할 수 있으며, 여러 메트릭을 함께 사용하는 경우 각 메트릭에 대해 필요한 Replica 개수를 계산한 후 가장 큰 값을 선택합니다.

예를 들어 CPU와 Memory를 동시에 기준으로 설정할 수 있습니다.

```yaml
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

이 경우 CPU와 Memory 사용률을 각각 계산하고, 두 메트릭이 제안하는 Replica 개수 중 더 큰 값을 선택합니다.

예를 들어 다음과 같은 결과가 나왔다고 가정하겠습니다.

| 메트릭 | 계산된 목표 Pod 개수 |
|---|---:|
| CPU | 6 |
| Memory | 8 |

이 경우 HPA는 8개의 Pod를 목표로 설정합니다.

다만 여러 메트릭 중 일부를 조회할 수 없는 경우에는 안전을 위해 Scale Down을 수행하지 않을 수 있습니다. 반면 정상적으로 조회된 메트릭이 Scale Up을 요구하면 해당 메트릭을 기준으로 확장할 수 있습니다.

## 7. HPA의 Scaling Behavior

HPA는 단순히 메트릭 변화에 따라 Pod 개수를 즉시 변경하는 것이 아니라 스케일링 속도와 안정성을 제어할 수 있는 기능을 제공합니다.

`autoscaling/v2`에서는 `behavior` 필드를 통해 Scale Up과 Scale Down의 동작을 각각 설정할 수 있습니다.

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
      - type: Percent
        value: 100
        periodSeconds: 60

  scaleDown:
    stabilizationWindowSeconds: 300
    policies:
      - type: Pods
        value: 2
        periodSeconds: 60
```

위 예시는 다음과 같은 동작을 수행합니다.

- Scale Up: 60초 동안 현재 Replica의 최대 100%까지 증가시킬 수 있습니다.
- Scale Down: 최근 5분의 스케일링 권장값을 고려하며, 60초 동안 최대 2개의 Pod만 감소시킵니다.

### 7.1. Stabilization Window

Stabilization Window는 메트릭의 급격한 변동으로 인해 Pod 개수가 반복적으로 증가하거나 감소하는 현상을 방지하기 위한 설정입니다.

예를 들어 트래픽이 순간적으로 감소했다고 해서 Pod를 즉시 제거하면, 곧바로 트래픽이 증가했을 때 다시 Pod를 생성해야 하는 상황이 발생할 수 있습니다.

이를 방지하기 위해 HPA는 일정 시간 동안의 스케일링 권장값을 고려하여 실제 스케일링을 수행합니다.

기본 설정은 다음과 같습니다.

| 구분 | 기본값 |
|---|---:|
| Scale Up Stabilization Window | 0초 |
| Scale Down Stabilization Window | 300초 |

Scale Down의 기본 Stabilization Window는 5분입니다.

Scale Down 시에는 해당 기간에 기록된 스케일링 권장값 중 가장 큰 값을 선택하여 급격하게 Pod를 제거하지 않도록 합니다.

반면 Scale Up은 기본적으로 Stabilization Window가 0초이므로 별도의 안정화 지연 없이 확장을 진행할 수 있습니다.

### 7.2. Scaling Policy

Scaling Policy는 일정 시간 동안 허용할 수 있는 Pod 개수의 변경 범위를 제한하는 기능입니다.

정책은 다음 두 가지 유형을 지원합니다.

| Type | 설명 |
|---|---|
| Pods | 일정 시간 동안 증가 또는 감소시킬 수 있는 Pod 개수를 절대값으로 제한합니다. |
| Percent | 현재 Pod 개수 대비 증가 또는 감소할 수 있는 비율을 제한합니다. |

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
behavior:
  scaleUp:
    policies:
      - type: Pods
        value: 4
        periodSeconds: 60

      - type: Percent
        value: 100
        periodSeconds: 60
```

위 설정은 60초 동안 최대 4개의 Pod를 추가하거나 현재 Replica의 최대 100%까지 증가시킬 수 있도록 허용합니다.

여러 정책을 설정한 경우 기본적으로 더 많은 변경을 허용하는 정책을 선택합니다.

정책 선택 방식은 `selectPolicy`로 변경할 수 있습니다.

| selectPolicy | 설명 |
|---|---|
| Max | 가장 많은 변경을 허용하는 정책을 선택합니다. 기본값입니다. |
| Min | 가장 적은 변경을 허용하는 정책을 선택합니다. |
| Disabled | 해당 방향의 스케일링을 비활성화합니다. |

### 7.3. Scale Up과 Scale Down 분리 설정

HPA는 Scale Up과 Scale Down에 서로 다른 정책을 적용할 수 있습니다.

일반적으로 트래픽 증가에는 빠르게 대응하고, 트래픽 감소에는 신중하게 대응하도록 구성합니다.

예를 들어 다음과 같은 정책을 사용할 수 있습니다.

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0
    selectPolicy: Max
    policies:
      - type: Percent
        value: 100
        periodSeconds: 60
      - type: Pods
        value: 4
        periodSeconds: 60

  scaleDown:
    stabilizationWindowSeconds: 300
    selectPolicy: Min
    policies:
      - type: Percent
        value: 10
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
```

이 설정은 다음과 같은 목적을 가집니다.

- Scale Up: 트래픽 증가에 빠르게 대응할 수 있도록 확장 속도를 높입니다.
- Scale Down: 트래픽이 일시적으로 감소하더라도 Pod를 급격하게 제거하지 않도록 제한합니다.

이러한 설정을 통해 애플리케이션의 응답성과 운영 안정성을 함께 고려할 수 있습니다.

## 8. HPA와 Deployment의 관계

HPA는 Deployment와 같은 워크로드 리소스의 Replica 개수를 변경하는 방식으로 동작합니다.

Deployment를 대상으로 HPA를 구성한 경우 일반적인 동작 구조는 다음과 같습니다.

```text
┌──────────────────────────────┐
│      Metrics Server          │
│                              │
│     CPU / Memory Metrics     │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      HPA Controller          │
│                              │
│  Current Metrics vs Target   │
│                              │
│     Desired Replicas         │
└──────────────┬───────────────┘
               │
               │ Update Scale Subresource
               ▼
┌──────────────────────────────┐
│         Deployment           │
│                              │
│       replicas: 6            │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│        ReplicaSet            │
│                              │
│     Desired Replicas: 6      │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│            Pods              │
│                              │
│  Pod 1  Pod 2  Pod 3          │
│  Pod 4  Pod 5  Pod 6          │
└──────────────────────────────┘
```

HPA는 Deployment의 Replica 개수를 변경하고, Deployment Controller는 변경된 Replica 개수를 ReplicaSet에 반영합니다.

이후 ReplicaSet Controller가 실제 Pod 개수를 목표 상태에 맞춥니다.

따라서 HPA, Deployment, ReplicaSet은 각각 다른 역할을 담당합니다.

| 구성 요소 | 역할 |
|---|---|
| HPA | 메트릭을 기반으로 필요한 Replica 개수를 계산합니다. |
| Deployment | 워크로드의 원하는 상태와 업데이트 전략을 관리합니다. |
| ReplicaSet | 지정된 개수의 Pod가 유지되도록 관리합니다. |
| Pod | 실제 애플리케이션 컨테이너를 실행합니다. |

## 9. HPA의 상태 (Status)

HPA 리소스는 스케일링 설정뿐만 아니라 현재 메트릭과 목표 Replica 개수, 스케일링 상태 등의 정보도 제공합니다.

HPA의 상태는 `.status` 필드에서 확인할 수 있습니다.

주요 항목은 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| currentReplicas | 현재 대상 워크로드의 Replica 개수입니다. |
| desiredReplicas | HPA가 목표로 설정한 Replica 개수입니다. |
| currentMetrics | 현재 관측된 메트릭 정보입니다. |
| lastScaleTime | 마지막으로 스케일링이 수행된 시간입니다. |
| conditions | HPA의 현재 동작 상태와 관련된 조건입니다. |

### 9.1. Conditions

HPA는 현재 스케일링 가능 여부와 메트릭 조회 상태 등을 Conditions로 표현합니다.

대표적인 Condition은 다음과 같습니다.

| Condition | 설명 |
|---|---|
| AbleToScale | HPA가 대상 워크로드의 Scale Subresource를 조회하거나 변경할 수 있는지를 나타냅니다. |
| ScalingActive | 메트릭을 정상적으로 조회하고 스케일링을 수행할 수 있는 상태인지를 나타냅니다. |
| ScalingLimited | 최소 또는 최대 Replica 제한 등의 조건으로 인해 계산된 스케일링 결과가 제한되었는지를 나타냅니다. |

각 Condition은 `True`, `False`, `Unknown` 등의 상태로 표현됩니다.

예를 들어 Metrics Server가 정상적으로 동작하지 않아 CPU 메트릭을 조회할 수 없다면 `ScalingActive`가 False로 표시될 수 있습니다.

이러한 상태 정보는 HPA가 예상대로 동작하지 않을 때 원인을 파악하는 데 활용됩니다.

## 10. HPA 운영 시 고려사항

HPA를 실제 운영 환경에 적용할 때는 메트릭 수집, 리소스 Requests, 스케일링 지연 및 클러스터 자원 등을 함께 고려해야 합니다.

### 10.1. Resource Requests 설정

CPU 또는 Memory의 Utilization을 기준으로 스케일링하려면 Pod에 해당 리소스의 Requests가 정의되어 있어야 합니다.

예를 들어 CPU Request가 설정되지 않은 컨테이너는 CPU 사용률을 계산할 기준이 없으므로 HPA가 해당 메트릭을 정상적으로 활용할 수 없습니다.

따라서 HPA를 사용하기 전에 Deployment 등의 리소스 정의에 적절한 Requests를 설정해야 합니다.

### 10.2. Metrics Server 및 Metrics Adapter 상태

HPA는 메트릭을 제공하는 API에 의존합니다.

Resource Metrics를 사용하는 경우 Metrics Server가 정상적으로 동작해야 하며, Custom Metrics 또는 External Metrics를 사용하는 경우 해당 API를 제공하는 Adapter가 정상적으로 동작해야 합니다.

메트릭 수집 시스템에 장애가 발생하면 HPA가 스케일링 판단을 수행하지 못하거나 Scale Down이 제한될 수 있습니다.

### 10.3. Node 자원과 Cluster Autoscaler

HPA가 Pod 개수를 증가시키더라도 클러스터에 해당 Pod를 실행할 수 있는 Node 자원이 충분하지 않으면 Pod가 정상적으로 실행되지 못할 수 있습니다.

예를 들어 HPA가 Pod를 5개에서 10개로 증가시켰지만 기존 Node의 CPU 및 Memory 자원이 부족하다면 일부 Pod는 `Pending` 상태에 머무를 수 있습니다.

이러한 상황에서는 Cluster Autoscaler와 같은 Node 자동 확장 기능을 함께 활용할 수 있습니다.

두 기능의 역할은 다음과 같습니다.

| 구분 | HPA | Cluster Autoscaler |
|---|---|---|
| 조정 대상 | Pod 개수 | Node 개수 |
| 확장 기준 | CPU, Memory 및 기타 메트릭 | 스케줄링할 수 없는 Pending Pod 및 Node 활용률 등의 조건 |
| 주요 역할 | 애플리케이션의 실행 인스턴스 확장 | Pod를 실행할 수 있는 컴퓨팅 자원 확보 |

HPA는 애플리케이션의 Pod 개수를 조정하고, Cluster Autoscaler는 Pod를 실행할 Node의 개수를 조정합니다.

따라서 두 기능을 함께 사용하면 애플리케이션의 부하 증가에 따라 Pod와 Node를 각각 확장하는 구조를 구성할 수 있습니다.

### 10.4. 스케일링 지연

HPA는 주기적으로 메트릭을 수집하고 스케일링을 수행하므로 부하가 증가한 시점과 실제 Pod가 준비되는 시점 사이에 지연이 발생할 수 있습니다.

실제 확장 시간에는 다음과 같은 요소가 영향을 미칩니다.

- HPA Controller의 동기화 주기
- 메트릭 수집 및 전달 지연
- 이미지 다운로드 시간
- 컨테이너 시작 시간
- 애플리케이션 초기화 시간
- Readiness Probe 통과 시간
- Node 자원 부족에 따른 스케줄링 지연

따라서 급격한 트래픽 증가가 예상되는 서비스에서는 최소 Replica 개수와 초기 확장 정책을 적절하게 설정해야 합니다.

### 10.5. HPA와 수동 Replica 설정의 충돌

HPA가 관리하는 Deployment의 Replica 개수를 별도로 수동 변경하면 설정이 서로 충돌할 수 있습니다.

예를 들어 Deployment에 `replicas: 3`이 설정되어 있어도 HPA가 해당 Deployment를 6개로 확장하면 HPA가 설정한 목표 Replica 개수가 적용됩니다.

이후 GitOps 도구나 배포 시스템이 Deployment의 Replica 개수를 다시 3개로 덮어쓰면 HPA와 외부 관리 시스템이 서로 다른 목표 상태를 반복적으로 적용하는 문제가 발생할 수 있습니다.

따라서 HPA가 관리하는 워크로드에서는 Replica 개수의 관리 주체를 명확하게 구분해야 합니다.

### 10.6. HPA와 PDB

PodDisruptionBudget(PDB)은 유지보수나 자발적 중단(Voluntary Disruption) 상황에서 애플리케이션이 유지해야 하는 최소 가용 Pod 개수를 제어합니다.

HPA는 메트릭에 따라 Replica 개수를 조정하고, PDB는 특정 작업으로 인해 Pod가 중단되는 상황에서 허용할 수 있는 중단 범위를 관리합니다.

두 리소스는 서로 다른 목적을 가지고 있지만 함께 사용하면 자동 스케일링과 워크로드 가용성을 함께 고려할 수 있습니다.

단, PDB가 HPA의 Scale Down 자체를 직접 제한하는 것은 아닙니다. PDB는 주로 Eviction API를 사용하는 자발적 중단 작업에 적용되며, HPA의 Replica 감소는 일반적으로 워크로드의 스케일 변경을 통해 처리됩니다.

## 11. HPA 관련 주요 명령어

| 명령어 | 설명 |
|---|---|
| `kubectl get hpa` | 클러스터의 HPA 목록을 확인합니다. |
| `kubectl get hpa -A` | 모든 Namespace의 HPA 목록을 확인합니다. |
| `kubectl describe hpa <name>` | 특정 HPA의 상세 설정, 메트릭 및 이벤트를 확인합니다. |
| `kubectl get hpa <name> -o yaml` | HPA 리소스의 전체 YAML 정보를 확인합니다. |
| `kubectl autoscale deployment <name> --cpu-percent=70 --min=2 --max=10` | Deployment에 CPU 사용률 기반 HPA를 생성합니다. |
| `kubectl top nodes` | Node의 CPU 및 Memory 사용량을 확인합니다. |
| `kubectl top pods` | Pod의 CPU 및 Memory 사용량을 확인합니다. |
| `kubectl get --raw "/apis/metrics.k8s.io/v1beta1/pods"` | Resource Metrics API에서 Pod 메트릭을 조회합니다. |

HPA의 동작 상태를 확인할 때는 다음 명령어를 사용할 수 있습니다.

```bash
kubectl get hpa
```

출력 예시는 다음과 같습니다.

```text
NAME              REFERENCE                TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
application-hpa   Deployment/application   85%/70%   2         10        6          10m
```

각 항목의 의미는 다음과 같습니다.

| 항목 | 설명 |
|---|---|
| NAME | HPA 리소스의 이름입니다. |
| REFERENCE | 스케일링 대상 워크로드입니다. |
| TARGETS | 현재 메트릭과 목표 메트릭입니다. |
| MINPODS | 최소 Pod 개수입니다. |
| MAXPODS | 최대 Pod 개수입니다. |
| REPLICAS | 현재 대상 워크로드의 Replica 개수입니다. |
| AGE | HPA 리소스가 생성된 이후 경과한 시간입니다. |

위 예시에서는 CPU 목표 사용률이 70%이지만 현재 사용률이 85%이므로 HPA가 Scale Up을 검토할 수 있는 상태입니다.

다만 실제 확장 여부와 목표 Pod 개수는 현재 Replica 개수, 메트릭 상태, Tolerance, Scaling Policy 및 기타 제한 조건에 따라 결정됩니다.

## 12. 정리

HorizontalPodAutoscaler(HPA)는 Kubernetes에서 애플리케이션의 부하에 따라 Pod 개수를 자동으로 조정하는 리소스입니다.

HPA Controller는 주기적으로 메트릭을 수집하고 현재 메트릭과 목표 메트릭의 차이를 계산하여 필요한 Replica 개수를 결정합니다.

CPU 및 Memory와 같은 Resource Metrics뿐만 아니라 Custom Metrics와 External Metrics를 활용할 수 있으며, `autoscaling/v2`의 `behavior` 설정을 통해 Scale Up과 Scale Down의 속도 및 안정성을 제어할 수 있습니다.

HPA는 대상 워크로드의 Scale Subresource를 변경하는 방식으로 동작하며, 실제 Pod의 생성 및 제거는 Deployment와 ReplicaSet 등의 Controller가 담당합니다.

또한 HPA는 Pod 개수만 조정하므로 클러스터의 Node 자원이 부족한 경우 Cluster Autoscaler와 같은 Node 자동 확장 기능을 함께 고려해야 합니다.

따라서 HPA는 애플리케이션의 부하 변화에 유연하게 대응하고 Kubernetes 클러스터의 컴퓨팅 자원을 효율적으로 활용하기 위한 핵심적인 자동 스케일링 리소스입니다.

## 13. 참고자료

- [Kubernetes 공식 문서 - Horizontal Pod Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)
- [Kubernetes 공식 문서 - HorizontalPodAutoscaler API Reference](https://kubernetes.io/docs/reference/kubernetes-api/autoscaling/horizontal-pod-autoscaler-v2/)
- [Kubernetes 공식 문서 - Autoscaling Workloads](https://kubernetes.io/docs/concepts/workloads/autoscaling/)
- [Kubernetes 공식 문서 - Resource Metrics Pipeline](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)
- [Kubernetes 공식 문서 - Metrics Server](https://github.com/kubernetes-sigs/metrics-server)
- [Kubernetes 공식 문서 - Scale a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#scaling-a-deployment)
- [Kubernetes 공식 문서 - Cluster Autoscaling](https://kubernetes.io/docs/concepts/cluster-administration/cluster-autoscaling/)