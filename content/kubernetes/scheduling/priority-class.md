# PriorityClass

## 1. PriorityClass란?

PriorityClass는 Kubernetes에서 Pod의 스케줄링 우선순위를 정의하는 클러스터 범위(Cluster-scoped)의 리소스입니다.

여러 Pod가 동시에 스케줄링을 기다리는 상황에서 Kubernetes Scheduler는 Pod의 우선순위를 고려하여 어떤 Pod를 먼저 스케줄링할지 결정합니다.

PriorityClass는 우선순위 이름과 정수 값을 매핑하며, 값이 클수록 높은 우선순위를 의미합니다.

예를 들어 여러 애플리케이션이 동일한 Node 자원을 사용하려고 할 때, 중요한 애플리케이션의 Pod에 높은 우선순위를 부여하면 일반 애플리케이션의 Pod보다 먼저 스케줄링을 시도하도록 할 수 있습니다.

또한 높은 우선순위의 Pod를 배치할 리소스가 부족한 경우 Kubernetes Scheduler는 낮은 우선순위의 Pod를 선점(Preemption)하여 필요한 리소스를 확보할 수 있습니다.

PriorityClass의 주요 특징은 다음과 같습니다.

- **스케줄링 우선순위 지정:** Pod마다 상대적인 중요도를 설정할 수 있습니다.
- **스케줄링 대기열 제어:** 높은 우선순위의 Pod를 낮은 우선순위의 Pod보다 먼저 스케줄링하도록 합니다.
- **Pod 선점:** 리소스 부족으로 스케줄링할 수 없는 경우 낮은 우선순위의 Pod를 선점할 수 있습니다.
- **클러스터 범위 설정:** Namespace에 종속되지 않으며 클러스터 전체에서 사용할 수 있습니다.
- **비선점 정책:** 다른 Pod를 축출하지 않고 스케줄링 우선순위만 높이는 방식도 지원합니다.

PriorityClass는 `scheduling.k8s.io/v1` API 그룹에 정의되어 있으며, Kubernetes v1.14부터 Stable 기능으로 제공됩니다.

## 2. PriorityClass의 동작 원리

PriorityClass는 우선순위 이름과 정수 값을 정의하고, Pod는 `priorityClassName` 필드를 통해 해당 우선순위 클래스를 참조합니다.

Kubernetes의 Priority Admission Controller는 Pod가 생성될 때 지정된 PriorityClass를 확인하고, 해당 클래스의 정수 우선순위 값을 Pod의 `.spec.priority`에 반영합니다.

이후 Kubernetes Scheduler는 Pod의 우선순위를 기준으로 스케줄링 대기열을 관리합니다.

전체적인 동작 과정은 다음과 같습니다.

```text
┌──────────────────────────────┐
│         PriorityClass        │
│                              │
│  name: high-priority         │
│  value: 1000000              │
└──────────────┬───────────────┘
               │
               │ priorityClassName
               ▼
┌──────────────────────────────┐
│          Pod Creation        │
│                              │
│  priorityClassName:          │
│    high-priority             │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│   Priority Admission         │
│        Controller            │
│                              │
│  Resolve PriorityClass       │
│  Populate spec.priority      │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      Kubernetes Scheduler   │
│                              │
│  Sort Scheduling Queue       │
│  by Pod Priority             │
└──────────────┬───────────────┘
               │
               ▼
       ┌───────────────┐
       │ Node Resources│
       │  Available?   │
       └───────┬───────┘
               │
       ┌───────┴────────┐
       │                │
       ▼                ▼
   Available         Insufficient
       │                │
       ▼                ▼
  Schedule Pod      Preemption
                       │
                       ▼
                Lower Priority
                   Pod Eviction
                       │
                       ▼
                 Schedule Pod
```

위 과정에서 PriorityClass는 Pod의 우선순위를 정의하고, Scheduler는 해당 우선순위를 활용하여 스케줄링 순서와 선점 여부를 판단합니다.

## 3. PriorityClass 리소스

PriorityClass는 다음과 같은 주요 필드로 구성됩니다.

| 필드 | 설명 |
|---|---|
| apiVersion | API 버전으로 `scheduling.k8s.io/v1`을 사용합니다. |
| kind | 리소스의 종류를 나타내며 `PriorityClass`를 사용합니다. |
| metadata.name | PriorityClass의 이름입니다. |
| value | 해당 클래스에 속하는 Pod에 부여할 정수 우선순위 값입니다. |
| globalDefault | PriorityClass를 지정하지 않은 Pod에 기본 우선순위로 사용할지 여부를 설정합니다. |
| description | 해당 PriorityClass의 사용 목적이나 적용 대상을 설명합니다. |
| preemptionPolicy | 해당 PriorityClass를 사용하는 Pod가 낮은 우선순위의 Pod를 선점할 수 있는지 지정합니다. |

