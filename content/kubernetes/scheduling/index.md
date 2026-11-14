# Scheduling Resources

## 개요

Kubernetes의 Scheduling은 Pod가 실행될 Node를 결정하고, 클러스터의 리소스를 효율적으로 배분하며, 애플리케이션의 가용성과 실행 조건을 관리하는 기능 영역입니다.

Kubernetes Scheduler는 Pod의 리소스 요청량, Node의 상태, 스케줄링 제약 조건 등을 고려하여 Pod를 배치합니다. 또한 HorizontalPodAutoscaler와 VerticalPodAutoscaler는 애플리케이션의 리소스 사용량에 따라 실행 규모와 리소스 할당량을 조정하며, PodDisruptionBudget은 자발적인 중단 상황에서 애플리케이션의 가용성을 보호합니다.

## 리소스 및 관련 개념

| 이름 | 역할 | 범위 |
|---|---|---|
| Node | Pod가 실행되는 워커 머신입니다. | Cluster |
| HorizontalPodAutoscaler (HPA) | 메트릭을 기반으로 Workload의 Pod 수를 자동으로 조정합니다. | Namespace |
| VerticalPodAutoscaler (VPA) | 컨테이너의 리소스 요청량과 제한값을 분석하고 조정합니다. | Namespace |
| PodDisruptionBudget (PDB) | 자발적인 중단 상황에서 유지해야 하는 최소 Pod 수 또는 최대 중단 Pod 수를 정의합니다. | Namespace |
| PriorityClass | Pod의 스케줄링 우선순위를 정의합니다. | Cluster |
| ResourceQuota | Namespace에서 사용할 수 있는 총 리소스 사용량을 제한합니다. | Namespace |
| LimitRange | Namespace 내 개별 컨테이너 및 Pod의 리소스 요청량과 제한값에 대한 기본값과 범위를 설정합니다. | Namespace |
| RuntimeClass | Pod가 사용할 컨테이너 런타임 구성을 지정합니다. | Cluster |
| Affinity / Anti-Affinity | Pod를 특정 Node 또는 다른 Pod와 함께 배치하거나 분리하는 조건을 정의합니다. | Pod 설정 |
| Taint / Toleration | Node에 배치 제한 조건을 설정하고, 해당 조건을 허용하는 Pod를 지정합니다. | Node / Pod 설정 |

> Node, PriorityClass, ResourceQuota, LimitRange 등은 Kubernetes API 리소스입니다. 반면 Affinity, Toleration은 Pod 설정에 포함되는 필드이며, Taint는 Node의 설정입니다.

## 리소스 및 개념 간 관계

```text
Scheduling
│
├── Node Scheduling
│   │
│   ├── Node
│   │   ├── Taint
│   │   └── Resource Capacity
│   │
│   ├── Affinity / Anti-Affinity
│   └── Toleration
│
├── Resource Management
│   │
│   ├── ResourceQuota
│   ├── LimitRange
│   ├── HorizontalPodAutoscaler (HPA)
│   └── VerticalPodAutoscaler (VPA)
│
├── Availability Management
│   │
│   └── PodDisruptionBudget (PDB)
│
├── Scheduling Priority
│   │
│   └── PriorityClass
│
└── Runtime Configuration
    │
    └── RuntimeClass
```

## 기능별 분류

### 1. Pod 배치 및 스케줄링 조건

Pod가 어떤 Node에서 실행될 수 있는지 결정하는 기능입니다.

- **Node**: Pod가 실제로 실행되는 머신이며, CPU와 메모리 등의 리소스 용량을 제공합니다.
- **Affinity / Anti-Affinity**: Node 또는 다른 Pod와의 관계를 기준으로 배치 조건을 정의합니다.
- **Taint / Toleration**: 특정 조건을 가진 Node에 Pod가 배치되는 것을 제한하거나 허용합니다.

Kubernetes Scheduler는 이러한 조건과 Pod의 리소스 요청량 등을 고려하여 적절한 Node를 선택합니다.

### 2. 리소스 관리 및 자동 조정

클러스터와 Namespace의 리소스 사용량을 제한하거나 애플리케이션의 실행 규모 및 리소스 할당량을 조정하는 기능입니다.

