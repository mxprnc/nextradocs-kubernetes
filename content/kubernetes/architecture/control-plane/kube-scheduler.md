# kube-scheduler

## 1. kube-scheduler란?

kube-scheduler는 Kubernetes Control Plane의 구성 요소로, 새롭게 생성되었거나 아직 Node가 할당되지 않은 Pod를 실행하기에 적절한 Node에 배치하는 역할을 담당합니다.

Kubernetes에서 Pod는 생성되었다고 해서 특정 Node에서 즉시 실행되는 것은 아닙니다. Pod가 실행될 Node를 결정하는 과정이 필요하며, 이 과정을 **Scheduling(스케줄링)**이라고 합니다.

kube-scheduler는 Pod가 요구하는 리소스와 스케줄링 조건을 확인하고, 클러스터 내에서 실행 가능한 Node를 탐색한 후 가장 적합한 Node를 선택합니다.

주요 역할은 다음과 같습니다.

- 스케줄링 대상 Pod를 감지합니다.
- Pod의 리소스 요구사항과 스케줄링 제약 조건을 확인합니다.
- 실행 가능한 Node를 필터링합니다.
- 필터링된 Node에 점수를 부여하여 적절한 Node를 선택합니다.
- 선택한 Node를 Kubernetes API Server에 전달하여 Pod와 Node를 연결합니다.

kube-scheduler는 Pod를 직접 실행하거나 컨테이너를 생성하지 않습니다. Pod가 실행될 Node를 결정하고 해당 정보를 API Server에 기록하는 역할을 담당합니다.

실제 Node에서 Pod를 실행하는 작업은 kubelet과 컨테이너 런타임이 수행합니다.

## 2. 주요 역할

### 2.1. 스케줄링 대상 Pod 감지

kube-scheduler는 Kubernetes API Server를 통해 Pod 상태를 감시하고, 아직 Node가 할당되지 않은 Pod를 스케줄링 대상으로 처리합니다.

일반적으로 다음 조건을 만족하는 Pod가 스케줄링 대상이 됩니다.

- Pod가 정상적으로 생성되었습니다.
- Pod의 `.spec.nodeName`이 비어 있습니다.
- 다른 스케줄러에 의해 이미 처리되지 않았습니다.
- Pod의 `.spec.schedulerName`이 해당 스케줄러의 프로파일과 일치합니다.

기본적으로 Kubernetes에서 생성되는 Pod는 `default-scheduler`를 사용합니다.

스케줄링 대상 Pod가 발견되면 kube-scheduler는 내부 스케줄링 큐에 해당 Pod를 추가하고 처리 순서에 따라 스케줄링을 수행합니다.

### 2.2. Node Filtering

Node Filtering은 Pod를 실행할 수 없는 Node를 제외하고, 스케줄링 조건을 만족하는 Node를 찾는 과정입니다.

예를 들어 Pod가 다음과 같은 조건을 요구한다고 가정하겠습니다.

- CPU 요청량: 2
- Memory 요청량: 4Gi
- 특정 Label을 가진 Node에 배치
- 특정 Taint가 적용된 Node는 제외

kube-scheduler는 클러스터의 Node 상태와 Pod의 스케줄링 조건을 비교하여 실행 가능한 Node만 남깁니다.

이 단계에서 모든 조건을 만족하는 Node가 없다면 Pod는 스케줄링되지 않은 상태로 남게 됩니다.

### 2.3. Node Scoring

Node Scoring은 Filtering 단계를 통과한 Node에 점수를 부여하여 가장 적합한 Node를 선택하는 과정입니다.

여러 Node에서 Pod를 실행할 수 있다면 kube-scheduler는 구성된 점수 계산 플러그인을 사용하여 각 Node의 적합도를 평가합니다.

예를 들어 다음과 같이 점수가 계산될 수 있습니다.

| Node | Scoring 결과 |
| --- | ---: |
| node-1 | 65 |
| node-2 | 90 |
| node-3 | 75 |

위와 같은 결과라면 kube-scheduler는 가장 높은 점수를 받은 `node-2`를 선택합니다.

여러 Node가 동일한 최종 점수를 받는 경우에는 그중 하나를 선택합니다.

### 2.4. Pod Binding

Pod Binding은 kube-scheduler가 선택한 Node를 Pod에 할당하는 과정입니다.

kube-scheduler는 API Server에 Binding 요청을 전달하여 Pod와 Node를 연결합니다.

Binding이 정상적으로 완료되면 Pod의 `.spec.nodeName`에 선택된 Node의 이름이 기록됩니다.

이후 해당 Node의 kubelet이 API Server를 통해 자신에게 할당된 Pod를 확인하고 컨테이너 실행을 진행합니다.

## 3. 스케줄링 처리 과정

kube-scheduler의 기본적인 스케줄링 과정은 다음과 같습니다.

