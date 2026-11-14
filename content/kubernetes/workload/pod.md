# **Pod**

## **1. Pod란?**

**Pod는 Kubernetes에서 생성하고 관리할 수 있는 가장 작은 배포 단위(Deployable Unit)입니다.**

Kubernetes에서 컨테이너는 일반적으로 독립적인 관리 단위가 아니라 Pod 내부에서 실행됩니다. 하나의 Pod에는 하나 이상의 컨테이너가 포함될 수 있으며, 동일한 Pod에 속한 컨테이너들은 네트워크 네임스페이스와 스토리지 등의 실행 환경을 공유합니다.

Pod는 일반적으로 하나의 애플리케이션 인스턴스를 실행하는 데 사용됩니다. 예를 들어 웹 애플리케이션을 실행하는 경우 하나의 Pod에 애플리케이션 컨테이너를 배치할 수 있습니다.

하나의 Pod에 여러 컨테이너를 배치하는 경우에는 애플리케이션 컨테이너와 긴밀하게 연관된 보조 컨테이너를 함께 실행할 수 있습니다. 대표적인 사례로 로그 수집, 프록시, 데이터 동기화 등을 담당하는 사이드카(Sidecar) 컨테이너가 있습니다.

Pod의 주요 특징은 다음과 같습니다.

- 하나 이상의 컨테이너를 포함할 수 있습니다.
- 동일한 Pod에 속한 컨테이너들은 하나의 네트워크 네임스페이스를 공유합니다.
- 동일한 Pod에 속한 컨테이너들은 Pod에 정의된 볼륨을 공유할 수 있습니다.
- Pod 내부 컨테이너들은 동일한 IP 주소와 포트 공간을 사용합니다.
- Pod는 자체적인 생명주기를 가지며, 일반적으로 생성된 노드에서 실행됩니다. 노드 장애 등의 상황에서 기존 Pod가 다른 노드로 이동하는 것이 아니라 새로운 Pod가 생성될 수 있습니다.
- Pod가 삭제되면 해당 Pod에 속한 컨테이너도 함께 종료됩니다.
- Pod의 IP 주소와 UID는 Pod 객체의 생명주기에 종속됩니다. 새로운 Pod가 생성되면 기존 Pod와는 별개의 객체로 취급됩니다.

Kubernetes에서 Pod는 애플리케이션을 실행하는 기본 단위이지만, 일반적으로 실제 운영 환경에서는 Pod를 직접 생성하고 관리하기보다는 Deployment, StatefulSet, DaemonSet 등의 상위 Workload 리소스를 통해 관리합니다.

## **2. Pod의 구조**

Pod는 하나 이상의 컨테이너와 이들이 공유하는 실행 환경으로 구성됩니다.

일반적인 Pod는 다음과 같은 구조를 가집니다.

```text
Pod
├── Shared Network Namespace
│   ├── Pod IP Address
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

### **2.1. 네트워크 공유**

동일한 Pod에 속한 컨테이너들은 하나의 네트워크 네임스페이스(Network Namespace)를 공유합니다.

따라서 Pod 내부의 모든 컨테이너는 동일한 IP 주소를 사용하며, `localhost`를 통해 서로 통신할 수 있습니다.

예를 들어 하나의 Pod에 애플리케이션 컨테이너와 프록시 컨테이너가 함께 실행되는 경우 다음과 같이 통신할 수 있습니다.

```text
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
│  │ Application Container  │  │
│  │ Port: 3000             │  │
│  └────────────────────────┘  │
│                              │
└──────────────────────────────┘
```

두 컨테이너는 별도의 Pod IP를 사용하지 않고 동일한 네트워크 네임스페이스에서 `localhost`를 통해 통신합니다.

다만 네트워크 네임스페이스를 공유하므로 컨테이너 간 포트 충돌에 주의해야 합니다. 예를 들어 두 컨테이너가 모두 `0.0.0.0:8080`에 바인딩하려고 하면 포트 충돌이 발생할 수 있습니다.

또한 `containerPort`는 컨테이너가 사용하는 포트를 선언하는 메타데이터이며, 해당 포트를 별도의 네트워크 인터페이스에 개방하거나 외부에 자동으로 노출하는 설정은 아닙니다.

### **2.2. 스토리지 공유**

동일한 Pod에 속한 컨테이너들은 Pod에 정의된 볼륨(Volume)을 공유할 수 있습니다.

각 컨테이너는 동일한 볼륨을 서로 다른 경로에 마운트할 수 있으며, 이를 통해 컨테이너 간 파일을 공유할 수 있습니다.

예를 들어 애플리케이션 컨테이너가 생성한 로그 파일을 로그 수집 컨테이너가 읽도록 구성할 수 있습니다.

```text
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

Pod에 정의된 볼륨의 종류에 따라 데이터의 유지 범위는 달라집니다.

- `emptyDir`: Pod가 노드에 할당되어 있는 동안 유지되는 임시 볼륨입니다. Pod가 삭제되면 해당 데이터도 제거됩니다.
- `persistentVolumeClaim`: PersistentVolumeClaim을 통해 영구 스토리지를 연결할 수 있습니다. Pod가 교체되더라도 스토리지 자체의 데이터는 유지될 수 있습니다.
- `configMap`, `secret`: 설정 정보나 민감한 데이터를 컨테이너에 파일 또는 환경 변수 형태로 제공할 수 있습니다.

따라서 Pod 내부 컨테이너들이 볼륨을 공유할 수 있다는 사실과 해당 데이터가 영구적으로 보존된다는 사실은 구분해야 합니다.

### **2.3. 생명주기 공유**

Pod는 하나의 생명주기를 가지며, Pod 내부 컨테이너들은 해당 Pod의 실행 환경에 종속됩니다.

Pod가 종료되거나 삭제되면 내부 컨테이너도 함께 종료됩니다.

