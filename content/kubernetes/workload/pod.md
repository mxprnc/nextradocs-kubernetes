# Pod

## 1. Pod란?

Pod는 Kubernetes에서 생성하고 관리할 수 있는 가장 작은 배포 단위(Deployable Unit)입니다.

Kubernetes에서 컨테이너는 독립적인 관리 단위가 아니라 Pod 내부에서 실행됩니다. 하나의 Pod에는 하나 이상의 컨테이너가 포함될 수 있으며, 동일한 Pod에 속한 컨테이너들은 네트워크와 스토리지 등의 실행 환경을 공유합니다.

Pod는 일반적으로 하나의 애플리케이션 인스턴스를 실행하는 데 사용됩니다. 예를 들어 웹 애플리케이션을 실행하는 경우 하나의 Pod에 애플리케이션 컨테이너를 배치할 수 있습니다.

하나의 Pod에 여러 컨테이너를 배치하는 경우에는 애플리케이션 컨테이너와 긴밀하게 연관된 보조 컨테이너를 함께 실행할 수 있습니다. 대표적인 사례로 로그 수집, 프록시, 데이터 동기화 등을 담당하는 사이드카(Sidecar) 컨테이너가 있습니다.

Pod의 주요 특징은 다음과 같습니다.

- 하나 이상의 컨테이너를 포함할 수 있습니다.
- 동일한 Pod에 속한 컨테이너들은 동일한 네트워크 네임스페이스를 공유합니다.
- 동일한 Pod에 속한 컨테이너들은 볼륨을 공유할 수 있습니다.
- Pod 내부 컨테이너들은 동일한 IP 주소와 포트 공간을 사용합니다.
- Pod는 자체적인 생명주기를 가지며, 일반적으로 한 번 생성된 Pod를 다른 노드로 이동시키지 않습니다.
- Pod가 삭제되면 해당 Pod에 속한 컨테이너도 함께 종료됩니다.

## 2. Pod의 구조

Pod는 하나 이상의 컨테이너와 이들이 공유하는 실행 환경으로 구성됩니다.

일반적인 Pod는 다음과 같은 구조를 가집니다.

```
Pod
├── Shared Network Namespace
│   ├── IP Address
│   └── Port Space
│
├── Shared Storage
│   └── Volumes
│
├── Container A
├── Container B
└── Container C
```

동일한 Pod에 속한 컨테이너들은 다음과 같은 환경을 공유합니다.

### 2.1. 네트워크 공유

동일한 Pod에 속한 컨테이너들은 하나의 네트워크 네임스페이스를 공유합니다.

따라서 Pod 내부의 모든 컨테이너는 동일한 IP 주소를 사용하며, `localhost`를 통해 서로 통신할 수 있습니다.

예를 들어 하나의 Pod에 애플리케이션 컨테이너와 프록시 컨테이너가 함께 실행되는 경우 다음과 같이 통신할 수 있습니다.

```
Pod IP: 10.0.1.10

┌──────────────────────────────┐
│ Pod                          │
│                              │
│  ┌────────────────────────┐  │
│  │ Proxy Container        │  │
│  │ Port: 8080             │  │
│  └───────────┬────────────┘  │
│              │               │
│              │ localhost     │
│              ▼               │
│  ┌────────────────────────┐  │
│  │ Application Container │  │
│  │ Port: 3000             │  │
│  └────────────────────────┘  │
│                              │
└──────────────────────────────┘
```

컨테이너 간 통신은 별도의 Pod IP를 사용하지 않고 `localhost`를 통해 이루어집니다.

단, 동일한 네트워크 네임스페이스를 공유하므로 컨테이너 간 포트 충돌에 주의해야 합니다.

### 2.2. 스토리지 공유

동일한 Pod에 속한 컨테이너들은 Pod에 정의된 볼륨(Volume)을 공유할 수 있습니다.

각 컨테이너는 동일한 볼륨을 서로 다른 경로에 마운트할 수 있으며, 이를 통해 컨테이너 간 파일을 공유할 수 있습니다.

예를 들어 애플리케이션 컨테이너가 생성한 로그 파일을 로그 수집 컨테이너가 읽도록 구성할 수 있습니다.