### 3.1. Priority Value

`value`는 PriorityClass를 사용하는 Pod에 부여할 정수 우선순위 값입니다.

값이 클수록 높은 우선순위를 가지며, Scheduler는 이 값을 기준으로 Pod의 상대적인 중요도를 판단합니다.

일반적인 PriorityClass에서 사용할 수 있는 값의 범위는 다음과 같습니다.

\[
-2147483648 \leq value \leq 1000000000
\]

1,000,000,000보다 큰 값은 Kubernetes의 중요한 시스템 Pod를 위한 예약 영역입니다.

따라서 일반적인 애플리케이션의 PriorityClass를 생성할 때는 1,000,000,000 이하의 값을 사용해야 합니다.

예를 들어 다음과 같이 세 가지 우선순위 클래스를 정의할 수 있습니다.

| PriorityClass | Value | 용도 |
|---|---:|---|
| low-priority | 1000 | 일반적인 백그라운드 작업 |
| medium-priority | 10000 | 일반 애플리케이션 |
| high-priority | 100000 | 중요한 서비스 |

위 예시에서는 `high-priority`가 가장 높은 우선순위를 가지며, 동일한 조건에서 스케줄링을 기다리는 다른 Pod보다 먼저 처리됩니다.

단, Priority Value가 높다고 해서 Node의 리소스를 예약하거나 실행 중인 Pod의 CPU 및 Memory를 강제로 확보하는 것은 아닙니다. 리소스가 부족한 경우 선점 정책과 Scheduler의 판단에 따라 별도의 처리가 필요합니다.

### 3.2. globalDefault

`globalDefault`는 Pod에서 `priorityClassName`을 지정하지 않았을 때 기본적으로 사용할 PriorityClass를 설정합니다.

다음과 같이 설정할 수 있습니다.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: default-priority
value: 1000
globalDefault: true
description: "Default priority for application workloads"
```

이 PriorityClass가 클러스터의 기본 우선순위 클래스로 설정되면, 이후 생성되는 Pod 중 `priorityClassName`을 지정하지 않은 Pod는 해당 클래스의 우선순위 값을 사용합니다.

주요 특징은 다음과 같습니다.

- `globalDefault: true`로 설정한 PriorityClass가 기본 우선순위 클래스로 사용됩니다.
- 기본 우선순위 클래스를 별도로 지정하지 않았다면 PriorityClass를 지정하지 않은 Pod의 우선순위는 0입니다.
- 기본 우선순위 클래스는 기존에 생성된 Pod의 우선순위를 변경하지 않습니다.
- 기본 우선순위 클래스를 추가한 이후 생성되는 Pod부터 적용됩니다.
- 클러스터 운영 시 기본 우선순위 클래스를 하나만 지정하는 것이 권장됩니다.

여러 PriorityClass에 `globalDefault: true`가 설정된 경우 Kubernetes API의 동작상 가장 작은 우선순위 값이 기본값으로 사용됩니다. 다만 설정의 혼란을 방지하기 위해 하나의 PriorityClass만 기본값으로 지정하는 것이 바람직합니다.

### 3.3. description

`description`은 PriorityClass의 사용 목적을 설명하는 문자열입니다.

```yaml
description: "Priority class for critical application workloads"
```

예를 들어 다음과 같은 내용을 작성할 수 있습니다.

- 중요한 서비스의 Pod에만 사용합니다.
- 백그라운드 Batch Job에는 사용하지 않습니다.
- 클러스터 운영에 필요한 시스템 컴포넌트에 사용합니다.

description은 설명을 위한 필드이며 Scheduler의 스케줄링 동작에 직접적인 영향을 주지는 않습니다.

### 3.4. preemptionPolicy

`preemptionPolicy`는 해당 PriorityClass를 사용하는 Pod가 낮은 우선순위의 Pod를 선점할 수 있는지 지정합니다.

지원하는 값은 다음과 같습니다.

| 값 | 설명 |
|---|---|
| PreemptLowerPriority | 낮은 우선순위의 Pod를 선점할 수 있습니다. |
| Never | 다른 Pod를 선점하지 않습니다. |

기본값은 `PreemptLowerPriority`입니다.

따라서 별도로 설정하지 않으면 높은 우선순위의 Pod는 필요한 경우 낮은 우선순위의 Pod를 선점할 수 있습니다.

반면 `Never`를 설정하면 Pod는 높은 스케줄링 우선순위를 가지지만 다른 Pod를 축출하지 않습니다.

이러한 방식은 실행 중인 작업을 중단하지 않으면서 새로운 작업의 스케줄링 순서를 높이고 싶은 경우에 활용할 수 있습니다.

## 4. PriorityClass 생성 및 적용

PriorityClass를 사용하려면 먼저 PriorityClass 리소스를 생성한 다음 Pod 또는 Pod를 생성하는 상위 워크로드에 해당 클래스의 이름을 지정해야 합니다.

### 4.1. PriorityClass 생성

다음은 높은 우선순위를 가진 PriorityClass를 생성하는 예시입니다.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
description: "Priority class for critical application workloads"
```

