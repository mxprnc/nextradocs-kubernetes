# DaemonSet

## 1. DaemonSet이란?

**DaemonSet은 Kubernetes 클러스터의 모든 노드 또는 지정된 일부 노드에서 Pod가 하나씩 실행되도록 관리하는 Workload 리소스입니다.**

일반적인 Deployment는 지정된 개수의 Pod를 클러스터에 배포하고 유지합니다. 이때 Pod가 어떤 노드에 배치되는지는 Kubernetes Scheduler가 결정합니다.

반면 DaemonSet은 특정 조건에 해당하는 각 노드에서 하나의 Pod가 실행되도록 관리합니다. 새로운 노드가 클러스터에 추가되면 해당 노드가 배치 조건에 부합하는지 확인한 후 Pod를 생성하고, 노드가 클러스터에서 제거되면 해당 노드에서 실행되던 Pod도 관리 대상에서 제외됩니다.

DaemonSet은 주로 노드 단위로 실행되어야 하는 인프라 구성 요소에 사용됩니다.

- 로그 수집 에이전트
- 모니터링 에이전트
- 노드 네트워크 플러그인
- 스토리지 관련 데몬
- 보안 및 시스템 관리 에이전트

대표적인 사용 사례는 다음과 같습니다.

| 사용 사례 | 설명 |
| --- | --- |
| 로그 수집 | 각 노드에서 컨테이너 로그를 수집하는 Fluent Bit 등의 에이전트를 실행합니다. |
| 모니터링 | 각 노드의 CPU, 메모리, 디스크 등의 상태를 수집하는 에이전트를 실행합니다. |
| 네트워크 | 각 노드에서 네트워크 설정이나 패킷 처리를 담당하는 구성 요소를 실행합니다. |
| 스토리지 | 각 노드에서 스토리지 연결 및 관리 작업을 수행하는 구성 요소를 실행합니다. |
| 보안 | 각 노드에서 시스템 상태를 감시하거나 보안 이벤트를 수집하는 에이전트를 실행합니다. |

DaemonSet의 핵심 목적은 **클러스터의 노드마다 필요한 Pod를 자동으로 배치하고 유지하는 것**입니다.

## 2. DaemonSet의 구조

DaemonSet은 Pod Template을 기반으로 대상 노드마다 Pod를 생성하고 관리합니다.

예를 들어 클러스터에 Worker Node가 3개 존재하고 모든 노드를 대상으로 DaemonSet을 생성하면 각 노드에 하나씩 Pod가 배치됩니다.

```text
Kubernetes Cluster
│
└── DaemonSet: fluent-bit
    │
    ├── Worker Node 1
    │   └── fluent-bit Pod
    │
    ├── Worker Node 2
    │   └── fluent-bit Pod
    │
    └── Worker Node 3
        └── fluent-bit Pod
```

DaemonSet은 단순히 Pod 개수를 지정하는 방식이 아니라, 대상 노드의 조건에 따라 필요한 Pod를 배치합니다.

따라서 노드가 추가되거나 제거되면 DaemonSet Controller가 이를 감지하고 관리 대상 Pod 구성을 조정합니다.

DaemonSet은 일반적으로 대상 노드마다 하나의 Pod를 생성합니다. 다만 Pod의 실제 실행 여부는 노드 상태, 리소스 여유, 스케줄링 조건 및 컨테이너 실행 가능 여부 등에 영향을 받습니다.

## 3. DaemonSet YAML 구성

다음은 클러스터의 각 대상 노드에서 Nginx 컨테이너를 실행하는 DaemonSet 예제입니다.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-daemonset
spec:
  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

각 필드의 역할은 다음과 같습니다.

| 필드 | 설명 |
| --- | --- |
| `apiVersion` | Kubernetes API 버전을 지정합니다. DaemonSet은 `apps/v1`을 사용합니다. |
| `kind` | 생성할 리소스의 종류를 지정합니다. |
| `metadata` | DaemonSet의 이름과 메타데이터를 정의합니다. |
| `spec` | DaemonSet의 원하는 상태를 정의합니다. |
| `selector` | DaemonSet이 관리할 Pod를 식별하는 조건을 지정합니다. |
| `template` | 각 대상 노드에 생성할 Pod의 구성을 정의합니다. |

### 3.1. selector