다만 Pod 내부의 개별 컨테이너는 독립적으로 재시작될 수 있습니다. 컨테이너 재시작 여부는 Pod에 설정된 `restartPolicy`와 컨테이너별 설정에 따라 결정됩니다.

예를 들어 애플리케이션 컨테이너가 오류로 종료되더라도 Pod 객체 자체가 삭제되는 것은 아닙니다. kubelet은 설정된 재시작 정책에 따라 동일한 Pod 안에서 해당 컨테이너를 다시 실행할 수 있습니다.

반면 Pod 객체가 삭제되거나 노드 장애 등으로 기존 Pod를 더 이상 사용할 수 없게 되면, Deployment나 ReplicaSet 등의 상위 컨트롤러가 새로운 Pod를 생성할 수 있습니다.

이 경우 새 Pod는 기존 Pod와 다른 UID를 가지며, Pod IP와 임시 스토리지 데이터도 달라질 수 있습니다.

## **3. Pod와 컨테이너의 관계**

Pod와 컨테이너는 서로 다른 관리 단위입니다.

컨테이너는 애플리케이션을 실행하는 환경이며, Pod는 하나 이상의 컨테이너를 묶어 Kubernetes가 관리할 수 있도록 하는 실행 단위입니다.

| 구분 | Pod | Container |
|---|---|---|
| 역할 | 컨테이너를 묶어 관리하는 실행 단위 | 애플리케이션 실행 단위 |
| 개수 | 하나 이상의 컨테이너 포함 | Pod 내부에 포함 |
| IP 주소 | Pod 단위로 할당 | 독립적인 Pod IP 없음 |
| 네트워크 | 네트워크 네임스페이스 공유 | Pod의 네트워크 네임스페이스 공유 |
| 스토리지 | 볼륨 정의 및 공유 | 볼륨을 컨테이너 경로에 마운트 |
| 생명주기 | Pod 자체의 생명주기 관리 | 개별 컨테이너 상태 및 재시작 관리 |

Kubernetes에서는 일반적으로 컨테이너를 직접 배포하기보다는 Pod를 정의하여 배포합니다.

또한 실제 운영 환경에서는 Pod를 직접 생성하기보다는 Deployment, StatefulSet, DaemonSet 등의 Workload 리소스를 통해 관리하는 방식을 권장합니다.

## **4. Pod의 종류**

Pod는 실행 목적과 구성 방식에 따라 다양한 형태로 사용됩니다.

### **4.1. 단일 컨테이너 Pod (Single Container Pod)**

하나의 Pod에 하나의 컨테이너만 포함하는 가장 일반적인 형태입니다.

대부분의 애플리케이션은 단일 컨테이너 Pod로 실행할 수 있습니다.

```text
Pod
└── Application Container
```

애플리케이션의 실행과 관리를 단순하게 유지할 수 있으며, Deployment와 같은 상위 Workload 리소스를 통해 여러 Pod를 생성하고 관리할 수 있습니다.

### **4.2. 다중 컨테이너 Pod (Multi-Container Pod)**

하나의 Pod에 여러 컨테이너를 포함하는 형태입니다.

동일한 생명주기를 가지면서 네트워크나 스토리지를 공유해야 하는 컨테이너들을 함께 배치할 때 사용합니다.

```text
Pod
├── Application Container
├── Sidecar Container
└── Log Collector Container
```

대표적인 사용 사례는 다음과 같습니다.

- **Sidecar:** 애플리케이션을 보조하는 프록시, 로그 수집기 등을 실행합니다.
- **Ambassador:** 애플리케이션의 외부 서비스 통신을 중계하거나 프록시 역할을 수행합니다.
- **Adapter:** 애플리케이션의 출력 데이터를 다른 시스템에서 사용할 수 있는 형식으로 변환합니다.

Sidecar, Ambassador, Adapter는 다중 컨테이너 Pod를 구성하는 대표적인 설계 패턴입니다. Kubernetes에서 각각 별도의 Workload 리소스 종류를 의미하는 것은 아닙니다.

다중 컨테이너 Pod는 컨테이너 간 밀접한 결합이 필요한 경우에 적합합니다. 서로 독립적으로 배포하거나 확장해야 하는 애플리케이션을 하나의 Pod에 배치하는 것은 적절하지 않습니다.

## **5. Pod의 생명주기 (Pod Lifecycle)**

Pod는 생성부터 종료까지 고유한 생명주기를 가집니다. Kubernetes는 Pod의 상태를 지속적으로 관찰하며, 현재 상태를 기반으로 필요한 작업을 수행합니다.

Pod의 생명주기는 크게 다음과 같은 개념으로 구분할 수 있습니다.

- **Pod Phase:** Pod의 전체적인 진행 단계를 나타냅니다.
- **Container State:** Pod 내부의 개별 컨테이너 실행 상태를 나타냅니다.
- **Pod Conditions:** Pod가 특정 조건을 충족하고 있는지를 나타냅니다.

### **5.1. Pod Phase**

Pod의 Phase는 Pod가 현재 어느 단계에 있는지를 나타내는 상위 수준의 상태입니다.

| Phase | 설명 |
|---|---|
| Pending | Pod가 Kubernetes에 생성되었지만, 컨테이너 실행이 아직 시작되지 않은 상태입니다. 노드 스케줄링, 이미지 다운로드, 볼륨 마운트, 초기화 작업 등이 진행될 수 있습니다. |
| Running | Pod가 노드에 바인딩되었으며 모든 컨테이너가 생성된 상태입니다. 하나 이상의 컨테이너가 실행 중이거나 시작 또는 재시작 중일 수 있습니다. |
| Succeeded | Pod에 포함된 모든 컨테이너가 성공적으로 종료된 상태입니다. |
| Failed | Pod에 포함된 모든 컨테이너가 종료되었으며 하나 이상의 컨테이너가 실패한 상태입니다. |
| Unknown | 일반적으로 노드와의 통신 문제 등으로 인해 Pod의 상태를 확인할 수 없는 상태입니다. |

