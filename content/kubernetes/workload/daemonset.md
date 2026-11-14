# DaemonSet

## 1. DaemonSet이란?

DaemonSet은 Kubernetes 클러스터의 모든 노드 또는 지정된 일부 노드에서 Pod가 하나씩 실행되도록 관리하는 Workload 리소스입니다.

일반적인 Deployment는 지정된 개수의 Pod를 클러스터 전체에 배포합니다. 이때 Pod가 어떤 노드에 배치되는지는 Kubernetes Scheduler가 결정합니다.

반면 DaemonSet은 노드마다 하나의 Pod가 실행되도록 관리합니다. 새로운 노드가 클러스터에 추가되면 해당 노드에도 Pod를 자동으로 생성하고, 노드가 클러스터에서 제거되면 해당 노드에서 실행되던 Pod도 정리됩니다.

DaemonSet은 주로 노드 단위로 실행되어야 하는 인프라 구성 요소에 사용됩니다.

- 로그 수집 에이전트
- 모니터링 에이전트
- 노드 네트워크 플러그인
- 스토리지 관련 데몬
- 보안 및 시스템 관리 에이전트

대표적인 사용 사례는 다음과 같습니다.

| 사용 사례 | 설명                                              |
| ----- | ----------------------------------------------- |
| 로그 수집 | 각 노드에서 컨테이너 로그를 수집하는 Fluent Bit 등의 에이전트를 실행합니다. |
| 모니터링  | 각 노드의 CPU, 메모리, 디스크 등의 상태를 수집하는 에이전트를 실행합니다.    |
| 네트워크  | 각 노드에서 네트워크 설정이나 패킷 처리를 담당하는 구성 요소를 실행합니다.      |
| 스토리지  | 각 노드에서 스토리지 연결 및 관리 작업을 수행하는 구성 요소를 실행합니다.      |
| 보안    | 각 노드에서 시스템 상태를 감시하거나 보안 이벤트를 수집하는 에이전트를 실행합니다.  |

DaemonSet의 핵심 목적은 클러스터의 노드마다 필요한 Pod를 자동으로 배치하고 유지하는 것입니다.

## 2. DaemonSet의 구조

DaemonSet은 Pod Template을 기반으로 대상 노드마다 Pod를 생성하고 관리합니다.

예를 들어 클러스터에 Worker Node가 3개 존재하고 모든 노드를 대상으로 DaemonSet을 생성하면 각 노드에 하나씩 Pod가 배치됩니다.

```
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

따라서 노드가 추가되거나 제거되면 DaemonSet Controller가 이를 감지하고 Pod 구성을 조정합니다.

## 3. DaemonSet YAML 구성

다음은 클러스터의 각 노드에서 Nginx 컨테이너를 실행하는 DaemonSet 예제입니다.

```
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

| 필드           | 설명                                                     |
| ------------ | ------------------------------------------------------ |
| `apiVersion` | Kubernetes API 버전을 지정합니다. DaemonSet은 `apps/v1`을 사용합니다. |
| `kind`       | 생성할 리소스의 종류를 지정합니다.                                    |
| `metadata`   | DaemonSet의 이름과 메타데이터를 정의합니다.                           |
| `spec`       | DaemonSet의 원하는 상태를 정의합니다.                              |
| `selector`   | DaemonSet이 관리할 Pod를 식별하는 조건을 지정합니다.                    |
| `template`   | 각 노드에 생성할 Pod의 구성을 정의합니다.                              |

### 3.1. selector

`selector`는 DaemonSet이 관리할 Pod를 식별하는 조건입니다.

```
spec:
  selector:
    matchLabels:
      app: nginx
```

Selector는 Pod Template에 정의된 레이블과 일치해야 합니다.

```
spec:
  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx
```

`apps/v1` API에서 Selector는 필수이며, DaemonSet 생성 이후 변경할 수 없습니다.

### 3.2. template

`template`은 DaemonSet이 각 대상 노드에 생성할 Pod의 구성을 정의합니다.

```
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

## 4. DaemonSet의 주요 특징

### 4.1. 노드마다 하나의 Pod 실행

DaemonSet의 가장 중요한 특징은 대상 노드마다 하나의 Pod를 실행한다는 것입니다.

예를 들어 클러스터에 Worker Node가 3개 존재한다면 다음과 같이 Pod가 배치됩니다.

```
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

### 4.2. 새로운 노드에 자동 배치