`selector`는 DaemonSet이 관리할 Pod를 식별하는 조건입니다.

```yaml
spec:
  selector:
    matchLabels:
      app: nginx
```

Selector는 Pod Template에 정의된 레이블과 일치해야 합니다.

```yaml
spec:
  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx
```

`apps/v1` API에서 Selector는 필수이며, Pod Template의 레이블과 일치해야 합니다.

또한 DaemonSet 생성 이후 Selector는 변경할 수 없습니다. 따라서 리소스를 생성하기 전에 관리할 Pod의 레이블과 Selector가 올바르게 설정되었는지 확인해야 합니다.

### 3.2. template

`template`은 DaemonSet이 각 대상 노드에 생성할 Pod의 구성을 정의합니다.

```yaml
spec:
  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

Pod Template에는 컨테이너 이미지, 환경 변수, 리소스 요청량, 볼륨 마운트, 보안 설정 등을 정의할 수 있습니다.

DaemonSet은 이 Template을 기반으로 대상 노드마다 Pod를 생성합니다.

Pod Template이 변경되면 설정된 업데이트 전략에 따라 기존 Pod를 새로운 구성으로 교체할 수 있습니다.

## 4. DaemonSet의 주요 특징

### 4.1. 노드마다 하나의 Pod 실행

DaemonSet의 가장 중요한 특징은 대상 노드마다 하나의 Pod를 실행하는 것입니다.

예를 들어 클러스터에 Worker Node가 3개 존재한다면 다음과 같이 Pod가 배치됩니다.

```text
DaemonSet: fluent-bit
│
├── Worker Node 1
│   └── fluent-bit Pod
│
├── Worker Node 2
│   └── fluent-bit Pod
│
└── Worker Node 3
    └── fluent-bit Pod
```

DaemonSet은 노드별 Pod 개수를 직접 지정하는 대신, DaemonSet의 배치 조건에 해당하는 노드 수에 따라 필요한 Pod를 관리합니다.

일반적으로 하나의 DaemonSet은 각 대상 노드에 하나의 Pod를 유지합니다.

단, 업데이트 전략에서 `maxSurge`를 설정하면 업데이트 과정에서 동일 노드에 기존 Pod와 새로운 Pod가 일시적으로 함께 존재할 수 있습니다.

### 4.2. 새로운 노드에 자동 배치

클러스터에 새로운 노드가 추가되면 DaemonSet Controller는 해당 노드가 DaemonSet의 배치 조건에 부합하는지 확인합니다.

조건에 부합한다면 해당 노드를 대상으로 새로운 Pod를 생성합니다.

```text
Before

Worker Node 1 ── fluent-bit Pod
Worker Node 2 ── fluent-bit Pod
Worker Node 3 ── fluent-bit Pod


Worker Node 4 추가


After

Worker Node 1 ── fluent-bit Pod
Worker Node 2 ── fluent-bit Pod
Worker Node 3 ── fluent-bit Pod
Worker Node 4 ── fluent-bit Pod (New)
```

이를 통해 노드가 추가되더라도 로그 수집이나 모니터링 에이전트를 별도로 배포할 필요가 없습니다.

### 4.3. 노드 제거 시 Pod 정리

DaemonSet의 대상 노드가 클러스터에서 제거되면 해당 노드는 DaemonSet의 관리 대상에서 제외됩니다.

DaemonSet Controller는 대상 노드의 변경 사항을 반영하여 관리 대상 Pod를 조정합니다.

다만 노드가 비정상적으로 연결이 끊어진 경우에는 노드 상태, Pod의 종료 처리 및 노드 삭제 여부에 따라 정리 시점이 달라질 수 있습니다.

### 4.4. 노드 선택 조건

DaemonSet은 모든 노드에 Pod를 배치할 수도 있지만, 특정 조건에 해당하는 노드에만 Pod를 배치하도록 구성할 수도 있습니다.

대표적으로 다음과 같은 설정을 사용할 수 있습니다.

- `nodeSelector`
- Node Affinity
- Taint와 Toleration

예를 들어 특정 노드에 `disk=ssd` 레이블이 지정되어 있다면 해당 노드에만 DaemonSet Pod를 배치할 수 있습니다.

```yaml
spec:
  template:
    spec:
      nodeSelector:
        disk: ssd

      containers:
        - name: agent
          image: nginx:1.27