```text
                 Pod 생성
                    |
                    v
              kube-apiserver
                    |
                    v
              kube-scheduler
                    |
                    v
             Scheduling Queue
                    |
                    v
             Node Filtering
                    |
                    v
              Node Scoring
                    |
                    v
              Node Selection
                    |
                    v
               Pod Binding
                    |
                    v
              kube-apiserver
                    |
                    v
              Pod.spec.nodeName
                    |
                    v
               대상 kubelet
                    |
                    v
              Pod 실행 준비
```

위 흐름은 스케줄링의 주요 개념을 설명하기 위한 단순화된 구조입니다. 실제 내부 처리 과정은 Scheduling Framework의 여러 단계와 플러그인을 통해 수행됩니다.

### 3.1. Scheduling Queue

kube-scheduler는 스케줄링이 필요한 Pod를 내부 큐에서 관리합니다.

스케줄링 큐는 Pod의 처리 순서와 재시도 상태를 관리하여 스케줄러가 효율적으로 작업을 수행할 수 있도록 합니다.

일반적으로 다음과 같은 큐 영역을 사용합니다.

| 큐 | 역할 |
| --- | --- |
| Active Queue | 스케줄링을 수행할 수 있는 Pod를 관리합니다. |
| Backoff Queue | 스케줄링에 실패하여 일정 시간 후 재시도할 Pod를 관리합니다. |
| Unschedulable Pod List | 현재 클러스터 상태에서는 스케줄링할 수 없는 Pod를 관리합니다. |

예를 들어 Pod가 요구하는 CPU를 제공할 수 있는 Node가 없다면 해당 Pod는 스케줄링에 실패할 수 있습니다.

이후 Node의 리소스 상태가 변경되거나 스케줄링 조건에 영향을 주는 이벤트가 발생하면 kube-scheduler는 해당 Pod를 다시 큐에 넣어 스케줄링을 시도할 수 있습니다.

### 3.2. Node Filtering

Node Filtering은 Pod를 실행할 수 없는 Node를 제외하는 단계입니다.

이 과정에서는 Pod의 리소스 요구사항, Node 상태, Affinity, Taint 및 Toleration, Volume 관련 조건 등을 확인합니다.

예를 들어 다음과 같은 조건을 확인할 수 있습니다.

- Node가 정상적으로 스케줄링 가능한 상태인지 확인합니다.
- Pod의 CPU 및 Memory 요청량을 수용할 수 있는지 확인합니다.
- Node Selector와 Node Affinity 조건을 만족하는지 확인합니다.
- Taint와 Toleration 조건이 일치하는지 확인합니다.
- Pod Affinity와 Pod Anti-Affinity 조건을 만족하는지 확인합니다.
- Topology Spread Constraints를 만족할 수 있는지 확인합니다.
- 필요한 Volume을 해당 Node에서 사용할 수 있는지 확인합니다.

Filtering을 통과한 Node는 **Feasible Node**라고 합니다.

모든 Node가 필터링에서 제외되면 Pod는 스케줄링되지 않습니다.

### 3.3. Node Scoring

Node Scoring은 Filtering을 통과한 Feasible Node를 대상으로 적합도를 평가하는 단계입니다.

각 Scoring Plugin은 Node별 점수를 계산하며, kube-scheduler는 각 플러그인의 점수와 가중치를 반영하여 최종 점수를 산출합니다.

일반적으로 다음과 같은 요소를 고려할 수 있습니다.

- Node의 리소스 사용량과 잔여 리소스
- Pod Affinity 및 Pod Anti-Affinity 선호 조건
- Topology Spread Constraints
- Volume 배치 조건
- 스케줄링 정책에 따른 Node 선호도

각 플러그인의 점수는 정해진 범위로 정규화될 수 있으며, 플러그인별 가중치가 최종 점수에 반영됩니다.

따라서 특정 조건을 만족하는 Node가 여러 개 존재하더라도 모든 Node가 동일한 우선순위로 선택되는 것은 아닙니다.

### 3.4. Node Selection

kube-scheduler는 Scoring 결과를 바탕으로 최종 Node를 선택합니다.

가장 높은 점수를 받은 Node가 선택되며, 동점인 Node가 존재하면 그중 하나를 선택합니다.

선택된 Node는 이후 Binding 단계에서 Pod에 할당됩니다.

### 3.5. Pod Binding

Node가 선택되면 kube-scheduler는 API Server에 Binding 요청을 전달합니다.

Binding은 Pod를 특정 Node에 할당하는 작업이며, 정상적으로 완료되면 Pod의 `.spec.nodeName`에 Node 이름이 기록됩니다.

예를 들어 다음과 같이 Pod에 Node가 할당됩니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeName: worker-node-1
  containers:
    - name: nginx
      image: nginx:latest