```
Pod
│
├── Shared Volume: log-volume
│
├── Application Container
│   └── /var/log/app
│       └── application.log
│
└── Log Collector Container
    └── /var/log/collector
        └── application.log
```

두 컨테이너는 동일한 볼륨을 서로 다른 경로에 마운트하지만, 실제로는 동일한 파일 데이터를 공유합니다.

### 2.3. 생명주기 공유

Pod는 하나의 생명주기를 가지며, Pod 내부 컨테이너들은 해당 Pod의 실행 환경에 종속됩니다.

Pod가 종료되거나 삭제되면 내부 컨테이너도 함께 종료됩니다.

다만 Pod 내부의 개별 컨테이너는 독립적으로 재시작될 수 있습니다. 컨테이너 재시작 여부는 Pod에 설정된 `restartPolicy`에 따라 결정됩니다.

또한 Pod 자체가 삭제된 후 새로운 Pod가 생성되는 경우에는 기존 Pod와 별개의 객체로 취급됩니다.

## 3. Pod와 컨테이너의 관계

Pod와 컨테이너는 서로 다른 관리 단위입니다.

컨테이너는 애플리케이션을 실행하는 환경이며, Pod는 하나 이상의 컨테이너를 묶어 Kubernetes가 관리할 수 있도록 하는 실행 단위입니다.

| 구분    | Pod              | Container           |
| ----- | ---------------- | ------------------- |
| 역할    | 컨테이너를 관리하는 실행 단위 | 애플리케이션 실행 단위        |
| 개수    | 하나 이상의 컨테이너 포함   | Pod 내부에 포함          |
| IP 주소 | Pod 단위로 할당       | 독립적인 Pod IP 없음      |
| 네트워크  | 네트워크 네임스페이스 공유   | Pod의 네트워크 공유        |
| 스토리지  | 볼륨 정의 및 공유       | 볼륨 마운트              |
| 생명주기  | Pod 자체의 생명주기 관리  | 개별 컨테이너 상태 및 재시작 관리 |

Kubernetes에서는 일반적으로 컨테이너를 직접 배포하기보다는 Pod를 정의하여 배포합니다.

또한 실제 운영 환경에서는 Pod를 직접 생성하기보다는 Deployment, StatefulSet, DaemonSet 등의 Workload 리소스를 통해 관리하는 방식을 권장합니다.

## 4. Pod의 종류

Pod는 실행 목적과 구성 방식에 따라 다양한 형태로 사용됩니다.

### 4.1. 단일 컨테이너 Pod (Single Container Pod)

하나의 Pod에 하나의 컨테이너만 포함하는 가장 일반적인 형태입니다.

대부분의 애플리케이션은 단일 컨테이너 Pod로 실행할 수 있습니다.

```
Pod
└── Application Container
```

애플리케이션의 실행과 관리를 단순하게 유지할 수 있으며, Deployment와 같은 상위 Workload 리소스를 통해 여러 Pod를 생성하고 관리할 수 있습니다.

### 4.2. 다중 컨테이너 Pod (Multi-Container Pod)

하나의 Pod에 여러 컨테이너를 포함하는 형태입니다.

동일한 생명주기를 가지면서 네트워크나 스토리지를 공유해야 하는 컨테이너들을 함께 배치할 때 사용합니다.

```
Pod
├── Application Container
├── Sidecar Container
└── Log Collector Container
```

대표적인 사용 사례는 다음과 같습니다.

- Sidecar: 애플리케이션을 보조하는 프록시, 로그 수집기 등을 실행합니다.
- Ambassador: 외부 서비스와의 통신을 중계하거나 프록시 역할을 수행합니다.
- Adapter: 애플리케이션의 출력 데이터를 다른 시스템에서 사용할 수 있는 형식으로 변환합니다.

다중 컨테이너 Pod는 컨테이너 간 밀접한 결합이 필요한 경우에 적합합니다. 서로 독립적으로 배포하거나 확장해야 하는 애플리케이션을 하나의 Pod에 배치하는 것은 적절하지 않습니다.

## 5. Pod의 생명주기 (Pod Lifecycle)