```

위 설정은 `disk=ssd` 레이블을 가진 노드에만 Pod를 배치하도록 구성합니다.

```text
Worker Node 1
disk=ssd
└── DaemonSet Pod 생성

Worker Node 2
disk=hdd
└── DaemonSet Pod 생성하지 않음

Worker Node 3
disk=ssd
└── DaemonSet Pod 생성
```

Node Affinity를 사용하면 단순한 레이블 일치뿐 아니라 여러 조건을 조합하여 대상 노드를 선택할 수 있습니다.

## 5. DaemonSet과 Taint 및 Toleration

Kubernetes에서는 특정 노드에 일반적인 Pod가 배치되지 않도록 Taint를 설정할 수 있습니다.

Taint가 설정된 노드에는 해당 Taint를 허용하는 Toleration이 없는 Pod가 스케줄링되지 않습니다.

DaemonSet Controller는 노드 단위로 실행되어야 하는 에이전트가 특정 노드 상태 때문에 배치되지 못하는 상황을 줄이기 위해 기본적인 Toleration을 자동으로 추가합니다.

대표적인 기본 Toleration은 다음과 같습니다.

| Taint Key | Effect | 설명 |
| --- | --- | --- |
| `node.kubernetes.io/not-ready` | `NoExecute` | 노드가 Ready 상태가 아닌 경우에도 DaemonSet Pod가 배치될 수 있으며, 해당 Taint만을 이유로 실행 중인 Pod가 제거되지 않도록 합니다. |
| `node.kubernetes.io/unreachable` | `NoExecute` | 노드에 접근할 수 없는 경우에도 DaemonSet Pod가 배치될 수 있으며, 해당 Taint만을 이유로 실행 중인 Pod가 제거되지 않도록 합니다. |
| `node.kubernetes.io/disk-pressure` | `NoSchedule` | 디스크 압박 상태인 노드에도 Pod를 배치할 수 있도록 허용합니다. |
| `node.kubernetes.io/memory-pressure` | `NoSchedule` | 메모리 압박 상태인 노드에도 Pod를 배치할 수 있도록 허용합니다. |
| `node.kubernetes.io/pid-pressure` | `NoSchedule` | 프로세스 ID 자원이 부족한 노드에도 Pod를 배치할 수 있도록 허용합니다. |
| `node.kubernetes.io/unschedulable` | `NoSchedule` | 스케줄링 불가능 상태로 표시된 노드에도 Pod를 배치할 수 있도록 허용합니다. |
| `node.kubernetes.io/network-unavailable` | `NoSchedule` | `hostNetwork: true`를 사용하는 DaemonSet Pod에 추가되며, 네트워크를 사용할 수 없는 노드에도 배치할 수 있도록 허용합니다. |

이러한 기본 Toleration은 DaemonSet Pod가 노드의 상태에 관계없이 무조건 실행된다는 의미는 아닙니다.

Toleration은 특정 Taint가 Pod의 배치를 차단하지 않도록 허용하는 설정입니다. 노드의 실제 리소스 부족, Pod의 다른 스케줄링 제약 조건, 컨테이너 실행 오류 등은 별도로 영향을 미칩니다.

또한 기본 Toleration이 모든 종류의 Taint를 허용하는 것은 아닙니다. 예를 들어 Control Plane 노드에 설정되는 다음과 같은 Taint는 별도의 Toleration을 추가해야 할 수 있습니다.

```text
node-role.kubernetes.io/control-plane:NoSchedule
```

### 5.1. 특정 Taint를 허용하는 설정

DaemonSet의 Pod Template에 Toleration을 직접 추가할 수도 있습니다.

```yaml
spec:
  template:
    spec:
      tolerations:
        - key: dedicated
          operator: Equal
          value: monitoring
          effect: NoSchedule

      containers:
        - name: monitoring-agent
          image: nginx:1.27