- **ResourceQuota**: Namespace 전체의 CPU, 메모리, 객체 수 등의 사용량을 제한합니다.
- **LimitRange**: 개별 컨테이너와 Pod의 리소스 요청량 및 제한값에 대한 기본값과 허용 범위를 설정합니다.
- **HorizontalPodAutoscaler (HPA)**: CPU, 메모리 또는 사용자 정의 메트릭 등을 기반으로 Pod의 복제본 수를 조정합니다.
- **VerticalPodAutoscaler (VPA)**: 컨테이너의 리소스 사용 패턴을 분석하여 CPU와 메모리 요청량 등을 권장하거나 조정합니다.

HPA는 수평 확장(Scale Out)을, VPA는 수직 확장(Scale Up)을 담당합니다. 두 기능은 조정 대상이 다르지만, 동일한 CPU 또는 메모리 메트릭을 기준으로 함께 동작하도록 구성할 때는 충돌 가능성을 고려해야 합니다.

### 3. 애플리케이션 가용성 관리

- **PodDisruptionBudget (PDB)**: 노드 유지보수나 클러스터 관리 작업 등 자발적인 중단 상황에서 동시에 중단될 수 있는 Pod 수를 제한합니다.

PDB는 애플리케이션의 가용성을 보호하기 위한 정책이며, 모든 장애나 비자발적인 Pod 중단을 방지하는 기능은 아닙니다.

### 4. 스케줄링 우선순위 관리

- **PriorityClass**: Pod의 우선순위를 정의합니다. 리소스가 부족한 경우 우선순위가 낮은 Pod를 선점(Preemption)하여 우선순위가 높은 Pod가 배치될 수 있도록 합니다.

PriorityClass는 클러스터 범위의 리소스이며, Pod에서 해당 클래스를 참조하여 우선순위를 지정합니다.

### 5. 컨테이너 런타임 설정

- **RuntimeClass**: Pod가 사용할 컨테이너 런타임 구성을 지정합니다. 이를 통해 서로 다른 런타임 환경을 사용하는 Pod를 구분할 수 있습니다.

예를 들어 일반적인 컨테이너 런타임과 추가적인 격리 기능을 제공하는 런타임을 구분하여 사용할 수 있습니다.

## 주요 구성 요소 간 관계

### HPA와 VPA

| 구분 | HPA | VPA |
|---|---|---|
| 조정 대상 | Pod 복제본 수 | 컨테이너 리소스 요청량 및 제한값 |
| 확장 방식 | 수평 확장 | 수직 확장 |
| 주요 목적 | 트래픽 증가에 대응 | Pod별 리소스 할당 최적화 |
| 주요 메트릭 | CPU, 메모리, 사용자 정의 메트릭 등 | 컨테이너 리소스 사용 패턴 |

### ResourceQuota와 LimitRange

| 구분 | ResourceQuota | LimitRange |
|---|---|---|
| 적용 범위 | Namespace 전체 | 개별 컨테이너 및 Pod |
| 주요 목적 | Namespace 총 사용량 제한 | 개별 리소스의 기본값과 허용 범위 설정 |
| 예시 | Namespace의 총 CPU 요청량을 10 Core로 제한 | 컨테이너의 메모리 요청량을 최소 128Mi, 최대 2Gi로 제한 |

두 리소스는 함께 사용하여 Namespace 전체의 리소스 소비량과 개별 Pod의 리소스 설정을 관리할 수 있습니다.

## 참고 사항

- Kubernetes Scheduler는 Pod의 리소스 요청량(`requests`)을 기준으로 Node의 배치 가능 여부를 판단합니다. 리소스 사용량(`usage`) 자체를 기준으로 일반적인 Pod 배치를 결정하는 것은 아닙니다.
- HPA는 메트릭을 수집할 수 있는 환경이 필요합니다. CPU 및 메모리 사용률 기반 자동 확장을 위해서는 일반적으로 Metrics Server가 필요합니다.
- VPA는 Kubernetes 기본 설치에 포함된 기능이 아니며, 별도로 설치해야 하는 확장 구성 요소입니다.
- PDB는 `kubectl drain`과 같은 자발적인 중단 작업에 영향을 주지만, Node 장애나 OOM 등 비자발적인 중단까지 방지하지는 않습니다.
- Taint는 Node에 설정하고 Toleration은 Pod에 설정합니다. Toleration이 있다고 해서 해당 Node에 반드시 배치되는 것은 아닙니다.
- Affinity와 Toleration은 서로 다른 목적을 가집니다. Affinity는 배치 선호도 또는 제약 조건을 정의하고, Toleration은 Node의 Taint에 의한 배치 제한을 허용합니다.

## 공식 문서

- [Scheduling, Preemption and Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/)
- [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Vertical Pod Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/)
- [Pod Disruption Budget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Limit Ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)
- [Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
- [RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/)