Pod는 생성부터 종료까지 고유한 생명주기를 가집니다. Kubernetes는 Pod의 상태를 지속적으로 관찰하며, 현재 상태를 기반으로 필요한 작업을 수행합니다.

Pod의 생명주기는 크게 다음과 같은 단계로 구분됩니다.

### 5.1. Pod Phase

Pod의 Phase는 Pod가 현재 어느 단계에 있는지를 나타내는 상위 수준의 상태입니다.

| Phase     | 설명                                                                                               |
| --------- | ------------------------------------------------------------------------------------------------ |
| Pending   | Pod가 Kubernetes에 생성되었지만, 컨테이너 실행이 아직 시작되지 않은 상태입니다. 노드 스케줄링, 이미지 다운로드, 볼륨 마운트 등의 작업이 진행될 수 있습니다. |
| Running   | Pod가 노드에 바인딩되었으며, 모든 컨테이너가 생성된 상태입니다. 하나 이상의 컨테이너가 실행 중이거나 시작 또는 재시작 중일 수 있습니다.                  |
| Succeeded | Pod에 포함된 모든 컨테이너가 성공적으로 종료된 상태입니다.                                                               |
| Failed    | Pod에 포함된 모든 컨테이너가 종료되었으며, 하나 이상의 컨테이너가 실패한 상태입니다.                                                |
| Unknown   | Pod의 상태를 확인할 수 없는 상태입니다. 일반적으로 노드와의 통신 문제 등으로 발생할 수 있습니다.                                        |

일반적인 Pod의 상태 변화는 다음과 같습니다.

```
         Pod 생성
            │
            ▼
         Pending
            │
            ▼
         Running
            │
      ┌─────┴─────┐
      │           │
      ▼           ▼
  Succeeded     Failed
```

`Succeeded`와 `Failed`는 일반적으로 종료된 Pod에서 나타나는 최종 상태입니다.

주의할 점은 Pod Phase와 컨테이너 상태(Container State)는 서로 다른 개념이라는 것입니다. Pod가 `Running` 상태이더라도 내부 컨테이너가 재시작 중이거나 준비되지 않은 상태일 수 있습니다.

### 5.2. Container State

Pod 내부의 각 컨테이너는 독립적인 상태를 가집니다.

컨테이너 상태는 다음 세 가지로 구분됩니다.

| 상태         | 설명                                                              |
| ---------- | --------------------------------------------------------------- |
| Waiting    | 컨테이너가 실행을 시작하기 전 대기 중인 상태입니다. 이미지 다운로드나 초기화 작업 등이 원인이 될 수 있습니다. |
| Running    | 컨테이너가 정상적으로 실행 중인 상태입니다.                                        |
| Terminated | 컨테이너의 실행이 종료된 상태입니다. 정상 종료 또는 오류로 인한 종료가 포함됩니다.                 |

컨테이너가 `Terminated` 상태가 되더라도 Pod의 `restartPolicy`에 따라 컨테이너가 다시 시작될 수 있습니다.

예를 들어 애플리케이션 컨테이너가 오류로 종료되었을 때 재시작 정책이 `Always`라면 Kubernetes는 해당 컨테이너를 다시 실행합니다.

### 5.3. Pod Conditions

Pod Conditions는 Pod가 현재 어떤 조건을 충족하고 있는지를 나타냅니다.

Pod Phase가 전체적인 진행 단계를 나타낸다면, Pod Conditions는 Pod의 세부적인 상태와 준비 여부를 표현합니다.

대표적인 Condition은 다음과 같습니다.

| Condition                 | 설명                                                   |
| ------------------------- | ---------------------------------------------------- |
| PodScheduled              | Pod가 실행될 노드에 정상적으로 스케줄링되었는지를 나타냅니다.                  |
| PodReadyToStartContainers | Pod의 샌드박스 생성 및 네트워크 설정이 완료되어 컨테이너를 시작할 수 있는지를 나타냅니다. |
| Initialized               | 모든 Init Container가 성공적으로 완료되었는지를 나타냅니다.              |
| ContainersReady           | Pod 내부의 모든 컨테이너가 Ready 상태인지를 나타냅니다.                  |
| Ready                     | Pod가 서비스 트래픽을 처리할 준비가 되었는지를 나타냅니다.                   |