일반적인 Pod의 상태 변화는 다음과 같습니다.

```text
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

다만 모든 Pod가 반드시 `Pending → Running → Succeeded` 또는 `Pending → Running → Failed` 순서로만 이동하는 것은 아닙니다. 예를 들어 Pod 생성 과정에서 문제가 발생하면 `Pending` 상태에서 종료될 수도 있습니다.

주의할 점은 Pod Phase와 컨테이너 상태(Container State)는 서로 다른 개념이라는 것입니다.

Pod가 `Running` 상태이더라도 내부 컨테이너가 재시작 중이거나 준비되지 않은 상태일 수 있습니다. 따라서 Pod가 실제로 요청을 처리할 수 있는지를 확인하려면 `Ready` Condition도 함께 확인해야 합니다.

### **5.2. Container State**

Pod 내부의 각 컨테이너는 독립적인 상태를 가집니다.

컨테이너 상태는 다음 세 가지로 구분됩니다.

| 상태 | 설명 |
|---|---|
| Waiting | 컨테이너가 실행을 시작하기 전 대기 중인 상태입니다. 이미지 다운로드, 컨테이너 생성, 설정 준비 등의 작업이 원인이 될 수 있습니다. |
| Running | 컨테이너가 정상적으로 실행 중인 상태입니다. |
| Terminated | 컨테이너의 실행이 종료된 상태입니다. 정상 종료 또는 오류로 인한 종료가 포함됩니다. |

컨테이너가 `Terminated` 상태가 되더라도 Pod의 `restartPolicy`에 따라 컨테이너가 다시 시작될 수 있습니다.

예를 들어 애플리케이션 컨테이너가 오류로 종료되었을 때 재시작 정책이 `Always`라면 kubelet은 해당 컨테이너를 다시 실행합니다.

컨테이너 상태를 확인할 때는 다음과 같은 세부 정보도 함께 확인할 수 있습니다.

- `reason`: 컨테이너 상태가 해당 값으로 설정된 이유입니다.
- `message`: 상태에 대한 추가 설명입니다.
- `exitCode`: 컨테이너가 종료된 경우 종료 코드입니다.
- `startedAt`: 컨테이너가 시작된 시각입니다.
- `finishedAt`: 컨테이너가 종료된 시각입니다.
- `lastState`: 이전 컨테이너 실행의 상태 정보입니다.

컨테이너가 반복적으로 종료되고 재시작되는 경우 `CrashLoopBackOff`라는 상태가 표시될 수 있습니다. 이는 컨테이너 상태 자체의 종류가 아니라, 반복적인 컨테이너 시작 실패로 인해 Kubernetes가 재시작 간격을 늘리며 대기하고 있음을 나타내는 상태 표시입니다.

### **5.3. Pod Conditions**

Pod Conditions는 Pod가 현재 어떤 조건을 충족하고 있는지를 나타냅니다.

Pod Phase가 전체적인 진행 단계를 나타낸다면, Pod Conditions는 Pod의 세부적인 상태와 준비 여부를 표현합니다.

대표적인 Condition은 다음과 같습니다.

| Condition | 설명 |
|---|---|
| PodScheduled | Pod가 실행될 노드에 정상적으로 스케줄링되었는지를 나타냅니다. |
| PodReadyToStartContainers | Pod의 실행 샌드박스 생성과 네트워크 설정 등이 완료되어 컨테이너를 시작할 수 있는지를 나타냅니다. |
| Initialized | 모든 일반 Init Container가 성공적으로 완료되었는지를 나타냅니다. |
| ContainersReady | Pod 내부의 모든 컨테이너가 Ready 상태인지를 나타냅니다. |
| Ready | Pod가 서비스 트래픽을 처리할 준비가 되었는지를 나타냅니다. |

`PodReadyToStartContainers`는 Kubernetes v1.28에서 처음 도입되었으며, v1.37부터 Stable 상태입니다. 이 Condition은 Pod의 네트워크 샌드박스 및 컨테이너 실행 환경 준비 상태를 파악하는 데 사용됩니다.

Condition의 상태 값은 일반적으로 다음과 같이 표현됩니다.

- `True`: 해당 조건을 충족합니다.
- `False`: 해당 조건을 충족하지 않습니다.
- `Unknown`: 해당 조건의 상태를 확인할 수 없습니다.

`Ready` Condition은 Service와 같은 다른 Kubernetes 리소스가 Pod를 트래픽 전달 대상으로 사용할 수 있는지를 판단하는 데 중요한 역할을 합니다.

예를 들어 Pod가 `Running` 상태이더라도 Readiness Probe가 실패하면 `Ready` Condition은 `False`가 될 수 있습니다.

이 경우 해당 Pod는 일반적으로 Service의 정상적인 트래픽 전달 대상에서 제외됩니다.

또한 `readinessGates`를 사용하면 기본 컨테이너 준비 상태 외에 사용자가 정의한 추가 조건을 Pod의 Ready 판단에 반영할 수 있습니다.

### **5.4. Pod 종료 과정**

Pod가 삭제되거나 종료되면 kubelet은 컨테이너를 즉시 강제 종료하는 대신 정상적인 종료 절차를 수행합니다.

일반적인 종료 과정은 다음과 같습니다.

```text
Pod 삭제 요청
     │
     ▼
DeletionTimestamp 설정
     │
     ▼
Pod 종료 절차 시작
     │
     ├── Endpoint에서 트래픽 제외 처리
     │
     ├── PreStop Hook 실행 (설정된 경우)
     │
     ├── 컨테이너에 SIGTERM 전달
     │
     ▼
Grace Period 동안 정상 종료 대기
     │
     ├── 정상 종료
     │
     └── 제한 시간 초과
             │
             ▼
         SIGKILL 전달
             │
             ▼
          Pod 종료
