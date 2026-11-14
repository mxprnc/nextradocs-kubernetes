# PodDisruptionBudget (PDB)

## 1. PodDisruptionBudget란?

PodDisruptionBudget(PDB)는 Kubernetes에서 애플리케이션을 구성하는 Pod가 자발적인 중단(Voluntary Disruption)으로 인해 동시에 종료되는 수를 제한하는 리소스입니다.

Kubernetes 클러스터에서는 Node 유지보수, 클러스터 축소, Node 업그레이드 등의 작업으로 인해 실행 중인 Pod를 종료해야 하는 상황이 발생할 수 있습니다.

이러한 작업이 여러 Pod에 동시에 발생하면 애플리케이션의 가용성이 저하되거나 서비스가 일시적으로 중단될 수 있습니다.

PDB는 이러한 상황에서 애플리케이션이 최소한으로 유지해야 하는 정상 Pod의 수 또는 허용할 수 있는 중단 Pod의 수를 정의합니다.

예를 들어 3개의 Replica로 구성된 애플리케이션에서 최소 2개의 Pod가 정상적으로 실행되어야 한다면 다음과 같이 PDB를 설정할 수 있습니다.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: application-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: application
```

위 설정에서는 `app: application` 레이블을 가진 Pod 중 최소 2개가 정상적으로 유지되어야 합니다.

따라서 Node 유지보수 등의 작업으로 Pod를 종료하려는 경우, PDB가 허용하는 범위를 초과하면 Kubernetes의 Eviction 요청이 거부되거나 지연될 수 있습니다.

PDB의 주요 특징은 다음과 같습니다.

- **가용성 보호:** 자발적인 중단으로 인해 동시에 종료되는 Pod의 수를 제한합니다.
- **최소 가용 Pod 보장:** 애플리케이션이 유지해야 하는 최소 정상 Pod 수를 지정할 수 있습니다.
- **중단 허용 범위 설정:** 동시에 사용할 수 없는 상태가 되어도 되는 Pod의 최대 수를 지정할 수 있습니다.
- **Eviction API 연동:** Kubernetes의 Eviction API를 사용하는 작업에서 PDB를 확인합니다.
- **운영 작업 보호:** Node Drain, 클러스터 축소 및 유지보수 작업 중 애플리케이션의 가용성을 보호하는 데 활용됩니다.

PDB는 Kubernetes의 `policy/v1` API 그룹에 속하는 리소스이며, Kubernetes v1.21부터 Stable 기능으로 제공됩니다.

## 2. Pod Disruption의 종류

PDB를 이해하려면 Kubernetes에서 발생하는 Pod Disruption을 먼저 구분해야 합니다.

Kubernetes의 Pod 중단은 크게 Voluntary Disruption과 Involuntary Disruption으로 구분됩니다.

### 2.1. Voluntary Disruption

Voluntary Disruption은 사용자 또는 클러스터 관리자가 의도적으로 수행하는 작업으로 인해 Pod가 종료되는 상황입니다.

대표적인 사례는 다음과 같습니다.

- Node 유지보수를 위한 `kubectl drain`
- Node 업그레이드
- 클러스터 축소를 위한 Node 제거
- 클러스터 관리 도구에 의한 Pod 축출
- 관리자가 직접 수행하는 Pod 삭제

예를 들어 클러스터 관리자가 Node의 커널을 업그레이드하기 위해 Node를 비우는 작업을 수행한다고 가정하겠습니다.

해당 Node에 애플리케이션 Pod가 3개 실행 중이라면, 모든 Pod를 한꺼번에 종료할 경우 서비스에 영향을 줄 수 있습니다.

PDB를 설정하면 Eviction API를 통해 요청된 Pod 종료가 허용 가능한 범위인지 확인할 수 있습니다.

### 2.2. Involuntary Disruption

Involuntary Disruption은 관리자가 의도하지 않았더라도 장애나 시스템 문제로 인해 Pod가 종료되는 상황입니다.

대표적인 사례는 다음과 같습니다.

- 물리 서버의 하드웨어 장애
- VM 또는 클라우드 인스턴스 장애
- Node의 Kernel Panic
- Node와 Control Plane 사이의 네트워크 단절
- Node의 리소스 부족으로 인한 Pod Eviction

이러한 상황은 PDB가 직접 방지할 수 없습니다.

다만 Involuntary Disruption으로 인해 정상 Pod의 수가 감소하면 해당 Pod도 PDB의 가용성 계산에 반영됩니다.

따라서 예상하지 못한 장애로 이미 가용 Pod 수가 감소한 상태라면 이후 발생하는 Voluntary Disruption이 제한될 수 있습니다.

### 2.3. Voluntary Disruption과 Involuntary Disruption 비교

| 구분 | Voluntary Disruption | Involuntary Disruption |
|---|---|---|
| 발생 원인 | 사용자 또는 관리자의 의도적인 작업 | 장애 또는 시스템 문제 |
| 주요 사례 | Node Drain, Node 업그레이드, 클러스터 축소 | 하드웨어 장애, Kernel Panic, Node 장애 |
| PDB의 보호 | Eviction API를 통한 중단 요청을 제한 | 직접 방지할 수 없음 |
| PDB 계산 | 허용 가능한 중단 수에 반영 | 가용 Pod 감소로 예산에 반영 |

PDB는 모든 종류의 Pod 종료를 방지하는 리소스가 아닙니다. 주로 Eviction API를 사용하는 자발적 중단 작업에서 애플리케이션의 가용성을 보호하는 역할을 담당합니다.

## 3. PDB의 동작 원리

PDB는 특정 레이블을 가진 Pod를 대상으로 최소 가용 Pod 수 또는 최대 중단 Pod 수를 정의합니다.

Kubernetes의 Disruption Controller는 PDB의 설정과 대상 Pod의 상태를 지속적으로 확인하여 현재 허용 가능한 중단 수를 계산합니다.

이때 중요한 개념은 다음과 같습니다.

| 항목 | 설명 |
|---|---|
| Expected Pods | PDB가 관리하는 대상 Pod의 기대 개수입니다. |
| Current Healthy | 현재 정상적으로 사용 가능한 Pod의 개수입니다. |
| Desired Healthy | 유지해야 하는 최소 정상 Pod의 개수입니다. |
| Disruptions Allowed | 현재 허용 가능한 추가 Pod 중단 개수입니다. |

예를 들어 Deployment가 5개의 Replica를 유지하고 있고 PDB에서 `minAvailable: 4`를 설정했다고 가정하겠습니다.

| 항목 | 값 |
|---|---:|
| Expected Pods | 5 |
| Current Healthy | 5 |
| Desired Healthy | 4 |
| Disruptions Allowed | 1 |

이 상태에서는 최대 1개의 Pod를 추가로 중단할 수 있습니다.

첫 번째 Pod의 Eviction이 허용되면 Current Healthy가 4로 감소하므로 추가 Eviction은 허용되지 않습니다.

새로운 Pod가 생성되어 다시 정상 상태가 되면 Disruptions Allowed가 증가할 수 있습니다.

### 3.1. Eviction API

PDB는 Kubernetes의 Eviction API와 연동하여 동작합니다.

Eviction API는 Pod를 정상적인 종료 절차에 따라 축출하기 위한 Kubernetes API입니다.

일반적인 Pod 삭제와 달리 Eviction API는 PDB를 확인하여 해당 Pod의 종료가 허용되는지 판단합니다.

예를 들어 `kubectl drain` 명령어는 Node에 실행 중인 Pod를 종료하기 위해 Eviction API를 사용합니다.

```bash
kubectl drain worker-node-01
```

위 명령어가 실행되면 Kubernetes는 해당 Node에서 실행 중인 Pod를 순차적으로 축출하려고 시도합니다.

이때 PDB가 설정되어 있다면 각 Eviction 요청이 허용 가능한 중단 범위에 해당하는지 확인합니다.

현재 PDB가 허용하는 중단 수를 초과하는 요청은 거부될 수 있으며, `kubectl drain`은 실패한 요청을 주기적으로 재시도합니다.

따라서 PDB가 허용하는 범위 내에서 Pod가 정상적으로 종료되고 새로운 Pod가 준비될 때까지 Node Drain 작업이 지연될 수 있습니다.

### 3.2. PDB가 적용되지 않는 작업

PDB는 모든 Pod 삭제 작업에 적용되는 것은 아닙니다.

특히 Kubernetes API를 통한 직접적인 Pod 삭제나 Deployment 삭제는 PDB의 Eviction 제한을 우회할 수 있습니다.

다음과 같은 작업은 PDB가 보호하는 대상이 아닙니다.

```bash
kubectl delete pod application-pod
```

```bash
kubectl delete deployment application
```

또한 Deployment나 StatefulSet의 Rolling Update는 PDB에 의해 직접 제한되지 않습니다.

워크로드의 Rolling Update 과정에서 발생하는 Pod 중단은 각 워크로드 리소스의 업데이트 전략을 통해 제어합니다.

예를 들어 Deployment의 경우 `maxUnavailable`과 `maxSurge`를 통해 Rolling Update 중 허용할 수 있는 Pod 수를 설정합니다.

따라서 PDB와 Deployment의 Rolling Update 설정은 서로 다른 목적과 제어 범위를 가진다는 점을 이해해야 합니다.

## 4. PDB 설정 (policy/v1)

PDB는 `policy/v1` API 그룹에 정의된 `PodDisruptionBudget` 리소스를 사용합니다.

다음은 Deployment의 Pod를 대상으로 최소 2개의 정상 Pod를 유지하도록 설정하는 예시입니다.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: application-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: application
```

