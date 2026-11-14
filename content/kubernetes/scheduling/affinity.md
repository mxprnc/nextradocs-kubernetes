

# Affinity

## 1. Affinity란?

`Affinity`는 Kubernetes Scheduler가 Pod를 배치할 Node를 선택할 때 적용하는 스케줄링 규칙입니다.

일반적으로 Scheduler는 Node의 가용 리소스, Pod의 리소스 Request, Taint 및 기타 스케줄링 조건을 고려하여 Pod를 배치합니다. 하지만 애플리케이션의 특성이나 인프라 구성에 따라 특정 Node를 선호하거나, 특정 Pod와 같은 위치 또는 다른 위치에 배치해야 하는 경우가 있습니다.

Affinity를 사용하면 이러한 배치 조건을 선언적으로 정의할 수 있습니다.

Kubernetes의 Affinity는 크게 다음 세 가지 유형으로 구분됩니다.

| 유형 | 설명 | 주요 목적 |
|---|---|---|
| Node Affinity | Node의 Label을 기준으로 Pod 배치 위치 결정 | 특정 하드웨어 또는 인프라를 갖춘 Node 선택 |
| Pod Affinity | 다른 Pod의 Label과 위치를 기준으로 Pod 배치 | 서로 연관된 Pod를 같은 위치에 배치 |
| Pod Anti-Affinity | 다른 Pod의 Label과 위치를 기준으로 Pod 배치 제한 | 특정 Pod와 같은 위치에 배치되지 않도록 제한 |

예를 들어 다음과 같은 환경에서 활용할 수 있습니다.

- SSD가 장착된 Node에 데이터베이스 Pod 배치
- 특정 Availability Zone에 애플리케이션 Pod 배치
- 서로 통신량이 많은 애플리케이션 Pod와 데이터베이스 Pod를 같은 Zone에 배치
- 동일한 애플리케이션의 Replica Pod를 서로 다른 Node에 분산 배치
- 특정 서비스의 Pod가 다른 서비스의 Pod와 동일한 Node에서 실행되지 않도록 제한

Affinity는 Pod의 `spec.affinity` 필드에 정의하며, Kubernetes Scheduler가 해당 조건을 평가하여 배치할 Node를 결정합니다.

## 2. Affinity의 동작 원리

Affinity는 Kubernetes Scheduler의 Filtering 및 Scoring 과정에서 적용됩니다.

기본적인 동작 과정은 다음과 같습니다.

1. 사용자가 Pod에 Affinity 규칙을 정의하여 생성합니다.
2. API Server가 Pod 생성 요청을 처리합니다.
3. Scheduler가 Pod를 배치할 수 있는 Node 후보를 확인합니다.
4. `required` 조건을 기준으로 조건을 만족하지 않는 Node를 제외합니다.
5. `preferred` 조건을 기준으로 각 Node의 선호도를 평가하고 점수를 계산합니다.
6. 다른 스케줄링 조건과 함께 최종 점수가 높은 Node를 선택합니다.
7. 선택된 Node에 Pod를 Binding합니다.

Affinity 규칙은 크게 필수 조건(Hard Constraint)과 선호 조건(Soft Constraint)으로 구분됩니다.

### 2.1. Required와 Preferred

Affinity에서 사용하는 주요 조건은 다음과 같습니다.

| 구분 | 필드 | 동작 |
|---|---|---|
| Required | `requiredDuringSchedulingIgnoredDuringExecution` | 조건을 만족하지 않으면 해당 Node에 Pod를 배치하지 않습니다. |
| Preferred | `preferredDuringSchedulingIgnoredDuringExecution` | 조건을 만족하는 Node를 우선적으로 선택하지만, 만족하지 않더라도 다른 조건이 허용하면 배치할 수 있습니다. |

`Required`는 반드시 만족해야 하는 스케줄링 조건이며, `Preferred`는 가능한 경우 우선적으로 만족시키려는 선호 조건입니다.

예를 들어 SSD가 장착된 Node에만 Pod를 실행해야 한다면 `Required`를 사용하고, SSD가 장착된 Node를 우선적으로 선택하되 다른 Node에도 배치할 수 있도록 하려면 `Preferred`를 사용할 수 있습니다.

### 2.2. IgnoredDuringExecution의 의미

`requiredDuringSchedulingIgnoredDuringExecution`과 `preferredDuringSchedulingIgnoredDuringExecution`에는 공통적으로 `IgnoredDuringExecution`이라는 표현이 포함되어 있습니다.

