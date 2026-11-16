# kube-controller-manager

## 1. kube-controller-manager란?

kube-controller-manager는 Kubernetes Control Plane의 구성 요소로, Kubernetes 클러스터의 여러 Controller를 실행하고 관리하는 역할을 담당합니다.

Kubernetes는 사용자가 선언한 원하는 상태(Desired State)를 기준으로 클러스터의 실제 상태(Current State)를 지속적으로 확인하고 조정하는 선언적 시스템입니다.

예를 들어 사용자가 Deployment의 Replica 수를 3으로 설정하면 Kubernetes는 해당 Deployment가 관리하는 Pod가 3개 실행되도록 유지해야 합니다. Pod 하나에 장애가 발생하거나 삭제되면 이를 감지하고 새로운 Pod를 생성하여 원하는 상태를 유지합니다.

이러한 상태 조정 작업을 수행하는 구성 요소가 Controller이며, kube-controller-manager는 Kubernetes에 내장된 여러 Controller를 하나의 프로세스에서 실행합니다.

주요 역할은 다음과 같습니다.

- Kubernetes 리소스의 현재 상태를 지속적으로 감시합니다.
- 리소스에 선언된 원하는 상태와 실제 상태를 비교합니다.
- 두 상태 사이에 차이가 발생하면 이를 조정하기 위한 작업을 수행합니다.
- 필요한 Kubernetes 리소스를 생성, 수정 또는 삭제합니다.
- Node 장애, Pod 상태 변경, 리소스 수명 주기 등의 클러스터 이벤트에 대응합니다.

kube-controller-manager 자체가 모든 작업을 직접 수행하는 것은 아닙니다. 각각의 Controller가 특정 리소스와 상태를 담당하며, API Server를 통해 필요한 작업을 수행합니다.

## 2. Controller란?

### 2.1. Controller의 개념

Controller는 Kubernetes 클러스터의 특정 리소스 상태를 감시하고, 현재 상태를 원하는 상태에 가깝게 유지하기 위해 동작하는 제어 루프(Control Loop)입니다.

Controller는 일반적으로 다음과 같은 과정을 반복합니다.

1. API Server를 통해 관리 대상 리소스의 상태를 감시합니다.
2. 리소스에 선언된 원하는 상태를 확인합니다.
3. 현재 상태와 원하는 상태 사이의 차이를 계산합니다.
4. 차이를 줄이기 위해 필요한 작업을 수행합니다.
5. 변경된 상태를 다시 확인하고 동일한 과정을 반복합니다.

이러한 구조를 **Reconciliation(조정)**이라고 합니다.

예를 들어 Deployment에 `replicas: 3`이 설정되어 있고 실제로 실행 중인 Pod가 2개라면 Deployment Controller는 부족한 Pod를 생성하도록 ReplicaSet을 조정합니다.

반대로 실제 Pod가 4개라면 원하는 상태에 맞게 Pod 수를 줄이도록 조정합니다.

### 2.2. Desired State와 Current State

Kubernetes Controller는 Desired State와 Current State의 차이를 지속적으로 확인합니다.

| 구분 | 설명 | 예시 |
| --- | --- | --- |
| Desired State | 사용자가 선언한 원하는 상태입니다. | Deployment의 Replica 수가 3개입니다. |
| Current State | 현재 클러스터에서 관찰되는 실제 상태입니다. | 실제 실행 중인 Pod가 2개입니다. |
| Reconciliation | 두 상태의 차이를 줄이기 위해 수행하는 조정 과정입니다. | 부족한 Pod 1개를 생성합니다. |

Kubernetes에서는 일반적으로 Desired State를 리소스의 `spec`에 선언하고, Controller가 관찰한 상태와 처리 결과를 `status`에 반영합니다.

예를 들어 Deployment는 다음과 같이 선언할 수 있습니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
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
          image: nginx:latest
```

위 리소스에서 `spec.replicas: 3`은 사용자가 원하는 Pod 수를 나타냅니다.

Deployment Controller와 ReplicaSet Controller는 해당 상태를 기준으로 필요한 Pod가 생성되고 유지되도록 조정합니다.

### 2.3. Reconciliation Loop

Reconciliation Loop는 Controller가 현재 상태와 원하는 상태를 비교하고 필요한 작업을 반복적으로 수행하는 구조입니다.

```text
          +----------------------+
          |   Desired State      |
          |   (spec.replicas: 3)  |
          +----------------------+
                     |
                     v
          +----------------------+
          |   Current State      |
          |   (Running Pods: 2)  |
          +----------------------+
                     |
                     v
          +----------------------+
          |    Reconciliation    |
          |  상태 차이 확인 및    |
          |    필요한 작업 결정   |
          +----------------------+
                     |
                     v
          +----------------------+
          |   API Server 요청    |
          |   Pod 생성 등 작업   |
          +----------------------+
                     |
                     v
          +----------------------+
          |   Updated State      |
          |   (Running Pods: 3)  |
          +----------------------+
                     |
                     +------ 반복