위 설정에서는 `high-priority`라는 PriorityClass를 생성하고 우선순위 값을 100000으로 지정합니다.

`globalDefault: false`이므로 해당 PriorityClass를 명시적으로 지정한 Pod에만 적용됩니다.

다음 명령어로 리소스를 생성할 수 있습니다.

```bash
kubectl apply -f high-priority.yaml
```

PriorityClass는 Cluster-scoped 리소스이므로 Namespace를 지정하지 않습니다.

### 4.2. Pod에 PriorityClass 적용

생성한 PriorityClass를 Pod에 적용하려면 `spec.priorityClassName`을 지정합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-application
spec:
  priorityClassName: high-priority
  containers:
    - name: application
      image: nginx:1.27
```

위 Pod는 `high-priority` PriorityClass를 참조합니다.

Priority Admission Controller는 해당 PriorityClass의 `value`를 확인하여 Pod의 `.spec.priority`에 반영합니다.

다음과 같이 확인할 수 있습니다.

```bash
kubectl get pod critical-application \
  -o jsonpath='{.spec.priority}'
```

출력 예시는 다음과 같습니다.

```text
100000
```

### 4.3. Deployment에 PriorityClass 적용

실제 운영 환경에서는 Pod를 직접 생성하기보다 Deployment, StatefulSet, Job 등의 워크로드 리소스를 통해 Pod를 생성하는 경우가 많습니다.

이러한 경우 Pod Template의 `spec.priorityClassName`에 PriorityClass 이름을 지정합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-application
spec:
  replicas: 3
  selector:
    matchLabels:
      app: critical-application
  template:
    metadata:
      labels:
        app: critical-application
    spec:
      priorityClassName: high-priority
      containers:
        - name: application
          image: nginx:1.27
```

이 Deployment가 생성하는 모든 Pod는 `high-priority` 클래스를 사용합니다.

따라서 동일한 클러스터에서 실행되는 다른 Pod보다 높은 스케줄링 우선순위를 가질 수 있습니다.

## 5. Pod Priority

Pod Priority는 개별 Pod에 실제로 적용된 정수 우선순위 값입니다.

PriorityClass가 우선순위 이름과 값을 정의한다면, Pod Priority는 해당 값을 실제 Pod에 적용한 결과입니다.

Pod의 `spec.priorityClassName`을 지정하면 Priority Admission Controller가 PriorityClass를 조회하고 정수 값을 `.spec.priority`에 반영합니다.

예를 들어 다음과 같은 PriorityClass가 있다고 가정하겠습니다.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
```

이 클래스를 사용하는 Pod는 다음과 같은 형태로 저장됩니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: application
spec:
  priorityClassName: high-priority
  priority: 100000
  containers:
    - name: application
      image: nginx:1.27
```

`priorityClassName`은 PriorityClass를 참조하는 이름이며, `priority`는 실제로 적용된 정수 우선순위입니다.

일반적으로 사용자는 `priorityClassName`을 지정하고 Kubernetes가 `priority` 값을 설정하도록 합니다.

### 5.1. PriorityClass가 지정되지 않은 Pod

Pod에 `priorityClassName`이 지정되지 않은 경우 기본 우선순위가 적용됩니다.

- `globalDefault: true`인 PriorityClass가 존재하면 해당 PriorityClass의 값이 적용됩니다.
- 기본 PriorityClass가 존재하지 않으면 우선순위는 0입니다.

따라서 별도의 PriorityClass를 지정하지 않은 일반 Pod는 기본적으로 우선순위 0을 가집니다.

### 5.2. 존재하지 않는 PriorityClass 지정

Pod에서 존재하지 않는 PriorityClass를 참조하면 Pod 생성이 거부됩니다.