```

위 예시에서 `worker-node-1`은 kube-scheduler가 선택하여 할당한 Node입니다.

Pod에 Node가 할당되면 해당 Node의 kubelet이 Pod를 감지하고 컨테이너 실행을 진행합니다.

## 4. 주요 스케줄링 조건

kube-scheduler는 단순히 Node의 CPU와 Memory 여유 공간만 확인하는 것이 아닙니다.

Pod가 요구하는 다양한 제약 조건과 클러스터의 상태를 함께 고려하여 Node를 선택합니다.

### 4.1. Resource Requests

Pod의 Resource Requests는 스케줄링 과정에서 Node의 리소스 수용 가능 여부를 판단하는 기준입니다.

예를 들어 다음과 같이 CPU와 Memory 요청량을 지정할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-example
spec:
  containers:
    - name: app
      image: nginx
      resources:
        requests:
          cpu: "2"
          memory: "4Gi"
        limits:
          cpu: "4"
          memory: "8Gi"
```

kube-scheduler는 Pod의 Resource Requests를 기반으로 Node가 해당 Pod를 수용할 수 있는지 확인합니다.

이때 단순히 현재 사용 중인 CPU와 Memory를 기준으로 판단하는 것이 아니라, Node에 이미 할당된 Pod들의 요청량과 Node의 Allocatable 리소스 등을 고려합니다.

주요 리소스는 다음과 같습니다.

- CPU
- Memory
- Ephemeral Storage
- Extended Resources

Resource Requests와 Limits는 서로 다른 목적을 가집니다.

| 구분 | 역할 |
| --- | --- |
| Requests | 스케줄링 시 Pod가 필요로 하는 리소스의 기준입니다. |
| Limits | 컨테이너가 사용할 수 있는 리소스의 상한을 설정합니다. |

따라서 일반적인 스케줄링 과정에서 Node의 수용 가능 여부를 판단할 때는 Requests가 중요한 기준이 됩니다.

### 4.2. Node Selector

Node Selector는 Pod를 특정 Label을 가진 Node에 배치하기 위한 간단한 스케줄링 조건입니다.

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-selector-example
spec:
  nodeSelector:
    disktype: ssd
  containers:
    - name: app
      image: nginx
```

위 Pod는 다음 조건을 만족하는 Node에만 배치될 수 있습니다.

```text
disktype=ssd
```

Node에 해당 Label이 존재하지 않거나 값이 일치하지 않으면 해당 Node는 스케줄링 후보에서 제외됩니다.

### 4.3. Node Affinity

Node Affinity는 Node의 Label을 기반으로 Pod가 배치될 Node를 선택하기 위한 기능입니다.

Node Selector보다 다양한 조건을 표현할 수 있으며, 필수 조건과 선호 조건을 구분할 수 있습니다.

주요 설정은 다음과 같습니다.

| 설정 | 역할 |
| --- | --- |
| `requiredDuringSchedulingIgnoredDuringExecution` | 스케줄링 시 반드시 만족해야 하는 조건입니다. |
| `preferredDuringSchedulingIgnoredDuringExecution` | 가능하면 만족하는 것이 좋은 선호 조건입니다. |

예를 들어 특정 Node Pool에 Pod를 배치하고 싶다면 다음과 같이 설정할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity-example
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: node-pool
                operator: In
                values:
                  - high-performance
  containers:
    - name: app
      image: nginx
```

위 설정은 `node-pool=high-performance` Label을 가진 Node에만 Pod를 배치하도록 합니다.

### 4.4. Pod Affinity와 Pod Anti-Affinity

Pod Affinity와 Pod Anti-Affinity는 다른 Pod의 위치를 기준으로 새로운 Pod의 배치 위치를 결정하는 기능입니다.

| 구분 | 역할 |
| --- | --- |
| Pod Affinity | 특정 조건을 만족하는 Pod와 같은 토폴로지 영역에 배치하도록 합니다. |
| Pod Anti-Affinity | 특정 조건을 만족하는 Pod와 다른 토폴로지 영역에 배치하도록 합니다. |

예를 들어 애플리케이션의 Web Pod와 Cache Pod를 동일한 Availability Zone에 배치하려면 Pod Affinity를 사용할 수 있습니다.

반대로 동일한 애플리케이션의 복제본을 서로 다른 Node에 분산하려면 Pod Anti-Affinity를 사용할 수 있습니다.

이러한 기능은 애플리케이션의 데이터 접근 지연이나 장애 격리 등을 고려한 배치 전략에 활용됩니다.

### 4.5. Taint와 Toleration

Taint와 Toleration은 특정 Node에 Pod가 배치되는 것을 제한하기 위한 기능입니다.

Taint는 Node에 설정하며, Toleration은 Pod에 설정합니다.

Pod가 Node의 Taint를 허용하지 않으면 해당 Node에 배치되지 않을 수 있습니다.

주요 Taint Effect는 다음과 같습니다.

| Effect | 설명 |
| --- | --- |
| `NoSchedule` | Taint를 허용하지 않는 새로운 Pod의 스케줄링을 차단합니다. |
| `PreferNoSchedule` | 가능한 경우 Taint가 없는 Node를 선호하도록 합니다. |
| `NoExecute` | 새로운 Pod의 스케줄링을 제한하며, 기존에 실행 중인 Pod도 Toleration 조건에 따라 퇴출할 수 있습니다. |