```

Controller는 특정 이벤트가 발생했을 때만 동작하는 것이 아니라, 지속적으로 상태를 확인하고 필요한 경우 조정 작업을 수행합니다.

또한 API Server의 Watch 이벤트를 통해 변경 사항을 빠르게 감지할 수 있으며, 재동기화(Reconciliation)를 통해 상태를 다시 확인할 수도 있습니다.

따라서 Kubernetes의 Controller는 특정 이벤트를 한 번 처리하고 종료하는 이벤트 핸들러와 구분해야 합니다.

### 2.4. Controller의 독립성

Kubernetes는 하나의 거대한 Controller가 모든 클러스터 상태를 관리하는 대신, 여러 Controller가 각각 특정 영역을 담당하도록 설계되어 있습니다.

예를 들어 다음과 같이 역할을 분리할 수 있습니다.

| Controller | 주요 책임 |
| --- | --- |
| Deployment Controller | Deployment의 원하는 Replica 상태를 유지합니다. |
| ReplicaSet Controller | ReplicaSet이 요구하는 Pod 수를 유지합니다. |
| Job Controller | Job이 완료될 때까지 필요한 Pod를 관리합니다. |
| Node Controller | Node의 상태를 감시하고 장애에 대응합니다. |
| Namespace Controller | Namespace 삭제 및 관련 리소스 정리를 관리합니다. |
| EndpointSlice Controller | Service와 연결되는 EndpointSlice를 관리합니다. |

각 Controller는 담당하는 리소스와 작업이 구분되어 있으므로 특정 Controller에 문제가 발생하더라도 다른 Controller의 작업이 반드시 함께 중단되는 것은 아닙니다.

## 3. kube-controller-manager의 구조

### 3.1. Controller Manager의 역할

kube-controller-manager는 여러 Controller를 하나의 실행 프로세스에서 관리하는 구성 요소입니다.

논리적으로 각 Controller는 독립적인 제어 루프이지만, 운영 및 배포의 복잡도를 줄이기 위해 하나의 바이너리에 포함되어 실행됩니다.

```text
+------------------------------------------+
|           kube-controller-manager        |
|                                          |
|  +------------------------------------+  |
|  |          Controller Loops          |  |
|  |                                    |  |
|  |  +------------------------------+  |  |
|  |  |   Deployment Controller      |  |  |
|  |  +------------------------------+  |  |
|  |  |   ReplicaSet Controller      |  |  |
|  |  +------------------------------+  |  |
|  |  |   Node Controller             |  |  |
|  |  +------------------------------+  |  |
|  |  |   Job Controller              |  |  |
|  |  +------------------------------+  |  |
|  |  |   Namespace Controller        |  |  |
|  |  +------------------------------+  |  |
|  |  |   ServiceAccount Controller   |  |  |
|  |  +------------------------------+  |  |
|  |  |   기타 Built-in Controllers   |  |  |
|  |  +------------------------------+  |  |
|  +------------------------------------+  |
|                                          |
+------------------------------------------+
                     |
                     v
              kube-apiserver