클러스터에 새로운 노드가 추가되면 DaemonSet Controller는 해당 노드가 DaemonSet의 배치 조건에 부합하는지 확인합니다.

조건에 부합한다면 해당 노드에 새로운 Pod를 생성합니다.

```
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

DaemonSet의 대상 노드가 클러스터에서 제거되면 해당 노드에서 실행되던 DaemonSet Pod도 더 이상 유지할 필요가 없습니다.

DaemonSet Controller는 대상 노드의 변경 사항을 반영하여 관리 대상 Pod를 조정합니다.

다만 노드가 비정상적으로 연결이 끊어진 경우에는 노드 상태와 Pod의 종료 처리 과정에 따라 정리 시점이 달라질 수 있습니다.

### 4.4. 노드 선택 조건

DaemonSet은 모든 노드에 Pod를 배치할 수도 있지만, 특정 조건에 해당하는 노드에만 Pod를 배치하도록 구성할 수도 있습니다.

대표적으로 다음과 같은 설정을 사용할 수 있습니다.

- `nodeSelector`
- Node Affinity
- Taint와 Toleration

예를 들어 특정 노드에 `disk=ssd` 레이블이 지정되어 있다면 해당 노드에만 DaemonSet Pod를 배치할 수 있습니다.

```
spec:
  template:
    spec:
      nodeSelector:
        disk: ssd

      containers:
        - name: agent
          image: nginx:1.27
```

위 설정은 다음과 같은 노드 중 `disk=ssd` 레이블을 가진 노드에만 Pod를 배치합니다.

```
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

## 5. DaemonSet과 Taint 및 Toleration

Kubernetes에서는 특정 노드에 일반적인 Pod가 배치되지 않도록 Taint를 설정할 수 있습니다.

Taint가 설정된 노드에는 해당 Taint를 허용하는 Toleration이 없는 Pod가 스케줄링되지 않습니다.

DaemonSet은 이러한 노드에도 필요한 Pod를 배치할 수 있도록 기본적인 Toleration을 자동으로 추가합니다.

대표적인 기본 Toleration은 다음과 같습니다.

| Taint                                | 설명                     |
| ------------------------------------ | ---------------------- |
| `node.kubernetes.io/not-ready`       | 노드가 Ready 상태가 아닌 경우    |
| `node.kubernetes.io/unreachable`     | 노드에 접근할 수 없는 경우        |
| `node.kubernetes.io/disk-pressure`   | 노드의 디스크 공간이 부족한 경우     |
| `node.kubernetes.io/memory-pressure` | 노드의 메모리 압박 상태가 발생한 경우  |
| `node.kubernetes.io/pid-pressure`    | 노드의 프로세스 ID 자원이 부족한 경우 |
| `node.kubernetes.io/unschedulable`   | 노드가 스케줄링 불가능한 상태인 경우   |

이러한 기본 Toleration은 DaemonSet Pod가 노드의 상태에 관계없이 무조건 실행된다는 의미는 아닙니다.

노드의 상태, Pod의 리소스 요구사항, 다른 스케줄링 제약 조건 등에 따라 Pod가 정상적으로 실행되지 못할 수 있습니다.

### 5.1. 특정 Taint를 허용하는 설정

DaemonSet의 Pod Template에 Toleration을 직접 추가할 수도 있습니다.

```
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

```
kubectl taint nodes worker-1 dedicated=monitoring:NoSchedule
```

이를 통해 모니터링 전용 노드처럼 특정 용도로 분리된 노드에도 DaemonSet Pod를 배치할 수 있습니다.

## 6. DaemonSet의 Pod 배치 과정

DaemonSet은 DaemonSet Controller가 대상 노드를 확인하고 각 노드에 필요한 Pod를 생성하는 방식으로 동작합니다.

일반적인 Deployment와 달리 DaemonSet Controller는 각 노드에 배치할 Pod를 직접 관리합니다.

대표적인 동작 과정은 다음과 같습니다.

```
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
각 대상 노드에
Pod 생성 요청
      │
      ▼
Pod가 지정된 노드에서 실행
      │
      ▼
노드 변경 사항 감지
      │
      ▼