예를 들어 특정 Node를 GPU 전용으로 사용하려면 다음과 같이 Taint를 설정할 수 있습니다.

```bash
kubectl taint nodes gpu-node-1 accelerator=nvidia:NoSchedule
```

이후 해당 Node에 Pod를 배치하려면 Pod에 적절한 Toleration을 설정해야 합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  tolerations:
    - key: "accelerator"
      operator: "Equal"
      value: "nvidia"
      effect: "NoSchedule"
  containers:
    - name: app
      image: nginx
```

Toleration은 해당 Taint를 허용한다는 의미이며, 반드시 해당 Node에 배치된다는 의미는 아닙니다.

실제 배치를 제한하거나 유도하려면 Node Selector 또는 Node Affinity 등을 함께 사용해야 합니다.

### 4.6. Topology Spread Constraints

Topology Spread Constraints는 Pod를 여러 토폴로지 영역에 균등하게 분산하기 위한 기능입니다.

토폴로지 영역은 일반적으로 다음과 같은 Label을 기준으로 구분할 수 있습니다.

- Node
- Availability Zone
- Region

예를 들어 여러 Availability Zone에 Pod 복제본을 분산하면 특정 Zone에 장애가 발생하더라도 전체 서비스가 중단될 가능성을 줄일 수 있습니다.

다음은 Pod를 서로 다른 Zone에 분산하기 위한 예시입니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: web
      containers:
        - name: web
          image: nginx
```

주요 설정은 다음과 같습니다.

| 설정 | 설명 |
| --- | --- |
| `maxSkew` | 토폴로지 영역 간 Pod 수의 허용 가능한 최대 차이를 지정합니다. |
| `topologyKey` | 토폴로지 영역을 구분할 Node Label을 지정합니다. |
| `whenUnsatisfiable` | 분산 조건을 만족할 수 없을 때의 처리 방식을 지정합니다. |
| `labelSelector` | 분산 대상 Pod를 식별하기 위한 Label 조건입니다. |

`whenUnsatisfiable`은 다음과 같이 설정할 수 있습니다.

- `DoNotSchedule`: 분산 조건을 만족하지 못하면 해당 Pod를 스케줄링하지 않습니다.
- `ScheduleAnyway`: 분산 조건을 만족하는 배치를 선호하지만, 조건을 만족하지 못하더라도 스케줄링을 허용합니다.

## 5. Scheduling Framework

### 5.1. Scheduling Framework란?

Scheduling Framework는 Kubernetes Scheduler의 스케줄링 동작을 플러그인 기반으로 확장할 수 있도록 설계된 아키텍처입니다.

기존의 스케줄링 기능을 하나의 고정된 로직으로 구현하는 대신, 여러 Plugin이 각 스케줄링 단계에 참여하도록 구성할 수 있습니다.

이를 통해 다음과 같은 기능을 제공합니다.

- 스케줄링 단계별 동작 확장
- 기본 스케줄링 로직의 변경 및 조합
- 사용자 정의 Scheduler Plugin 개발
- 스케줄링 프로파일별 동작 구성

Scheduling Framework는 Kubernetes v1.19부터 Stable 상태로 제공됩니다.

### 5.2. Scheduling Cycle과 Binding Cycle

Scheduling Framework는 하나의 Pod를 처리하는 과정을 크게 Scheduling Cycle과 Binding Cycle로 구분합니다.

| 구분 | 역할 |
| --- | --- |
| Scheduling Cycle | Pod를 실행할 Node를 선택합니다. |
| Binding Cycle | 선택된 Node에 Pod를 실제로 할당합니다. |

Scheduling Cycle은 순차적으로 실행되며, Binding Cycle은 서로 독립적으로 병렬 실행될 수 있습니다.

이러한 구조를 통해 Node 선택 과정의 일관성을 유지하면서 Binding 작업의 처리 효율을 높일 수 있습니다.

### 5.3. Scheduling Framework Extension Points

Scheduling Framework는 각 처리 단계에 Plugin이 참여할 수 있도록 Extension Point를 제공합니다.

주요 Extension Point는 다음과 같습니다.

| Extension Point | 역할 |
| --- | --- |
| `PreEnqueue` | Pod가 스케줄링 큐에 들어가기 전에 처리합니다. |
| `QueueSort` | 스케줄링 큐에서 Pod의 처리 순서를 결정합니다. |
| `PreFilter` | Filtering 전에 Pod 또는 클러스터 정보를 사전 처리합니다. |
| `Filter` | Pod를 실행할 수 없는 Node를 제외합니다. |
| `PostFilter` | 실행 가능한 Node가 없을 때 추가 처리를 수행합니다. |
| `PreScore` | Scoring 전에 필요한 정보를 사전 처리합니다. |
| `Score` | Filtering을 통과한 Node에 점수를 부여합니다. |
| `NormalizeScore` | Scoring 결과를 정규화합니다. |
| `Reserve` | Binding 전에 선택된 Node의 리소스 예약과 관련된 내부 상태를 관리합니다. |
| `Permit` | Pod의 Binding을 허용하거나 지연 또는 거부합니다. |
| `PreBind` | Binding 전에 필요한 작업을 수행합니다. |
| `Bind` | Pod를 Node에 할당합니다. |
| `PostBind` | Binding이 완료된 이후 후속 작업을 수행합니다. |