예를 들어 다음과 같이 설정했다고 가정하겠습니다.

```yaml
spec:
  priorityClassName: unknown-priority
```

클러스터에 `unknown-priority`라는 PriorityClass가 없다면 Priority Admission Controller가 해당 참조를 확인할 수 없으므로 Pod 생성 요청이 실패합니다.

따라서 워크로드를 배포하기 전에 PriorityClass가 정상적으로 생성되어 있는지 확인해야 합니다.

## 6. Preemption

Preemption은 높은 우선순위의 Pod를 스케줄링하기 위해 낮은 우선순위의 Pod를 선점하는 기능입니다.

Kubernetes Scheduler는 Pod를 배치할 Node를 찾지 못한 경우 리소스 부족이 원인인지 확인하고, 필요한 리소스를 확보할 수 있는 Node에서 낮은 우선순위의 Pod를 선점할 수 있는지 판단합니다.

예를 들어 다음과 같은 상황을 가정하겠습니다.

- Node의 CPU Allocatable: 4 Core
- 현재 실행 중인 Pod의 CPU Requests 합계: 4 Core
- 새로운 Pod의 CPU Request: 2 Core
- 새로운 Pod의 우선순위: 100000
- 기존 Pod의 우선순위: 1000

새로운 Pod가 배치되려면 CPU 2 Core가 추가로 필요하지만 Node에는 여유 리소스가 없습니다.

이때 Scheduler는 기존 Pod 중 새로운 Pod보다 우선순위가 낮은 Pod를 선점하여 리소스를 확보할 수 있습니다.

### 6.1. Preemption 동작 과정

Preemption은 다음과 같은 순서로 동작합니다.

1. 높은 우선순위의 Pod가 생성됩니다.
2. Scheduler가 해당 Pod를 스케줄링하려고 시도합니다.
3. Node의 가용 리소스가 부족하여 스케줄링에 실패합니다.
4. Scheduler가 낮은 우선순위의 Pod를 선점할 수 있는 Node를 탐색합니다.
5. 선점 후보 Node에서 제거할 수 있는 낮은 우선순위의 Pod를 판단합니다.
6. 선점할 Pod를 결정하고 종료를 요청합니다.
7. 기존 Pod가 종료되면서 리소스가 확보됩니다.
8. 높은 우선순위의 Pod가 해당 Node에 스케줄링됩니다.

```text
┌──────────────────────────────┐
│      High Priority Pod       │
│                              │
│       Pending State          │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      Kubernetes Scheduler   │
│                              │
│    Check Available Nodes     │
└──────────────┬───────────────┘
               │
               ▼
       ┌───────────────┐
       │  Resources    │
       │  Available?   │
       └───────┬───────┘
               │
               │ No
               ▼
┌──────────────────────────────┐
│          Preemption          │
│                              │
│  Find Lower Priority Pods    │
│                              │
│  Select Victim Pods          │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│   Lower Priority Pod         │
│                              │
│   Graceful Termination       │
└──────────────┬───────────────┘
               │
               │ Resource Released
               ▼
┌──────────────────────────────┐
│      High Priority Pod       │
│                              │
│      Scheduled on Node       │
└──────────────────────────────┘
```

### 6.2. Preemption 대상 선정

Scheduler는 단순히 가장 낮은 우선순위의 Pod 하나를 무조건 종료하는 것이 아닙니다.

새로운 Pod를 배치할 수 있도록 필요한 리소스를 확보할 수 있는지 판단하고, 가능한 선점 후보 중 적절한 대상을 선택합니다.

이때 다음과 같은 요소를 고려합니다.

- 새로운 Pod보다 우선순위가 낮은 Pod인지 확인합니다.
- 해당 Pod를 제거하면 Node의 리소스 요구량을 충족할 수 있는지 확인합니다.
- 선점 이후에도 Node의 스케줄링 조건을 만족할 수 있는지 확인합니다.
- PodDisruptionBudget(PDB) 위반 여부를 고려합니다.
- 선점으로 인해 발생하는 영향을 최소화할 수 있는 후보를 선택합니다.

Scheduler는 가능한 경우 PDB 위반을 최소화하는 선점 대상을 선택하려고 합니다.

다만 PDB는 Preemption을 절대적으로 차단하는 장치는 아닙니다. 적절한 선점 후보를 찾을 수 없는 상황에서는 PDB의 보호 조건을 위반하는 선점이 발생할 수도 있습니다.

### 6.3. Preemption과 Graceful Termination