각 필드의 의미는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| apiVersion | PDB 리소스의 API 버전입니다. |
| kind | 리소스의 종류를 나타내며 `PodDisruptionBudget`을 사용합니다. |
| metadata.name | PDB 리소스의 이름입니다. |
| minAvailable | 중단 이후에도 유지해야 하는 최소 정상 Pod 수입니다. |
| maxUnavailable | 동시에 중단될 수 있는 최대 Pod 수입니다. |
| selector | PDB가 관리할 Pod를 선택하는 레이블 조건입니다. |
| unhealthyPodEvictionPolicy | 정상 상태가 아닌 Pod의 Eviction 허용 정책을 지정합니다. |

`minAvailable`과 `maxUnavailable`은 동시에 설정할 수 없습니다. 둘 중 하나만 지정해야 합니다.

## 5. minAvailable

`minAvailable`은 자발적인 중단 이후에도 최소한으로 유지해야 하는 정상 Pod의 수를 지정합니다.

절대적인 개수 또는 백분율(Percentage)을 사용할 수 있습니다.

### 5.1. 절대 개수 설정

다음 예시는 최소 2개의 Pod가 정상 상태로 유지되도록 설정합니다.

```yaml
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: application
```

Deployment가 5개의 Replica를 유지하고 있다면 다음과 같이 동작합니다.