이는 Pod가 Node에 배치된 이후 스케줄링 조건이 변경되더라도 이미 실행 중인 Pod를 자동으로 종료하거나 다른 Node로 이동시키지 않는다는 의미입니다.

예를 들어 다음과 같은 상황을 가정하겠습니다.

1. `disktype=ssd` Label이 설정된 Node에 Pod가 배치됩니다.
2. 이후 관리자가 해당 Node의 `disktype=ssd` Label을 제거합니다.
3. 이미 실행 중인 Pod는 해당 Label이 제거되었다는 이유만으로 종료되지 않습니다.

즉, `IgnoredDuringExecution`은 스케줄링 시점에만 조건을 평가하고 실행 중인 Pod에는 해당 조건을 지속적으로 강제하지 않는다는 의미입니다.

## 3. Node Affinity

Node Affinity는 Node에 설정된 Label을 기준으로 Pod가 배치될 Node를 선택하는 기능입니다.

`nodeSelector`와 유사하지만, 더 다양한 조건과 선호도를 정의할 수 있습니다.

예를 들어 다음과 같은 Node가 존재한다고 가정하겠습니다.

| Node | Label |
|---|---|
| node-1 | `disktype=ssd` |
| node-2 | `disktype=hdd` |
| node-3 | `disktype=ssd` |

SSD가 장착된 Node에서만 Pod를 실행하려면 Node Affinity를 사용하여 `disktype=ssd` 조건을 정의할 수 있습니다.

Node Affinity는 다음 두 가지 필드를 지원합니다.

- `requiredDuringSchedulingIgnoredDuringExecution`
- `preferredDuringSchedulingIgnoredDuringExecution`

### 3.1. Required Node Affinity

`requiredDuringSchedulingIgnoredDuringExecution`은 반드시 만족해야 하는 Node 선택 조건을 정의합니다.

다음은 `disktype=ssd` Label이 설정된 Node에만 Pod를 배치하는 예제입니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: required-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
  containers:
    - name: app
      image: nginx
```

위 설정에서는 `disktype=ssd` 조건을 만족하는 Node만 스케줄링 대상이 됩니다.

해당 조건을 만족하는 Node가 존재하지 않거나 다른 스케줄링 조건까지 고려했을 때 배치 가능한 Node가 없다면 Pod는 `Pending` 상태로 남을 수 있습니다.

### 3.2. Preferred Node Affinity

`preferredDuringSchedulingIgnoredDuringExecution`은 특정 Node를 우선적으로 선택하도록 선호 조건을 정의합니다.

다음은 `disktype=ssd` Label이 설정된 Node를 우선적으로 선택하는 예제입니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: preferred-node-affinity
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
  containers:
    - name: app
      image: nginx
```

주요 필드는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `weight` | 해당 선호 조건에 부여할 가중치입니다. |
| `preference` | 선호하는 Node의 Label 조건을 정의합니다. |
| `matchExpressions` | Label Key, Operator, Value를 사용하여 조건을 정의합니다. |

`weight`는 1부터 100까지 지정할 수 있습니다.

Scheduler는 Node가 만족하는 Preferred 조건의 가중치를 계산하여 다른 스케줄링 점수와 함께 평가합니다.

따라서 가중치가 높을수록 해당 조건을 만족하는 Node가 우선적으로 선택될 가능성이 높아집니다.

다만 `weight`는 Node 선택을 절대적으로 보장하는 값이 아닙니다. 다른 스케줄링 점수와 함께 사용되는 상대적인 선호도입니다.

### 3.3. Node Affinity의 Operator

Node Affinity의 `operator`는 Node Label을 어떤 방식으로 비교할지 결정합니다.

| Operator | 설명 | 예시 |
|---|---|---|
| `In` | Label 값이 지정된 값 중 하나와 일치해야 합니다. | `disktype In [ssd]` |
| `NotIn` | Label 값이 지정된 값과 일치하지 않아야 합니다. | `disktype NotIn [hdd]` |
| `Exists` | 지정한 Label Key가 존재해야 합니다. | `gpu Exists` |
| `DoesNotExist` | 지정한 Label Key가 존재하지 않아야 합니다. | `gpu DoesNotExist` |
| `Gt` | Label 값이 지정한 정수보다 커야 합니다. | `cores Gt 8` |
| `Lt` | Label 값이 지정한 정수보다 작아야 합니다. | `cores Lt 16` |

`Gt`와 `Lt`는 정수 비교에 사용하며, `values`에는 비교할 정수 값을 문자열로 지정합니다.

