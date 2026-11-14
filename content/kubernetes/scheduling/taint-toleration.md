

# Taint와 Toleration

## 1. Taint와 Toleration이란?

`Taint`와 `Toleration`은 Kubernetes에서 특정 Node에 Pod가 배치되는 것을 제한하거나 허용하기 위한 스케줄링 기능입니다.

일반적으로 Kubernetes Scheduler는 Pod의 리소스 요구사항, Node의 가용 리소스 및 다양한 스케줄링 조건을 고려하여 Pod를 배치합니다. 하지만 클러스터 운영 환경에서는 특정 Node를 특정 용도의 Pod만 사용하도록 제한해야 하는 경우가 있습니다.

예를 들어 다음과 같은 상황에서 활용할 수 있습니다.

- GPU가 장착된 Node를 GPU 작업을 수행하는 Pod 전용으로 사용
- 데이터베이스 전용 Node에 일반 애플리케이션 Pod가 배치되지 않도록 제한
- 유지보수 대상 Node에 새로운 Pod가 배치되지 않도록 설정
- 장애가 발생한 Node에서 실행 중인 Pod를 제거
- 특정 환경이나 워크로드에 적합한 Node만 사용하도록 제한

Taint와 Toleration은 서로 다른 역할을 수행합니다.

| 구분 | 설명 |
|---|---|
| Taint | Node에 설정하여 특정 Pod가 해당 Node에 배치되는 것을 제한합니다. |
| Toleration | Pod에 설정하여 특정 Taint가 있는 Node에 배치될 수 있도록 허용합니다. |

Taint는 Node에 적용하고, Toleration은 Pod에 적용합니다.

다만 Toleration은 특정 Taint를 허용하는 조건일 뿐, 해당 Node에 Pod가 반드시 배치되도록 보장하지는 않습니다. 실제 Node 선택은 `nodeSelector`, Node Affinity 및 Scheduler의 다른 조건에 따라 결정됩니다.

## 2. Taint의 동작 원리

Taint는 Node에 설정하는 속성으로, 해당 Node에 배치될 수 있는 Pod를 제한합니다.

Taint는 다음 세 가지 요소로 구성됩니다.

- `key`: Taint의 식별자입니다.
- `value`: Taint에 연결되는 값입니다.
- `effect`: 해당 Taint가 Pod에 미치는 영향을 정의합니다.

기본적인 형식은 다음과 같습니다.

```text
key=value:effect
```

예를 들어 다음과 같은 Taint를 설정할 수 있습니다.

```text
dedicated=database:NoSchedule
```

이 설정은 `dedicated=database` Taint를 허용하지 않는 Pod가 해당 Node에 스케줄링되지 않도록 제한합니다.

Taint는 Node에 여러 개 설정할 수 있으며, Scheduler는 Pod의 Toleration과 Node의 Taint를 비교하여 배치 가능 여부를 판단합니다.

## 3. Taint Effect

Taint의 `effect`는 해당 Taint가 Pod에 미치는 영향을 결정합니다.

Kubernetes는 다음 세 가지 Effect를 지원합니다.

| Effect | 설명 |
|---|---|
| `NoSchedule` | Taint를 허용하지 않는 새로운 Pod를 해당 Node에 스케줄링하지 않습니다. |
| `PreferNoSchedule` | Taint를 허용하지 않는 Pod가 해당 Node에 배치되는 것을 가능한 한 피합니다. |
| `NoExecute` | Taint를 허용하지 않는 새로운 Pod의 스케줄링을 제한하고, 이미 실행 중인 Pod도 제거합니다. |

### 3.1. NoSchedule

`NoSchedule`은 Taint를 허용하지 않는 Pod가 해당 Node에 새롭게 배치되지 않도록 제한합니다.

예를 들어 다음과 같이 설정할 수 있습니다.

```bash
kubectl taint nodes node-1 dedicated=database:NoSchedule
```

이후 `dedicated=database:NoSchedule` Taint를 허용하지 않는 Pod는 `node-1`에 스케줄링되지 않습니다.

다만 Taint를 설정하기 전에 이미 해당 Node에서 실행 중이던 Pod는 자동으로 제거되지 않습니다.

따라서 `NoSchedule`은 새로운 Pod의 배치를 제한하는 용도로 사용합니다.