필요한 Pod 생성 및 정리
```

DaemonSet은 대상 노드에 Pod가 배치되도록 `spec.nodeName`을 설정하는 방식으로 동작할 수 있습니다.

이러한 방식은 일반적인 Pod와 달리 DaemonSet Pod가 일반적인 Scheduler의 노드 선택 과정에 의존하지 않도록 합니다.

다만 Pod의 실제 실행 가능 여부는 kubelet, 컨테이너 런타임, 노드 상태 및 리소스 상황 등에 영향을 받습니다.

## 7. DaemonSet의 업데이트 전략

DaemonSet은 Pod Template이 변경되었을 때 기존 Pod를 새로운 구성으로 교체할 수 있습니다.

대표적인 업데이트 전략은 다음과 같습니다.

- RollingUpdate
- OnDelete

### 7.1. RollingUpdate

RollingUpdate는 DaemonSet의 기본 업데이트 전략입니다.

Pod Template이 변경되면 기존 Pod를 새로운 구성으로 순차적으로 교체합니다.

```
spec:
  updateStrategy:
    type: RollingUpdate
```

예를 들어 컨테이너 이미지를 `nginx:1.27`에서 `nginx:1.28`로 변경하면 각 노드에서 실행 중인 Pod가 새로운 이미지로 교체됩니다.

```
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

```
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 0
```

| 필드               | 설명                                                        |
| ---------------- | --------------------------------------------------------- |
| `maxUnavailable` | 업데이트 과정에서 사용할 수 없는 상태가 허용되는 Pod의 최대 개수를 지정합니다.            |
| `maxSurge`       | 업데이트 과정에서 원하는 Pod 수를 초과하여 추가로 생성할 수 있는 Pod의 최대 개수를 지정합니다. |

DaemonSet의 RollingUpdate에서는 `maxUnavailable`과 `maxSurge`를 사용할 수 있습니다.

`maxSurge`를 사용하면 기존 Pod를 종료하기 전에 새로운 Pod를 추가로 생성하여 교체할 수 있습니다. 다만 동일 노드에 두 Pod가 동시에 실행될 수 있는지, 포트 충돌이나 리소스 부족이 발생하지 않는지 고려해야 합니다.

### 7.2. OnDelete

OnDelete는 Pod Template이 변경되더라도 기존 Pod를 자동으로 교체하지 않는 업데이트 전략입니다.

```
spec:
  updateStrategy:
    type: OnDelete
```

기존 Pod를 직접 삭제해야 새로운 Template을 적용한 Pod가 생성됩니다.

예를 들어 다음과 같이 동작합니다.

```
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

| 구분       | RollingUpdate                | OnDelete                 |
| -------- | ---------------------------- | ------------------------ |
| 업데이트 방식  | 기존 Pod를 자동으로 교체합니다.          | Pod를 직접 삭제해야 교체됩니다.      |
| 업데이트 시점  | DaemonSet Controller가 관리합니다. | 운영자가 직접 제어합니다.           |
| 주요 사용 사례 | 일반적인 에이전트 업데이트               | 업데이트 시점을 수동으로 제어해야 하는 경우 |

## 8. DaemonSet과 Deployment의 차이

DaemonSet과 Deployment는 모두 Pod를 생성하고 관리하는 Workload 리소스입니다.

하지만 Pod를 배치하는 기준에 차이가 있습니다.

Deployment는 지정된 복제본 개수를 유지하는 데 집중하며, Pod가 어떤 노드에 배치되는지는 Scheduler가 결정합니다.

반면 DaemonSet은 대상 노드마다 하나의 Pod가 실행되도록 관리합니다.

| 구분        | Deployment              | DaemonSet               |
| --------- | ----------------------- | ----------------------- |
| 주요 목적     | 애플리케이션의 복제본 관리          | 노드별 Pod 배치 및 관리         |
| Pod 개수    | `replicas`로 지정          | 대상 노드 수와 배치 조건에 따라 결정   |
| Pod 배치    | Scheduler가 배치 노드를 결정    | 대상 노드마다 Pod를 생성하도록 관리   |
| 신규 노드 추가  | 기존 복제본 수를 유지하며 배치 조정    | 조건에 부합하는 신규 노드에 Pod 추가  |
| 노드별 Pod 수 | 특정 노드에 제한 없이 배치될 수 있음   | 대상 노드마다 하나의 Pod 유지      |
| 업데이트      | RollingUpdate, Recreate | RollingUpdate, OnDelete |
| 주요 사용 사례  | 웹 애플리케이션, API 서버        | 로그 수집, 모니터링, 네트워크 에이전트  |

### 8.1. Deployment의 Pod 배치

Deployment는 `replicas`에 지정된 개수만큼 Pod를 유지합니다.

```
spec:
  replicas: 3