`Ready` Condition은 Service와 같은 다른 Kubernetes 리소스가 Pod를 트래픽 전달 대상으로 사용할 수 있는지를 판단하는 데 중요한 역할을 합니다.

예를 들어 Pod가 `Running` 상태이더라도 Readiness Probe가 실패하면 `Ready` Condition은 `False`가 될 수 있습니다.

이 경우 해당 Pod는 일반적으로 Service의 정상적인 트래픽 전달 대상에서 제외됩니다.

## 6. Pod의 주요 설정

Pod는 YAML Manifest를 통해 정의할 수 있습니다.

다음은 Pod를 생성하는 기본적인 YAML 예제입니다.

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

각 필드의 주요 역할은 다음과 같습니다.

| 필드         | 설명                                          |
| ---------- | ------------------------------------------- |
| apiVersion | Kubernetes API 버전을 지정합니다. Pod는 `v1`을 사용합니다. |
| kind       | 생성할 리소스의 종류를 지정합니다.                         |
| metadata   | Pod의 이름, 레이블 등의 메타데이터를 정의합니다.               |
| spec       | Pod의 실행 환경과 컨테이너 구성을 정의합니다.                 |
| containers | Pod에서 실행할 컨테이너 목록을 정의합니다.                   |
| name       | 컨테이너 이름을 지정합니다.                             |
| image      | 컨테이너에서 사용할 이미지를 지정합니다.                      |
| ports      | 컨테이너가 사용하는 포트 정보를 정의합니다.                    |

Pod에서 자주 사용하는 주요 설정은 다음과 같습니다.

### 6.1. restartPolicy

`restartPolicy`는 Pod 내부 컨테이너가 종료되었을 때 컨테이너를 다시 시작할지 결정하는 정책입니다.

| 정책        | 설명                               |
| --------- | -------------------------------- |
| Always    | 컨테이너가 종료되면 종료 코드와 관계없이 다시 시작합니다. |
| OnFailure | 컨테이너가 비정상적으로 종료된 경우 다시 시작합니다.    |
| Never     | 컨테이너가 종료되어도 다시 시작하지 않습니다.        |

별도로 지정하지 않으면 기본값은 `Always`입니다.

다음은 `restartPolicy`를 설정한 예제입니다.

```
apiVersion: v1
kind: Pod
metadata:
  name: batch-pod
spec:
  restartPolicy: OnFailure
  containers:
    - name: batch
      image: busybox:1.36
      command: ["sh", "-c", "echo Batch processing"]
```

`restartPolicy`는 Pod 단위로 설정하며, Pod 내부의 모든 컨테이너에 적용됩니다.

특히 Job과 CronJob은 일반적으로 작업 완료 후 컨테이너가 불필요하게 다시 실행되지 않도록 `Never` 또는 `OnFailure`를 사용합니다.

### 6.2. Init Container

Init Container는 애플리케이션 컨테이너가 실행되기 전에 초기화 작업을 수행하는 컨테이너입니다.

일반 컨테이너와 달리 Init Container는 정의된 순서대로 실행되며, 이전 Init Container가 성공적으로 종료되어야 다음 Init Container가 실행됩니다.

모든 Init Container가 성공적으로 완료된 후 일반 애플리케이션 컨테이너가 실행됩니다.

```
Pod
│
├── Init Container 1
│       │
│       ▼
├── Init Container 2
│       │
│       ▼
├── Application Container
│
└── Sidecar Container
```

다음은 Init Container를 사용하는 예제입니다.