```

Pod 삭제 요청이 전달되면 Kubernetes는 기본적으로 30초의 `terminationGracePeriodSeconds`를 기준으로 정상 종료를 기다립니다.

주요 설정은 다음과 같습니다.

| 설정 | 설명 |
|---|---|
| `terminationGracePeriodSeconds` | Pod가 종료되기 전 정상 종료를 기다리는 최대 시간을 초 단위로 지정합니다. 기본값은 30초입니다. |
| `preStop` | 컨테이너 종료 직전에 실행할 작업을 정의합니다. |
| `SIGTERM` | 컨테이너에 정상 종료를 요청하는 시그널입니다. |
| `SIGKILL` | 정상 종료 대기 시간이 초과되었을 때 컨테이너를 강제로 종료하는 시그널입니다. |

`preStop` Hook이 설정된 경우 kubelet은 컨테이너 종료 과정에서 해당 Hook을 실행합니다. Hook의 실행 시간도 Pod의 종료 유예 시간에 포함됩니다.

애플리케이션은 `SIGTERM`을 수신하면 새로운 요청 수신을 중단하고, 처리 중인 요청을 마무리한 후 종료하도록 구현하는 것이 좋습니다.

이를 Graceful Shutdown이라고 합니다.

특히 HTTP 서버나 메시지 소비자처럼 종료 시점에 처리 중인 작업이 존재할 수 있는 애플리케이션은 종료 유예 시간과 애플리케이션의 정상 종료 절차를 함께 설계해야 합니다.

## **6. Pod의 주요 설정**

Pod는 YAML Manifest를 통해 정의할 수 있습니다.

다음은 Pod를 생성하는 기본적인 YAML 예제입니다.

```yaml
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

| 필드 | 설명 |
|---|---|
| `apiVersion` | Kubernetes API 버전을 지정합니다. Pod는 `v1`을 사용합니다. |
| `kind` | 생성할 리소스의 종류를 지정합니다. |
| `metadata` | Pod의 이름, 레이블 등의 메타데이터를 정의합니다. |
| `spec` | Pod의 실행 환경과 컨테이너 구성을 정의합니다. |
| `containers` | Pod에서 실행할 일반 컨테이너 목록을 정의합니다. |
| `name` | 컨테이너 이름을 지정합니다. |
| `image` | 컨테이너에서 사용할 이미지를 지정합니다. |
| `ports` | 컨테이너가 사용하는 포트 정보를 선언합니다. |

Pod에서 자주 사용하는 주요 설정은 다음과 같습니다.

### **6.1. restartPolicy**

`restartPolicy`는 Pod 내부 컨테이너가 종료되었을 때 컨테이너를 다시 시작할지 결정하는 정책입니다.

| 정책 | 설명 |
|---|---|
| `Always` | 컨테이너가 종료되면 종료 코드와 관계없이 다시 시작합니다. |
| `OnFailure` | 컨테이너가 비정상적으로 종료된 경우 다시 시작합니다. |
| `Never` | 컨테이너가 종료되어도 다시 시작하지 않습니다. |

별도로 지정하지 않으면 기본값은 `Always`입니다.

다음은 `restartPolicy`를 설정한 예제입니다.

```yaml
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

`restartPolicy`는 Pod 단위로 설정하며, 기본적으로 해당 Pod의 일반 컨테이너와 일반 Init Container의 재시작 동작에 영향을 줍니다. 단, Sidecar로 사용하는 재시작 가능한 Init Container는 컨테이너 수준의 `restartPolicy`를 별도로 가질 수 있습니다.

특히 Job과 CronJob은 일반적으로 작업 완료 후 컨테이너가 불필요하게 다시 실행되지 않도록 `Never` 또는 `OnFailure`를 사용합니다.

컨테이너가 반복적으로 종료되면 kubelet은 재시작 사이에 지수형 백오프(Exponential Backoff)를 적용합니다. 따라서 컨테이너가 종료되었다고 해서 즉시 다시 시작되는 것은 아닙니다.

### **6.2. Init Container**

Init Container는 애플리케이션 컨테이너가 실행되기 전에 초기화 작업을 수행하는 컨테이너입니다.

일반 컨테이너와 달리 Init Container는 정의된 순서대로 실행되며, 이전 Init Container가 성공적으로 종료되어야 다음 Init Container가 실행됩니다.

모든 일반 Init Container가 성공적으로 완료된 후 일반 애플리케이션 컨테이너가 실행됩니다.

```text
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

위 구조에서 일반 Init Container는 순서대로 실행된 후 종료되며, Application Container와 함께 계속 실행되는 것은 아닙니다. Sidecar는 별도의 실행 방식으로 Pod의 생명주기 동안 실행될 수 있습니다.

다음은 Init Container를 사용하는 예제입니다.

```yaml
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

일반 Init Container는 작업이 성공적으로 완료될 때까지 재시도될 수 있으므로, 초기화 작업은 여러 번 실행되어도 문제가 발생하지 않도록 멱등성(Idempotency)을 고려하여 구현하는 것이 좋습니다.

#### **Sidecar Container와의 차이**

Sidecar Container는 애플리케이션 컨테이너와 함께 실행되면서 보조 기능을 제공하는 컨테이너입니다.

Kubernetes에서는 재시작 가능한 Init Container를 사용하여 Sidecar를 구성할 수 있습니다. 이 방식은 일반 Init Container와 달리 초기화 과정에서 시작된 후 애플리케이션 컨테이너와 함께 계속 실행됩니다.

예제는 다음과 같습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-pod
spec:
  initContainers:
    - name: log-collector
      image: busybox:1.36
      restartPolicy: Always
      command:
        - sh
        - -c
        - tail -F /var/log/app/application.log
      volumeMounts:
        - name: log-volume
          mountPath: /var/log/app

  containers:
    - name: application
      image: nginx:1.27
      volumeMounts:
        - name: log-volume
          mountPath: /var/log/app

  volumes:
    - name: log-volume
      emptyDir: {}
```