### 3.2. PreferNoSchedule

`PreferNoSchedule`은 Taint를 허용하지 않는 Pod가 해당 Node에 배치되는 것을 가능한 한 피하도록 설정합니다.

```bash
kubectl taint nodes node-1 dedicated=database:PreferNoSchedule
```

Scheduler는 해당 Taint를 허용하지 않는 Pod를 다른 Node에 배치하는 방향으로 고려합니다.

하지만 다른 Node에 리소스가 부족하거나 스케줄링 조건을 만족하는 Node가 없다면 해당 Node에 Pod가 배치될 수도 있습니다.

즉, `PreferNoSchedule`은 강제적인 스케줄링 제한이 아니라 선호도에 가까운 조건입니다.

### 3.3. NoExecute

`NoExecute`는 Taint를 허용하지 않는 Pod의 신규 배치를 제한할 뿐 아니라, 이미 실행 중인 Pod도 해당 Node에서 제거합니다.

```bash
kubectl taint nodes node-1 maintenance=true:NoExecute
```

위 명령어를 실행하면 다음과 같이 동작합니다.

- Taint를 허용하지 않는 새로운 Pod는 해당 Node에 스케줄링되지 않습니다.
- 이미 실행 중인 Pod 중 해당 Taint를 허용하지 않는 Pod는 제거됩니다.
- 해당 Taint를 허용하는 Pod는 계속 실행될 수 있습니다.

`NoExecute`는 Node의 상태가 비정상적이거나 특정 Node에서 Pod를 제거해야 하는 상황에서 활용할 수 있습니다.

다만 Pod에 `tolerationSeconds`가 설정되어 있다면 해당 시간 동안 Taint를 허용한 후 제거됩니다.

## 4. Toleration

Toleration은 Pod에 설정하여 특정 Taint가 있는 Node에 배치될 수 있도록 허용하는 조건입니다.

예를 들어 다음과 같은 Taint가 설정된 Node가 있다고 가정하겠습니다.

```text
dedicated=database:NoSchedule
```

일반적인 Pod는 해당 Taint를 허용하지 않으므로 Node에 스케줄링되지 않습니다.

하지만 Pod에 다음과 같은 Toleration을 추가하면 해당 Taint를 허용할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database-pod
spec:
  tolerations:
    - key: dedicated
      operator: Equal
      value: database
      effect: NoSchedule
  containers:
    - name: database
      image: nginx
```

이 Pod는 `dedicated=database:NoSchedule` Taint가 설정된 Node에 배치될 수 있습니다.

다만 Toleration을 설정했다고 해서 반드시 해당 Node가 선택되는 것은 아닙니다.

예를 들어 클러스터에 여러 Node가 존재하고 다른 스케줄링 조건에 의해 다른 Node가 선택될 수 있습니다.

특정 Node에만 배치하려면 Node Affinity나 `nodeSelector`를 함께 사용해야 합니다.

## 5. Toleration의 주요 필드

Toleration은 다음과 같은 필드로 구성됩니다.

| 필드 | 설명 |
|---|---|
| `key` | 허용할 Taint의 Key입니다. |
| `operator` | Taint의 Key와 Value를 비교하는 방식을 지정합니다. |
| `value` | 허용할 Taint의 Value입니다. |
| `effect` | 허용할 Taint의 Effect를 지정합니다. |
| `tolerationSeconds` | `NoExecute` Taint를 허용하는 시간을 초 단위로 지정합니다. |

### 5.1. operator

`operator`는 Taint의 Key와 Value를 어떤 방식으로 비교할지 결정합니다.

주요 Operator는 다음과 같습니다.

| Operator | 설명 |
|---|---|
| `Equal` | Taint의 Key와 Value가 지정한 값과 일치해야 합니다. |
| `Exists` | Taint의 Key가 존재하면 허용합니다. Value는 비교하지 않습니다. |

`Equal`은 Toleration의 기본 Operator입니다.

다음 예제는 `dedicated=database:NoSchedule` Taint를 허용합니다.

```yaml
tolerations:
  - key: dedicated
    operator: Equal
    value: database
    effect: NoSchedule
```

반면 `Exists`를 사용하면 동일한 Key를 가진 Taint의 Value와 관계없이 허용할 수 있습니다.

```yaml
tolerations:
  - key: dedicated
    operator: Exists
    effect: NoSchedule