선점 대상으로 선정된 Pod는 즉시 강제 종료되는 것이 아니라 일반적인 Pod 종료 절차에 따라 종료를 요청받습니다.

기본적으로 Pod의 `terminationGracePeriodSeconds`에 지정된 유예 시간이 적용되며, 별도로 지정하지 않았다면 기본값은 30초입니다.

이 과정에서 다음과 같은 동작이 발생할 수 있습니다.

1. Scheduler가 기존 Pod를 선점 대상으로 결정합니다.
2. Kubernetes가 기존 Pod에 종료를 요청합니다.
3. 기존 Pod는 설정된 종료 유예 시간 동안 종료 절차를 수행합니다.
4. Pod가 종료되면서 Node의 리소스가 해제됩니다.
5. 높은 우선순위의 Pod가 해당 Node에 배치됩니다.

따라서 Preemption이 발생했다고 해서 새로운 Pod가 즉시 Running 상태가 되는 것은 아닙니다.

기존 Pod의 종료가 완료되기까지 시간이 필요하며, 그 사이 더 높은 우선순위의 Pod가 생성되면 먼저 스케줄링될 수도 있습니다.

## 7. Non-preempting PriorityClass

Non-preempting PriorityClass는 높은 스케줄링 우선순위를 유지하면서 다른 Pod를 선점하지 않도록 설정하는 방식입니다.

일반적인 PriorityClass는 기본적으로 `PreemptLowerPriority` 정책을 사용하므로 리소스가 부족하면 낮은 우선순위의 Pod를 선점할 수 있습니다.