```

위 구조에서 각 Controller는 담당하는 리소스의 상태를 감시하고 API Server를 통해 필요한 변경 사항을 반영합니다.

### 3.2. Built-in Controllers

Kubernetes에는 클러스터의 기본 동작을 유지하기 위한 여러 Built-in Controller가 포함되어 있습니다.

대표적인 Controller는 다음과 같습니다.

| Controller | 주요 역할 |
| --- | --- |
| Deployment Controller | Deployment의 Replica 상태를 관리합니다. |
| ReplicaSet Controller | ReplicaSet이 요구하는 Pod 수를 유지합니다. |
| ReplicationController | ReplicationController가 요구하는 Pod 수를 유지합니다. |
| StatefulSet Controller | StatefulSet의 Pod 생성, 삭제 및 순서와 관련된 상태를 관리합니다. |
| DaemonSet Controller | 지정된 Node에 DaemonSet Pod가 실행되도록 관리합니다. |
| Job Controller | Job의 작업 완료를 위해 필요한 Pod를 관리합니다. |
| CronJob Controller | 예약된 일정에 따라 Job이 생성되도록 관리합니다. |
| Node Controller | Node 상태를 감시하고 장애에 대응합니다. |
| Namespace Controller | Namespace의 종료 및 관련 리소스 정리를 관리합니다. |
| EndpointSlice Controller | Service와 연결되는 EndpointSlice를 관리합니다. |
| ServiceAccount Controller | Namespace의 기본 ServiceAccount 등 관련 리소스를 관리합니다. |
| ResourceQuota Controller | ResourceQuota의 사용량과 상태를 관리합니다. |
| PersistentVolume Controller | PersistentVolume과 PersistentVolumeClaim의 바인딩 및 관련 상태를 관리합니다. |
| Garbage Collector Controller | 더 이상 필요하지 않은 Kubernetes 리소스를 정리합니다. |
| Horizontal Pod Autoscaler Controller | 메트릭을 기반으로 대상 워크로드의 Replica 수를 조정합니다. |

위 목록은 대표적인 예시이며, Kubernetes 버전과 활성화된 Controller에 따라 실제 구성은 달라질 수 있습니다.

### 3.3. kube-controller-manager와 개별 Controller의 관계

kube-controller-manager와 개별 Controller는 서로 다른 수준의 개념입니다.

- kube-controller-manager는 여러 Built-in Controller를 실행하고 관리하는 프로세스입니다.
- 개별 Controller는 특정 Kubernetes 리소스의 상태를 감시하고 조정하는 제어 루프입니다.

따라서 Deployment Controller나 Node Controller를 kube-controller-manager와 동등한 수준의 별도 Control Plane 컴포넌트로 이해해서는 안 됩니다.

일반적으로 이들은 kube-controller-manager 내부에서 실행되는 Controller입니다.

## 4. 주요 Controller 상세

### 4.1. Deployment Controller

Deployment Controller는 Deployment 리소스의 원하는 상태를 유지하기 위한 Controller입니다.

Deployment는 애플리케이션의 배포 및 업데이트를 선언적으로 관리하기 위한 리소스입니다.

Deployment Controller는 Deployment의 상태를 감시하고, 필요한 ReplicaSet이 생성되거나 업데이트되도록 관리합니다.

Deployment Controller가 Pod를 직접 생성하는 것은 아닙니다. 일반적으로 ReplicaSet을 생성하고 관리하며, 실제 Pod 생성과 삭제는 ReplicaSet Controller가 담당합니다.

전체 관계는 다음과 같습니다.

```text
        Deployment
             |
             v
    Deployment Controller
             |
             | ReplicaSet 생성 및 관리
             v
         ReplicaSet
             |
             v
     ReplicaSet Controller
             |
             | Pod 생성 및 관리
             v
            Pod
```

예를 들어 다음과 같이 Deployment를 생성할 수 있습니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
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
          image: nginx:latest
```

Deployment Controller는 해당 Deployment를 관찰하고 필요한 ReplicaSet 상태를 유지합니다.

이후 ReplicaSet Controller가 실제 Pod 수를 조정합니다.

### 4.2. ReplicaSet Controller

ReplicaSet Controller는 ReplicaSet에 선언된 Replica 수를 유지하는 Controller입니다.

예를 들어 ReplicaSet의 원하는 Pod 수가 3개인데 실제로 관리 중인 Pod가 2개라면 새로운 Pod를 생성하도록 API Server에 요청합니다.

반대로 실제 Pod가 4개라면 초과한 Pod를 삭제하여 원하는 상태에 맞춥니다.

```text
Desired Replicas = 3
Current Replicas = 2

          |
          v

   ReplicaSet Controller
          |
          | Pod 1개 생성 요청
          v

Desired Replicas = 3
Current Replicas = 3
```

ReplicaSet Controller는 Pod의 개수를 유지하는 역할을 담당하며, 실제 Pod를 실행할 Node를 선택하는 작업은 kube-scheduler가 수행합니다.

### 4.3. Node Controller

Node Controller는 Kubernetes 클러스터에 등록된 Node의 상태를 감시하고 장애 상황에 대응하는 Controller입니다.

주요 역할은 다음과 같습니다.

- Node의 상태를 감시합니다.
- Node가 정상적으로 응답하지 않는 경우 상태를 변경합니다.
- Node 장애와 관련된 Taint를 관리합니다.
- 장애가 발생한 Node의 Pod 처리와 관련된 후속 조치를 수행합니다.

일반적으로 kubelet은 주기적으로 API Server에 Node 상태를 보고합니다.

Node Controller는 이러한 상태 정보를 확인하여 Node가 정상적으로 동작하는지 판단합니다.

예를 들어 Node가 일정 시간 동안 응답하지 않으면 Node의 상태가 `Ready`에서 `Unknown` 또는 `False`로 변경될 수 있습니다.

이후 Kubernetes는 설정된 정책에 따라 해당 Node에 실행 중인 Pod의 퇴출(Eviction) 등을 진행할 수 있습니다.

Node Controller는 Node 장애를 감지하고 상태를 조정하는 역할을 담당하며, 실제 Node에서 실행 중인 컨테이너의 생명주기를 직접 관리하는 것은 아닙니다.