예를 들어 다음과 같이 정의할 수 있습니다.

```yaml
matchExpressions:
  - key: cpu-cores
    operator: Gt
    values:
      - "8"
```

이 경우 `cpu-cores` Label 값이 정수 8보다 큰 Node가 선택 대상이 됩니다.

`NotIn`과 `DoesNotExist`는 특정 Label 조건을 제외하는 방식으로 Node Anti-Affinity와 유사한 동작을 구성할 때도 활용할 수 있습니다.

### 3.4. NodeSelectorTerms와 MatchExpressions의 논리 연산

Node Affinity에서는 `nodeSelectorTerms`와 `matchExpressions`의 조합에 따라 논리 연산 방식이 달라집니다.

- 여러 `nodeSelectorTerms`는 OR 조건으로 평가됩니다.
- 하나의 `nodeSelectorTerms` 내부에 있는 여러 `matchExpressions`는 AND 조건으로 평가됩니다.

예를 들어 다음 설정을 살펴보겠습니다.

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
    - matchExpressions:
        - key: disktype
          operator: In
          values:
            - ssd
        - key: environment
          operator: In
          values:
            - production
    - matchExpressions:
        - key: gpu
          operator: Exists
```

위 설정은 다음과 같은 조건을 의미합니다.

```text
(disktype=ssd AND environment=production)
OR
(gpu Label이 존재함)
```

따라서 다음 중 하나를 만족하는 Node가 선택 대상이 됩니다.

1. `disktype=ssd`와 `environment=production`을 모두 만족하는 Node
2. `gpu` Label이 존재하는 Node

### 3.5. NodeSelector와 Node Affinity 함께 사용하기

Pod에는 기존 `nodeSelector`와 Node Affinity를 함께 정의할 수 있습니다.

이 경우 두 조건을 모두 만족해야 합니다.

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-selector-affinity
spec:
  nodeSelector:
    environment: production
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
  containers:
    - name: app
      image: nginx
```

이 Pod는 다음 두 조건을 모두 만족하는 Node에 배치됩니다.

- `environment=production`
- `disktype=ssd`

`nodeSelector`는 단순한 Label 일치 조건을 정의할 때 적합하며, 복잡한 조건이나 선호도를 정의해야 한다면 Node Affinity를 사용하는 것이 좋습니다.

## 4. Pod Affinity

Pod Affinity는 이미 실행 중인 다른 Pod의 Label을 기준으로 새로운 Pod를 어디에 배치할지 결정하는 기능입니다.

Node Affinity가 Node 자체의 Label을 기준으로 배치 위치를 결정한다면, Pod Affinity는 기존 Pod의 Label과 해당 Pod가 실행 중인 Node의 위치를 기준으로 배치 위치를 결정합니다.

예를 들어 다음과 같은 상황에서 사용할 수 있습니다.

- 애플리케이션 Pod와 데이터베이스 Pod를 동일한 Availability Zone에 배치
- 서로 통신량이 많은 Pod를 동일한 Node 또는 Rack에 배치
- 특정 서비스의 Pod가 이미 실행 중인 Zone에 새로운 Pod 배치

Pod Affinity는 다음 두 가지 필드를 지원합니다.

- `requiredDuringSchedulingIgnoredDuringExecution`
- `preferredDuringSchedulingIgnoredDuringExecution`

### 4.1. Required Pod Affinity

`requiredDuringSchedulingIgnoredDuringExecution`은 특정 Pod와 동일한 Topology Domain에 배치되어야 한다는 필수 조건을 정의합니다.

예를 들어 `app=database` Label이 설정된 Pod가 존재하는 Availability Zone에 애플리케이션 Pod를 배치하려면 다음과 같이 설정할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  labels:
    app: frontend
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - database
          topologyKey: topology.kubernetes.io/zone
  containers:
    - name: app
      image: nginx
```

이 설정은 다음 조건을 의미합니다.

- `app=database` Label을 가진 Pod를 찾습니다.
- 해당 Pod가 실행 중인 Node의 `topology.kubernetes.io/zone` Label을 확인합니다.
- 동일한 Zone에 새로운 Pod를 배치합니다.

예를 들어 다음과 같은 환경을 가정하겠습니다.

| Node | Zone | 실행 중인 Pod |
|---|---|---|
| node-1 | zone-a | database |
| node-2 | zone-a | 없음 |
| node-3 | zone-b | 없음 |

위 Pod Affinity가 적용되면 새로운 `app-pod`는 `zone-a`에 속한 Node에만 배치될 수 있습니다.

따라서 `node-1` 또는 `node-2`가 선택 대상이 되며, 다른 스케줄링 조건에 따라 최종 Node가 결정됩니다.

### 4.2. Preferred Pod Affinity

`preferredDuringSchedulingIgnoredDuringExecution`은 특정 Pod와 동일한 Topology Domain에 배치하는 것을 우선적으로 고려하도록 설정합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: preferred-app-pod
spec:
  affinity:
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values:
                    - database
            topologyKey: topology.kubernetes.io/zone
  containers:
    - name: app
      image: nginx
```