```

위 설정은 다음 Taint가 적용된 노드에도 Pod를 배치할 수 있도록 허용합니다.

```bash
kubectl taint nodes worker-1 dedicated=monitoring:NoSchedule
```

이를 통해 모니터링 전용 노드처럼 특정 용도로 분리된 노드에도 DaemonSet Pod를 배치할 수 있습니다.

단, Toleration은 해당 노드에 Pod를 배치할 수 있도록 허용하는 설정이지, 해당 노드를 반드시 선택하도록 지정하는 설정은 아닙니다. 특정 노드에만 배치하려면 `nodeSelector` 또는 Node Affinity 등의 조건도 함께 사용해야 합니다.

## 6. DaemonSet의 Pod 배치 과정

DaemonSet은 DaemonSet Controller가 대상 노드를 확인하고 각 노드에 필요한 Pod를 생성하는 방식으로 동작합니다.

일반적인 Deployment와 달리 DaemonSet Controller는 단순히 전체 Pod 개수를 유지하는 것이 아니라 각 대상 노드에 Pod가 배치되도록 관리합니다.

대표적인 동작 과정은 다음과 같습니다.

```text
DaemonSet 생성
      │
      ▼
DaemonSet Controller가
대상 노드 확인
      │
      ▼
Node Selector 및
Affinity 조건 확인
      │
      ▼
각 대상 노드를 대상으로
Pod 생성
      │
      ▼
Pod에 대상 노드를 지정하는
Node Affinity 설정
      │
      ▼
Kubernetes Scheduler가
대상 노드에 Pod 배치
      │
      ▼
kubelet이 Pod 실행
      │
      ▼
노드 변경 사항 감지
      │
      ▼
필요한 Pod 생성 및 정리
```

DaemonSet Controller는 대상 노드마다 Pod를 생성하고, 생성된 Pod의 `spec.affinity.nodeAffinity`에 해당 노드를 대상으로 하는 조건을 설정합니다.

이후 일반적으로 Kubernetes Scheduler가 해당 Node Affinity를 확인하고 대상 노드에 Pod를 바인딩합니다. 바인딩 과정에서 Pod의 `spec.nodeName`이 설정됩니다.

따라서 DaemonSet Controller가 일반적으로 `spec.nodeName`을 직접 지정하여 Pod를 배치하는 것은 아닙니다.

또한 DaemonSet Pod는 기본 Scheduler를 사용하는 것이 일반적이지만, Pod Template의 `schedulerName`을 통해 다른 Scheduler를 지정할 수도 있습니다.

DaemonSet Pod가 대상 노드에 배치되더라도 실제 컨테이너 실행은 kubelet과 컨테이너 런타임이 담당합니다. 노드의 리소스가 부족하거나 컨테이너 이미지 다운로드에 실패하는 등의 상황에서는 Pod가 정상적으로 실행되지 못할 수 있습니다.

## 7. DaemonSet의 업데이트 전략

DaemonSet은 Pod Template이 변경되었을 때 기존 Pod를 새로운 구성으로 교체할 수 있습니다.

대표적인 업데이트 전략은 다음과 같습니다.

- RollingUpdate
- OnDelete

### 7.1. RollingUpdate

RollingUpdate는 DaemonSet의 기본 업데이트 전략입니다.

Pod Template이 변경되면 기존 Pod를 새로운 구성으로 순차적으로 교체합니다.

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
```

예를 들어 컨테이너 이미지를 `nginx:1.27`에서 `nginx:1.28`로 변경하면 각 노드에서 실행 중인 Pod가 새로운 이미지로 교체됩니다.

```text
Before Update

Worker Node 1 ── nginx:1.27
Worker Node 2 ── nginx:1.27
Worker Node 3 ── nginx:1.27


Rolling Update

Worker Node 1 ── nginx:1.28
Worker Node 2 ── nginx:1.27
Worker Node 3 ── nginx:1.27

          │
          ▼

Worker Node 1 ── nginx:1.28
Worker Node 2 ── nginx:1.28
Worker Node 3 ── nginx:1.27

          │
          ▼

Worker Node 1 ── nginx:1.28
Worker Node 2 ── nginx:1.28
Worker Node 3 ── nginx:1.28
```