### 4.4. Job Controller

Job Controller는 Job 리소스가 지정한 작업을 완료하도록 Pod를 관리하는 Controller입니다.

Deployment나 ReplicaSet은 지속적으로 실행되는 애플리케이션의 상태를 유지하는 데 주로 사용되지만, Job은 특정 작업을 실행하고 완료하는 데 사용됩니다.

예를 들어 데이터 처리, 배치 작업, 일회성 스크립트 실행 등에 활용할 수 있습니다.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  completions: 1
  parallelism: 1
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: busybox
          command:
            - sh
            - -c
            - echo "Job completed"
```

Job Controller는 Job의 완료 조건을 확인하고 필요한 Pod를 생성하거나 실패한 작업을 재시도하도록 관리합니다.

Job이 요구하는 작업이 완료되면 Job의 상태를 업데이트합니다.

### 4.5. CronJob Controller

CronJob Controller는 지정된 일정에 따라 Job이 생성되도록 관리하는 Controller입니다.

CronJob은 Linux의 Cron 표현식과 유사한 스케줄을 사용하여 주기적인 작업을 실행할 수 있습니다.

예를 들어 매일 자정에 백업 작업을 수행하도록 구성할 수 있습니다.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 0 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: busybox
              command:
                - sh
                - -c
                - echo "Backup started"
```

CronJob Controller는 CronJob의 실행 일정을 확인하고 해당 시점에 Job을 생성합니다.

실제 작업 실행과 완료 여부 관리는 Job Controller가 담당합니다.

### 4.6. StatefulSet Controller

StatefulSet Controller는 StatefulSet에 정의된 Pod를 관리하는 Controller입니다.

StatefulSet은 Pod의 안정적인 식별자와 순서, 영속적인 스토리지 연결 등이 필요한 애플리케이션에 사용됩니다.

대표적인 사용 사례는 다음과 같습니다.

- 데이터베이스 클러스터
- 분산 메시징 시스템
- 상태를 유지하는 분산 애플리케이션

StatefulSet Controller는 다음과 같은 작업을 수행합니다.

- StatefulSet에 정의된 Pod 수를 유지합니다.
- Pod의 순서와 식별자를 관리합니다.
- Pod 생성 및 삭제 순서를 관리합니다.
- 관련 PersistentVolumeClaim의 상태를 관리합니다.

### 4.7. DaemonSet Controller

DaemonSet Controller는 지정된 Node에 DaemonSet Pod가 실행되도록 관리하는 Controller입니다.

일반적인 Deployment는 전체 Replica 수를 유지하는 데 목적이 있지만, DaemonSet은 특정 조건을 만족하는 Node마다 Pod를 배치하는 데 목적이 있습니다.

주요 사용 사례는 다음과 같습니다.

- Node 로그 수집 에이전트
- 모니터링 에이전트
- 네트워크 플러그인
- 스토리지 관련 에이전트

DaemonSet Controller는 Node 상태와 DaemonSet 설정을 확인하여 필요한 Pod가 각 Node에 존재하도록 조정합니다.

Pod를 특정 Node에 배치하는 과정에는 kube-scheduler가 관여할 수 있으며, DaemonSet Controller가 Node별 Pod를 직접 실행하는 것은 아닙니다.

### 4.8. Namespace Controller

Namespace Controller는 Namespace의 생명주기를 관리하는 Controller입니다.

Namespace가 삭제 요청을 받으면 해당 Namespace의 상태가 `Terminating`으로 변경됩니다.

Namespace Controller는 해당 Namespace에 포함된 리소스가 정리되도록 처리하고, 정리 작업이 완료되면 Namespace 삭제가 마무리되도록 관리합니다.

이러한 과정을 통해 Namespace 삭제 시 관련 리소스가 함께 정리될 수 있도록 합니다.

### 4.9. EndpointSlice Controller

EndpointSlice Controller는 Service와 연결되는 Pod의 네트워크 Endpoint 정보를 관리하는 Controller입니다.

Service는 일반적으로 Label Selector를 사용하여 연결할 Pod를 선택합니다.

EndpointSlice Controller는 Service의 Selector와 Pod 상태 등을 확인하여 Service가 사용할 수 있는 EndpointSlice를 생성하거나 업데이트합니다.

```text
          Service
             |
             | Selector
             v
            Pod
             |
             v
    EndpointSlice Controller
             |
             v
        EndpointSlice
             |
             v
         kube-proxy
```

EndpointSlice에는 Service가 연결할 수 있는 Endpoint의 주소와 관련 상태 정보가 포함됩니다.

이후 kube-proxy 또는 대체 네트워크 구현체가 EndpointSlice 정보를 활용하여 Service 트래픽을 처리합니다.