위 예제에서 `log-collector`는 `initContainers`에 정의되어 있지만, `restartPolicy: Always`가 설정되어 있으므로 재시작 가능한 Init Container, 즉 Sidecar로 동작합니다.

Kubernetes의 Sidecar Container 기능은 v1.33부터 Stable 상태입니다.

일반 Init Container와 Sidecar Container의 차이는 다음과 같습니다.

| 구분 | 일반 Init Container | Sidecar Container |
|---|---|---|
| 실행 시점 | 애플리케이션 컨테이너보다 먼저 실행 | 애플리케이션 컨테이너보다 먼저 시작할 수 있음 |
| 실행 방식 | 작업 완료 후 종료 | Pod가 종료될 때까지 실행 |
| 애플리케이션과 동시 실행 | 일반적으로 실행되지 않음 | 실행됨 |
| 설정 위치 | `initContainers` | `initContainers` |
| 재시작 설정 | Pod의 `restartPolicy`에 따름 | 컨테이너 수준의 `restartPolicy: Always` |
| 주요 목적 | 초기화 작업 | 로그 수집, 프록시 등 보조 기능 |

### **6.3. Probe**

Probe는 Kubernetes가 컨테이너의 상태를 확인하기 위해 사용하는 검사 방법입니다.

kubelet은 Probe를 주기적으로 실행하여 애플리케이션이 정상적으로 동작하고 있는지, 트래픽을 처리할 준비가 되었는지 등을 판단할 수 있습니다.

대표적인 Probe는 다음 세 가지입니다.

| Probe | 설명 |
|---|---|
| Liveness Probe | 컨테이너가 정상적으로 동작하고 있는지 확인합니다. 검사에 실패하면 kubelet이 컨테이너를 재시작할 수 있습니다. |
| Readiness Probe | 컨테이너가 요청을 처리할 준비가 되었는지 확인합니다. 실패하면 해당 Pod는 일반적으로 Service의 트래픽 전달 대상에서 제외됩니다. |
| Startup Probe | 애플리케이션의 초기 시작이 완료되었는지 확인합니다. 성공하기 전까지 Liveness Probe와 Readiness Probe의 실행을 유예합니다. |

다음은 Probe를 설정한 예제입니다.

```yaml
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
        periodSeconds: 10

      readinessProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 5
```

Probe는 다음과 같은 방식으로 컨테이너 상태를 검사할 수 있습니다.

| 검사 방식 | 설명 |
|---|---|
| `httpGet` | HTTP 요청을 전송하여 응답 상태를 확인합니다. |
| `tcpSocket` | 지정된 TCP 포트에 연결할 수 있는지 확인합니다. |
| `exec` | 컨테이너 내부에서 명령어를 실행하고 종료 코드를 확인합니다. |
| `grpc` | gRPC Health Checking Protocol을 사용하여 상태를 확인합니다. |

Probe의 주요 설정은 다음과 같습니다.

| 설정 | 설명 |
|---|---|
| `initialDelaySeconds` | 컨테이너 시작 후 첫 번째 Probe를 실행하기까지 대기할 시간입니다. |
| `periodSeconds` | Probe를 실행하는 주기입니다. |
| `timeoutSeconds` | Probe 응답을 기다리는 최대 시간입니다. |
| `successThreshold` | 성공으로 판단하기 위해 필요한 연속 성공 횟수입니다. |
| `failureThreshold` | 실패로 판단하기 위해 필요한 연속 실패 횟수입니다. |
| `terminationGracePeriodSeconds` | Probe 실패로 컨테이너를 재시작할 때 정상 종료를 기다리는 시간을 지정할 수 있습니다. |

각 Probe는 목적이 다르므로 검사 기준을 구분하여 설계해야 합니다.

- **Liveness Probe:** 애플리케이션이 더 이상 정상적으로 동작하지 않는 상태를 감지하는 데 사용합니다. 단순히 외부 DB나 API가 일시적으로 응답하지 않는다는 이유로 실패하도록 구성하면 불필요한 컨테이너 재시작이 발생할 수 있습니다.
- **Readiness Probe:** 애플리케이션이 실제 요청을 처리할 수 있는 상태인지 확인하는 데 사용합니다. 애플리케이션이 실행 중이더라도 초기화가 완료되지 않았거나 필수 의존성이 준비되지 않았다면 실패하도록 구성할 수 있습니다.
- **Startup Probe:** 초기 구동 시간이 긴 애플리케이션에서 시작 과정이 완료되기 전에 Liveness Probe가 실패하는 문제를 방지하는 데 사용합니다.

Startup Probe가 설정된 경우 Startup Probe가 성공할 때까지 Liveness Probe와 Readiness Probe가 실행되지 않습니다.

### **6.4. Resources**

Pod 내부 컨테이너에는 CPU와 메모리 등의 리소스 요구량과 사용 제한을 설정할 수 있습니다.

리소스 설정은 `resources` 필드에서 정의합니다.

주요 설정은 다음과 같습니다.

| 설정 | 설명 |
|---|---|
| `requests` | 컨테이너 실행에 필요한 리소스 요구량을 지정합니다. |
| `limits` | 컨테이너가 사용할 수 있는 리소스의 상한을 지정합니다. |

예제는 다음과 같습니다.