위 설정은 `app=database` Label을 가진 Pod가 실행 중인 Zone을 우선적으로 선택하도록 합니다.

해당 조건을 만족하는 Zone이 없더라도 다른 스케줄링 조건을 만족하는 Node가 있다면 Pod를 배치할 수 있습니다.

`weight`는 1부터 100까지 지정하며, Node Affinity와 마찬가지로 다른 스케줄링 점수와 함께 평가됩니다.

### 4.3. labelSelector

`labelSelector`는 배치 기준이 되는 기존 Pod를 선택하는 조건입니다.

Pod의 Label을 기준으로 비교하며, `matchLabels` 또는 `matchExpressions`를 사용할 수 있습니다.

예를 들어 다음과 같이 정의할 수 있습니다.

```yaml
labelSelector:
  matchLabels:
    app: database
```

또는 다음과 같이 정의할 수도 있습니다.

```yaml
labelSelector:
  matchExpressions:
    - key: app
      operator: In
      values:
        - database
        - cache
```

이 경우 `app=database` 또는 `app=cache` Label을 가진 Pod가 선택 대상이 됩니다.

### 4.4. topologyKey

`topologyKey`는 기존 Pod가 실행 중인 Node의 Label 중 어떤 값을 기준으로 동일한 위치를 판단할지 지정하는 필드입니다.

즉, Pod Affinity에서 말하는 동일한 위치는 반드시 같은 Node를 의미하지 않습니다.

대표적인 Topology Key는 다음과 같습니다.

| topologyKey | 의미 | 활용 예시 |
|---|---|---|
| `kubernetes.io/hostname` | 동일한 Node | 같은 Node에 Pod 배치 |
| `topology.kubernetes.io/zone` | 동일한 Availability Zone | 같은 Zone에 Pod 배치 |
| `topology.kubernetes.io/region` | 동일한 Region | 같은 Region에 Pod 배치 |

예를 들어 `topologyKey: kubernetes.io/hostname`을 사용하면 기존 Pod와 동일한 Node에 배치하는 조건이 됩니다.

반면 `topologyKey: topology.kubernetes.io/zone`을 사용하면 기존 Pod와 동일한 Zone에 속한 Node 중 하나에 배치됩니다.

따라서 `topologyKey`는 Pod Affinity가 적용되는 위치의 범위를 결정하는 중요한 필드입니다.

## 5. Pod Anti-Affinity

Pod Anti-Affinity는 특정 Label을 가진 Pod와 동일한 Topology Domain에 배치되지 않도록 제한하는 기능입니다.

Pod Affinity가 서로 연관된 Pod를 가까운 위치에 배치하기 위한 기능이라면, Pod Anti-Affinity는 특정 Pod끼리 같은 위치에 배치되지 않도록 분산하기 위한 기능입니다.

예를 들어 다음과 같은 상황에서 활용할 수 있습니다.

- 동일한 Deployment의 Replica Pod를 서로 다른 Node에 배치
- 장애 발생 시 여러 Replica가 동시에 영향을 받지 않도록 Zone별로 분산
- 특정 서비스의 Pod가 동일한 Node에 집중되지 않도록 제한

Pod Anti-Affinity도 다음 두 가지 필드를 지원합니다.

- `requiredDuringSchedulingIgnoredDuringExecution`
- `preferredDuringSchedulingIgnoredDuringExecution`

### 5.1. Required Pod Anti-Affinity

`requiredDuringSchedulingIgnoredDuringExecution`은 특정 Pod와 동일한 Topology Domain에 배치되지 않아야 한다는 필수 조건을 정의합니다.

다음은 `app=frontend` Label을 가진 Pod가 동일한 Node에 중복 배치되지 않도록 설정하는 예제입니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
  labels:
    app: frontend
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: frontend
          topologyKey: kubernetes.io/hostname
  containers:
    - name: app
      image: nginx