```

3개의 Pod가 생성되더라도 반드시 서로 다른 노드에 하나씩 배치되는 것은 아닙니다.

```
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

```
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

따라서 모든 노드에서 동일한 에이전트가 실행되어야 하는 경우 DaemonSet이 적합합니다.

## 9. DaemonSet과 다른 Workload 리소스의 관계

Kubernetes의 Workload 리소스는 애플리케이션의 실행 목적에 따라 구분됩니다.

| 리소스         | 주요 역할                     | 주요 사용 사례         |
| ----------- | ------------------------- | ---------------- |
| Pod         | 컨테이너 실행 단위                | 단일 애플리케이션 실행     |
| Deployment  | Stateless 애플리케이션 배포 및 관리  | 웹 애플리케이션, API 서버 |
| ReplicaSet  | 지정된 Pod 복제본 개수 유지         | Pod 복제본 관리       |
| StatefulSet | 안정적인 식별자와 스토리지를 가진 Pod 관리 | 데이터베이스, 분산 시스템   |
| DaemonSet   | 대상 노드마다 Pod 배치            | 로그 수집, 모니터링 에이전트 |
| Job         | 작업이 완료될 때까지 Pod 실행        | 배치 작업            |
| CronJob     | 일정에 따라 Job 실행             | 정기적인 배치 작업       |

DaemonSet은 Deployment처럼 애플리케이션의 복제본 개수를 기준으로 관리하지 않습니다.

대신 클러스터의 노드 구성을 기준으로 Pod를 배치한다는 점에서 차이가 있습니다.

## 10. DaemonSet 사용 시 고려사항

### 10.1. 모든 노드에 반드시 배치되는 것은 아닙니다.

DaemonSet은 기본적으로 모든 노드에 Pod를 배치할 수 있지만, 실제 배치 대상은 Node Selector, Node Affinity, Taint 및 Toleration 등의 설정에 따라 달라집니다.

또한 Pod의 리소스 요구사항이나 노드 상태 등에 따라 Pod가 정상적으로 실행되지 못할 수 있습니다.

따라서 DaemonSet을 생성했다고 해서 모든 노드에서 Pod가 반드시 실행되는 것은 아닙니다.

### 10.2. DaemonSet Pod의 리소스 사용량을 고려해야 합니다.

DaemonSet은 대상 노드마다 Pod를 실행하므로 노드 수가 증가하면 전체 Pod 개수와 리소스 사용량도 증가합니다.

예를 들어 100개의 노드에서 CPU 100m, 메모리 128Mi를 사용하는 모니터링 에이전트를 실행한다면 전체적으로 다음과 같은 리소스가 필요합니다.

- CPU 요청량: 10 CPU
- 메모리 요청량: 12.5Gi

따라서 DaemonSet에 배포하는 에이전트의 CPU 및 메모리 요청량과 제한값을 적절하게 설정해야 합니다.

### 10.3. HostPath 사용 시 보안에 주의해야 합니다.

로그 수집 에이전트나 모니터링 에이전트는 노드의 파일 시스템에 접근해야 하는 경우가 있습니다.

이때 `hostPath` Volume을 사용하여 노드의 디렉터리를 Pod에 마운트할 수 있습니다.

```
volumes:
  - name: varlog
    hostPath:
      path: /var/log
      type: Directory
```

하지만 HostPath를 사용하면 컨테이너가 노드의 파일 시스템에 접근할 수 있으므로 보안 위험이 발생할 수 있습니다.

필요한 경로에만 접근하도록 제한하고, 컨테이너의 권한과 보안 설정을 함께 검토해야 합니다.

### 10.4. 업데이트 과정에서 서비스 중단 가능성을 고려해야 합니다.

DaemonSet은 노드마다 하나의 Pod를 실행하는 구조이므로 RollingUpdate 과정에서 기존 Pod가 종료되고 새로운 Pod가 준비되는 동안 해당 노드의 에이전트가 일시적으로 실행되지 않을 수 있습니다.

특히 로그 수집이나 네트워크 관련 에이전트는 업데이트 중 발생할 수 있는 데이터 수집 공백이나 통신 영향을 고려해야 합니다.

필요하다면 `maxUnavailable`, `maxSurge`, Readiness Probe 등을 적절하게 설정해야 합니다.

### 10.5. DaemonSet은 애플리케이션의 고가용성을 자동으로 보장하지 않습니다.

DaemonSet은 노드마다 Pod를 배치하는 기능을 제공하지만, 해당 애플리케이션 자체의 고가용성이나 데이터 복제 기능을 제공하지는 않습니다.

예를 들어 DaemonSet으로 각 노드에 데이터베이스를 하나씩 실행하더라도 데이터베이스 클러스터가 자동으로 구성되는 것은 아닙니다.

애플리케이션의 복제, 장애 조치, 데이터 동기화 등은 별도의 구성이 필요합니다.

## 11. kubectl 명령어

### 11.1. DaemonSet 조회

```
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