```

위 설정은 `dedicated`라는 Key를 가진 `NoSchedule` Taint를 Value와 관계없이 허용합니다.

또한 다음과 같이 `key`와 `effect`를 모두 생략하고 `operator: Exists`를 지정하면 모든 Taint를 허용하는 형태가 됩니다.

```yaml
tolerations:
  - operator: Exists
```

이 설정은 모든 Key와 Value 및 Effect의 Taint를 허용하므로, Node에 설정된 Taint를 통한 스케줄링 제한을 사실상 무력화할 수 있습니다.

따라서 특별한 목적이 없다면 무분별하게 사용하지 않는 것이 좋습니다.

### 5.2. effect 생략

Toleration에서 `effect`를 생략하면 해당 Key와 Operator 조건에 일치하는 Taint의 모든 Effect를 허용합니다.

예를 들어 다음 설정은 `dedicated=database` Taint의 Effect가 `NoSchedule`, `PreferNoSchedule`, `NoExecute` 중 무엇이든 허용합니다.

```yaml
tolerations:
  - key: dedicated
    operator: Equal
    value: database
```

특정 Effect만 허용해야 한다면 `effect`를 명시하는 것이 좋습니다.

### 5.3. tolerationSeconds

`tolerationSeconds`는 `NoExecute` Taint를 일정 시간 동안 허용한 뒤 Pod를 제거하도록 설정하는 필드입니다.

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
tolerations:
  - key: node.kubernetes.io/unreachable
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 300
```

이 설정은 `node.kubernetes.io/unreachable:NoExecute` Taint가 적용된 Node에서 Pod가 최대 300초 동안 유지될 수 있도록 합니다.

지정한 시간이 지나면 Pod는 해당 Node에서 제거 대상이 됩니다.

`tolerationSeconds`는 `NoExecute` Effect에만 적용됩니다.

또한 다음과 같이 동작합니다.

- 필드를 생략하면 해당 Taint를 시간 제한 없이 허용합니다.
- `0`으로 설정하면 해당 Taint가 적용될 때 즉시 제거 대상이 됩니다.
- 양의 정수로 설정하면 지정한 시간 동안 허용합니다.

## 6. 여러 Taint가 설정된 Node의 동작

하나의 Node에는 여러 Taint를 설정할 수 있습니다.

예를 들어 다음과 같은 Taint가 설정된 Node를 가정하겠습니다.

```text
dedicated=database:NoSchedule
gpu=nvidia:NoSchedule
maintenance=true:NoExecute
```

이 Node에 Pod를 배치하려면 해당 Pod가 모든 Taint를 허용해야 합니다.

즉, 하나의 Taint라도 허용하지 못하면 해당 Taint의 Effect에 따라 Pod의 스케줄링이 제한되거나 제거될 수 있습니다.

예를 들어 다음과 같이 Toleration을 설정하면 두 개의 `NoSchedule` Taint는 허용하지만 `NoExecute` Taint는 허용하지 않습니다.

```yaml
tolerations:
  - key: dedicated
    operator: Equal
    value: database
    effect: NoSchedule
  - key: gpu
    operator: Equal
    value: nvidia
    effect: NoSchedule
```

이 경우 새로운 Pod는 `maintenance=true:NoExecute` Taint 때문에 해당 Node에 배치되지 않으며, 이미 실행 중인 Pod라면 해당 Taint를 허용하지 않는 경우 제거 대상이 됩니다.

## 7. Taint와 Toleration의 조합

Taint와 Toleration은 Effect에 따라 서로 다른 동작을 수행합니다.

| Node Taint | Pod Toleration | 결과 |
|---|---|---|
| `NoSchedule` | 일치하는 Toleration 없음 | 새로운 Pod 배치 제한 |
| `NoSchedule` | 일치하는 Toleration 있음 | 다른 조건을 만족하면 배치 가능 |
| `PreferNoSchedule` | 일치하는 Toleration 없음 | 가능한 한 배치 회피 |
| `PreferNoSchedule` | 일치하는 Toleration 있음 | 해당 Taint에 의한 회피 조건 없음 |
| `NoExecute` | 일치하는 Toleration 없음 | 신규 배치 제한 및 기존 Pod 제거 |
| `NoExecute` | 일치하는 Toleration 있음 | 실행 유지 가능 |
| `NoExecute` | 시간 제한이 있는 Toleration | 지정한 시간 이후 제거 대상 |