| 항목 | 값 |
|---|---:|
| Expected Pods | 5 |
| Current Healthy | 5 |
| Desired Healthy | 2 |
| Disruptions Allowed | 3 |

이 상태에서는 최대 3개의 Pod를 추가로 중단할 수 있습니다.

다만 실제 서비스 가용성을 고려하면 최소 2개의 Pod만 유지하는 설정이 적절한지는 별도로 판단해야 합니다.

### 5.2. 백분율 설정

다음과 같이 백분율을 지정할 수도 있습니다.

```yaml
spec:
  minAvailable: "80%"
  selector:
    matchLabels:
      app: application
```

Deployment가 5개의 Replica를 유지하고 있다면 최소 80%에 해당하는 Pod가 정상 상태로 유지되어야 합니다.

계산 결과가 정수가 아닌 경우 최소 유지 개수는 올림(ROUND UP)하여 계산합니다.

예를 들어 다음과 같습니다.

| Expected Pods | minAvailable | Desired Healthy |
|---:|---:|---:|
| 5 | 80% | 4 |
| 6 | 80% | 5 |
| 10 | 80% | 8 |
| 7 | 50% | 4 |

백분율을 사용하면 Replica 개수가 변경되더라도 전체 Pod 수에 비례하여 최소 가용 개수를 유지할 수 있습니다.