반면 `preemptionPolicy: Never`를 설정하면 다른 Pod를 선점하지 않습니다.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-nonpreempting
value: 100000
preemptionPolicy: Never
globalDefault: false
description: "High scheduling priority without preemption"
```

위 설정을 사용하는 Pod는 다른 낮은 우선순위의 Pod보다 먼저 스케줄링 대기열에서 처리되지만, 기존 Pod를 강제로 축출하지 않습니다.

### 7.1. 동작 방식

Non-preempting PriorityClass의 동작 방식은 다음과 같습니다.

1. 높은 우선순위의 Pod가 생성됩니다.
2. Scheduler는 해당 Pod를 낮은 우선순위의 Pod보다 먼저 스케줄링하려고 시도합니다.
3. Node의 가용 리소스가 부족하면 Pod는 Pending 상태로 대기합니다.
4. 기존 Pod가 종료되거나 Node에 충분한 리소스가 확보되면 높은 우선순위의 Pod를 스케줄링합니다.
5. 기존 Pod를 선점하지 않고 스케줄링을 완료합니다.

즉, Non-preempting PriorityClass는 스케줄링 대기열에서의 우선순위는 높이지만 실행 중인 Pod의 리소스를 강제로 확보하지는 않습니다.

다만 다음과 같은 특징이 있습니다.

- 다른 Pod보다 먼저 스케줄링을 시도합니다.
- 리소스가 부족하면 Pending 상태로 대기합니다.
- Scheduler의 Backoff 정책에 따라 스케줄링 재시도 간격이 달라질 수 있습니다.
- 더 높은 우선순위의 Pod가 생성되면 해당 Pod에 의해 선점될 수 있습니다.

Non-preempting PriorityClass는 Kubernetes v1.24부터 Stable 기능으로 제공됩니다.

### 7.2. 주요 활용 사례

Non-preempting PriorityClass는 기존 작업을 강제로 중단하지 않으면서 새로운 작업의 스케줄링 순서를 높이고 싶은 경우에 활용할 수 있습니다.

예를 들어 다음과 같은 상황을 생각할 수 있습니다.

- 기존 Batch Job의 실행을 중단하고 싶지 않은 경우
- 새로운 데이터 분석 작업을 일반 작업보다 먼저 처리하고 싶은 경우
- 리소스가 확보될 때까지 기다릴 수 있지만 다른 작업보다 높은 우선순위를 부여하고 싶은 경우

이러한 환경에서는 `preemptionPolicy: Never`를 사용하여 실행 중인 작업을 보호하면서 스케줄링 우선순위를 조정할 수 있습니다.

## 8. Built-in PriorityClass

Kubernetes는 클러스터의 핵심 시스템 컴포넌트를 보호하기 위해 기본 PriorityClass를 제공합니다.

대표적인 클래스는 다음과 같습니다.

| PriorityClass | 용도 |
|---|---|
| system-cluster-critical | 클러스터 전체의 핵심 기능을 담당하는 시스템 컴포넌트 |
| system-node-critical | 개별 Node의 핵심 기능을 담당하는 시스템 컴포넌트 |

이러한 PriorityClass는 일반 애플리케이션보다 훨씬 높은 우선순위 값을 사용합니다.

예를 들어 다음과 같은 시스템 컴포넌트가 해당 우선순위 클래스를 사용할 수 있습니다.

- CoreDNS
- Kubernetes 네트워크 플러그인
- Node의 핵심 시스템 컴포넌트
- 클러스터 운영에 필요한 주요 서비스

정확한 적용 대상과 우선순위 값은 Kubernetes 배포 환경 및 버전에 따라 다를 수 있습니다.

일반적으로 애플리케이션 개발자가 직접 시스템 PriorityClass를 사용하는 것은 권장되지 않습니다.

클러스터 운영에 필요한 핵심 시스템 컴포넌트에 예약된 우선순위이므로 일반 워크로드에는 별도의 PriorityClass를 생성하여 사용하는 것이 적절합니다.

## 9. PriorityClass와 Scheduler의 관계

PriorityClass는 Kubernetes Scheduler의 스케줄링 대기열 및 선점 판단에 영향을 줍니다.

Scheduler는 단순히 Pod가 생성된 순서대로 스케줄링하지 않습니다.

여러 Pod가 스케줄링을 기다리는 경우 Priority를 고려하여 처리 순서를 결정합니다.

### 9.1. Scheduling Queue

Kubernetes Scheduler는 스케줄링이 필요한 Pod를 대기열에 저장하고 처리합니다.

높은 우선순위의 Pod는 낮은 우선순위의 Pod보다 먼저 스케줄링 대상으로 처리됩니다.

예를 들어 다음과 같은 Pod가 대기 중이라고 가정하겠습니다.

| Pod | Priority | 생성 순서 |
|---|---:|---:|
| pod-a | 1000 | 1 |
| pod-b | 100000 | 2 |
| pod-c | 10000 | 3 |

이 경우 일반적인 우선순위 기준의 처리 순서는 다음과 같습니다.

1. pod-b
2. pod-c
3. pod-a

즉, Pod가 생성된 순서보다 우선순위가 높은 Pod가 먼저 처리됩니다.

단, PriorityClass는 스케줄링 대기열에서의 처리 순서를 결정하는 요소이며, 높은 우선순위가 항상 먼저 Running 상태가 된다는 의미는 아닙니다.

Node의 리소스 부족이나 Affinity, Taint/Toleration 등의 스케줄링 조건을 만족하지 못하면 높은 우선순위의 Pod도 Pending 상태로 대기할 수 있습니다.

### 9.2. Scheduling과 Preemption의 관계

Priority와 Preemption은 서로 밀접하게 연결되어 있지만 동일한 개념은 아닙니다.

| 구분 | Priority | Preemption |
|---|---|---|
| 역할 | Pod의 상대적인 중요도 지정 | 리소스 확보를 위한 낮은 우선순위 Pod 선점 |
| 주요 시점 | 스케줄링 대기열에서 Pod 처리 시 | 스케줄링에 필요한 리소스가 부족할 때 |
| 동작 | 높은 우선순위 Pod를 먼저 처리 | 낮은 우선순위 Pod를 종료하여 리소스 확보 |
| 설정 | PriorityClass의 value | PriorityClass의 preemptionPolicy 및 Scheduler 판단 |

따라서 높은 우선순위의 Pod가 항상 다른 Pod를 선점하는 것은 아닙니다.

Node에 충분한 리소스가 있다면 일반적인 스케줄링을 통해 배치되고, 리소스가 부족한 경우에만 Preemption이 고려됩니다.

## 10. PriorityClass와 다른 Scheduling 리소스의 관계

PriorityClass는 Pod의 스케줄링 순서와 선점 여부를 제어하는 기능입니다.

다른 Scheduling 리소스와 함께 사용하면 보다 세밀한 워크로드 스케줄링 정책을 구성할 수 있습니다.

### 10.1. PriorityClass와 ResourceQuota

PriorityClass를 사용하면 특정 워크로드에 높은 우선순위를 부여할 수 있습니다.

그러나 모든 사용자가 높은 우선순위의 Pod를 생성할 수 있다면 일반 워크로드가 반복적으로 선점되거나 스케줄링되지 못하는 문제가 발생할 수 있습니다.

따라서 ResourceQuota를 활용하여 특정 Namespace에서 사용할 수 있는 PriorityClass의 범위와 Pod 개수를 제한할 수 있습니다.

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: priority-quota
  namespace: production
spec:
  hard:
    pods: "20"
    requests.cpu: "20"
    requests.memory: 40Gi
    count/pods.high-priority: "5"
```