```

이 설정은 `app=frontend` Label을 가진 기존 Pod가 실행 중인 Node에는 새로운 `frontend-pod`를 배치하지 않는다는 의미입니다.

예를 들어 다음과 같은 환경을 가정하겠습니다.

| Node | 실행 중인 Pod |
|---|---|
| node-1 | frontend |
| node-2 | 없음 |
| node-3 | 없음 |

위 조건이 적용되면 새로운 `frontend-pod`는 `node-1`을 제외한 `node-2` 또는 `node-3`에 배치됩니다.

동일한 Node에 Replica Pod가 집중되는 것을 방지할 수 있습니다.

다만 조건을 만족하는 Node가 존재하지 않으면 Pod는 `Pending` 상태로 남을 수 있습니다.

### 5.2. Preferred Pod Anti-Affinity

`preferredDuringSchedulingIgnoredDuringExecution`은 특정 Pod와 동일한 Topology Domain에 배치되지 않는 것을 우선적으로 고려하도록 설정합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
  labels:
    app: frontend
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: frontend
            topologyKey: kubernetes.io/hostname
  containers:
    - name: app
      image: nginx
```

위 설정은 `app=frontend` Label을 가진 Pod가 이미 실행 중인 Node를 가능한 한 피하도록 합니다.

하지만 다른 스케줄링 조건이나 Node의 가용 리소스 상황에 따라 동일한 Node에 배치될 수도 있습니다.

따라서 Replica Pod를 가능한 한 분산시키되, 분산 조건 때문에 Pod가 스케줄링되지 않는 상황은 피하고 싶을 때 유용합니다.

### 5.3. Node 단위와 Zone 단위 분산

Pod Anti-Affinity의 `topologyKey`에 따라 분산 범위가 달라집니다.

**Node 단위 분산**

```yaml
topologyKey: kubernetes.io/hostname
```

동일한 Node에 같은 Label을 가진 Pod가 중복 배치되지 않도록 합니다.

**Zone 단위 분산**

```yaml
topologyKey: topology.kubernetes.io/zone
```

동일한 Zone에 같은 Label을 가진 Pod가 중복 배치되지 않도록 합니다.

예를 들어 3개의 Zone에 각각 여러 Node가 존재하는 클러스터에서 Zone 단위 Anti-Affinity를 사용하면 동일한 애플리케이션의 Replica Pod가 서로 다른 Zone에 배치되도록 제한할 수 있습니다.

다만 Zone 단위의 필수 Anti-Affinity는 동일 Zone에 두 번째 Replica를 배치하지 못하게 하므로, Replica 수와 Zone 수를 함께 고려해야 합니다.

## 6. Pod Affinity와 Anti-Affinity의 Namespace 범위

Pod Affinity와 Pod Anti-Affinity는 기존 Pod의 Label을 기준으로 동작하므로, 어떤 Namespace의 Pod를 검색할 것인지 지정할 수 있습니다.

이를 위해 다음 필드를 사용할 수 있습니다.

- `namespaces`
- `namespaceSelector`

### 6.1. namespaces

`namespaces`는 Label Selector를 적용할 Namespace 목록을 지정하는 필드입니다.

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
podAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: database
      namespaces:
        - database
        - production
      topologyKey: topology.kubernetes.io/zone
```

이 경우 `database`와 `production` Namespace에 존재하는 `app=database` Label의 Pod를 기준으로 Affinity 조건을 평가합니다.

`namespaces`를 생략하거나 빈 목록으로 설정하면 Affinity 규칙을 정의한 Pod 자신의 Namespace를 기본 범위로 사용합니다.

### 6.2. namespaceSelector

`namespaceSelector`는 Namespace 자체의 Label을 기준으로 검색 대상 Namespace를 선택하는 필드입니다.

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
podAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: database
      namespaceSelector:
        matchLabels:
          environment: production
      topologyKey: topology.kubernetes.io/zone
```

이 경우 `environment=production` Label이 설정된 Namespace에서 `app=database` Label을 가진 Pod를 검색합니다.

`namespaceSelector: {}`와 같이 빈 Selector를 지정하면 모든 Namespace가 검색 대상이 됩니다.

`namespaceSelector` 기능은 Kubernetes v1.24부터 Stable 상태로 제공됩니다.

### 6.3. namespaces와 namespaceSelector 함께 사용하기

`namespaces`와 `namespaceSelector`를 함께 지정할 수도 있습니다.

이 경우 두 필드가 선택하는 Namespace의 합집합이 검색 대상이 됩니다.