```yaml
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

예를 들어 CPU `250m`, 메모리 `128Mi`를 요청하는 컨테이너가 포함된 Pod는 해당 리소스 요구량을 충족할 수 있는 노드에 스케줄링되어야 합니다.

`limits`는 컨테이너의 리소스 사용량을 제한하는 기준으로 사용됩니다.

- **CPU:** 설정된 제한을 초과하여 사용하려는 경우 CPU Throttling이 발생합니다.
- **Memory:** 설정된 제한을 초과하여 메모리를 사용하면 OOM(Out Of Memory)으로 컨테이너가 종료될 수 있습니다.

CPU와 메모리는 리소스 제한을 적용하는 방식에도 차이가 있습니다. CPU는 사용량을 제한하는 방식으로 동작하지만, 메모리는 사용량이 제한을 초과하면 컨테이너가 종료될 수 있는 리소스입니다.

또한 컨테이너에 `requests`와 `limits`를 적절하게 설정하면 Scheduler의 배치 판단과 노드의 리소스 관리에 도움이 됩니다.

## **7. Pod와 상위 Workload 리소스의 관계**

Kubernetes에서는 Pod를 직접 생성하여 관리할 수도 있지만, 실제 운영 환경에서는 Deployment, StatefulSet, DaemonSet 등의 상위 Workload 리소스를 통해 관리하는 것이 일반적입니다.

이러한 리소스들은 Pod의 생성과 삭제, 복제, 업데이트 등의 작업을 자동화합니다.

대표적인 Workload 리소스와 Pod의 관계는 다음과 같습니다.

```text
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

| 리소스 | Pod 관리 목적 |
|---|---|
| Deployment | 애플리케이션의 배포, 복제본 관리 및 Rolling Update를 수행합니다. |
| ReplicaSet | 지정된 수의 Pod 복제본이 유지되도록 관리합니다. |
| StatefulSet | 안정적인 네트워크 식별자와 영구 스토리지를 사용하는 상태 기반 애플리케이션을 관리합니다. |
| DaemonSet | 각 노드 또는 지정된 노드 집합에 Pod가 실행되도록 관리합니다. |
| Job | 특정 작업이 성공적으로 완료될 때까지 Pod를 실행합니다. |
| CronJob | 지정된 일정에 따라 Job을 생성하고 실행합니다. |

예를 들어 Deployment에 복제본을 3개로 설정하면 Deployment Controller가 ReplicaSet을 관리하고, ReplicaSet Controller가 실제 Pod 3개를 생성하고 유지합니다.

따라서 Pod는 애플리케이션의 실제 실행 단위이며, Deployment 등의 상위 리소스는 Pod의 생성과 유지보수를 담당하는 관리 단위라고 이해할 수 있습니다.

상위 Workload 리소스가 Pod를 관리하는 방식은 리소스마다 다릅니다. Deployment는 ReplicaSet을 통해 Pod를 관리하지만, StatefulSet과 DaemonSet은 각자의 Controller가 Pod를 직접 관리합니다.

## **8. Pod의 네트워킹**

Kubernetes에서 Pod는 고유한 IP 주소를 할당받으며, 클러스터 내부의 다른 Pod와 직접 통신할 수 있습니다.

일반적인 Kubernetes 네트워크 모델은 다음과 같은 특성을 가집니다.

- 모든 Pod는 클러스터 네트워크에서 고유한 IP 주소를 가집니다.
- 동일한 클러스터 내의 Pod들은 NAT 없이 서로 통신할 수 있습니다.
- Pod는 동일한 노드에 있는지 여부와 관계없이 통신할 수 있습니다.
- 동일한 Pod 내부의 컨테이너들은 네트워크 네임스페이스를 공유합니다.
- Pod IP는 Pod의 생명주기에 종속되며, Pod가 교체되면 IP 주소가 변경될 수 있습니다.

다음은 Pod 간 통신 구조의 예시입니다.

```text
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

이러한 네트워크 모델은 일반적으로 CNI(Container Network Interface) 플러그인을 통해 구현됩니다.

Pod가 생성되면 컨테이너 런타임과 CNI 플러그인이 Pod의 네트워크 환경을 구성하고 IP 주소를 할당합니다.

다만 Pod IP는 Pod의 생명주기에 종속됩니다. Pod가 삭제되고 새로운 Pod가 생성되면 IP 주소가 변경될 수 있습니다.

따라서 일반적으로 Pod IP를 직접 참조하기보다는 Service를 통해 안정적인 네트워크 접근 지점을 제공하는 방식을 사용합니다.

Service는 레이블 셀렉터를 기반으로 대상 Pod를 선택하고, 해당 Pod들의 IP가 변경되더라도 동일한 서비스 주소를 통해 애플리케이션에 접근할 수 있도록 지원합니다.

또한 Pod가 `Ready` 상태가 아닌 경우 일반적인 Service 트래픽 전달 대상에서 제외됩니다. 단, `publishNotReadyAddresses` 등의 설정에 따라 동작이 달라질 수 있습니다.

## **9. Pod의 스케줄링과 실행**

Pod가 생성되면 Kubernetes의 Scheduler가 Pod를 실행할 노드를 결정합니다.

일반적인 Pod 생성 및 실행 과정은 다음과 같습니다.

```text
kubectl apply
      │
      ▼
Kubernetes API Server
      │
      ▼
Pod 생성 및 저장
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
Container 생성 및 실행
      │
      ▼