위 예시는 `production` Namespace에서 `high-priority` 클래스를 사용하는 Pod의 개수를 최대 5개로 제한하는 형태입니다.

PriorityClass에 대한 ResourceQuota를 적용하면 높은 우선순위의 Pod가 무분별하게 생성되는 상황을 방지할 수 있습니다.

### 10.2. PriorityClass와 PodDisruptionBudget

PriorityClass는 높은 우선순위의 Pod를 스케줄링하기 위해 낮은 우선순위의 Pod를 선점할 수 있습니다.

이때 선점 대상 Pod에 PDB가 설정되어 있다면 Scheduler는 가능한 경우 PDB 위반을 최소화하는 대상을 선택합니다.

그러나 PDB는 선점을 완전히 차단하지 않습니다.

따라서 PriorityClass와 PDB를 함께 사용하는 환경에서는 높은 우선순위의 워크로드가 기존 애플리케이션의 가용성에 미치는 영향을 고려해야 합니다.

### 10.3. PriorityClass와 Node Affinity

Node Affinity는 Pod가 배치될 수 있는 Node의 조건을 지정합니다.

PriorityClass는 Pod의 스케줄링 우선순위를 결정하지만 Node Affinity 조건을 무시하지는 않습니다.

예를 들어 높은 우선순위의 Pod라도 특정 Node에만 배치되도록 설정되어 있고 해당 Node에 필요한 리소스가 없다면 스케줄링되지 못할 수 있습니다.

따라서 PriorityClass와 Node Affinity를 함께 사용할 때는 우선순위뿐만 아니라 실제 배치 가능한 Node의 범위도 고려해야 합니다.

### 10.4. PriorityClass와 Taint/Toleration

Taint/Toleration은 특정 Node에 Pod가 배치될 수 있는 조건을 제어합니다.

PriorityClass는 Taint/Toleration을 무시하거나 우회하는 기능이 아닙니다.

높은 우선순위의 Pod라도 해당 Node의 Taint를 허용하지 않으면 그 Node에 배치될 수 없습니다.

따라서 PriorityClass는 Pod의 상대적인 중요도를 지정하고, Taint/Toleration은 Pod가 배치될 수 있는 Node의 조건을 제어하는 기능으로 구분해야 합니다.

## 11. PriorityClass 운영 시 고려사항

PriorityClass를 운영 환경에 적용할 때는 다음 사항을 고려해야 합니다.

### 11.1. Priority Value의 계층화

Priority Value를 무분별하게 설정하면 워크로드 간 우선순위 정책이 복잡해질 수 있습니다.

따라서 서비스의 중요도와 운영 정책에 따라 우선순위 구간을 구분하는 것이 좋습니다.

예를 들어 다음과 같이 설정할 수 있습니다.

| 구분 | Priority Value | 대상 |
|---|---:|---|
| Critical | 100000 | 핵심 서비스 |
| High | 50000 | 중요 서비스 |
| Normal | 10000 | 일반 서비스 |
| Low | 1000 | 백그라운드 작업 |

이러한 계층을 정의하면 워크로드별 우선순위 정책을 일관되게 유지할 수 있습니다.

### 11.2. Preemption에 따른 서비스 영향

높은 우선순위의 Pod가 낮은 우선순위의 Pod를 선점하면 기존 애플리케이션의 처리량이 감소하거나 서비스에 영향을 줄 수 있습니다.

따라서 모든 중요한 워크로드에 높은 Priority Value를 부여하는 것은 적절하지 않습니다.

실제 서비스 중요도와 선점으로 인해 발생할 수 있는 영향을 함께 고려해야 합니다.

### 11.3. 무분별한 높은 우선순위 사용 방지

사용자가 임의로 높은 우선순위의 Pod를 생성할 수 있다면 다른 워크로드가 반복적으로 선점될 수 있습니다.

특히 여러 팀이 하나의 Kubernetes 클러스터를 공유하는 환경에서는 PriorityClass 사용 권한을 관리해야 합니다.

다음과 같은 방법을 활용할 수 있습니다.

- Namespace별 ResourceQuota 설정
- PriorityClass 사용 정책 수립
- RBAC를 통한 PriorityClass 생성 및 관리 권한 제한
- 높은 우선순위 클래스를 사용할 수 있는 워크로드 제한

### 11.4. PriorityClass 변경 및 삭제

PriorityClass의 `value`는 생성 후 변경할 수 없는 필드입니다.