일반적인 스케줄링 흐름은 다음과 같습니다.

```text
             Scheduling Queue
                    |
                    v
                PreFilter
                    |
                    v
                  Filter
                    |
                    v
               PostFilter
          (Feasible Node가 없을 때)
                    |
                    v
                PreScore
                    |
                    v
                  Score
                    |
                    v
             NormalizeScore
                    |
                    v
                 Reserve
                    |
                    v
                 Permit
                    |
                    v
              Scheduling Cycle
                    |
                    v
                 PreBind
                    |
                    v
                  Bind
                    |
                    v
                PostBind
                    |
                    v
              Binding Cycle
```

위 그림은 주요 Extension Point를 개념적으로 표현한 것입니다. 실제 실행 여부와 세부 순서는 Plugin의 구현 및 구성에 따라 달라질 수 있습니다.

또한 `Reserve`와 `Permit`은 Scheduling Cycle과 Binding Cycle 사이에서 스케줄링 상태와 Binding 진행 여부를 조정하는 역할을 담당합니다.

### 5.4. 주요 Extension Point 상세

#### PreFilter

Filter 단계에서 반복적으로 사용할 정보를 사전에 계산하거나 Pod의 스케줄링 조건을 확인합니다.

예를 들어 Pod의 리소스 요청량이나 Affinity 조건 등을 사전에 계산하여 이후 Filter 단계에서 활용할 수 있습니다.

#### Filter

각 Node가 Pod의 스케줄링 조건을 만족하는지 확인합니다.

하나의 Filter Plugin이라도 특정 Node를 실행 불가능한 대상으로 판단하면 해당 Node는 후보에서 제외됩니다.

#### PostFilter

Filter 단계에서 실행 가능한 Node를 하나도 찾지 못한 경우 호출됩니다.

대표적인 기능은 **Preemption(선점)**입니다.

Preemption은 우선순위가 높은 Pod를 실행하기 위해 기존에 실행 중인 우선순위가 낮은 Pod를 퇴출하는 방식입니다.

다만 Preemption을 수행하더라도 Pod가 반드시 스케줄링된다는 보장은 없습니다.

#### Score

Filter를 통과한 Node를 대상으로 적합도를 계산합니다.

여러 Scoring Plugin이 각각 점수를 계산하며, 설정된 가중치를 반영하여 최종 Node 점수를 결정합니다.

#### Reserve

선택된 Node에 Pod를 할당하기 전에 Plugin이 필요한 내부 상태를 예약하는 단계입니다.

예를 들어 여러 스케줄링 작업이 동시에 진행될 때 특정 리소스의 예약 상태를 관리하여 경쟁 조건을 방지할 수 있습니다.

Reserve 단계 이후 Binding이 실패하면 `Unreserve`가 호출되어 예약 상태를 정리할 수 있습니다.

#### Permit

Pod의 Binding을 허용하거나 지연 또는 거부할 수 있는 단계입니다.

Permit Plugin은 다음 세 가지 결과를 반환할 수 있습니다.

- `Approve`: Binding을 허용합니다.
- `Deny`: Binding을 거부하고 Pod를 스케줄링 큐로 되돌립니다.
- `Wait`: 일정 시간 동안 Binding을 대기시킵니다.

여러 Pod가 함께 준비되어야 하는 작업이나 별도의 승인 조건이 필요한 스케줄링 로직에 활용할 수 있습니다.

#### PreBind

실제 Binding을 수행하기 전에 필요한 작업을 처리합니다.

예를 들어 특정 Volume을 준비하거나 Node에 연결하기 위한 작업 등을 수행할 수 있습니다.

#### Bind

선택된 Node에 Pod를 할당하는 단계입니다.

기본적으로 kube-scheduler는 API Server에 Binding 요청을 전달하여 Pod와 Node를 연결합니다.

사용자 정의 Bind Plugin을 통해 기본 Binding 동작을 확장할 수도 있습니다.

#### PostBind

Pod가 정상적으로 Binding된 이후 호출되는 단계입니다.

Binding 이후 필요한 정리 작업이나 후속 처리를 수행하는 데 활용됩니다.

## 6. Scheduling Plugin

### 6.1. 기본 Scheduling Plugin

Kubernetes는 Scheduling Framework를 통해 다양한 기본 Plugin을 제공합니다.

대표적인 Plugin은 다음과 같습니다.