여기서 중요한 점은 Toleration이 Taint에 의한 제한을 허용할 뿐, 다른 스케줄링 조건까지 무시하는 것은 아니라는 점입니다.

예를 들어 Pod가 특정 Taint를 허용하더라도 다음과 같은 조건을 만족하지 못하면 해당 Node에 배치될 수 없습니다.

- CPU 및 Memory Request를 수용할 수 있는 가용 리소스
- Node Affinity 및 `nodeSelector`
- Pod Affinity 및 Pod Anti-Affinity
- 기타 Scheduler의 스케줄링 제약 조건

## 8. Node에 Taint 설정 및 제거하기

Taint는 `kubectl taint` 명령어를 사용하여 Node에 설정하거나 제거할 수 있습니다.

### 8.1. Taint 추가

```bash
kubectl taint nodes node-1 dedicated=database:NoSchedule
```

`node-1`에 `dedicated=database:NoSchedule` Taint를 추가합니다.

### 8.2. 여러 Taint 추가

```bash
kubectl taint nodes node-1 dedicated=database:NoSchedule
kubectl taint nodes node-1 gpu=nvidia:NoSchedule
```

동일한 Node에 여러 Taint를 추가할 수 있습니다.

### 8.3. Taint 확인

```bash
kubectl describe node node-1
```

Node의 상세 정보에서 `Taints` 항목을 확인할 수 있습니다.

전체 Node의 Taint를 확인하려면 다음 명령어를 사용할 수 있습니다.

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

### 8.4. Taint 제거

Taint를 제거할 때는 Key와 Effect를 지정하고 마지막에 `-`를 붙입니다.

```bash
kubectl taint nodes node-1 dedicated:NoSchedule-
```

`node-1`에서 `dedicated` Key와 `NoSchedule` Effect를 가진 Taint를 제거합니다.

특정 Value까지 포함하여 지정하려면 다음과 같이 사용할 수도 있습니다.

```bash
kubectl taint nodes node-1 dedicated=database:NoSchedule-
```

## 9. Taint와 Toleration 활용 예제

### 9.1. GPU 전용 Node 구성

GPU가 장착된 Node를 GPU 작업을 수행하는 Pod 전용으로 사용하려는 상황을 가정하겠습니다.

먼저 GPU Node에 Taint를 설정합니다.

```bash
kubectl taint nodes gpu-node-1 accelerator=nvidia:NoSchedule
```

이후 GPU 작업을 수행하는 Pod에 Toleration을 추가합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  tolerations:
    - key: accelerator
      operator: Equal
      value: nvidia
      effect: NoSchedule
  containers:
    - name: app
      image: nginx
      resources:
        limits:
          nvidia.com/gpu: 1
```

이 Pod는 `accelerator=nvidia:NoSchedule` Taint를 허용하므로 GPU Node에 배치될 수 있습니다.

다만 Toleration만으로 GPU Node가 반드시 선택되는 것은 아닙니다.

GPU Node에만 배치되도록 보장하려면 Node Affinity 또는 `nodeSelector`를 함께 설정해야 합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  nodeSelector:
    accelerator: nvidia
  tolerations:
    - key: accelerator
      operator: Equal
      value: nvidia
      effect: NoSchedule
  containers:
    - name: app
      image: nginx
      resources:
        limits:
          nvidia.com/gpu: 1
```

이 구성은 다음 두 가지 조건을 함께 적용합니다.

- `nodeSelector`: `accelerator=nvidia` Label이 설정된 Node를 선택합니다.
- `tolerations`: `accelerator=nvidia:NoSchedule` Taint를 허용합니다.

따라서 GPU Node를 일반 Pod로부터 격리하면서 GPU 작업을 수행하는 Pod만 해당 Node에 배치되도록 구성할 수 있습니다.

### 9.2. 데이터베이스 전용 Node 구성

데이터베이스 전용 Node에 일반 애플리케이션 Pod가 배치되지 않도록 구성할 수 있습니다.

먼저 Node에 Taint를 설정합니다.