즉, `namespaces`에 명시된 Namespace와 `namespaceSelector`에 일치하는 Namespace를 모두 포함합니다.

두 필드를 모두 생략하거나 빈 값으로 두면 Affinity 규칙을 정의한 Pod 자신의 Namespace가 기본 검색 범위가 됩니다.

## 7. Node Affinity와 Pod Affinity 비교

Node Affinity와 Pod Affinity는 모두 Pod의 배치 위치를 제어하지만, 선택 기준이 다릅니다.

| 구분 | Node Affinity | Pod Affinity |
|---|---|---|
| 선택 기준 | Node Label | 기존 Pod Label |
| 주요 대상 | Node | Pod가 실행 중인 Topology Domain |
| 주요 목적 | 특정 인프라를 갖춘 Node 선택 | 연관된 Pod를 가까운 위치에 배치 |
| 주요 필드 | `nodeAffinity` | `podAffinity` |
| 예시 | SSD Node 선택 | Database와 동일 Zone에 배치 |

Pod Anti-Affinity는 Pod Affinity와 반대 방향의 배치 조건을 정의합니다.

| 구분 | Pod Affinity | Pod Anti-Affinity |
|---|---|---|
| 목적 | 특정 Pod와 같은 위치에 배치 | 특정 Pod와 다른 위치에 배치 |
| 대표 활용 | 통신량이 많은 Pod의 근접 배치 | Replica Pod의 분산 배치 |
| 주요 필드 | `podAffinity` | `podAntiAffinity` |

## 8. Affinity와 다른 Kubernetes 스케줄링 기능의 관계

### 8.1. Affinity와 nodeSelector

`nodeSelector`는 Node의 Label을 기준으로 Pod를 배치하는 가장 단순한 방식입니다.

다음과 같은 조건을 지정할 수 있습니다.

```yaml
spec:
  nodeSelector:
    disktype: ssd
```

Node Affinity는 `nodeSelector`보다 더 다양한 조건을 정의할 수 있습니다.

| 구분 | nodeSelector | Node Affinity |
|---|---|---|
| Label 일치 | 지원 | 지원 |
| 여러 조건 조합 | AND 조건 | AND / OR 조건 |
| 필수 조건 | 지원 | 지원 |
| 선호 조건 | 미지원 | 지원 |
| 다양한 Operator | 미지원 | 지원 |

단순히 특정 Label을 가진 Node를 선택한다면 `nodeSelector`를 사용할 수 있으며, 복잡한 조건이나 선호도를 적용해야 한다면 Node Affinity를 사용하는 것이 적합합니다.

### 8.2. Affinity와 Taint / Toleration

Affinity와 Taint / Toleration은 모두 Pod의 배치 위치에 영향을 주지만 목적이 다릅니다.

- Affinity: Pod가 특정 Node 또는 다른 Pod와 어떤 관계를 가져야 하는지 정의합니다.
- Taint: 특정 Node에 Pod가 배치되는 것을 제한하거나 회피하도록 합니다.
- Toleration: Pod가 특정 Taint를 허용할 수 있도록 설정합니다.

Toleration은 해당 Node에 Pod를 반드시 배치하도록 보장하지 않습니다.

예를 들어 GPU Node에 다음과 같은 Taint가 설정되어 있다고 가정하겠습니다.

```bash
kubectl taint nodes gpu-node-1 accelerator=nvidia:NoSchedule
```

GPU를 사용하는 Pod가 해당 Node에 배치되려면 Toleration을 설정해야 합니다.

```yaml
spec:
  tolerations:
    - key: accelerator
      operator: Equal
      value: nvidia
      effect: NoSchedule
```

여기에 Node Affinity를 함께 사용하면 GPU Node만 선택하도록 제한할 수 있습니다.

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: accelerator
                operator: In
                values:
                  - nvidia
  tolerations:
    - key: accelerator
      operator: Equal
      value: nvidia
      effect: NoSchedule