| Plugin | 주요 역할 |
| --- | --- |
| `NodeResourcesFit` | Node가 Pod의 CPU, Memory 등 리소스 요청량을 수용할 수 있는지 확인합니다. |
| `NodeName` | Pod의 `nodeName`과 Node 이름이 일치하는지 확인합니다. |
| `NodeAffinity` | Node Selector 및 Node Affinity 조건을 평가합니다. |
| `TaintToleration` | Node의 Taint와 Pod의 Toleration 조건을 평가합니다. |
| `PodTopologySpread` | Topology Spread Constraints를 평가합니다. |
| `InterPodAffinity` | Pod Affinity 및 Pod Anti-Affinity 조건을 평가합니다. |
| `VolumeBinding` | Volume의 배치 및 바인딩 가능 여부를 확인합니다. |
| `DefaultPreemption` | 실행 가능한 Node가 없을 때 Preemption을 시도합니다. |
| `ImageLocality` | Node에 이미 존재하는 이미지 정보를 고려하여 점수를 계산합니다. |
| `NodeResourcesBalancedAllocation` | Node 간 리소스 사용 균형을 고려하여 점수를 계산합니다. |
| `NodeResourcesFit` | Node의 리소스 수용 가능 여부를 확인하고 구성에 따라 점수를 계산할 수 있습니다. |

각 Plugin은 하나 이상의 Extension Point에 참여할 수 있습니다.

예를 들어 `NodeResourcesFit`은 Filter와 Score 단계에 모두 참여할 수 있습니다.

### 6.2. 사용자 정의 Scheduling Plugin

Kubernetes는 기본 Plugin 외에도 사용자 정의 Scheduling Plugin을 개발할 수 있도록 확장 지점을 제공합니다.

사용자 정의 Plugin은 Go 언어와 Kubernetes Scheduler Framework API를 사용하여 구현하며, 일반적으로 kube-scheduler 바이너리에 포함되어야 합니다.

이를 통해 조직의 인프라 환경이나 서비스 특성에 맞는 스케줄링 로직을 구현할 수 있습니다.

예를 들어 다음과 같은 요구사항을 처리할 수 있습니다.

- 특정 하드웨어를 사용하는 Node 우선 배치
- 서비스별 리소스 배치 정책
- 특수한 네트워크 토폴로지 고려
- 외부 시스템의 배치 정보를 반영한 스케줄링

## 7. Scheduling Profile과 다중 Scheduler

### 7.1. Scheduling Profile

Scheduling Profile은 하나의 kube-scheduler 인스턴스에서 스케줄링 동작을 다르게 구성할 수 있도록 제공하는 기능입니다.

각 Profile은 서로 다른 Plugin 구성을 가질 수 있으며, Pod의 `.spec.schedulerName`을 통해 사용할 Profile을 지정할 수 있습니다.

예를 들어 다음과 같이 두 개의 Profile을 구성할 수 있습니다.

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler

  - schedulerName: custom-scheduler
    plugins:
      score:
        disabled:
          - name: NodeResourcesBalancedAllocation
```

위 구성에서는 하나의 kube-scheduler가 두 가지 Profile을 사용합니다.

- `default-scheduler`: 기본 스케줄링 Plugin 구성을 사용합니다.
- `custom-scheduler`: 별도의 Plugin 구성을 사용합니다.

Pod는 다음과 같이 사용할 Scheduler를 지정할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-scheduler-example
spec:
  schedulerName: custom-scheduler
  containers:
    - name: app
      image: nginx
```

위 Pod는 `custom-scheduler`라는 이름의 Profile을 사용하여 스케줄링됩니다.

### 7.2. Multiple Scheduler

Kubernetes에서는 기본 kube-scheduler 외에 별도의 Scheduler를 실행할 수 있습니다.

각 Scheduler는 자신이 처리할 Pod를 구분하기 위해 `schedulerName`을 사용합니다.

예를 들어 다음과 같은 구성이 가능합니다.

```text
              Kubernetes API Server
                       |
             +---------+---------+
             |                   |
             v                   v
      kube-scheduler       custom-scheduler
             |                   |
             v                   v
      default-scheduler     custom-scheduler
             |                   |
             v                   v
       기본 Pod 처리         사용자 정의 Pod 처리
```

기본 kube-scheduler는 `default-scheduler`를 사용하는 Pod를 처리하고, 별도로 실행한 Scheduler는 자신에게 지정된 이름의 Pod를 처리하도록 구성할 수 있습니다.

Scheduling Profile과 Multiple Scheduler는 서로 다른 개념입니다.

| 구분 | Scheduling Profile | Multiple Scheduler |
| --- | --- | --- |
| 실행 구조 | 하나의 Scheduler 인스턴스에서 여러 Profile 실행 | 여러 Scheduler 인스턴스 실행 |
| Plugin 구성 | Profile별로 다르게 구성 가능 | Scheduler별로 독립적으로 구성 가능 |
| Pod 선택 | `schedulerName`으로 Profile 선택 | `schedulerName`으로 Scheduler 선택 |
| 주요 목적 | 하나의 Scheduler 내에서 스케줄링 정책 분리 | 별도의 스케줄링 로직과 실행 환경 구성 |

따라서 단순히 서로 다른 스케줄링 정책을 적용하려는 목적이라면 Scheduling Profile을 고려할 수 있으며, 별도의 Scheduler 구현이나 독립적인 실행 환경이 필요한 경우 Multiple Scheduler를 고려할 수 있습니다.