```bash
kubectl taint nodes db-node-1 dedicated=database:NoSchedule
```

데이터베이스 Pod에는 다음과 같이 Toleration을 추가합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database
spec:
  tolerations:
    - key: dedicated
      operator: Equal
      value: database
      effect: NoSchedule
  containers:
    - name: database
      image: nginx
```

이제 해당 Taint를 허용하는 Pod만 `db-node-1`에 배치될 수 있습니다.

다만 실제 데이터베이스 Pod만 해당 Node에 배치하려면 별도의 Node Affinity 또는 `nodeSelector`를 함께 적용하는 것이 좋습니다.

### 9.3. 유지보수 대상 Node에서 Pod 제거

특정 Node를 유지보수하기 위해 새로운 Pod의 배치를 막고 기존 Pod도 제거해야 하는 상황을 가정하겠습니다.

다음과 같이 `NoExecute` Taint를 설정할 수 있습니다.

```bash
kubectl taint nodes node-1 maintenance=true:NoExecute
```

이후 해당 Taint를 허용하지 않는 기존 Pod는 Node에서 제거 대상이 됩니다.

다만 실제 Node 유지보수 작업에서는 Pod의 종료 및 재생성, PersistentVolume 연결 상태, PodDisruptionBudget 등을 함께 고려해야 합니다.

특히 `NoExecute` Taint는 Eviction API를 통한 일반적인 Node Drain과 동일한 동작이 아니므로, 유지보수 목적이라면 `kubectl drain`과의 차이를 이해하고 적절한 절차를 선택해야 합니다.

## 10. Kubernetes의 기본 Taint와 Toleration

Kubernetes는 Node 상태에 따라 특정 Taint를 자동으로 설정할 수 있습니다.

대표적인 Taint는 다음과 같습니다.

| Taint | 설명 |
|---|---|
| `node.kubernetes.io/not-ready` | Node가 Ready 상태가 아닙니다. |
| `node.kubernetes.io/unreachable` | Node Controller가 Node와 통신할 수 없습니다. |
| `node.kubernetes.io/memory-pressure` | Node에 Memory Pressure가 발생했습니다. |
| `node.kubernetes.io/disk-pressure` | Node에 Disk Pressure가 발생했습니다. |
| `node.kubernetes.io/pid-pressure` | Node에 PID Pressure가 발생했습니다. |
| `node.kubernetes.io/network-unavailable` | Node의 네트워크를 사용할 수 없습니다. |
| `node.kubernetes.io/unschedulable` | Node가 스케줄링 불가능한 상태입니다. |
| `node.kubernetes.io/out-of-service` | Node가 서비스에서 제외된 상태입니다. |

이러한 Taint는 Node의 상태나 운영 상황에 따라 Kubernetes 구성요소 또는 관리자가 설정할 수 있습니다.

### 10.1. NotReady와 Unreachable

Node가 정상적으로 동작하지 않거나 Control Plane과 통신할 수 없는 경우 다음과 같은 Taint가 설정될 수 있습니다.

```text
node.kubernetes.io/not-ready:NoExecute
node.kubernetes.io/unreachable:NoExecute
```

`NoExecute`가 적용되면 해당 Taint를 허용하지 않는 Pod는 Node에서 제거 대상이 됩니다.

Kubernetes는 일반 Pod에 대해 다음과 같은 기본 Toleration을 자동으로 추가할 수 있습니다.

```yaml
tolerations:
  - key: node.kubernetes.io/not-ready
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 300
  - key: node.kubernetes.io/unreachable
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 300
```

이 설정은 Node가 `NotReady` 또는 `Unreachable` 상태가 되었을 때 일반 Pod가 최대 300초 동안 해당 Taint를 허용하도록 합니다.

따라서 Node에 장애가 발생하더라도 Pod가 즉시 제거되는 것이 아니라 일정 시간 동안 상태를 유지할 수 있습니다.

다만 이 기본값은 모든 Pod에 동일하게 적용되는 것은 아닙니다. Kubernetes가 자동으로 추가하는 기본 Toleration은 Pod 유형 및 클러스터 설정에 따라 달라질 수 있습니다.

### 10.2. DaemonSet의 기본 Toleration

DaemonSet Pod에는 일반적인 Pod와 달리 여러 Node 상태 관련 Taint를 허용하는 기본 Toleration이 자동으로 추가됩니다.

대표적으로 다음과 같은 Taint를 허용합니다.

- `node.kubernetes.io/not-ready`
- `node.kubernetes.io/unreachable`
- `node.kubernetes.io/disk-pressure`
- `node.kubernetes.io/memory-pressure`
- `node.kubernetes.io/pid-pressure`
- `node.kubernetes.io/unschedulable`
- `node.kubernetes.io/network-unavailable` (HostNetwork Pod 관련 기본 허용)

이러한 기본 Toleration은 Node 상태가 일시적으로 변경되더라도 DaemonSet Pod가 가능한 한 해당 Node에 유지되거나 실행될 수 있도록 지원합니다.

예를 들어 로그 수집기, 모니터링 에이전트, 네트워크 플러그인처럼 Node 단위로 실행되어야 하는 구성요소는 DaemonSet으로 배포하는 경우가 많습니다.

### 10.3. Control Plane Node의 Taint

일반적인 kubeadm 기반 Kubernetes 클러스터에서는 Control Plane Node에 다음과 같은 Taint가 설정될 수 있습니다.

```text
node-role.kubernetes.io/control-plane:NoSchedule
```

이 Taint는 일반 애플리케이션 Pod가 Control Plane Node에 배치되는 것을 방지합니다.

Control Plane Node에도 일반 Pod를 배치하려면 해당 Taint를 허용하는 Toleration을 설정하거나 Taint를 제거해야 합니다.

다만 Control Plane의 안정성과 리소스 확보를 위해 일반 애플리케이션 Pod를 배치하지 않는 구성이 일반적입니다.

## 11. Taint와 Node Affinity의 관계

Taint와 Node Affinity는 특정 Node에 Pod를 배치하도록 제어할 때 함께 사용하는 경우가 많습니다.

두 기능은 서로 다른 방향에서 스케줄링 조건을 제어합니다.

| 구분 | Taint / Toleration | Node Affinity |
|---|---|---|
| 적용 대상 | Taint는 Node, Toleration은 Pod | Pod |
| 주요 목적 | 특정 Pod의 Node 접근 제한 또는 허용 | 특정 조건을 가진 Node 선택 |
| 동작 방향 | Node가 Pod를 거부하거나 허용 | Pod가 Node를 선택 |
| Node 배치 보장 | Toleration만으로는 보장하지 않음 | Required 조건으로 배치 대상 제한 |

예를 들어 GPU Node를 일반 Pod로부터 격리하면서 GPU 작업을 수행하는 Pod만 해당 Node에 배치하려면 두 기능을 함께 사용하는 것이 적합합니다.

- Taint: 일반 Pod가 GPU Node에 배치되는 것을 제한합니다.
- Toleration: GPU 작업 Pod가 해당 Taint를 허용하도록 설정합니다.
- Node Affinity: GPU 작업 Pod가 GPU Node를 선택하도록 제한합니다.

이처럼 Taint와 Toleration은 Node를 특정 워크로드로부터 보호하고, Node Affinity는 Pod가 적절한 Node를 선택하도록 하는 역할을 수행합니다.

## 12. Taint와 PodDisruptionBudget의 관계

`NoExecute` Taint는 Taint를 허용하지 않는 Pod를 Node에서 제거할 수 있습니다.

하지만 이러한 제거 동작은 일반적인 `kubectl drain`과 동일한 방식으로 수행되는 것은 아닙니다.

`kubectl drain`은 Eviction API를 사용하여 Pod를 제거하므로 PodDisruptionBudget(PDB)의 영향을 받습니다.

반면 Taint 기반의 Pod 제거는 Taint Manager가 처리하는 별도의 경로를 사용하며, 일반적인 Eviction API 기반의 PDB 보호와 동일하게 취급되지 않습니다.

따라서 PDB가 설정되어 있더라도 `NoExecute` Taint에 의한 Pod 제거를 동일한 방식으로 보호한다고 가정해서는 안 됩니다.

Node 유지보수 작업에서 PDB를 고려해야 한다면 `kubectl drain`을 사용하는 절차와 Taint 기반 제거의 차이를 확인해야 합니다.

## 13. Taint와 Toleration 적용 시 주의사항

### 13.1. Toleration은 Node 선택을 보장하지 않습니다.

Toleration은 특정 Taint를 허용하는 조건입니다.

따라서 Toleration만 설정하면 해당 Node뿐 아니라 다른 Node에도 Pod가 배치될 수 있습니다.

특정 Node에만 배치하려면 Node Affinity 또는 `nodeSelector`를 함께 사용해야 합니다.

### 13.2. NoExecute는 기존 Pod에도 영향을 줍니다.

`NoExecute`는 새로운 Pod의 배치뿐 아니라 이미 실행 중인 Pod의 제거에도 영향을 줍니다.

따라서 운영 중인 Node에 `NoExecute` Taint를 설정할 때는 기존 Pod의 종료와 재스케줄링에 미치는 영향을 확인해야 합니다.

### 13.3. 모든 Taint를 허용하는 설정에 주의해야 합니다.

다음과 같은 설정은 모든 Taint를 허용합니다.

```yaml
tolerations:
  - operator: Exists