## 6. maxUnavailable

`maxUnavailable`은 동시에 중단될 수 있는 최대 Pod 수를 지정합니다.

절대적인 개수 또는 백분율을 사용할 수 있습니다.

### 6.1. 절대 개수 설정

다음 예시는 동시에 중단될 수 있는 Pod를 최대 1개로 제한합니다.

```yaml
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: application
```

Deployment가 5개의 Replica를 유지하고 있다면 다음과 같이 동작합니다.

| 항목 | 값 |
|---|---:|
| Expected Pods | 5 |
| Current Healthy | 5 |
| Max Unavailable | 1 |
| Desired Healthy | 4 |
| Disruptions Allowed | 1 |

이 상태에서는 최대 1개의 Pod에 대한 추가 Eviction을 허용합니다.

### 6.2. 백분율 설정

다음과 같이 백분율을 지정할 수도 있습니다.

```yaml
spec:
  maxUnavailable: "20%"
  selector:
    matchLabels:
      app: application
```

Deployment가 5개의 Replica를 유지하고 있다면 최대 20%에 해당하는 Pod의 중단을 허용합니다.

백분율 계산 결과가 정수가 아닌 경우 허용할 수 있는 최대 중단 개수는 올림(ROUND UP)하여 계산합니다.

예를 들어 다음과 같습니다.

| Expected Pods | maxUnavailable | 허용 가능한 중단 개수 |
|---:|---:|---:|
| 5 | 20% | 1 |
| 6 | 20% | 2 |
| 10 | 20% | 2 |
| 7 | 50% | 4 |

백분율을 사용하면 Replica 개수에 따라 허용 가능한 중단 개수를 비례적으로 조정할 수 있습니다.

### 6.3. minAvailable과 maxUnavailable 비교

두 필드는 서로 반대되는 관점에서 PDB의 가용성 조건을 정의합니다.

| 구분 | minAvailable | maxUnavailable |
|---|---|---|
| 설정 기준 | 최소 정상 Pod 수 | 최대 중단 Pod 수 |
| 주요 목적 | 최소 가용성 보장 | 허용 가능한 중단 범위 제한 |
| 절대 개수 | 지원 | 지원 |
| 백분율 | 지원 | 지원 |
| 동시 설정 | 불가능 | 불가능 |

예를 들어 Replica가 5개인 Deployment에서 다음 두 설정은 동일한 초기 상태의 가용성 조건을 나타냅니다.

```yaml
spec:
  minAvailable: 4
  selector:
    matchLabels:
      app: application
```

```yaml
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: application
```

다만 백분율 계산이나 대상 Controller의 Replica 변경에 따라 결과가 달라질 수 있으므로 실제 운영 환경에서는 각 필드의 의미를 기준으로 선택해야 합니다.

## 7. selector

`selector`는 PDB가 관리할 Pod를 선택하는 레이블 조건입니다.

일반적으로 Deployment 또는 StatefulSet의 Pod Template에 지정된 레이블과 동일하게 설정합니다.