```
apiVersion: v1
kind: Pod
metadata:
  name: init-container-pod
spec:
  initContainers:
    - name: init
      image: busybox:1.36
      command:
        - sh
        - -c
        - echo "Initialization completed"

  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

Init Container는 다음과 같은 작업에 활용할 수 있습니다.

- 애플리케이션 실행에 필요한 초기 파일 생성
- 외부 서비스의 준비 상태 확인
- 초기 데이터 다운로드
- 애플리케이션 실행 전 환경 구성

Init Container가 성공적으로 완료되지 않으면 일반 애플리케이션 컨테이너는 실행되지 않습니다.

### 6.3. Probe

Probe는 Kubernetes가 컨테이너의 상태를 확인하기 위해 사용하는 검사 방법입니다.

Kubernetes는 Probe를 통해 애플리케이션이 정상적으로 실행되고 있는지, 트래픽을 처리할 준비가 되었는지 등을 판단할 수 있습니다.

대표적인 Probe는 다음 세 가지입니다.

| Probe           | 설명                                                                                |
| --------------- | --------------------------------------------------------------------------------- |
| Liveness Probe  | 컨테이너가 정상적으로 동작하고 있는지 확인합니다. 검사에 실패하면 kubelet이 컨테이너를 재시작할 수 있습니다.                  |
| Readiness Probe | 컨테이너가 요청을 처리할 준비가 되었는지 확인합니다. 실패하면 해당 Pod는 일반적으로 Service의 트래픽 전달 대상에서 제외됩니다.      |
| Startup Probe   | 애플리케이션의 초기 시작이 완료되었는지 확인합니다. 성공하기 전까지 Liveness Probe와 Readiness Probe의 실행을 유예합니다. |

다음은 Probe를 설정한 예제입니다.

```
apiVersion: v1
kind: Pod
metadata:
  name: probe-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80

      startupProbe:
        httpGet:
          path: /
          port: 80
        failureThreshold: 30
        periodSeconds: 10

      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 10

      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5
```

Probe는 HTTP, TCP, gRPC 및 명령어 실행 방식 등을 통해 컨테이너 상태를 검사할 수 있습니다.

### 6.4. Resources

Pod 내부 컨테이너에는 CPU와 메모리 등의 리소스 요구량과 사용 제한을 설정할 수 있습니다.

리소스 설정은 `resources` 필드에서 정의합니다.

주요 설정은 다음과 같습니다.

| 설정       | 설명                              |
| -------- | ------------------------------- |
| requests | 컨테이너 실행에 필요한 최소 리소스 요구량을 지정합니다. |
| limits   | 컨테이너가 사용할 수 있는 리소스의 상한을 지정합니다.  |

예제는 다음과 같습니다.

```
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      resources:
        requests:
          cpu: "250m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

`requests`는 Kubernetes Scheduler가 Pod를 어느 노드에 배치할지 결정할 때 중요한 기준으로 사용됩니다.

`limits`는 컨테이너의 리소스 사용량을 제한하는 기준으로 사용됩니다.

CPU는 일반적으로 설정된 제한을 초과하여 사용할 수 없도록 스로틀링(Throttling)되며, 메모리는 제한을 초과하면 OOM(Out Of Memory)으로 컨테이너가 종료될 수 있습니다.

## 7. Pod와 상위 Workload 리소스의 관계

Kubernetes에서는 Pod를 직접 생성하여 관리할 수도 있지만, 실제 운영 환경에서는 Deployment, StatefulSet, DaemonSet 등의 상위 Workload 리소스를 통해 관리하는 것이 일반적입니다.

이러한 리소스들은 Pod의 생성과 삭제, 복제, 업데이트 등의 작업을 자동화합니다.

대표적인 Workload 리소스와 Pod의 관계는 다음과 같습니다.

```
Workload Resources
│
├── Deployment
│   └── ReplicaSet
│       └── Pod
│
├── StatefulSet
│   └── Pod
│
├── DaemonSet
│   └── Pod
│
├── Job
│   └── Pod
│
└── CronJob
    └── Job
        └── Pod
```

각 리소스의 역할은 다음과 같습니다.

| 리소스         | Pod 관리 목적                                         |
| ----------- | ------------------------------------------------- |
| Deployment  | 애플리케이션의 배포, 복제본 관리 및 Rolling Update를 수행합니다.       |
| ReplicaSet  | 지정된 수의 Pod 복제본이 유지되도록 관리합니다.                      |
| StatefulSet | 안정적인 네트워크 식별자와 영구 스토리지를 사용하는 상태 기반 애플리케이션을 관리합니다. |
| DaemonSet   | 각 노드 또는 지정된 노드 집합에 Pod가 실행되도록 관리합니다.              |
| Job         | 특정 작업이 성공적으로 완료될 때까지 Pod를 실행합니다.                  |
| CronJob     | 지정된 일정에 따라 Job을 생성하고 실행합니다.                       |