RollingUpdate는 다음 설정을 통해 업데이트 동작을 제어할 수 있습니다.

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 0
```

| 필드 | 설명 |
| --- | --- |
| `maxUnavailable` | 업데이트 과정에서 사용할 수 없는 상태가 허용되는 DaemonSet Pod의 최대 개수를 지정합니다. |
| `maxSurge` | 업데이트 과정에서 기존 Pod가 실행 중인 노드에 새로운 Pod를 추가로 생성할 수 있는 최대 개수를 지정합니다. |

`maxUnavailable`은 기본값이 `1`이며, 정수 또는 백분율로 지정할 수 있습니다.

예를 들어 `maxUnavailable: 1`은 업데이트 과정에서 동시에 사용할 수 없는 DaemonSet Pod를 최대 1개로 제한합니다.

`maxSurge`는 기본값이 `0`입니다. 기본값을 사용하면 기존 Pod를 종료한 후 새로운 Pod를 생성하는 방식으로 업데이트가 진행됩니다.

반면 `maxSurge`를 1 이상으로 설정하면 기존 Pod가 실행 중인 상태에서 새로운 Pod를 추가로 생성할 수 있습니다. 새로운 Pod가 Ready 상태가 된 후 기존 Pod를 종료하는 방식으로 업데이트를 진행할 수 있습니다.

이 경우 동일 노드에 기존 Pod와 새로운 Pod가 일시적으로 함께 실행될 수 있으므로 추가 리소스 사용량과 포트 충돌 등을 고려해야 합니다.

`maxUnavailable`과 `maxSurge`는 동시에 0으로 설정할 수 없습니다. 하나의 업데이트 전략에서 최소 하나의 Pod 교체 방식을 허용해야 합니다.

또한 `minReadySeconds`를 설정하면 새로운 Pod가 Ready 상태로 지정된 시간 동안 유지된 후 사용 가능한 Pod로 인정됩니다.

### 7.2. OnDelete

OnDelete는 Pod Template이 변경되더라도 기존 Pod를 자동으로 교체하지 않는 업데이트 전략입니다.

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

기존 Pod를 직접 삭제해야 새로운 Template을 적용한 Pod가 생성됩니다.

예를 들어 다음과 같이 동작합니다.

```text
Before

Worker Node 1 ── nginx:1.27
Worker Node 2 ── nginx:1.27
Worker Node 3 ── nginx:1.27


Template 변경

nginx:1.27 → nginx:1.28


기존 Pod 유지

Worker Node 1 ── nginx:1.27
Worker Node 2 ── nginx:1.27
Worker Node 3 ── nginx:1.27


Worker Node 2의 Pod 삭제


새로운 Pod 생성

Worker Node 1 ── nginx:1.27
Worker Node 2 ── nginx:1.28
Worker Node 3 ── nginx:1.27
```

OnDelete는 업데이트 시점을 운영자가 직접 제어해야 하는 경우에 사용할 수 있습니다.

| 구분 | RollingUpdate | OnDelete |
| --- | --- | --- |
| 업데이트 방식 | 기존 Pod를 자동으로 교체합니다. | Pod를 직접 삭제해야 교체됩니다. |
| 업데이트 시점 | DaemonSet Controller가 관리합니다. | 운영자가 직접 제어합니다. |
| 주요 사용 사례 | 일반적인 에이전트 업데이트 | 업데이트 시점을 수동으로 제어해야 하는 경우 |

## 8. DaemonSet과 Deployment의 차이

DaemonSet과 Deployment는 모두 Pod를 생성하고 관리하는 Workload 리소스입니다.

하지만 Pod를 배치하는 기준에 차이가 있습니다.

Deployment는 지정된 복제본 개수를 유지하는 데 집중하며, Pod가 어떤 노드에 배치되는지는 Scheduler가 결정합니다.

반면 DaemonSet은 대상 노드마다 하나의 Pod가 실행되도록 관리합니다.

| 구분 | Deployment | DaemonSet |
| --- | --- | --- |
| 주요 목적 | 애플리케이션의 복제본 관리 | 노드별 Pod 배치 및 관리 |
| Pod 개수 | `replicas`로 지정 | 대상 노드 수와 배치 조건에 따라 결정 |
| Pod 배치 | Scheduler가 배치 노드를 결정 | 대상 노드마다 Pod가 배치되도록 관리 |
| 신규 노드 추가 | 기존 복제본 수를 유지하며 배치 조정 | 조건에 부합하는 신규 노드에 Pod 추가 |
| 노드별 Pod 수 | 특정 노드에 여러 Pod가 배치될 수 있음 | 기본적으로 대상 노드마다 하나의 Pod 유지 |
| 업데이트 | RollingUpdate, Recreate | RollingUpdate, OnDelete |
| 주요 사용 사례 | 웹 애플리케이션, API 서버 | 로그 수집, 모니터링, 네트워크 에이전트 |

### 8.1. Deployment의 Pod 배치

Deployment는 `replicas`에 지정된 개수만큼 Pod를 유지합니다.

```yaml
spec:
  replicas: 3