## 8. 고가용성 및 운영

### 8.1. Scheduler High Availability

kube-scheduler는 여러 인스턴스를 실행하여 고가용성을 구성할 수 있습니다.

일반적으로 동일한 스케줄링 설정을 가진 여러 kube-scheduler 인스턴스를 실행하고 Leader Election을 사용하여 하나의 인스턴스만 활성 스케줄러로 동작하도록 구성합니다.

```text
             Kubernetes API Server
                       |
              +--------+--------+
              |        |        |
              v        v        v
          Scheduler Scheduler Scheduler
              1        2        3
              |        |        |
              +--------+--------+
                       |
                  Leader Election
                       |
                       v
                Active Scheduler
```

Leader가 아닌 Scheduler 인스턴스는 대기 상태로 유지되며, 활성 Leader에 장애가 발생하면 다른 인스턴스가 새로운 Leader로 선출될 수 있습니다.

이러한 구성은 Scheduler 프로세스 장애에 대비하기 위한 것이며, etcd의 데이터 복제 및 Quorum을 기반으로 하는 고가용성과는 별개의 메커니즘입니다.

### 8.2. Scheduler 장애 영향

kube-scheduler에 장애가 발생하면 새로운 Pod를 Node에 배치하는 작업이 중단되거나 지연될 수 있습니다.

그러나 이미 Node에 할당되어 실행 중인 Pod가 즉시 종료되는 것은 아닙니다.

| 상황 | 영향 |
| --- | --- |
| kube-scheduler 정상 동작 | 새로운 Pod의 스케줄링을 수행합니다. |
| kube-scheduler 프로세스 장애 | 새로운 Pod의 스케줄링이 중단됩니다. |
| Scheduler Leader 장애 | 다른 Scheduler 인스턴스가 Leader로 선출될 때까지 스케줄링이 지연될 수 있습니다. |
| 스케줄링 조건 불충족 | Pod가 Node를 할당받지 못하고 Pending 상태로 남을 수 있습니다. |
| kube-scheduler 복구 | 스케줄링 큐에 남아 있는 Pod의 처리를 재개할 수 있습니다. |

이미 실행 중인 Pod의 생명주기는 주로 kubelet과 컨테이너 런타임이 관리하므로 Scheduler 장애와 기존 Pod 실행 여부를 구분해야 합니다.

### 8.3. 주요 실행 옵션

kube-scheduler는 실행 옵션과 `KubeSchedulerConfiguration` 설정 파일을 통해 동작을 구성할 수 있습니다.

대표적인 옵션은 다음과 같습니다.

| 옵션 | 설명 |
| --- | --- |
| `--config` | Scheduler 설정 파일 경로를 지정합니다. |
| `--kubeconfig` | API Server에 접근할 때 사용할 kubeconfig 파일을 지정합니다. |
| `--master` | API Server 주소를 지정합니다. |
| `--leader-elect` | Leader Election 사용 여부를 설정합니다. |
| `--leader-elect-resource-name` | Leader Election에 사용할 리소스 이름을 지정합니다. |
| `--leader-elect-lease-duration` | Leader Lease의 유효 시간을 지정합니다. |
| `--leader-elect-renew-deadline` | Leader가 Lease를 갱신하기 위해 허용되는 시간을 지정합니다. |
| `--leader-elect-retry-period` | Leader Election 관련 작업의 재시도 간격을 지정합니다. |

실제 지원 옵션은 Kubernetes 버전에 따라 달라질 수 있으므로 운영 중인 버전의 공식 Command Line Reference를 확인해야 합니다.

## 9. 다른 Control Plane 컴포넌트와의 관계

kube-scheduler는 독립적으로 동작하는 것이 아니라 API Server를 중심으로 다른 Control Plane 컴포넌트와 협력합니다.

### 9.1. kube-apiserver

kube-apiserver는 kube-scheduler가 클러스터의 리소스 정보를 조회하고 Pod의 Node 할당 결과를 기록하기 위해 사용하는 API 인터페이스입니다.

kube-scheduler는 API Server를 통해 다음 작업을 수행합니다.

- 스케줄링 대상 Pod 감지
- Node 및 리소스 정보 조회
- 스케줄링 결과 기록
- Pod Binding 요청

kube-scheduler가 etcd에 직접 접근하는 것은 일반적인 구조가 아닙니다. Kubernetes 리소스에 대한 접근은 API Server를 통해 이루어집니다.

### 9.2. kube-controller-manager

kube-controller-manager는 Kubernetes 리소스의 실제 상태를 원하는 상태에 맞추기 위한 컨트롤러들을 실행합니다.

예를 들어 Deployment Controller는 Deployment의 Replica 수를 확인하고 필요한 ReplicaSet과 Pod가 생성되도록 조정합니다.

이후 생성된 Pod는 아직 Node가 할당되지 않은 상태이므로 kube-scheduler의 스케줄링 대상이 됩니다.