예를 들어 Deployment에 복제본을 3개로 설정하면 Deployment Controller가 ReplicaSet을 관리하고, ReplicaSet Controller가 실제 Pod 3개를 생성하고 유지합니다.

따라서 Pod는 애플리케이션의 실제 실행 단위이며, Deployment 등의 상위 리소스는 Pod의 생성과 유지보수를 담당하는 관리 단위라고 이해할 수 있습니다.

## 8. Pod의 네트워킹

Kubernetes에서 Pod는 고유한 IP 주소를 할당받으며, 클러스터 내부의 다른 Pod와 직접 통신할 수 있습니다.

일반적으로 Kubernetes 네트워크 모델은 다음과 같은 특성을 가집니다.

- 모든 Pod는 고유한 IP 주소를 가집니다.
- 동일한 클러스터 내의 Pod들은 NAT 없이 서로 통신할 수 있습니다.
- Pod는 동일한 노드에 있는지 여부와 관계없이 통신할 수 있습니다.
- 컨테이너 간 통신은 동일한 Pod의 네트워크 네임스페이스를 공유하여 이루어집니다.

다음은 Pod 간 통신 구조의 예시입니다.

```
Kubernetes Cluster
│
├── Node 1
│   └── Pod A
│       └── IP: 10.0.1.10
│
└── Node 2
    └── Pod B
        └── IP: 10.0.2.20

Pod A (10.0.1.10)
        │
        │ Pod-to-Pod Communication
        ▼
Pod B (10.0.2.20)
```

다만 Pod IP는 Pod의 생명주기에 종속됩니다. Pod가 삭제되고 새로운 Pod가 생성되면 IP 주소가 변경될 수 있습니다.

따라서 일반적으로 Pod IP를 직접 참조하기보다는 Service를 통해 안정적인 네트워크 접근 지점을 제공하는 방식을 사용합니다.

Service는 Pod의 IP가 변경되더라도 동일한 서비스 주소를 통해 애플리케이션에 접근할 수 있도록 지원합니다.

## 9. Pod의 스케줄링과 실행

Pod가 생성되면 Kubernetes의 Scheduler가 Pod를 실행할 노드를 결정합니다.

일반적인 Pod 생성 및 실행 과정은 다음과 같습니다.

```
kubectl apply
      │
      ▼
Kubernetes API Server
      │
      ▼
Pod 생성
      │
      ▼
Scheduler
      │
      ▼
Node 선택 및 바인딩
      │
      ▼
Kubelet
      │
      ▼
Container Runtime
      │
      ▼
Container 실행
      │
      ▼
Pod Running
```

각 구성 요소의 역할은 다음과 같습니다.

1. API Server: Pod 생성 요청을 받아 Kubernetes API 객체로 저장합니다.
2. Scheduler: Pod의 리소스 요구량과 스케줄링 조건 등을 고려하여 실행할 노드를 선택합니다.
3. Kubelet: 선택된 노드에서 Pod의 실행 상태를 관리합니다.
4. Container Runtime: 컨테이너 이미지를 가져오고 실제 컨테이너를 생성 및 실행합니다.

Pod가 실행될 노드가 결정된 이후에는 해당 노드의 Kubelet이 Pod의 실행 상태를 지속적으로 관리합니다.

## 10. Pod 관리 명령어

Pod를 조회하고 관리할 때 사용하는 주요 kubectl 명령어는 다음과 같습니다.