```

이렇게 구성하면 Taint를 허용하면서 GPU Label을 가진 Node에만 배치할 수 있습니다.

### 8.3. Affinity와 Topology Spread Constraints

`Topology Spread Constraints`는 Pod를 Node, Zone 등의 Topology Domain에 걸쳐 균등하게 분산시키기 위한 기능입니다.

Pod Anti-Affinity도 Replica Pod를 분산하는 데 사용할 수 있지만, 두 기능의 목적과 동작 방식에는 차이가 있습니다.

| 구분 | Pod Anti-Affinity | Topology Spread Constraints |
|---|---|---|
| 주요 목적 | 특정 Pod와 동일한 위치에 배치되지 않도록 제한 | 여러 Topology Domain에 Pod를 균등하게 분산 |
| 배치 기준 | 기존 Pod Label | 기존 Pod의 분포 |
| 제약 방식 | Required / Preferred | `whenUnsatisfiable` |
| 주요 필드 | `podAntiAffinity` | `topologySpreadConstraints` |
| 활용 사례 | 동일 Node에 Replica 중복 배치 방지 | Zone별 Replica 분산 |

예를 들어 3개의 Zone에 동일한 애플리케이션 Pod를 균등하게 배치하려면 Topology Spread Constraints를 사용할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
  labels:
    app: frontend
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: frontend
  containers:
    - name: app
      image: nginx
```

`maxSkew`는 Topology Domain 사이에서 허용할 Pod 개수 차이를 정의합니다.

위 예제에서는 Zone별 Pod 개수 차이가 1을 초과하지 않도록 제한합니다.

`whenUnsatisfiable`은 분산 조건을 만족하지 못할 때의 동작을 결정합니다.

- `DoNotSchedule`: 조건을 만족하지 못하면 Pod를 스케줄링하지 않습니다.
- `ScheduleAnyway`: 조건을 만족하는 방향으로 점수를 부여하지만, 조건을 만족하지 못해도 스케줄링할 수 있습니다.

Replica Pod를 여러 Node 또는 Zone에 균등하게 분산시키는 것이 목적이라면 Topology Spread Constraints를 우선적으로 고려할 수 있습니다.

## 9. Affinity 적용 시 주의사항

### 9.1. Required 조건으로 인한 Pending 상태

`requiredDuringSchedulingIgnoredDuringExecution`은 반드시 만족해야 하는 조건입니다.

따라서 조건을 만족하는 Node가 없거나 다른 스케줄링 제약 조건과 충돌하면 Pod가 `Pending` 상태로 남을 수 있습니다.

특히 Pod Anti-Affinity를 필수 조건으로 설정하면 Replica 수가 Node 또는 Zone의 수보다 많을 때 모든 Pod를 배치하지 못할 수 있습니다.

### 9.2. Topology Label의 일관성

Pod Affinity와 Anti-Affinity는 `topologyKey`에 지정한 Node Label을 기준으로 위치를 판단합니다.

따라서 해당 Label이 Node에 일관되게 설정되어 있어야 합니다.

예를 들어 다음과 같은 Topology Key를 사용한다면 각 Node에 해당 Label이 적절하게 존재해야 합니다.

- `kubernetes.io/hostname`
- `topology.kubernetes.io/zone`
- `topology.kubernetes.io/region`

특히 Pod Anti-Affinity는 일부 Node에 Topology Label이 누락되어 있으면 의도하지 않은 스케줄링 결과가 발생할 수 있습니다.

### 9.3. Pod Affinity와 Anti-Affinity의 성능

Pod Affinity와 Anti-Affinity는 기존 Pod의 Label 및 위치 정보를 기준으로 조건을 평가하므로 Node Affinity보다 많은 스케줄링 연산이 필요할 수 있습니다.

Kubernetes 공식 문서에서는 이러한 규칙이 수백 개 이상의 Node로 구성된 대규모 클러스터에서 스케줄링 성능을 저하시킬 수 있다고 안내합니다.

따라서 대규모 클러스터에서는 Pod Affinity와 Anti-Affinity를 필요한 범위에서만 사용하는 것이 좋습니다.

### 9.4. Required Pod Anti-Affinity의 Topology 제한

Kubernetes에는 `LimitPodHardAntiAffinityTopology` Admission Controller가 있으며, 기본 설정에서는 Required Pod Anti-Affinity의 `topologyKey`를 `kubernetes.io/hostname`으로 제한합니다.

따라서 다음과 같이 Node 단위가 아닌 Zone 단위의 Required Pod Anti-Affinity를 사용하려면 클러스터의 Admission Controller 설정을 확인해야 합니다.

```yaml
topologyKey: topology.kubernetes.io/zone
```

클러스터 관리자가 해당 Admission Controller의 설정을 변경하거나 비활성화한 경우에는 다른 Topology Key를 사용할 수 있습니다.

### 9.5. Pod Affinity의 Self-Affinity

Pod Affinity를 사용하는 Pod 그룹에서 최초 Pod가 아직 존재하지 않는 경우, 모든 Pod가 서로를 기다리면서 스케줄링되지 못하는 문제가 발생할 수 있습니다.