예를 들어 다음과 같은 Deployment가 있다고 가정하겠습니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: application
spec:
  replicas: 3
  selector:
    matchLabels:
      app: application
  template:
    metadata:
      labels:
        app: application
    spec:
      containers:
        - name: application
          image: nginx:1.27
```

위 Deployment가 생성한 Pod는 `app: application` 레이블을 가지고 있습니다.

따라서 PDB에서도 동일한 레이블을 지정해야 합니다.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: application-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: application
```

이 경우 `app: application` 레이블을 가진 Pod가 PDB의 관리 대상이 됩니다.

### 7.1. Selector 주의사항

PDB의 Selector를 설정할 때는 대상 Pod가 의도한 워크로드에만 포함되는지 확인해야 합니다.

특히 동일한 레이블을 여러 Deployment가 사용하면 예상하지 못한 Pod가 PDB의 관리 대상에 포함될 수 있습니다.

또한 `policy/v1`에서 다음과 같은 빈 Selector는 해당 Namespace의 모든 Pod를 선택합니다.

```yaml
spec:
  selector: {}
```

반면 Selector가 `null`인 경우에는 어떤 Pod도 선택하지 않습니다.

따라서 PDB를 생성할 때는 빈 Selector와 Selector 생략 또는 `null`의 의미를 구분해야 합니다.

## 8. unhealthyPodEvictionPolicy

`unhealthyPodEvictionPolicy`는 정상 상태가 아닌 Pod를 Eviction할 수 있는 조건을 지정합니다.

PDB는 기본적으로 Ready 상태인 Pod를 정상 Pod로 판단합니다.

그러나 애플리케이션이 정상 상태로 복구되지 못하거나 Readiness Probe가 계속 실패하는 경우 Node Drain 작업이 지연될 수 있습니다.

이러한 상황을 처리하기 위해 다음 두 가지 정책을 제공합니다.

| 정책 | 설명 |
|---|---|
| IfHealthyBudget | 애플리케이션이 PDB의 최소 가용성 조건을 만족하는 경우에만 정상 상태가 아닌 Pod의 Eviction을 허용합니다. |
| AlwaysAllow | 정상 상태가 아닌 Running Pod의 Eviction을 PDB의 가용성 조건과 관계없이 허용합니다. |

### 8.1. IfHealthyBudget

IfHealthyBudget는 기본 정책입니다.

애플리케이션이 이미 PDB의 가용성 조건을 충족하지 못하고 있다면 정상 상태가 아닌 Pod의 Eviction도 제한될 수 있습니다.

예를 들어 다음과 같은 상황을 가정하겠습니다.

- Deployment Replica: 3
- PDB minAvailable: 2
- 정상 Pod: 1
- 비정상 Pod: 2

이 상태에서는 이미 최소 가용 Pod 수를 충족하지 못하고 있으므로 비정상 Pod의 Eviction이 제한될 수 있습니다.

### 8.2. AlwaysAllow

AlwaysAllow는 정상 상태가 아닌 Running Pod의 Eviction을 허용하는 정책입니다.

다음과 같이 설정할 수 있습니다.

```yaml
spec:
  minAvailable: 2
  unhealthyPodEvictionPolicy: AlwaysAllow
  selector:
    matchLabels:
      app: application
```

이 설정에서는 애플리케이션의 PDB 조건을 만족하지 못하더라도 정상 상태가 아닌 Running Pod의 Eviction을 허용합니다.

예를 들어 Readiness Probe가 지속적으로 실패하는 Pod가 존재하는 경우 Node Drain 작업이 해당 Pod 때문에 무기한 지연되는 상황을 방지할 수 있습니다.

다만 정상 상태가 아닌 Pod를 Eviction하면 애플리케이션이 복구될 기회를 잃을 수 있으므로 워크로드의 특성을 고려하여 설정해야 합니다.