Pod Running
```

각 구성 요소의 역할은 다음과 같습니다.

1. **API Server:** Pod 생성 요청을 받아 Kubernetes API 객체로 저장합니다.
2. **Scheduler:** Pod의 리소스 요구량과 스케줄링 조건 등을 고려하여 실행할 노드를 선택합니다.
3. **Kubelet:** 선택된 노드에서 Pod의 실행 상태를 관리합니다.
4. **Container Runtime:** 컨테이너 이미지를 가져오고 실제 컨테이너를 생성 및 실행합니다.

Pod가 실행될 노드가 결정되면 해당 노드의 Kubelet이 Pod의 실행 상태를 지속적으로 관리합니다.

실제 실행 과정에는 볼륨 마운트, 네트워크 샌드박스 구성, 이미지 다운로드, Init Container 실행 등의 작업이 포함될 수 있습니다.

따라서 Pod가 생성되었다고 해서 즉시 애플리케이션 컨테이너가 실행되는 것은 아닙니다.

### **9.1. Scheduler의 노드 선택**

Scheduler는 Pod의 리소스 요구량과 스케줄링 조건을 기반으로 적절한 노드를 선택합니다.

대표적인 고려 요소는 다음과 같습니다.

- CPU 및 메모리 등의 리소스 요구량
- Node Selector 및 Node Affinity
- Taint와 Toleration
- Pod Affinity 및 Pod Anti-Affinity
- Topology Spread Constraints
- 볼륨 관련 스케줄링 조건

스케줄링 조건을 만족하는 노드가 없으면 Pod는 `Pending` 상태에 머무를 수 있습니다.

이 경우 다음 명령어를 통해 Pod의 상세 정보와 이벤트를 확인할 수 있습니다.

```bash
kubectl describe pod <pod-name>
```

예를 들어 `Insufficient cpu`, `Insufficient memory`, `FailedScheduling` 등의 이벤트를 통해 스케줄링 실패 원인을 확인할 수 있습니다.

## **10. Pod 관리 명령어**

Pod를 조회하고 관리할 때 사용하는 주요 kubectl 명령어는 다음과 같습니다.

| 명령어 | 설명 |
|---|---|
| `kubectl get pods` | 현재 네임스페이스의 Pod 목록을 조회합니다. |
| `kubectl get pods -A` | 모든 네임스페이스의 Pod 목록을 조회합니다. |
| `kubectl get pods -o wide` | Pod 목록과 함께 노드, IP 등의 추가 정보를 조회합니다. |
| `kubectl describe pod <pod-name>` | Pod의 상세 정보와 이벤트를 조회합니다. |
| `kubectl logs <pod-name>` | Pod 내부 컨테이너의 로그를 조회합니다. |
| `kubectl logs <pod-name> -c <container-name>` | 특정 컨테이너의 로그를 조회합니다. |
| `kubectl logs <pod-name> --previous` | 이전에 종료된 컨테이너 인스턴스의 로그를 조회합니다. |
| `kubectl exec -it <pod-name> -- sh` | Pod 내부 컨테이너에서 명령어를 실행합니다. |
| `kubectl delete pod <pod-name>` | Pod를 삭제합니다. |
| `kubectl apply -f pod.yaml` | YAML Manifest를 기반으로 Pod를 생성하거나 변경합니다. |
| `kubectl get pod <pod-name> -o yaml` | Pod의 전체 YAML 정보를 조회합니다. |

다중 컨테이너 Pod에서는 `-c` 옵션을 사용하여 특정 컨테이너를 지정할 수 있습니다.

예를 들어 다음 명령어는 Pod 내부의 `nginx` 컨테이너에서 셸을 실행합니다.

```bash
kubectl exec -it nginx-pod -c nginx -- sh
```

컨테이너가 반복적으로 재시작되는 경우에는 다음과 같이 이전 컨테이너의 로그를 확인할 수 있습니다.

```bash
kubectl logs nginx-pod -c nginx --previous
```

Pod의 상태와 이벤트를 확인할 때는 다음 명령어도 유용합니다.

```bash
kubectl get pod nginx-pod -o yaml
kubectl describe pod nginx-pod
kubectl get events --sort-by=.metadata.creationTimestamp
```

`kubectl delete pod`를 실행했을 때 해당 Pod가 Deployment나 ReplicaSet 등의 관리 대상이라면 상위 Controller가 원하는 복제본 수를 유지하기 위해 새로운 Pod를 생성할 수 있습니다.

반면 직접 생성한 독립 Pod는 삭제 후 이를 대신할 Pod가 자동으로 생성되지 않습니다.

## **11. Pod 사용 시 고려사항**

Pod를 설계하고 운영할 때 다음 사항을 고려해야 합니다.

### **11.1. Pod는 일시적인 실행 단위입니다.**

Pod는 영구적인 실행 단위가 아닙니다.

노드 장애, 애플리케이션 오류, 스케일링, 업데이트 등의 이유로 Pod가 삭제되거나 새로운 Pod로 교체될 수 있습니다.

따라서 Pod의 IP 주소나 이름, UID가 항상 유지된다고 가정해서는 안 됩니다.

애플리케이션이 특정 Pod의 IP 주소를 직접 참조하도록 설계하면 Pod가 교체될 때 통신 문제가 발생할 수 있습니다. 일반적으로 Service를 통해 안정적인 접근 지점을 제공하는 것이 적절합니다.

### **11.2. Pod 내부 컨테이너는 밀접하게 결합되어야 합니다.**

여러 컨테이너를 하나의 Pod에 배치하면 네트워크와 스토리지를 공유할 수 있지만, Pod의 생명주기도 함께 관리됩니다.

따라서 다음과 같은 특성을 가진 컨테이너들을 하나의 Pod에 배치하는 것이 적절합니다.

- 동일한 네트워크 환경을 공유해야 하는 경우
- 동일한 볼륨을 통해 데이터를 주고받아야 하는 경우
- 함께 배포되고 종료되어야 하는 경우
- 서로 긴밀하게 연관된 보조 기능을 제공하는 경우

반대로 독립적으로 배포하거나 확장해야 하는 컨테이너는 별도의 Pod로 분리하는 것이 적절합니다.

### **11.3. 상태가 필요한 데이터는 별도로 관리해야 합니다.**

Pod가 삭제되거나 교체되더라도 데이터가 유지되어야 하는 경우에는 PersistentVolumeClaim(PVC) 등을 사용하여 영구 스토리지를 연결해야 합니다.

컨테이너 내부의 쓰기 가능한 파일 시스템에만 데이터를 저장하면 컨테이너 재생성 시 해당 데이터가 유지되지 않을 수 있습니다.

특히 `emptyDir`는 동일한 Pod 내부 컨테이너 간 임시 데이터를 공유하는 데 적합하지만, Pod가 삭제되면 데이터도 함께 제거됩니다.

영구 데이터가 필요한 애플리케이션은 PersistentVolume과 PersistentVolumeClaim을 사용하거나, StatefulSet과 같은 Workload 리소스를 함께 고려해야 합니다.

### **11.4. Pod를 직접 관리하기보다는 상위 Workload 리소스를 활용해야 합니다.**

직접 생성한 Pod는 삭제되거나 노드 장애가 발생했을 때 이를 대신할 Pod를 자동으로 생성하지 않습니다.

Deployment, StatefulSet, DaemonSet 등의 상위 리소스를 사용하면 원하는 상태를 지속적으로 유지할 수 있습니다.

애플리케이션의 특성에 따라 적절한 Workload 리소스를 선택해야 합니다.

- **Deployment:** 일반적인 Stateless 애플리케이션을 배포하고 복제본을 관리하는 데 적합합니다.
- **StatefulSet:** 안정적인 네트워크 식별자와 영구 스토리지가 필요한 애플리케이션에 적합합니다.
- **DaemonSet:** 각 노드에서 실행되어야 하는 에이전트나 모니터링 프로그램 등에 적합합니다.
- **Job:** 한 번 실행하여 완료해야 하는 작업에 적합합니다.
- **CronJob:** 일정한 주기로 실행해야 하는 작업에 적합합니다.

### **11.5. Readiness와 Liveness를 구분해야 합니다.**

Readiness Probe와 Liveness Probe는 목적이 다르므로 동일한 검사 로직을 무조건 재사용하는 것은 적절하지 않을 수 있습니다.

Readiness Probe는 트래픽을 처리할 수 있는 상태인지를 판단하고, Liveness Probe는 컨테이너가 정상적으로 동작하고 있는지를 판단합니다.

예를 들어 데이터베이스 연결이 일시적으로 끊겼다는 이유로 Liveness Probe가 실패하도록 구성하면, 실제로는 복구 가능한 애플리케이션이 반복적으로 재시작될 수 있습니다.

반면 Readiness Probe에서 해당 상태를 검사하면 애플리케이션을 재시작하지 않고도 일시적으로 트래픽 전달 대상에서 제외할 수 있습니다.

### **11.6. 정상 종료 절차를 고려해야 합니다.**

Pod가 삭제될 때 애플리케이션이 처리 중인 요청이나 작업을 정상적으로 마무리할 수 있도록 Graceful Shutdown을 구현하는 것이 좋습니다.

특히 다음 사항을 함께 고려해야 합니다.

- `terminationGracePeriodSeconds` 설정
- `preStop` Hook 사용 여부
- SIGTERM 수신 후 애플리케이션 종료 처리
- 진행 중인 HTTP 요청이나 메시지 처리 작업의 완료
- Service Endpoint에서 트래픽이 제외되는 과정

종료 유예 시간이 부족하면 처리 중인 요청이 강제로 중단되거나 메시지가 정상적으로 처리되지 못할 수 있습니다.

### **11.7. 리소스 요청량과 제한을 적절하게 설정해야 합니다.**

CPU와 메모리의 `requests` 및 `limits`는 Pod의 스케줄링과 실행 안정성에 영향을 줍니다.

`requests`를 실제 사용량보다 지나치게 높게 설정하면 노드에 배치할 수 있는 Pod의 수가 줄어들 수 있습니다.

반대로 메모리 `limits`를 지나치게 낮게 설정하면 애플리케이션이 정상적으로 동작하는 중에도 OOM으로 종료될 수 있습니다.

애플리케이션의 실제 사용량과 부하 특성을 고려하여 적절한 값을 설정해야 합니다.

## **12. 핵심 정리**

Pod는 Kubernetes에서 컨테이너를 실행하기 위한 가장 작은 배포 단위입니다.

- 하나 이상의 컨테이너를 포함할 수 있습니다.
- 동일한 Pod 내부의 컨테이너들은 네트워크 네임스페이스와 볼륨을 공유할 수 있습니다.
- Pod는 고유한 IP 주소와 생명주기를 가집니다.
- Pod 내부 컨테이너들은 동일한 IP 주소와 포트 공간을 사용하며, `localhost`를 통해 통신할 수 있습니다.
- Pod Phase, Container State, Pod Conditions를 통해 실행 상태를 확인할 수 있습니다.
- 컨테이너 재시작과 Pod 객체 교체는 서로 다른 동작입니다.
- Init Container와 Sidecar Container를 통해 초기화 작업 및 보조 기능을 구성할 수 있습니다.
- Probe를 사용하여 컨테이너의 정상 동작 여부와 트래픽 처리 준비 상태를 확인할 수 있습니다.
- Resource Requests와 Limits를 통해 컨테이너의 CPU 및 메모리 사용량을 관리할 수 있습니다.
- Pod는 일시적인 객체이므로 영구적인 네트워크 식별자나 데이터 저장소로 사용해서는 안 됩니다.
- 실제 운영 환경에서는 Deployment, StatefulSet, DaemonSet 등의 상위 Workload 리소스를 통해 Pod를 관리하는 것이 일반적입니다.

**Pod는 Kubernetes의 모든 Workload 리소스가 실제 애플리케이션을 실행하기 위해 사용하는 기본 단위입니다.** Deployment, StatefulSet, DaemonSet, Job 등의 리소스를 이해하기 위해서는 먼저 Pod의 구조와 생명주기를 이해해야 합니다.

## **13. 참고자료**

- [Kubernetes Documentation - Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Kubernetes Documentation - Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Kubernetes Documentation - Pod Conditions](https://kubernetes.io/docs/concepts/workloads/pods/pod-condition/)
- [Kubernetes Documentation - Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
- [Kubernetes Documentation - Sidecar Containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
- [Kubernetes Documentation - Container Probes](https://kubernetes.io/docs/concepts/workloads/pods/probes/)
- [Kubernetes Documentation - Pod API Reference](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/)
- [Kubernetes Documentation - Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes Documentation - Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes Documentation - Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)