| 명령어                                           | 설명                                    |
| --------------------------------------------- | ------------------------------------- |
| `kubectl get pods`                            | 현재 네임스페이스의 Pod 목록을 조회합니다.             |
| `kubectl get pods -A`                         | 모든 네임스페이스의 Pod 목록을 조회합니다.             |
| `kubectl describe pod <pod-name>`             | Pod의 상세 정보와 이벤트를 조회합니다.               |
| `kubectl logs <pod-name>`                     | Pod 내부 컨테이너의 로그를 조회합니다.               |
| `kubectl logs <pod-name> -c <container-name>` | 특정 컨테이너의 로그를 조회합니다.                   |
| `kubectl exec -it <pod-name> -- sh`           | Pod 내부 컨테이너에서 명령어를 실행합니다.             |
| `kubectl delete pod <pod-name>`               | Pod를 삭제합니다.                           |
| `kubectl apply -f pod.yaml`                   | YAML Manifest를 기반으로 Pod를 생성하거나 변경합니다. |
| `kubectl get pod <pod-name> -o yaml`          | Pod의 전체 YAML 정보를 조회합니다.               |

다중 컨테이너 Pod에서는 `-c` 옵션을 사용하여 특정 컨테이너를 지정할 수 있습니다.

예를 들어 다음 명령어는 Pod 내부의 `nginx` 컨테이너에서 셸을 실행합니다.

```
kubectl exec -it nginx-pod -c nginx -- sh
```

## 11. Pod 사용 시 고려사항

Pod를 설계하고 운영할 때 다음 사항을 고려해야 합니다.

### 11.1. Pod는 일시적인 실행 단위입니다.

Pod는 영구적인 실행 단위가 아닙니다.

노드 장애, 애플리케이션 오류, 스케일링, 업데이트 등의 이유로 Pod가 삭제되거나 새로운 Pod로 교체될 수 있습니다.

따라서 Pod의 IP 주소나 이름이 항상 유지된다고 가정해서는 안 됩니다.

### 11.2. Pod 내부 컨테이너는 밀접하게 결합되어야 합니다.

여러 컨테이너를 하나의 Pod에 배치하면 네트워크와 스토리지를 공유할 수 있지만, Pod의 생명주기도 함께 관리됩니다.

따라서 독립적으로 배포하거나 확장해야 하는 컨테이너는 별도의 Pod로 분리하는 것이 적절합니다.

### 11.3. 상태가 필요한 데이터는 별도로 관리해야 합니다.

Pod가 삭제되거나 교체되더라도 데이터가 유지되어야 하는 경우에는 PersistentVolumeClaim(PVC) 등을 사용하여 영구 스토리지를 연결해야 합니다.

컨테이너 내부 파일 시스템에만 데이터를 저장하면 컨테이너 재생성 시 데이터가 유지되지 않을 수 있습니다.

### 11.4. Pod를 직접 관리하기보다는 상위 Workload 리소스를 활용해야 합니다.

직접 생성한 Pod는 삭제되거나 노드 장애가 발생했을 때 이를 대신할 Pod를 자동으로 생성하지 않습니다.

Deployment, StatefulSet, DaemonSet 등의 상위 리소스를 사용하면 원하는 상태를 지속적으로 유지할 수 있습니다.

## 12. 핵심 정리

Pod는 Kubernetes에서 컨테이너를 실행하기 위한 가장 작은 배포 단위입니다.

- 하나 이상의 컨테이너를 포함할 수 있습니다.
- 동일한 Pod 내부의 컨테이너들은 네트워크 네임스페이스와 볼륨을 공유할 수 있습니다.
- Pod는 고유한 IP 주소와 생명주기를 가집니다.
- Pod Phase, Container State, Pod Conditions를 통해 실행 상태를 확인할 수 있습니다.
- Init Container, Probe, Resource Requests/Limits 등의 설정을 통해 컨테이너 실행 환경을 제어할 수 있습니다.
- Pod는 일시적인 객체이므로 영구적인 네트워크 식별자나 데이터 저장소로 사용해서는 안 됩니다.
- 실제 운영 환경에서는 Deployment, StatefulSet, DaemonSet 등의 상위 Workload 리소스를 통해 Pod를 관리하는 것이 일반적입니다.

Pod는 Kubernetes의 모든 Workload 리소스가 실제 애플리케이션을 실행하기 위해 사용하는 기본 단위입니다. Deployment, StatefulSet, DaemonSet, Job 등의 리소스를 이해하기 위해서는 먼저 Pod의 구조와 생명주기를 이해해야 합니다.