## 5. Controller의 동작 방식

### 5.1. API Server와 Watch

Kubernetes Controller는 일반적으로 API Server를 통해 리소스의 상태를 감시합니다.

각 Controller는 담당하는 리소스의 변경 사항을 감지하고, 변경된 상태를 기준으로 Reconciliation을 수행합니다.

이때 Kubernetes API의 Watch 메커니즘을 활용하여 리소스 변경 사항을 감지할 수 있습니다.

대표적인 처리 흐름은 다음과 같습니다.

1. Controller가 API Server를 통해 관리 대상 리소스를 조회합니다.
2. Watch를 통해 리소스 변경 이벤트를 감지합니다.
3. 변경된 리소스의 식별 정보를 내부 작업 큐에 등록합니다.
4. Worker가 큐에서 작업을 가져옵니다.
5. 현재 상태와 원하는 상태를 비교합니다.
6. 필요한 변경 작업을 API Server에 요청합니다.

Controller는 API Server를 통해 리소스를 관리하므로 etcd에 직접 접근하지 않습니다.

### 5.2. Informer

Informer는 Kubernetes Controller가 리소스의 변경 사항을 효율적으로 감지하고 로컬 캐시를 유지할 수 있도록 지원하는 Kubernetes Client 라이브러리의 구성 요소입니다.

일반적으로 Informer는 다음과 같은 기능을 제공합니다.

- API Server를 통해 리소스 목록을 조회합니다.
- Watch를 통해 리소스 변경 사항을 감지합니다.
- 변경된 리소스를 로컬 캐시에 반영합니다.
- 리소스 변경 이벤트를 등록된 Event Handler에 전달합니다.
- Controller가 캐시를 활용하여 리소스 상태를 조회할 수 있도록 합니다.

Informer를 사용하면 Controller가 매번 API Server에 직접 요청하여 전체 리소스를 조회하는 부담을 줄일 수 있습니다.

일반적인 구조는 다음과 같습니다.

```text
             kube-apiserver
                    |
                    | List / Watch
                    v
                Informer
                    |
                    v
               Local Cache
                    |
                    v
              Event Handler
                    |
                    v
              Work Queue
                    |
                    v
                 Worker
                    |
                    v
              Reconciliation
                    |
                    v
             kube-apiserver
```

위 구조는 Kubernetes Controller에서 흔히 사용하는 Client-go 기반 처리 패턴을 개념적으로 표현한 것입니다.

### 5.3. Work Queue

Work Queue는 Controller가 처리해야 하는 작업을 관리하는 내부 큐입니다.

Informer가 리소스 변경 이벤트를 감지하면 일반적으로 해당 리소스의 식별 정보를 Work Queue에 등록합니다.

Worker는 큐에서 작업을 가져와 실제 Reconciliation을 수행합니다.

Work Queue를 사용하면 다음과 같은 장점이 있습니다.

- 이벤트 처리와 실제 작업 수행을 분리할 수 있습니다.
- 여러 Worker를 통해 작업을 병렬 처리할 수 있습니다.
- 동일한 리소스에 대한 중복 작업을 효율적으로 관리할 수 있습니다.
- 실패한 작업을 재시도할 수 있습니다.
- API Server 요청이 일시적으로 실패하더라도 작업을 다시 수행할 수 있습니다.

Work Queue에는 일반적으로 리소스 객체 전체보다 Namespace와 Name을 조합한 Key를 등록합니다.

Worker는 해당 Key를 사용하여 캐시에서 최신 리소스 상태를 조회한 후 조정 작업을 수행합니다.

### 5.4. Reconciliation과 Idempotency

Controller는 동일한 작업이 여러 번 수행되더라도 최종 상태가 일관되게 유지되도록 설계하는 것이 중요합니다.

이를 **Idempotency(멱등성)**라고 합니다.

예를 들어 Deployment Controller가 필요한 ReplicaSet을 생성하는 과정에서 동일한 이벤트가 여러 번 전달되더라도 중복된 ReplicaSet이 계속 생성되어서는 안 됩니다.

Controller는 현재 상태를 다시 확인하고 이미 원하는 상태가 충족되어 있다면 추가 변경 작업을 수행하지 않아야 합니다.

이러한 특성은 Kubernetes에서 이벤트가 중복 전달되거나 작업이 재시도되는 상황에서도 안정적으로 상태를 유지할 수 있도록 합니다.

또한 Controller는 모든 작업이 한 번만 실행된다고 가정해서는 안 됩니다. 일반적으로 **At-least-once 처리와 재시도에 안전한 조정 로직**을 고려하여 구현합니다.

## 6. kube-controller-manager와 cloud-controller-manager

### 6.1. 역할 분리