```

이 설정은 Node의 Taint를 통한 격리 정책을 우회할 수 있으므로 필요한 경우에만 사용해야 합니다.

### 13.4. 기본 Toleration을 확인해야 합니다.

Kubernetes는 Pod 유형이나 클러스터 구성에 따라 기본 Toleration을 자동으로 추가할 수 있습니다.

따라서 Pod에 명시적으로 Toleration을 작성하지 않았더라도 특정 Taint를 허용하는 경우가 있습니다.

실제 Pod에 적용된 Toleration은 다음 명령어로 확인할 수 있습니다.

```bash
kubectl get pod <pod-name> -o yaml
```

### 13.5. Taint와 Toleration만으로는 워크로드 격리가 완성되지 않습니다.

Taint와 Toleration은 특정 Node에 대한 Pod의 접근을 제한하는 스케줄링 기능입니다.

하지만 Node에 접근할 수 있는 Pod를 허용하는 것만으로 보안 격리나 리소스 격리가 완성되는 것은 아닙니다.

필요에 따라 다음 기능을 함께 고려해야 합니다.

- Node Affinity: 특정 Node에 Pod 배치
- ResourceQuota: Namespace의 리소스 사용량 제한
- LimitRange: 개별 Container 및 Pod의 리소스 기본값과 범위 설정
- NetworkPolicy: Pod 간 네트워크 통신 제어
- Pod Security Admission: Pod의 보안 관련 설정 제한

## 14. 정리

Taint와 Toleration은 Kubernetes에서 Node에 배치할 수 있는 Pod를 제어하기 위한 스케줄링 기능입니다.

- **Taint:** Node에 설정하여 특정 Pod가 해당 Node에 배치되는 것을 제한합니다.
- **Toleration:** Pod에 설정하여 특정 Taint가 있는 Node에 배치될 수 있도록 허용합니다.

Taint의 Effect는 다음 세 가지로 구분됩니다.

- `NoSchedule`: 새로운 Pod의 배치를 제한합니다.
- `PreferNoSchedule`: 가능한 한 해당 Node에 배치하지 않도록 합니다.
- `NoExecute`: 새로운 Pod의 배치를 제한하고, 기존 Pod도 제거합니다.

Toleration은 `key`, `operator`, `value`, `effect`, `tolerationSeconds` 등을 사용하여 허용할 Taint의 조건과 유지 시간을 정의합니다.

특히 Toleration은 Node에 Pod를 배치할 수 있도록 허용하는 조건일 뿐, 해당 Node를 선택하도록 강제하지는 않습니다. 특정 Node에 워크로드를 배치하려면 Node Affinity 또는 `nodeSelector`를 함께 사용해야 합니다.

Taint와 Toleration을 적절히 활용하면 GPU 전용 Node, 데이터베이스 전용 Node, Control Plane Node 등 서로 다른 용도의 Node를 논리적으로 구분하고 워크로드별 배치 정책을 구성할 수 있습니다.

## 참고자료

- [Kubernetes 공식 문서 - Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Kubernetes 공식 문서 - Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Kubernetes 공식 문서 - Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
- [Kubernetes 공식 문서 - DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
- [Kubernetes API Reference - Pod v1](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/)
- [Kubernetes API Reference - Node v1](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/node-v1/)