```
# YAML Manifest 적용
kubectl apply -f daemonset.yaml

# DaemonSet 삭제
kubectl delete daemonset fluent-bit
```

### 11.3. DaemonSet의 Pod 확인

```
# DaemonSet에 의해 생성된 Pod 조회
kubectl get pods -l app=fluent-bit

# Pod가 배치된 노드 확인
kubectl get pods -l app=fluent-bit -o wide
```

### 11.4. DaemonSet 업데이트

```
# 컨테이너 이미지 변경
kubectl set image daemonset/fluent-bit \
  fluent-bit=fluent/fluent-bit:3.2
```

```
# 업데이트 진행 상태 확인
kubectl rollout status daemonset/fluent-bit

# 업데이트 이력 확인
kubectl rollout history daemonset/fluent-bit

# 이전 버전으로 롤백
kubectl rollout undo daemonset/fluent-bit
```

### 11.5. DaemonSet 스케일 및 배치 상태 확인

DaemonSet은 Deployment처럼 `replicas` 값을 지정하여 스케일링하지 않습니다. 대상 노드의 수와 배치 조건에 따라 Pod 개수가 결정됩니다.

다음 명령어로 DaemonSet의 배치 상태를 확인할 수 있습니다.

```
kubectl get daemonset fluent-bit
```

출력 예시는 다음과 같습니다.

```
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR
fluent-bit   3         3         3       3            3           <none>
```

| 항목              | 설명                                  |
| --------------- | ----------------------------------- |
| `DESIRED`       | DaemonSet이 Pod를 실행해야 하는 대상 노드 수입니다. |
| `CURRENT`       | 현재 생성된 DaemonSet Pod 수입니다.          |
| `READY`         | Ready 상태인 Pod 수입니다.                 |
| `UP-TO-DATE`    | 최신 Pod Template을 사용하는 Pod 수입니다.     |
| `AVAILABLE`     | 사용 가능한 상태인 Pod 수입니다.                |
| `NODE SELECTOR` | Pod 배치 대상 노드를 선택하는 조건입니다.           |

## 12. 핵심 정리

DaemonSet은 Kubernetes 클러스터의 모든 노드 또는 지정된 일부 노드에서 Pod가 하나씩 실행되도록 관리하는 Workload 리소스입니다.

- 대상 노드마다 하나의 Pod가 실행되도록 관리합니다.
- 새로운 노드가 추가되면 배치 조건에 따라 Pod를 자동으로 생성합니다.
- 노드가 제거되면 해당 노드에서 실행되던 Pod도 관리 대상에서 제외됩니다.
- Node Selector, Node Affinity, Taint 및 Toleration 등을 활용하여 배치 대상 노드를 제어할 수 있습니다.
- 기본적으로 RollingUpdate 전략을 사용하여 Pod를 순차적으로 업데이트합니다.
- OnDelete 전략을 사용하면 운영자가 직접 Pod를 삭제하여 업데이트 시점을 제어할 수 있습니다.
- Deployment와 달리 `replicas` 값으로 Pod 개수를 직접 조절하지 않습니다.
- 로그 수집, 모니터링, 네트워크 및 스토리지 에이전트처럼 노드 단위로 실행되어야 하는 애플리케이션에 적합합니다.

DaemonSet은 애플리케이션의 복제본 개수가 아니라 클러스터의 노드 구성을 기준으로 Pod를 배치하고 관리하는 Workload 리소스입니다.

Deployment가 애플리케이션의 실행 인스턴스 수를 유지하는 데 적합하다면, DaemonSet은 각 노드에서 반드시 실행되어야 하는 공통 에이전트나 인프라 구성 요소를 배포하는 데 적합합니다.