```

3개의 Pod가 생성되더라도 반드시 서로 다른 노드에 하나씩 배치되는 것은 아닙니다.

```text
Deployment
│
├── Worker Node 1
│   ├── nginx Pod
│   └── nginx Pod
│
└── Worker Node 2
    └── nginx Pod
```

Pod의 분산 배치가 필요하다면 Pod Anti-Affinity나 Topology Spread Constraints 등을 추가로 구성해야 합니다.

### 8.2. DaemonSet의 Pod 배치

DaemonSet은 대상 노드마다 하나의 Pod가 실행되도록 관리합니다.

```text
DaemonSet
│
├── Worker Node 1
│   └── fluent-bit Pod
│
├── Worker Node 2
│   └── fluent-bit Pod
│
└── Worker Node 3
    └── fluent-bit Pod
```

따라서 모든 대상 노드에서 동일한 에이전트가 실행되어야 하는 경우 DaemonSet이 적합합니다.

## 9. DaemonSet과 다른 Workload 리소스의 관계

Kubernetes의 Workload 리소스는 애플리케이션의 실행 목적에 따라 구분됩니다.

| 리소스 | 주요 역할 | 주요 사용 사례 |
| --- | --- | --- |
| Pod | 컨테이너 실행 단위 | 단일 애플리케이션 실행 |
| Deployment | Stateless 애플리케이션 배포 및 관리 | 웹 애플리케이션, API 서버 |
| ReplicaSet | 지정된 Pod 복제본 개수 유지 | Pod 복제본 관리 |
| StatefulSet | 안정적인 식별자와 스토리지를 가진 Pod 관리 | 데이터베이스, 분산 시스템 |
| DaemonSet | 대상 노드마다 Pod 배치 | 로그 수집, 모니터링 에이전트 |
| Job | 작업이 완료될 때까지 Pod 실행 | 배치 작업 |
| CronJob | 일정에 따라 Job 생성 | 정기적인 배치 작업 |

DaemonSet은 Deployment처럼 애플리케이션의 복제본 개수를 기준으로 관리하지 않습니다.

대신 클러스터의 노드 구성을 기준으로 Pod를 배치한다는 점에서 차이가 있습니다.

## 10. DaemonSet 사용 시 고려사항

### 10.1. 모든 노드에 반드시 배치되는 것은 아닙니다.

DaemonSet은 기본적으로 모든 적격 노드에 Pod를 배치할 수 있지만, 실제 배치 대상은 Node Selector, Node Affinity, Taint 및 Toleration 등의 설정에 따라 달라집니다.

또한 Pod의 리소스 요구사항이나 노드 상태 등에 따라 Pod가 정상적으로 실행되지 못할 수 있습니다.

따라서 DaemonSet을 생성했다고 해서 모든 노드에서 Pod가 반드시 실행되는 것은 아닙니다.

### 10.2. DaemonSet Pod의 리소스 사용량을 고려해야 합니다.

DaemonSet은 대상 노드마다 Pod를 실행하므로 노드 수가 증가하면 전체 Pod 개수와 리소스 사용량도 증가합니다.

예를 들어 100개의 노드에서 CPU 100m, 메모리 128Mi를 사용하는 모니터링 에이전트를 실행한다면 전체적으로 다음과 같은 리소스 요청량이 발생합니다.

- CPU 요청량: 10 CPU
- 메모리 요청량: 12.5Gi

따라서 DaemonSet에 배포하는 에이전트의 CPU 및 메모리 요청량과 제한값을 적절하게 설정해야 합니다.

또한 `maxSurge`를 사용하는 업데이트에서는 일부 노드에서 기존 Pod와 새로운 Pod가 동시에 실행될 수 있으므로 업데이트 중 추가 리소스 사용량도 고려해야 합니다.

### 10.3. HostPath 사용 시 보안에 주의해야 합니다.

로그 수집 에이전트나 모니터링 에이전트는 노드의 파일 시스템에 접근해야 하는 경우가 있습니다.

이때 `hostPath` Volume을 사용하여 노드의 디렉터리를 Pod에 마운트할 수 있습니다.

```yaml
volumes:
  - name: varlog
    hostPath:
      path: /var/log
      type: Directory