Kubernetes에는 kube-controller-manager 외에도 cloud-controller-manager라는 구성 요소가 있습니다.

두 컴포넌트 모두 Controller를 실행하지만, 담당하는 영역이 다릅니다.

- kube-controller-manager: Kubernetes 클러스터 내부의 일반적인 리소스 상태와 동작을 관리합니다.
- cloud-controller-manager: 클라우드 제공자의 API와 연동하여 클라우드 인프라와 관련된 상태를 관리합니다.

이러한 분리는 Kubernetes의 핵심 제어 로직과 클라우드 제공자별 구현을 분리하기 위한 구조입니다.

### 6.2. kube-controller-manager가 담당하는 영역

kube-controller-manager는 Kubernetes 자체의 핵심 동작을 유지하기 위한 Controller를 실행합니다.

대표적인 영역은 다음과 같습니다.

- Deployment 및 ReplicaSet 관리
- StatefulSet 및 DaemonSet 관리
- Job 및 CronJob 관리
- Node 상태 관리
- Namespace 관리
- EndpointSlice 관리
- ResourceQuota 관리
- PersistentVolume 관련 관리

### 6.3. cloud-controller-manager가 담당하는 영역

cloud-controller-manager는 클라우드 환경에 종속적인 Controller를 실행합니다.

대표적인 영역은 다음과 같습니다.

- Cloud Node Controller
- Route Controller
- Service Controller

예를 들어 Service의 Type이 `LoadBalancer`인 경우 cloud-controller-manager의 Service Controller가 클라우드 제공자의 API를 호출하여 외부 Load Balancer를 생성하고 관리할 수 있습니다.

두 컴포넌트의 상세 역할과 동작 방식은 별도의 `cloud-controller-manager.md` 문서에서 다룹니다.

## 7. 고가용성 및 운영

### 7.1. Leader Election

kube-controller-manager는 여러 인스턴스를 실행하여 고가용성을 구성할 수 있습니다.

일반적으로 여러 kube-controller-manager 인스턴스를 실행하고 Leader Election을 통해 하나의 인스턴스만 활성 Leader로 동작하도록 구성합니다.

```text
              kube-apiserver
                     |
          +----------+----------+
          |          |          |
          v          v          v
      KCM Node 1  KCM Node 2  KCM Node 3
          |          |          |
          +----------+----------+
                     |
                Leader Election
                     |
                     v
               Active Leader
                     |
                     v
            Controller 실행 및 조정
```

Leader Election은 일반적으로 Kubernetes API의 Lease 리소스를 사용하여 수행합니다.

Leader가 아닌 인스턴스는 대기 상태로 유지되며, 기존 Leader가 정상적으로 Lease를 갱신하지 못하면 다른 인스턴스가 새로운 Leader로 선출될 수 있습니다.

이러한 구조는 여러 kube-controller-manager 인스턴스가 동시에 동일한 Controller 작업을 수행하는 상황을 방지하고, 장애 발생 시 다른 인스턴스가 작업을 이어받을 수 있도록 합니다.

단, Leader Election이 활성화되어 있다고 해서 모든 Controller의 작업이 완전히 중단 없이 이어지는 것은 아닙니다. Leader 변경 과정에서 일시적인 지연이 발생할 수 있습니다.

### 7.2. Controller별 동시 처리

kube-controller-manager는 Controller별로 작업을 병렬 처리할 수 있도록 동시 실행 Worker 수를 설정할 수 있습니다.

예를 들어 Deployment Controller는 여러 Deployment를 동시에 처리할 수 있으며, Job Controller 역시 여러 Job을 병렬로 관리할 수 있습니다.

대표적인 설정 옵션은 다음과 같습니다.

| 옵션 | 설명 |
| --- | --- |
| `--concurrent-deployment-syncs` | 동시에 처리할 Deployment 작업 수를 지정합니다. |
| `--concurrent-replicaset-syncs` | 동시에 처리할 ReplicaSet 작업 수를 지정합니다. |
| `--concurrent-job-syncs` | 동시에 처리할 Job 작업 수를 지정합니다. |
| `--concurrent-daemonset-syncs` | 동시에 처리할 DaemonSet 작업 수를 지정합니다. |
| `--concurrent-namespace-syncs` | 동시에 처리할 Namespace 작업 수를 지정합니다. |
| `--concurrent-horizontal-pod-autoscaler-syncs` | 동시에 처리할 HPA 작업 수를 지정합니다. |

동시 처리 수를 늘리면 리소스 변경에 대한 응답성이 향상될 수 있지만, API Server 요청량과 kube-controller-manager의 CPU 및 Memory 사용량도 증가할 수 있습니다.

따라서 클러스터 규모와 Controller의 처리량을 고려하여 설정해야 합니다.

### 7.3. Controller 활성화 및 비활성화