Kubernetes 공식 문서에서는 Node Drain 작업이 비정상 Pod 때문에 지연되는 상황을 방지하기 위한 방법으로 `AlwaysAllow` 사용을 권장하고 있습니다.

## 9. PDB와 Deployment Rolling Update의 관계

PDB와 Deployment의 Rolling Update 설정은 모두 애플리케이션의 가용성과 관련이 있지만 서로 다른 작업을 제어합니다.

PDB는 Eviction API를 사용하는 자발적 중단 작업을 제한합니다.

반면 Deployment의 Rolling Update는 Deployment Controller가 Pod를 교체하는 과정이며, `maxUnavailable`과 `maxSurge`를 통해 업데이트 중 Pod의 개수와 가용성을 제어합니다.

예를 들어 다음과 같은 Deployment 설정을 사용할 수 있습니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: application
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: application
  template:
    metadata:
      labels:
        app: application
    spec:
      containers:
        - name: application
          image: nginx:1.27
```

각 필드의 의미는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| replicas | 유지하려는 Pod의 개수입니다. |
| maxUnavailable | Rolling Update 중 사용할 수 없는 상태가 될 수 있는 최대 Pod 수입니다. |
| maxSurge | Rolling Update 중 원하는 Replica 수를 초과하여 추가로 생성할 수 있는 최대 Pod 수입니다. |

위 설정에서는 5개의 Replica를 유지하면서 Rolling Update 중 최대 1개의 Pod가 사용할 수 없는 상태가 될 수 있으며, 최대 1개의 추가 Pod를 생성할 수 있습니다.

PDB와 Deployment Rolling Update의 차이는 다음과 같습니다.

| 구분 | PDB | Deployment Rolling Update |
|---|---|---|
| 제어 대상 | 자발적인 Pod 중단 | Deployment의 업데이트 과정 |
| 주요 목적 | 유지보수 등의 작업 중 가용성 보호 | 애플리케이션 버전 변경 시 가용성 유지 |
| 주요 설정 | minAvailable, maxUnavailable | maxUnavailable, maxSurge |
| 적용 주체 | Eviction API 및 Disruption Controller | Deployment Controller |

따라서 PDB를 설정했다고 해서 Deployment의 Rolling Update가 PDB에 의해 제한되는 것은 아닙니다.

애플리케이션의 업데이트 가용성은 Deployment의 Rolling Update 설정으로 별도로 관리해야 합니다.

## 10. PDB와 VPA의 관계

VPA의 Recreate Update Mode는 기존 Pod를 Eviction하고 새로운 Pod를 생성하는 방식으로 리소스 변경을 적용합니다.

이 과정에서 Updater는 Kubernetes의 Eviction API를 사용하므로 PDB의 제약을 고려합니다.

예를 들어 Deployment가 3개의 Replica를 유지하고 있고 다음과 같은 PDB가 설정되어 있다고 가정하겠습니다.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: application-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: application
```

VPA가 기존 Pod의 리소스 변경을 위해 Eviction을 요청하면 다음과 같이 동작합니다.

1. VPA Recommender가 새로운 리소스 추천값을 계산합니다.
2. VPA Updater가 기존 Pod의 리소스 설정과 추천값을 비교합니다.
3. 리소스 변경이 필요한 Pod를 선택합니다.
4. Updater가 Eviction API를 통해 Pod 축출을 요청합니다.
5. Kubernetes가 PDB의 현재 가용성 조건을 확인합니다.
6. Eviction이 허용되면 기존 Pod가 종료되고 새로운 Pod가 생성됩니다.
7. 새로운 Pod에 VPA의 추천 리소스 값이 적용됩니다.

PDB가 허용하는 중단 수를 초과하면 Eviction 요청이 거부될 수 있습니다.

따라서 VPA의 Recreate 모드를 사용하는 환경에서는 PDB의 최소 가용 Pod 수와 Deployment의 Replica 개수를 함께 고려해야 합니다.