따라서 우선순위 값을 변경하려면 기존 PriorityClass를 삭제하고 새로운 값으로 다시 생성해야 합니다.

다만 기존 Pod는 이미 적용된 `.spec.priority` 값을 유지하므로 PriorityClass를 변경하거나 삭제했다고 해서 실행 중인 Pod의 우선순위가 자동으로 변경되지는 않습니다.

새로운 PriorityClass를 적용하려면 해당 클래스를 참조하는 Pod를 새로 생성해야 합니다.

또한 기존 Pod가 삭제된 PriorityClass의 이름을 참조하고 있더라도 이미 생성된 Pod 자체는 유지됩니다. 다만 해당 이름으로 새로운 Pod를 생성하려면 PriorityClass가 다시 존재해야 합니다.

## 12. PriorityClass 관련 주요 명령어

| 명령어 | 설명 |
|---|---|
| `kubectl get priorityclass` | 클러스터의 PriorityClass 목록을 확인합니다. |
| `kubectl describe priorityclass <name>` | 특정 PriorityClass의 상세 정보를 확인합니다. |
| `kubectl get priorityclass <name> -o yaml` | PriorityClass의 전체 YAML 정보를 확인합니다. |
| `kubectl get pods -o wide` | Pod의 상태와 배치된 Node를 확인합니다. |
| `kubectl get pod <name> -o yaml` | Pod의 전체 YAML 정보를 확인합니다. |
| `kubectl get pod <name> -o jsonpath='{.spec.priority}'` | Pod에 적용된 실제 우선순위 값을 확인합니다. |
| `kubectl get events --sort-by=.lastTimestamp` | 스케줄링 및 선점 관련 이벤트를 확인합니다. |

PriorityClass 목록은 다음 명령어로 확인할 수 있습니다.

```bash
kubectl get priorityclass
```

Pod에 적용된 우선순위는 다음 명령어로 확인할 수 있습니다.

```bash
kubectl get pod application \
  -o jsonpath='{.spec.priorityClassName}{" "}{.spec.priority}'
```

출력 예시는 다음과 같습니다.

```text
high-priority 100000
```

스케줄링 실패 및 선점 관련 이벤트는 다음 명령어로 확인할 수 있습니다.

```bash
kubectl get events --sort-by=.lastTimestamp
```

Pod의 상세 정보를 확인하면 `FailedScheduling` 등의 이벤트를 통해 리소스 부족이나 스케줄링 조건 불충족 여부를 파악할 수 있습니다.

## 13. 정리

PriorityClass는 Kubernetes에서 Pod의 상대적인 스케줄링 우선순위를 정의하는 Cluster-scoped 리소스입니다.

PriorityClass는 이름과 정수 우선순위 값을 매핑하며, Pod의 `priorityClassName`을 통해 적용됩니다.

Kubernetes의 Priority Admission Controller는 해당 PriorityClass의 값을 Pod의 `.spec.priority`에 반영하고, Scheduler는 이 값을 기준으로 스케줄링 대기열의 처리 순서를 결정합니다.

또한 높은 우선순위의 Pod를 배치할 리소스가 부족한 경우 낮은 우선순위의 Pod를 선점하는 Preemption 기능을 활용할 수 있습니다.

`preemptionPolicy: Never`를 설정하면 다른 Pod를 선점하지 않으면서 스케줄링 대기열에서 높은 우선순위를 유지할 수도 있습니다.

다만 PriorityClass는 Node의 리소스를 예약하거나 모든 스케줄링 조건을 우회하는 기능이 아닙니다. 높은 우선순위의 Pod라도 Node Affinity, Taint/Toleration 및 리소스 요구량 등의 조건을 충족하지 못하면 스케줄링되지 않을 수 있습니다.

따라서 PriorityClass는 ResourceQuota, PodDisruptionBudget, Affinity 및 Taint/Toleration 등의 기능과 함께 고려하여 워크로드의 중요도와 클러스터 자원을 효율적으로 관리하는 데 활용해야 합니다.

## 14. 참고자료

- [Kubernetes 공식 문서 - Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
- [Kubernetes 공식 문서 - PriorityClass API Reference](https://kubernetes.io/docs/reference/kubernetes-api/scheduling/priority-class-v1/)
- [Kubernetes 공식 문서 - Advanced Pod Configuration](https://kubernetes.io/docs/concepts/workloads/pods/advanced-pod-config/)
- [Kubernetes 공식 문서 - Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Kubernetes 공식 문서 - Pod Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Kubernetes 공식 문서 - Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Kubernetes 공식 문서 - Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)