kube-controller-manager는 실행 옵션을 통해 특정 Controller를 활성화하거나 비활성화할 수 있습니다.

대표적인 옵션은 다음과 같습니다.

```bash
kube-controller-manager \
  --controllers=*,bootstrapsigner-controller,-tokencleaner-controller
```

주요 설정 방식은 다음과 같습니다.

| 설정 | 설명 |
| --- | --- |
| `*` | 기본적으로 활성화되는 Controller를 활성화합니다. |
| `controller-name` | 지정한 Controller를 활성화합니다. |
| `-controller-name` | 지정한 Controller를 비활성화합니다. |

Controller의 기본 활성화 여부와 지원 목록은 Kubernetes 버전에 따라 달라질 수 있습니다.

또한 일부 Controller는 다른 Control Plane 컴포넌트의 설정과 함께 관리해야 하므로, 운영 환경에서 임의로 비활성화하는 것은 주의해야 합니다.

### 7.4. 주요 실행 옵션

kube-controller-manager는 실행 옵션과 설정을 통해 API Server 연결, Leader Election, Controller 동시 처리, 클러스터 네트워크 등의 동작을 구성할 수 있습니다.

대표적인 옵션은 다음과 같습니다.

| 옵션 | 설명 |
| --- | --- |
| `--kubeconfig` | API Server에 접근할 때 사용할 kubeconfig 파일을 지정합니다. |
| `--master` | API Server 주소를 지정합니다. |
| `--leader-elect` | Leader Election 사용 여부를 설정합니다. |
| `--leader-elect-lease-duration` | Leader Lease의 유효 시간을 지정합니다. |
| `--leader-elect-renew-deadline` | Leader가 Lease를 갱신하기 위해 허용되는 시간을 지정합니다. |
| `--leader-elect-retry-period` | Leader Election 작업의 재시도 간격을 지정합니다. |
| `--controllers` | 실행할 Controller 목록을 지정합니다. |
| `--cluster-cidr` | 클러스터 Pod 네트워크에 사용할 CIDR을 지정합니다. |
| `--allocate-node-cidrs` | Node별 Pod CIDR 할당 여부를 설정합니다. |
| `--node-monitor-period` | Node 상태를 확인하는 주기를 지정합니다. |
| `--node-monitor-grace-period` | Node가 응답하지 않는 상태를 허용하는 시간을 지정합니다. |
| `--use-service-account-credentials` | 개별 Controller가 별도의 ServiceAccount 자격 증명을 사용하도록 설정합니다. |

실제 지원 옵션과 기본값은 Kubernetes 버전에 따라 달라질 수 있으므로 운영 중인 버전의 Command Line Reference를 확인해야 합니다.

## 8. 다른 Control Plane 컴포넌트와의 관계

kube-controller-manager는 API Server를 중심으로 다른 Control Plane 컴포넌트와 협력합니다.

### 8.1. kube-apiserver

kube-apiserver는 kube-controller-manager가 Kubernetes 리소스를 조회하고 변경하기 위해 사용하는 API 인터페이스입니다.

kube-controller-manager 내부의 Controller는 API Server를 통해 다음 작업을 수행합니다.

- 리소스 상태 조회
- 리소스 변경 사항 감시
- 새로운 리소스 생성
- 기존 리소스 수정
- 불필요한 리소스 삭제
- 리소스의 Status 업데이트

Controller는 etcd에 직접 접근하지 않으며, Kubernetes API를 통해 리소스 상태를 관리합니다.

### 8.2. etcd

etcd는 Kubernetes 클러스터의 리소스 상태를 영구적으로 저장하는 분산 Key-Value 저장소입니다.

kube-controller-manager가 API Server를 통해 리소스를 생성하거나 수정하면 API Server가 해당 변경 사항을 etcd에 저장합니다.

따라서 kube-controller-manager는 etcd에 직접 데이터를 저장하는 구성 요소가 아니라, API Server를 통해 리소스 상태를 조정하는 구성 요소입니다.

### 8.3. kube-scheduler

kube-scheduler는 아직 Node가 할당되지 않은 Pod를 실행할 적절한 Node에 배치하는 역할을 담당합니다.

일반적으로 Deployment Controller가 Deployment에 필요한 ReplicaSet을 관리하고, ReplicaSet Controller가 필요한 Pod를 생성하면 kube-scheduler가 해당 Pod를 실행할 Node를 선택합니다.

두 컴포넌트의 책임은 다음과 같이 구분됩니다.

| 컴포넌트 | 역할 |
| --- | --- |
| kube-controller-manager | 원하는 상태를 유지하기 위해 필요한 Kubernetes 리소스를 생성하고 조정합니다. |
| kube-scheduler | 생성된 Pod를 실행할 적절한 Node를 선택하고 할당합니다. |