반면 VPA의 InPlace 또는 InPlaceOrRecreate 모드에서 수행되는 In-Place 리소스 변경은 Pod Eviction을 사용하지 않으므로 PDB의 Eviction 제한과 동일한 방식으로 동작하지 않습니다.

## 11. PDB 운영 시 고려사항

PDB를 운영 환경에 적용할 때는 다음 사항을 고려해야 합니다.

### 11.1. Replica 개수와 최소 가용 Pod 수

PDB의 `minAvailable`이 Deployment의 Replica 개수와 같다면 자발적인 중단을 허용하지 않습니다.

예를 들어 다음과 같이 설정하면 모든 Pod가 정상 상태로 유지되어야 합니다.

```yaml
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: application
```

Deployment가 3개의 Replica를 유지하는 경우 Disruptions Allowed는 0이 됩니다.

이 상태에서는 PDB를 준수하는 Node Drain 작업이 새로운 Pod가 정상 상태가 되거나 PDB 설정이 변경될 때까지 지연될 수 있습니다.

따라서 유지보수 작업이 필요한 환경에서는 최소 가용 Pod 수와 허용 가능한 중단 범위를 적절하게 설정해야 합니다.

### 11.2. 단일 Replica 애플리케이션

Replica가 1개인 애플리케이션에 다음과 같은 설정을 적용했다고 가정하겠습니다.

```yaml
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: application
```

이 경우 유일한 Pod가 중단되면 최소 가용 Pod 수를 만족하지 못하므로 Eviction이 허용되지 않습니다.

따라서 단일 Replica 애플리케이션에 PDB를 설정하더라도 무중단 유지보수가 가능해지는 것은 아닙니다.

높은 가용성이 필요한 애플리케이션은 적절한 Replica 개수와 분산 배치를 함께 고려해야 합니다.

### 11.3. Node 장애에 대한 보호 한계

PDB는 하드웨어 장애나 Node 장애와 같은 비자발적인 중단을 방지하지 못합니다.

예를 들어 3개의 Replica가 각각 다른 Node에서 실행되고 있더라도 Node 자체에 장애가 발생하면 해당 Node의 Pod는 정상적으로 실행될 수 없습니다.

따라서 PDB만으로 고가용성을 보장할 수는 없습니다.

다음과 같은 기능을 함께 활용하는 것이 중요합니다.

- 여러 Replica를 통한 애플리케이션 복제
- Pod Anti-Affinity
- Topology Spread Constraints
- Multi-AZ Node 배치
- 적절한 Readiness Probe 및 Liveness Probe

### 11.4. Selector 충돌

하나의 Pod가 여러 PDB의 Selector에 동시에 포함되면 해당 Pod의 Eviction이 거부될 수 있습니다.

따라서 여러 PDB를 운영할 때는 서로 다른 PDB가 동일한 Pod를 중복해서 관리하지 않도록 주의해야 합니다.

### 11.5. Node Drain 지연

PDB가 허용하는 중단 수를 초과하는 경우 Node Drain 작업은 Pod의 Eviction을 진행하지 못하고 대기할 수 있습니다.

특히 다음과 같은 상황에서는 유지보수 작업이 지연될 수 있습니다.

- 현재 정상 Pod 수가 PDB의 최소 가용 개수와 동일한 경우
- 새로운 Pod가 생성되었지만 아직 Ready 상태가 아닌 경우
- 애플리케이션의 시작 시간이 긴 경우
- Readiness Probe가 지속적으로 실패하는 경우
- 비정상 Pod의 Eviction 정책이 기본값인 IfHealthyBudget인 경우

따라서 Node 유지보수 작업을 수행하기 전에 PDB의 상태와 현재 정상 Pod 수를 확인하는 것이 좋습니다.

## 12. PDB 관련 주요 명령어