두 컴포넌트의 역할은 다음과 같이 구분할 수 있습니다.

| 컴포넌트 | 역할 |
| --- | --- |
| kube-controller-manager | 원하는 상태에 맞게 Kubernetes 리소스를 생성하고 조정합니다. |
| kube-scheduler | 생성된 Pod를 실행할 적절한 Node를 선택합니다. |

### 9.3. kubelet

kubelet은 각 Worker Node에서 실행되며, 해당 Node에 할당된 Pod의 실행을 관리합니다.

kube-scheduler가 Pod를 특정 Node에 할당하면 해당 Node의 kubelet이 API Server를 통해 할당된 Pod를 확인합니다.

이후 컨테이너 런타임과 협력하여 컨테이너를 실행합니다.

| 컴포넌트 | 역할 |
| --- | --- |
| kube-scheduler | Pod를 실행할 Node를 선택하고 할당합니다. |
| kubelet | 할당된 Pod의 컨테이너 실행과 상태 관리를 수행합니다. |

### 9.4. 전체 컴포넌트 상호작용

Deployment를 생성하여 새로운 Pod를 실행하는 과정을 기준으로 각 컴포넌트의 역할을 정리하면 다음과 같습니다.

```text
                 User
                  |
                  | kubectl apply
                  v
             kube-apiserver
                  |
                  v
                 etcd
                  |
                  | 리소스 상태 저장
                  |
                  v
       kube-controller-manager
                  |
                  | Pod 생성 및 조정
                  v
             kube-apiserver
                  |
                  v
             kube-scheduler
                  |
                  | Node 선택 및 Binding
                  v
             kube-apiserver
                  |
                  | Pod.spec.nodeName 기록
                  v
                kubelet
                  |
                  | 컨테이너 실행 요청
                  v
           Container Runtime
                  |
                  v
             Running Pod
```

각 컴포넌트의 책임을 구분하면 다음과 같습니다.

1. kube-apiserver는 Kubernetes API 요청을 처리합니다.
2. etcd는 Kubernetes 리소스 상태를 저장합니다.
3. kube-controller-manager는 원하는 상태에 맞게 Pod를 생성하고 조정합니다.
4. kube-scheduler는 Pod를 실행할 Node를 선택하고 할당합니다.
5. kubelet은 할당된 Pod를 확인하고 컨테이너를 실행합니다.

이러한 역할 분리를 통해 Kubernetes는 리소스 생성, 스케줄링, 실제 실행을 독립적인 컴포넌트가 담당하는 구조를 유지합니다.

## 10. 정리

kube-scheduler는 Kubernetes Control Plane에서 Pod를 실행할 적절한 Node를 선택하고 할당하는 역할을 담당합니다.

Pod의 Resource Requests, Node 상태, Affinity, Taint와 Toleration, Topology Spread Constraints, Volume 조건 등 다양한 요소를 고려하여 스케줄링을 수행합니다.

kube-scheduler를 이해할 때는 다음 개념을 특히 구분해서 이해해야 합니다.

- **Filtering과 Scoring:** Filtering은 Pod를 실행할 수 없는 Node를 제외하는 과정이며, Scoring은 남은 Node의 적합도를 평가하여 최적의 Node를 선택하는 과정입니다.
- **Scheduling Cycle과 Binding Cycle:** Scheduling Cycle은 Pod를 실행할 Node를 결정하고, Binding Cycle은 결정된 Node를 Pod에 실제로 할당하는 과정입니다.
- **Scheduling Framework와 Plugin:** Scheduling Framework는 스케줄링 단계별 동작을 Plugin으로 확장할 수 있도록 제공하는 구조입니다. 이를 통해 기본 스케줄링 로직을 조합하거나 사용자 정의 정책을 적용할 수 있습니다.
- **Scheduling Profile과 Multiple Scheduler:** Scheduling Profile은 하나의 Scheduler 인스턴스에서 여러 스케줄링 정책을 구성하는 기능이며, Multiple Scheduler는 여러 Scheduler를 독립적으로 실행하는 방식입니다.

kube-scheduler는 Pod를 직접 실행하는 컴포넌트가 아닙니다. **Pod가 실행될 Node를 결정하는 역할은 kube-scheduler가 담당하고, 실제 컨테이너 실행은 해당 Node의 kubelet과 컨테이너 런타임이 담당합니다.**

이러한 책임 분리를 이해하면 Kubernetes에서 Pod가 생성된 이후 Node에 배치되고 실제로 실행되기까지의 전체 흐름을 이해할 수 있습니다.

## 11. 참고자료

### Kubernetes 공식 문서

- [Kubernetes Scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
- [Scheduler Configuration](https://kubernetes.io/docs/reference/scheduling/config/)
- [Scheduler Performance Tuning](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduler-perf-tuning/)
- [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)

### Kubernetes API 및 실행 옵션

- [kube-scheduler Command Line Reference](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)
- [KubeSchedulerConfiguration API](https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/)
- [Scheduling Framework Design Proposal](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/624-scheduling-framework)