Kubernetes는 이러한 상황을 방지하기 위해 Self-Affinity 관련 예외 동작을 지원합니다.

현재 스케줄링 중인 Pod가 자신의 Affinity Selector에 일치하고 다른 조건도 만족하며, 해당 Topology 조건을 충족하는 경우 최초 Pod가 배치될 수 있습니다.

이를 통해 동일한 Pod Affinity 규칙을 가진 Pod 그룹이 최초 Pod부터 스케줄링될 수 있도록 지원합니다.

### 9.6. Affinity는 실행 중인 Pod를 자동으로 이동시키지 않습니다.

Node Affinity와 Pod Affinity 및 Anti-Affinity의 일반적인 `IgnoredDuringExecution` 규칙은 이미 배치된 Pod에 대해 스케줄링 조건을 지속적으로 강제하지 않습니다.

따라서 Node Label이 변경되거나 다른 Pod의 배치 상태가 변경되더라도 기존 Pod가 자동으로 다른 Node로 이동하지 않습니다.

배치 조건을 변경하려면 Pod를 재생성하거나 별도의 재배치 절차를 수행해야 합니다.

## 10. Affinity 확인 및 관리 명령어

### 10.1. Node Label 확인

```bash
kubectl get nodes --show-labels
```

클러스터의 Node에 설정된 Label을 확인합니다.

특정 Node의 Label만 확인하려면 다음 명령어를 사용할 수 있습니다.

```bash
kubectl get node node-1 --show-labels
```

### 10.2. Node Label 추가

```bash
kubectl label nodes node-1 disktype=ssd
```

Node에 `disktype=ssd` Label을 추가합니다.

### 10.3. Node Label 삭제

```bash
kubectl label nodes node-1 disktype-
```

Node에 설정된 `disktype` Label을 삭제합니다.

### 10.4. Pod의 Affinity 설정 확인

```bash
kubectl get pod frontend-pod -o yaml
```

Pod에 설정된 Node Affinity, Pod Affinity 및 Pod Anti-Affinity를 확인합니다.

### 10.5. Pod 스케줄링 상태 확인

```bash
kubectl describe pod frontend-pod
```

Pod가 스케줄링되지 않는 경우 Events 영역에서 Affinity 조건이나 다른 스케줄링 제약 조건으로 인해 발생한 문제를 확인할 수 있습니다.

### 10.6. Pod의 Node 배치 확인

```bash
kubectl get pods -o wide
```

Pod가 실제로 어떤 Node에 배치되었는지 확인합니다.

### 10.7. Node별 Pod 배치 현황 확인

```bash
kubectl get pods -A -o wide
```

Namespace 전체의 Pod와 배치된 Node를 확인하여 Replica Pod의 분산 상태 등을 점검할 수 있습니다.

## 11. 정리

`Affinity`는 Kubernetes Scheduler가 Pod를 배치할 Node를 선택할 때 적용하는 스케줄링 규칙입니다.

주요 유형은 다음과 같습니다.

- **Node Affinity:** Node Label을 기준으로 Pod가 배치될 Node를 선택합니다.
- **Pod Affinity:** 특정 Label을 가진 Pod와 동일한 Topology Domain에 배치합니다.
- **Pod Anti-Affinity:** 특정 Label을 가진 Pod와 동일한 Topology Domain에 배치되지 않도록 제한합니다.

각 Affinity 유형은 필수 조건인 `requiredDuringSchedulingIgnoredDuringExecution`과 선호 조건인 `preferredDuringSchedulingIgnoredDuringExecution`을 지원합니다.

Node Affinity는 특정 하드웨어나 인프라를 갖춘 Node를 선택하는 데 적합하며, Pod Affinity와 Anti-Affinity는 Pod 간의 배치 관계를 제어하는 데 적합합니다.

또한 Replica Pod를 여러 Node나 Zone에 균등하게 분산하는 것이 목적이라면 Topology Spread Constraints와의 차이를 고려하여 적절한 기능을 선택해야 합니다.

Affinity를 적절히 활용하면 애플리케이션의 통신 효율성, 장애 격리, 리소스 활용도 및 고가용성을 고려한 Pod 배치 정책을 구성할 수 있습니다.

## 참고자료

- [Kubernetes 공식 문서 - Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Kubernetes 공식 문서 - Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Kubernetes 공식 문서 - Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [Kubernetes 공식 문서 - Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Kubernetes API Reference - Pod v1](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/)