| 명령어 | 설명 |
|---|---|
| `kubectl get pdb` | 현재 Namespace의 PDB 목록을 확인합니다. |
| `kubectl get pdb -A` | 모든 Namespace의 PDB 목록을 확인합니다. |
| `kubectl describe pdb <name>` | 특정 PDB의 상세 설정과 상태를 확인합니다. |
| `kubectl get pdb <name> -o yaml` | PDB 리소스의 전체 YAML 정보를 확인합니다. |
| `kubectl get pods --show-labels` | Pod의 레이블을 확인합니다. |
| `kubectl get pods -l app=application` | 특정 레이블을 가진 Pod를 조회합니다. |
| `kubectl drain <node-name>` | Node의 Pod를 Eviction하여 Node를 비웁니다. |

PDB의 현재 상태를 확인하려면 다음 명령어를 사용할 수 있습니다.

```bash
kubectl get pdb
```

출력 예시는 다음과 같습니다.

```text
NAME             MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
application-pdb  2               N/A               1                     10m
```

각 항목의 의미는 다음과 같습니다.

| 항목 | 설명 |
|---|---|
| MIN AVAILABLE | 최소 유지해야 하는 정상 Pod 수입니다. |
| MAX UNAVAILABLE | 동시에 중단될 수 있는 최대 Pod 수입니다. |
| ALLOWED DISRUPTIONS | 현재 허용 가능한 추가 Pod 중단 수입니다. |
| AGE | PDB가 생성된 이후 경과한 시간입니다. |

보다 상세한 상태를 확인하려면 다음 명령어를 사용할 수 있습니다.

```bash
kubectl describe pdb application-pdb
```

출력 결과에서 다음 항목을 확인할 수 있습니다.

```text
Status:
  Current Healthy:     3
  Desired Healthy:     2
  Disruptions Allowed: 1
  Expected Pods:       3
```

이를 통해 현재 정상 Pod 수와 최소 요구 Pod 수, 추가로 허용할 수 있는 중단 수를 확인할 수 있습니다.

## 13. 정리

PodDisruptionBudget(PDB)은 Kubernetes에서 자발적인 Pod 중단으로 인해 애플리케이션의 가용성이 저하되는 상황을 방지하기 위한 리소스입니다.

PDB는 `minAvailable` 또는 `maxUnavailable`을 사용하여 애플리케이션이 유지해야 하는 최소 정상 Pod 수 또는 허용할 수 있는 최대 중단 Pod 수를 정의합니다.

Kubernetes의 Disruption Controller는 대상 Pod의 상태와 PDB 설정을 기반으로 허용 가능한 중단 수를 계산하며, Eviction API를 사용하는 작업은 해당 조건을 확인한 후 Pod 축출을 진행합니다.

따라서 Node Drain, Node 유지보수, 클러스터 축소 및 VPA의 Recreate 모드와 같이 Eviction API를 사용하는 작업에서 애플리케이션의 가용성을 보호하는 데 활용할 수 있습니다.

다만 PDB는 모든 Pod 삭제 작업을 제한하지 않으며, 하드웨어 장애와 같은 비자발적인 중단도 방지할 수 없습니다. 또한 Deployment의 Rolling Update는 PDB가 아니라 Deployment의 업데이트 전략을 통해 별도로 제어됩니다.

따라서 PDB는 Replica 개수, Pod 배치 전략, Readiness Probe 및 Deployment의 Rolling Update 설정 등과 함께 고려해야 하는 Kubernetes의 가용성 관리 리소스입니다.

## 14. 참고자료

- [Kubernetes 공식 문서 - Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Kubernetes 공식 문서 - Specifying a Disruption Budget for your Application](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [Kubernetes 공식 API Reference - PodDisruptionBudget v1](https://kubernetes.io/docs/reference/kubernetes-api/policy/pod-disruption-budget-v1/)
- [Kubernetes 공식 문서 - Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes 공식 문서 - Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
- [Kubernetes 공식 문서 - Vertical Pod Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/)