```

하지만 HostPath를 사용하면 컨테이너가 노드의 파일 시스템에 접근할 수 있으므로 보안 위험이 발생할 수 있습니다.

필요한 경로에만 접근하도록 제한하고, 컨테이너의 권한과 보안 설정을 함께 검토해야 합니다.

특히 노드의 루트 파일 시스템이나 민감한 디렉터리를 불필요하게 마운트하지 않도록 주의해야 합니다.

### 10.4. 업데이트 과정에서 서비스 중단 가능성을 고려해야 합니다.

DaemonSet은 노드마다 하나의 Pod를 실행하는 구조이므로 RollingUpdate 과정에서 기존 Pod가 종료되고 새로운 Pod가 준비되는 동안 해당 노드의 에이전트가 일시적으로 실행되지 않을 수 있습니다.

특히 로그 수집이나 네트워크 관련 에이전트는 업데이트 중 발생할 수 있는 데이터 수집 공백이나 통신 영향을 고려해야 합니다.

필요하다면 다음 설정을 적절하게 구성해야 합니다.

- `maxUnavailable`: 업데이트 중 사용할 수 없는 Pod의 최대 개수 제한
- `maxSurge`: 기존 Pod를 유지하면서 새로운 Pod를 추가로 실행할 수 있는 개수 지정
- `minReadySeconds`: 새로운 Pod가 Ready 상태를 유지해야 하는 시간 지정
- Readiness Probe: 애플리케이션이 실제로 요청을 처리할 수 있는 상태인지 확인

단, `maxSurge`를 사용하면 동일 노드에서 기존 Pod와 새로운 Pod가 함께 실행될 수 있으므로 포트 충돌이나 리소스 부족 가능성을 검토해야 합니다.

### 10.5. DaemonSet은 애플리케이션의 고가용성을 자동으로 보장하지 않습니다.

DaemonSet은 노드마다 Pod를 배치하는 기능을 제공하지만, 해당 애플리케이션 자체의 고가용성이나 데이터 복제 기능을 제공하지는 않습니다.

예를 들어 DaemonSet으로 각 노드에 데이터베이스를 하나씩 실행하더라도 데이터베이스 클러스터가 자동으로 구성되는 것은 아닙니다.

애플리케이션의 복제, 장애 조치, 데이터 동기화 등은 별도의 구성이 필요합니다.

### 10.6. Control Plane 노드의 배치 여부를 확인해야 합니다.

Kubernetes 클러스터의 Control Plane 노드에는 일반적으로 일반 워크로드의 배치를 제한하기 위한 Taint가 설정되어 있습니다.

DaemonSet Controller가 기본 Toleration을 추가하더라도 Control Plane 노드에 설정된 모든 Taint를 자동으로 허용하는 것은 아닙니다.

Control Plane 노드에서도 로그 수집이나 모니터링 에이전트를 실행해야 한다면 해당 노드의 Taint를 확인하고 필요한 Toleration을 추가해야 합니다.

예를 들어 다음과 같이 구성할 수 있습니다.

```yaml
spec:
  template:
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
```

이를 통해 Control Plane 노드에 해당 Taint가 설정되어 있더라도 DaemonSet Pod가 배치될 수 있도록 허용합니다.

## 11. kubectl 명령어

### 11.1. DaemonSet 조회

```bash
# 현재 Namespace의 DaemonSet 조회
kubectl get daemonsets

# 모든 Namespace의 DaemonSet 조회
kubectl get daemonsets -A

# 특정 DaemonSet 상세 조회
kubectl describe daemonset fluent-bit

# DaemonSet의 YAML 확인
kubectl get daemonset fluent-bit -o yaml
```

### 11.2. DaemonSet 생성 및 삭제

```bash
# YAML Manifest 적용
kubectl apply -f daemonset.yaml

# DaemonSet 삭제
kubectl delete daemonset fluent-bit
```

DaemonSet을 삭제하면 해당 DaemonSet이 관리하던 Pod도 함께 정리됩니다.

### 11.3. DaemonSet의 Pod 확인

```bash
# DaemonSet에 의해 생성된 Pod 조회
kubectl get pods -l app=fluent-bit