### 8.4. kubelet

kubelet은 각 Node에서 실행되며, 해당 Node에 할당된 Pod의 실제 실행 상태를 관리합니다.

kube-controller-manager가 Pod를 생성하면 kube-scheduler가 Node를 할당하고, 해당 Node의 kubelet이 Pod를 확인하여 컨테이너 실행을 진행합니다.

| 컴포넌트 | 역할 |
| --- | --- |
| kube-controller-manager | 원하는 상태에 맞게 Pod 등의 리소스를 생성하고 조정합니다. |
| kube-scheduler | Pod가 실행될 Node를 선택합니다. |
| kubelet | 할당된 Pod의 컨테이너 실행과 상태 관리를 수행합니다. |

### 8.5. 전체 컴포넌트 상호작용

Deployment를 생성하고 Pod가 실행되는 과정을 기준으로 각 컴포넌트의 역할을 정리하면 다음과 같습니다.

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
                     | Deployment 상태 저장
                     v
          kube-controller-manager
                     |
                     | Deployment Controller
                     | ReplicaSet 생성 및 관리
                     v
                kube-apiserver
                     |
                     v
                ReplicaSet
                     |
                     | ReplicaSet Controller
                     | Pod 생성
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
                     | 컨테이너 실행
                     v
             Container Runtime
                     |
                     v
                Running Pod
```

위 과정에서 중요한 점은 각 컴포넌트가 서로의 역할을 대신하지 않는다는 것입니다.

1. kube-apiserver는 Kubernetes API 요청을 처리합니다.
2. etcd는 Kubernetes 리소스 상태를 저장합니다.
3. kube-controller-manager는 선언된 원하는 상태를 유지하기 위해 리소스를 조정합니다.
4. kube-scheduler는 Pod가 실행될 Node를 결정합니다.
5. kubelet은 할당된 Pod를 실제로 실행합니다.

이러한 역할 분리를 통해 Kubernetes는 선언적인 리소스 관리와 분산된 제어 루프를 기반으로 클러스터를 운영합니다.

## 9. 정리

kube-controller-manager는 Kubernetes Control Plane에서 여러 Built-in Controller를 실행하고 관리하는 구성 요소입니다.

각 Controller는 담당하는 리소스의 현재 상태와 원하는 상태를 비교하고, 그 차이를 줄이기 위한 Reconciliation을 지속적으로 수행합니다.

kube-controller-manager를 이해할 때는 다음 개념을 특히 구분해서 이해해야 합니다.

- **Controller와 kube-controller-manager:** Controller는 특정 리소스의 상태를 조정하는 제어 루프이며, kube-controller-manager는 여러 Built-in Controller를 하나의 프로세스에서 실행하고 관리하는 구성 요소입니다.
- **Desired State와 Current State:** Desired State는 사용자가 선언한 원하는 상태이며, Current State는 현재 클러스터에서 관찰되는 상태입니다. Controller는 두 상태 사이의 차이를 지속적으로 조정합니다.
- **Reconciliation Loop:** Controller가 상태를 감시하고 필요한 작업을 수행한 뒤 다시 상태를 확인하는 반복적인 제어 과정입니다. Kubernetes는 이러한 구조를 통해 장애와 상태 변경에 대응합니다.
- **Informer와 Work Queue:** Informer는 리소스 변경 사항을 감지하고 캐시를 유지하며, Work Queue는 Controller가 처리해야 할 작업을 관리합니다. 이를 통해 이벤트 감지와 실제 상태 조정 작업을 효율적으로 분리할 수 있습니다.
- **kube-controller-manager와 cloud-controller-manager:** kube-controller-manager는 Kubernetes 자체의 핵심 상태 관리 로직을 담당하고, cloud-controller-manager는 클라우드 제공자별 인프라와 관련된 제어 로직을 담당합니다.

kube-controller-manager는 Kubernetes에서 선언한 원하는 상태가 실제 클러스터에 반영되고 유지되도록 하는 핵심 구성 요소입니다.

**Kubernetes의 선언적 운영 방식은 Controller가 Desired State와 Current State의 차이를 지속적으로 조정하는 Reconciliation 메커니즘을 통해 구현됩니다.**

## 10. 참고자료

### Kubernetes 공식 문서

- [Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)
- [Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
- [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)
- [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Horizontal Pod Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)

### Kubernetes API 및 개발 관련 문서

- [kube-controller-manager Command Line Reference](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)
- [Client-go Informers](https://pkg.go.dev/k8s.io/client-go/tools/cache)
- [Kubernetes Sample Controller](https://github.com/kubernetes/sample-controller)
- [Kubernetes Controller Runtime Concepts](https://book.kubebuilder.io/reference/good-practices.html)
- [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)