# Pod가 배치된 노드 확인
kubectl get pods -l app=fluent-bit -o wide
```

### 11.4. DaemonSet 업데이트

```bash
# 컨테이너 이미지 변경
kubectl set image daemonset/fluent-bit \
  fluent-bit=fluent/fluent-bit:3.2
```

```bash
# 업데이트 진행 상태 확인
kubectl rollout status daemonset/fluent-bit

# 업데이트 이력 확인
kubectl rollout history daemonset/fluent-bit

# 이전 버전으로 롤백
kubectl rollout undo daemonset/fluent-bit
```

### 11.5. DaemonSet 배치 상태 확인

DaemonSet은 Deployment처럼 `replicas` 값을 지정하여 스케일링하지 않습니다. 대상 노드의 수와 배치 조건에 따라 Pod 개수가 결정됩니다.

다음 명령어로 DaemonSet의 배치 상태를 확인할 수 있습니다.

```bash
kubectl get daemonset fluent-bit
```

출력 예시는 다음과 같습니다.

```text
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR
fluent-bit   3         3         3       3            3           <none>
```

| 항목 | 설명 |
| --- | --- |
| `DESIRED` | DaemonSet이 Pod를 실행해야 하는 대상 노드 수입니다. |
| `CURRENT` | 현재 생성된 DaemonSet Pod 수입니다. |
| `READY` | Ready 상태인 Pod 수입니다. |
| `UP-TO-DATE` | 최신 Pod Template을 사용하는 Pod 수입니다. |
| `AVAILABLE` | 사용 가능한 상태인 Pod 수입니다. |
| `NODE SELECTOR` | Pod 배치 대상 노드를 선택하는 조건입니다. |

`DESIRED`, `CURRENT`, `READY` 등의 값은 DaemonSet Controller가 파악한 배치 및 실행 상태를 나타냅니다. 따라서 값이 일치하지 않는다면 Pod의 스케줄링 상태, 노드 조건 및 컨테이너 실행 상태를 추가로 확인해야 합니다.

## 12. 핵심 정리

DaemonSet은 Kubernetes 클러스터의 모든 노드 또는 지정된 일부 노드에서 Pod가 하나씩 실행되도록 관리하는 Workload 리소스입니다.

- 대상 노드마다 하나의 Pod가 실행되도록 관리합니다.
- 새로운 노드가 추가되면 배치 조건에 따라 Pod를 자동으로 생성합니다.
- 노드가 제거되면 해당 노드는 관리 대상에서 제외됩니다.
- Node Selector, Node Affinity, Taint 및 Toleration 등을 활용하여 배치 대상 노드를 제어할 수 있습니다.
- DaemonSet Controller는 대상 노드에 대한 Node Affinity를 설정하고, 일반적으로 Kubernetes Scheduler가 해당 노드에 Pod를 배치합니다.
- DaemonSet Controller는 특정 노드 상태와 관련된 기본 Toleration을 Pod에 자동으로 추가합니다.
- 기본적으로 RollingUpdate 전략을 사용하여 Pod를 순차적으로 업데이트합니다.
- `maxUnavailable`과 `maxSurge`를 통해 RollingUpdate의 동작을 제어할 수 있습니다.
- OnDelete 전략을 사용하면 운영자가 직접 Pod를 삭제하여 업데이트 시점을 제어할 수 있습니다.
- Deployment와 달리 `replicas` 값으로 Pod 개수를 직접 조절하지 않습니다.
- 로그 수집, 모니터링, 네트워크 및 스토리지 에이전트처럼 노드 단위로 실행되어야 하는 애플리케이션에 적합합니다.

**DaemonSet은 애플리케이션의 복제본 개수가 아니라 클러스터의 노드 구성을 기준으로 Pod를 배치하고 관리하는 Workload 리소스입니다.**

Deployment가 애플리케이션의 실행 인스턴스 수를 유지하는 데 적합하다면, DaemonSet은 각 노드에서 실행되어야 하는 공통 에이전트나 인프라 구성 요소를 배포하는 데 적합합니다.

## 13. 참고자료

- [Kubernetes 공식 문서 - DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
- [Kubernetes 공식 문서 - Perform a Rolling Update on a DaemonSet](https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/)
- [Kubernetes 공식 문서 - DaemonSet API Reference](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/daemon-set-v1/)
- [Kubernetes 공식 문서 - Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Kubernetes 공식 문서 - Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)