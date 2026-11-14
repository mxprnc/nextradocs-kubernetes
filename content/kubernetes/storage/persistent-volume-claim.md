# PersistentVolumeClaim (PVC)

## 1. PersistentVolumeClaim이란?

PersistentVolumeClaim(PVC)은 Kubernetes에서 애플리케이션이 필요한 영구 스토리지를 요청하기 위한 리소스입니다.

PersistentVolume(PV)이 클러스터에서 사용할 수 있는 실제 스토리지를 나타낸다면, PVC는 애플리케이션이 요구하는 스토리지의 용량, 접근 모드 및 스토리지 유형 등을 선언하는 역할을 수행합니다.

애플리케이션은 실제 스토리지의 위치나 구현 방식을 직접 지정하는 대신 PVC를 통해 필요한 스토리지를 요청할 수 있습니다.

Kubernetes는 PVC의 요구사항을 확인하여 적합한 PV를 찾아 바인딩하거나, StorageClass를 통해 새로운 PV를 동적으로 생성합니다.

PVC의 주요 특징은 다음과 같습니다.

- 애플리케이션이 필요한 스토리지의 용량을 요청할 수 있습니다.
- 접근 모드와 Volume Mode를 지정할 수 있습니다.
- 특정 StorageClass를 지정하여 원하는 스토리지 유형을 요청할 수 있습니다.
- 기존 PV를 지정하거나 Kubernetes가 적합한 PV를 선택하도록 할 수 있습니다.
- 동적 프로비저닝을 통해 필요한 스토리지를 자동으로 생성할 수 있습니다.
- Pod의 생명주기와 독립적으로 관리되므로 여러 Pod에서 동일한 PVC를 사용할 수 있습니다.

PVC는 Namespace에 종속되는 리소스(Namespace-scoped)입니다. 따라서 Pod에서 PVC를 참조할 때는 해당 Pod와 동일한 Namespace에 존재하는 PVC를 사용해야 합니다.

반면 PV는 클러스터 범위의 리소스(Cluster-scoped)이므로 특정 Namespace에 종속되지 않습니다.

## 2. PersistentVolumeClaim의 기본 구조

PersistentVolumeClaim은 `apiVersion: v1`, `kind: PersistentVolumeClaim`으로 정의합니다.

다음은 10Gi 용량의 스토리지를 요청하는 PVC 예제입니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: application-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce

  volumeMode: Filesystem

  storageClassName: standard

  resources:
    requests:
      storage: 10Gi
```

각 필드의 역할은 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `metadata.name` | PVC의 이름입니다. |
| `metadata.namespace` | PVC가 생성될 Namespace입니다. |
| `accessModes` | 요청하는 스토리지의 접근 모드입니다. |
| `volumeMode` | 파일 시스템 또는 원시 블록 장치 형태를 지정합니다. |
| `storageClassName` | 요청할 StorageClass를 지정합니다. |
| `resources.requests.storage` | 필요한 스토리지 용량을 지정합니다. |

위 예제에서는 `standard` StorageClass에 속하는 10Gi 용량의 `ReadWriteOnce` 스토리지를 요청합니다.

Kubernetes는 해당 요구사항을 충족하는 PV를 탐색하거나 동적 프로비저닝을 통해 새로운 스토리지를 생성합니다.

## 3. PersistentVolumeClaim과 PersistentVolume의 관계

PVC와 PV는 Binding 과정을 통해 연결됩니다.

각 리소스의 역할은 다음과 같습니다.

| 리소스 | 역할 |
|---|---|
| PersistentVolume (PV) | 클러스터에서 사용할 수 있는 영구 스토리지를 나타냅니다. |
| PersistentVolumeClaim (PVC) | 애플리케이션이 필요한 스토리지의 조건을 요청합니다. |
| StorageClass | 스토리지를 동적으로 생성하기 위한 프로비저닝 방식을 정의합니다. |

일반적인 연결 구조는 다음과 같습니다.

```text
              Administrator
                    │
                    │ Prepare Storage
                    ▼
              Actual Storage
                    │
                    │
                    ▼
            PersistentVolume
                    ▲
                    │ Binding
                    ▼
         PersistentVolumeClaim
                    ▲
                    │ Reference
                    ▼
                   Pod
                    │
                    ▼
               Application
```

정적 프로비저닝에서는 관리자가 사전에 생성한 PV와 PVC가 바인딩됩니다.

동적 프로비저닝에서는 PVC가 요청한 조건에 따라 StorageClass의 프로비저너가 실제 스토리지를 생성하고, 이에 대응하는 PV가 자동으로 생성되어 PVC와 바인딩됩니다.

### 3.1. Binding 조건

Kubernetes는 PVC와 PV의 요구사항을 비교하여 바인딩을 수행합니다.

주요 조건은 다음과 같습니다.

- 스토리지 용량: PV가 제공하는 용량이 PVC의 요청 용량 이상이어야 합니다.
- 접근 모드: PV가 지원하는 접근 모드가 PVC의 요청 조건을 충족해야 합니다.
- Volume Mode: PV와 PVC의 Volume Mode가 호환되어야 합니다.
- StorageClass: PV와 PVC의 StorageClass가 일치해야 합니다.
- Selector: PVC에 Label Selector가 지정된 경우 PV의 레이블이 해당 조건을 충족해야 합니다.

하나의 PV는 동시에 하나의 PVC에만 바인딩될 수 있습니다.

반면 하나의 PVC는 여러 PV를 동시에 연결하는 용도로 사용할 수 없습니다.

따라서 여러 Pod가 동일한 영구 스토리지를 사용해야 하는 경우에는 동일한 PVC를 참조하도록 구성할 수 있습니다. 실제 동시 접근 가능 여부는 해당 PV의 Access Modes와 스토리지 구현 방식에 따라 달라집니다.

## 4. PersistentVolumeClaim의 주요 설정

### 4.1. resources.requests.storage

`resources.requests.storage`는 PVC가 요청하는 스토리지 용량을 지정합니다.

```yaml
spec:
  resources:
    requests:
      storage: 10Gi
```

위 구성에서는 10Gi 이상의 스토리지를 요청합니다.

Kubernetes는 요청한 용량 이상의 스토리지를 제공할 수 있는 PV를 탐색합니다.

예를 들어 다음과 같은 PV가 존재한다고 가정합니다.

| PV | 제공 용량 |
|---|---:|
| pv-small | 5Gi |
| pv-medium | 10Gi |
| pv-large | 20Gi |

PVC가 10Gi를 요청하면 `pv-small`은 조건을 충족하지 못하며, `pv-medium`과 `pv-large`는 용량 조건을 충족합니다.

다만 실제 바인딩에는 용량뿐 아니라 Access Modes, Volume Mode, StorageClass 등의 조건도 함께 적용됩니다.

현재 PVC에서 요청할 수 있는 주요 리소스는 스토리지 용량입니다.

### 4.2. accessModes

`accessModes`는 PVC가 요청하는 스토리지의 접근 모드를 지정합니다.

PV와 동일한 접근 모드를 사용하며, 주요 유형은 다음과 같습니다.

| 접근 모드 | 설명 |
|---|---|
| `ReadWriteOnce` | 하나의 노드에서 읽기 및 쓰기 접근을 요청합니다. |
| `ReadOnlyMany` | 여러 노드에서 읽기 전용 접근을 요청합니다. |
| `ReadWriteMany` | 여러 노드에서 읽기 및 쓰기 접근을 요청합니다. |
| `ReadWriteOncePod` | 클러스터 전체에서 하나의 Pod만 읽기 및 쓰기 접근을 요청합니다. |

다음은 여러 노드에서 읽기 및 쓰기가 가능한 스토리지를 요청하는 예제입니다.

```yaml
spec:
  accessModes:
    - ReadWriteMany

  resources:
    requests:
      storage: 20Gi
```

이 경우 Kubernetes는 `ReadWriteMany` 접근 모드를 지원하는 PV를 탐색합니다.

단, 모든 스토리지 유형이 모든 접근 모드를 지원하는 것은 아닙니다. 실제 지원 여부는 스토리지 시스템과 CSI 드라이버에 따라 달라집니다.

### 4.3. volumeMode

`volumeMode`는 요청하는 스토리지를 파일 시스템 또는 원시 블록 장치 형태로 사용할지 지정합니다.

지원하는 값은 다음과 같습니다.

| 값 | 설명 |
|---|---|
| `Filesystem` | 파일 시스템 형태로 스토리지를 사용합니다. |
| `Block` | 원시 블록 장치 형태로 스토리지를 사용합니다. |

별도로 지정하지 않으면 기본값은 `Filesystem`입니다.

다음은 파일 시스템 형태의 스토리지를 요청하는 예제입니다.

```yaml
spec:
  volumeMode: Filesystem
```

원시 블록 장치 형태로 스토리지를 요청하려면 다음과 같이 설정합니다.

```yaml
spec:
  volumeMode: Block
```

PV와 PVC는 동일한 Volume Mode를 사용해야 정상적으로 바인딩될 수 있습니다.

### 4.4. storageClassName

`storageClassName`은 PVC가 요청할 StorageClass를 지정합니다.

StorageClass는 스토리지의 유형과 동적 프로비저닝에 사용할 드라이버 및 설정을 정의하는 리소스입니다.

다음은 `standard`라는 StorageClass를 요청하는 예제입니다.

```yaml
spec:
  storageClassName: standard

  resources:
    requests:
      storage: 10Gi
```

Kubernetes는 해당 StorageClass를 사용하는 PV를 탐색하거나, StorageClass에 지정된 프로비저너를 통해 새로운 PV를 생성합니다.

`storageClassName`의 설정 여부에 따라 동작 방식이 달라집니다.

| 설정 | 동작 |
|---|---|
| 특정 이름 지정 | 해당 StorageClass에 속하는 PV를 요청합니다. |
| 필드 생략 | 기본 StorageClass가 적용될 수 있습니다. |
| `""` 지정 | StorageClass가 없는 PV를 요청합니다. |

특히 필드를 생략하는 것과 빈 문자열을 지정하는 것은 서로 다른 의미입니다.

**필드를 생략한 경우**

```yaml
spec:
  resources:
    requests:
      storage: 10Gi
```

PVC에 `storageClassName`이 지정되지 않으면 클러스터에 설정된 기본 StorageClass가 적용될 수 있습니다.

기본 StorageClass가 없는 경우에는 PVC가 StorageClass를 지정하지 않은 상태로 남을 수 있으며, 이후 기본 StorageClass가 설정되면 해당 PVC에 기본값이 소급 적용될 수도 있습니다.

**빈 문자열을 지정한 경우**

```yaml
spec:
  storageClassName: ""

  resources:
    requests:
      storage: 10Gi
```

빈 문자열을 명시적으로 지정하면 StorageClass가 없는 PV만 요청합니다.

따라서 기본 StorageClass가 설정되어 있더라도 해당 PVC에 자동으로 적용되지 않습니다.

이 차이는 정적 프로비저닝 환경에서 특정 PV를 사용해야 하는 경우 중요합니다.

### 4.5. volumeName

`volumeName`은 PVC가 바인딩할 특정 PV의 이름을 직접 지정하는 필드입니다.

일반적으로 Kubernetes는 PVC의 조건에 맞는 PV를 탐색하여 바인딩하지만, `volumeName`을 사용하면 특정 PV를 대상으로 바인딩을 시도할 수 있습니다.

다음은 `application-pv`라는 PV를 직접 지정하는 예제입니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: application-pvc
spec:
  volumeName: application-pv

  accessModes:
    - ReadWriteOnce

  storageClassName: manual

  resources:
    requests:
      storage: 10Gi
```

위 구성에서는 `application-pv`를 바인딩 대상으로 지정합니다.

다만 `volumeName`을 지정하더라도 스토리지 용량, Access Modes, StorageClass 등의 기본적인 호환 조건은 확인됩니다.

또한 특정 PV를 지정하는 것만으로 해당 PV가 다른 PVC에 의해 사용되는 것을 완전히 방지하는 것은 아닙니다.

특정 PVC만 해당 PV를 사용할 수 있도록 사전에 예약해야 하는 경우에는 PV의 `claimRef`를 함께 설정할 수 있습니다.

### 4.6. selector

`selector`는 PV의 Label을 기준으로 바인딩 대상을 필터링하는 설정입니다.

PVC에 Selector를 지정하면 해당 조건에 일치하는 PV만 바인딩 대상으로 고려됩니다.

Selector는 다음 두 가지 방식으로 구성할 수 있습니다.

- `matchLabels`
- `matchExpressions`

다음은 `environment: production` 레이블을 가진 PV를 요청하는 예제입니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: production-pvc
spec:
  accessModes:
    - ReadWriteOnce

  storageClassName: manual

  selector:
    matchLabels:
      environment: production

  resources:
    requests:
      storage: 10Gi
```

위 구성에서는 `environment=production` 레이블을 가진 PV만 바인딩 대상으로 고려합니다.

`matchExpressions`를 사용하면 보다 다양한 조건을 지정할 수 있습니다.

```yaml
selector:
  matchExpressions:
    - key: environment
      operator: In
      values:
        - production
        - staging

    - key: storage-type
      operator: Exists
```

지원하는 주요 연산자는 다음과 같습니다.

| 연산자 | 설명 |
|---|---|
| `In` | 지정한 값 중 하나와 일치해야 합니다. |
| `NotIn` | 지정한 값에 포함되지 않아야 합니다. |
| `Exists` | 해당 레이블 키가 존재해야 합니다. |
| `DoesNotExist` | 해당 레이블 키가 존재하지 않아야 합니다. |

`matchLabels`와 `matchExpressions`에 지정한 조건은 모두 AND 조건으로 평가됩니다.

즉, 모든 조건을 충족하는 PV만 선택됩니다.

참고로 비어 있지 않은 Selector가 지정된 PVC에는 동적 프로비저닝이 수행되지 않습니다. Selector는 기존 PV 중 조건에 맞는 대상을 찾는 용도로 사용해야 합니다.

## 5. PersistentVolumeClaim의 Binding 방식

PVC는 PV와 연결되는 방식에 따라 일반적인 자동 바인딩과 특정 PV를 지정하는 사전 바인딩 방식으로 구분할 수 있습니다.

### 5.1. 자동 바인딩

자동 바인딩은 PVC가 특정 PV를 지정하지 않고 Kubernetes가 요구사항에 적합한 PV를 탐색하여 연결하는 방식입니다.

다음은 자동 바인딩을 사용하는 PVC 예제입니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: automatic-pvc
spec:
  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 10Gi
```

위 구성에서는 `volumeName`을 지정하지 않았으므로 Kubernetes가 PVC의 요구사항에 적합한 PV를 탐색합니다.

적합한 PV가 존재하면 해당 PV와 바인딩됩니다.

적합한 PV가 존재하지 않고 StorageClass를 통해 동적 프로비저닝이 가능한 경우에는 새로운 스토리지를 생성할 수 있습니다.

### 5.2. 사전 바인딩(Pre-binding)

사전 바인딩은 PVC와 PV 사이의 연결을 명시적으로 지정하는 방식입니다.

PVC의 `volumeName`을 사용하여 특정 PV를 지정하거나, PV의 `claimRef`를 사용하여 특정 PVC를 예약할 수 있습니다.

**PVC에서 PV 지정**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: reserved-pvc
spec:
  volumeName: reserved-pv

  accessModes:
    - ReadWriteOnce

  storageClassName: manual

  resources:
    requests:
      storage: 10Gi
```

**PV에서 PVC 예약**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: reserved-pv
spec:
  capacity:
    storage: 10Gi

  accessModes:
    - ReadWriteOnce

  storageClassName: manual

  persistentVolumeReclaimPolicy: Retain

  claimRef:
    namespace: default
    name: reserved-pvc

  hostPath:
    path: /mnt/data/reserved
    type: DirectoryOrCreate
```

위 구성에서는 `reserved-pv`가 `default` Namespace의 `reserved-pvc`에 예약됩니다.

이 방식은 특정 PVC가 반드시 특정 PV를 사용해야 하는 경우나, Retain 정책으로 보존한 기존 PV를 다시 사용해야 하는 경우에 유용합니다.

## 6. Volume Binding Mode

Volume Binding Mode는 PVC 생성 이후 PV의 바인딩과 동적 프로비저닝이 수행되는 시점을 제어하는 설정입니다.

이 설정은 PVC가 아니라 StorageClass의 `volumeBindingMode` 필드에 정의합니다.

주요 설정값은 다음과 같습니다.

| 설정값 | 설명 |
|---|---|
| `Immediate` | PVC가 생성되면 즉시 PV 바인딩 및 동적 프로비저닝을 수행합니다. |
| `WaitForFirstConsumer` | PVC를 사용하는 Pod가 생성될 때까지 바인딩 및 동적 프로비저닝을 지연합니다. |

### 6.1. Immediate

`Immediate`는 PVC가 생성되는 즉시 PV 바인딩 및 동적 프로비저닝을 수행하는 방식입니다.

별도로 지정하지 않으면 기본적으로 `Immediate`가 사용됩니다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: example.com/csi-driver
volumeBindingMode: Immediate
```

동작 과정은 다음과 같습니다.

```text
PVC Created
    │
    ▼
Storage Provisioning
    │
    ▼
PV Created
    │
    ▼
PVC Bound
    │
    ▼
Pod Scheduled
```

이 방식은 스토리지가 특정 노드에 종속되지 않고 클러스터의 여러 노드에서 접근 가능한 경우에 적합합니다.

반면 특정 노드나 가용 영역(Availability Zone)에 종속된 스토리지에서는 문제가 발생할 수 있습니다.

PVC가 생성되었을 때 Pod의 스케줄링 조건을 고려하지 않고 스토리지가 먼저 생성되거나 바인딩될 수 있기 때문입니다.

그 결과 스토리지가 특정 영역에 생성된 이후 Pod가 해당 스토리지에 접근할 수 없는 다른 영역으로 스케줄링되는 상황이 발생할 수 있습니다.

### 6.2. WaitForFirstConsumer

`WaitForFirstConsumer`는 PVC를 사용하는 Pod가 생성될 때까지 PV 바인딩 및 동적 프로비저닝을 지연하는 방식입니다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: topology-aware
provisioner: example.com/csi-driver
volumeBindingMode: WaitForFirstConsumer
```

동작 과정은 다음과 같습니다.

```text
PVC Created
    │
    ▼
PVC Pending
    │
    │ Pod Created
    ▼
Scheduler Evaluates Pod
    │
    ├── Node Resources
    ├── Node Selector
    ├── Affinity / Anti-Affinity
    └── Storage Topology
    │
    ▼
Storage Provisioning / Binding
    │
    ▼
PV Created or Selected
    │
    ▼
PVC Bound
    │
    ▼
Pod Scheduled
```

Kubernetes 스케줄러는 Pod의 노드 배치 조건과 스토리지의 토폴로지 조건을 함께 고려합니다.

이를 통해 스토리지가 특정 노드나 가용 영역에 종속되어 있더라도 Pod가 해당 스토리지를 사용할 수 있는 위치에 배치되도록 할 수 있습니다.

특히 Local Volume처럼 특정 노드에 종속된 스토리지를 사용할 때 유용합니다.

### 6.3. Immediate와 WaitForFirstConsumer 비교

| 구분 | Immediate | WaitForFirstConsumer |
|---|---|---|
| 바인딩 시점 | PVC 생성 직후 | PVC를 사용하는 Pod가 생성된 이후 |
| Pod 스케줄링 조건 | 고려하지 않습니다. | 함께 고려합니다. |
| 토폴로지 제약 | 제약이 있는 스토리지에서 스케줄링 문제가 발생할 수 있습니다. | 스토리지 위치와 Pod 배치를 함께 고려합니다. |
| 주요 용도 | 여러 노드에서 접근 가능한 스토리지 | 노드 또는 가용 영역에 종속된 스토리지 |

## 7. PersistentVolumeClaim의 상태

PVC는 생성 및 바인딩 과정에서 다음과 같은 상태를 가질 수 있습니다.

| 상태 | 설명 |
|---|---|
| `Pending` | PVC가 생성되었지만 아직 PV와 바인딩되지 않은 상태입니다. |
| `Bound` | PV와 정상적으로 바인딩된 상태입니다. |
| `Lost` | PVC가 참조하는 PV를 더 이상 사용할 수 없는 상태입니다. |

### 7.1. Pending

PVC가 생성되었지만 아직 사용할 수 있는 PV가 연결되지 않은 상태입니다.

다음과 같은 상황에서 발생할 수 있습니다.

- PVC의 요구사항을 충족하는 PV가 존재하지 않는 경우
- 동적 프로비저닝이 아직 완료되지 않은 경우
- StorageClass 또는 CSI 프로비저너에 문제가 발생한 경우
- `WaitForFirstConsumer`가 설정되어 있고 PVC를 사용하는 Pod가 아직 생성되지 않은 경우
- 스토리지의 토폴로지 또는 용량 조건이 충족되지 않는 경우

Pending 상태 자체가 반드시 오류를 의미하지는 않습니다.

특히 `WaitForFirstConsumer`를 사용하는 경우 Pod가 생성되기 전까지 PVC가 Pending 상태로 유지되는 것은 정상적인 동작입니다.

### 7.2. Bound

PVC가 적합한 PV와 정상적으로 바인딩된 상태입니다.

Bound 상태가 되면 Pod에서 해당 PVC를 참조하여 스토리지를 사용할 수 있습니다.

### 7.3. Lost

PVC가 바인딩되어 있던 PV를 더 이상 사용할 수 없는 상태입니다.

예를 들어 PVC가 참조하는 PV 리소스가 비정상적으로 삭제되거나 스토리지 연결이 유실되는 경우 발생할 수 있습니다.

Lost 상태에서는 기존 PV 및 실제 스토리지의 상태를 확인하고 복구 작업을 수행해야 합니다.

### 7.4. PVC 상태 확인

다음 명령어를 사용하여 PVC의 상태를 확인할 수 있습니다.

```bash
kubectl get pvc
```

특정 PVC의 상세 정보를 확인하려면 다음 명령어를 사용합니다.

```bash
kubectl describe pvc application-pvc
```

PVC와 연결된 PV를 확인하려면 다음 명령어를 사용합니다.

```bash
kubectl get pvc application-pvc \
  -o jsonpath='{.spec.volumeName}'
```

## 8. Pod에서 PersistentVolumeClaim 사용하기

Pod는 PVC를 직접 참조하여 영구 스토리지를 사용합니다.

Pod의 `spec.volumes`에 `persistentVolumeClaim`을 지정하고, 컨테이너의 `volumeMounts`를 통해 원하는 경로에 마운트합니다.

다음은 PVC를 사용하는 Pod 예제입니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: application-pod
spec:
  containers:
    - name: app
      image: nginx

      volumeMounts:
        - name: application-data
          mountPath: /app/data

  volumes:
    - name: application-data
      persistentVolumeClaim:
        claimName: application-pvc
```

위 구성에서는 다음과 같은 과정으로 스토리지가 연결됩니다.

1. `application-pvc`라는 PVC를 생성합니다.
2. Kubernetes가 PVC의 요구사항에 적합한 PV를 바인딩합니다.
3. Pod의 `persistentVolumeClaim` 설정에서 해당 PVC를 참조합니다.
4. Kubernetes가 PV에 연결된 실제 스토리지를 Pod에 연결합니다.
5. 컨테이너의 `/app/data` 경로에 스토리지가 마운트됩니다.

전체 연결 구조는 다음과 같습니다.

```text
                 Pod
                  │
                  │ spec.volumes
                  ▼
         PersistentVolumeClaim
                  │
                  │ Binding
                  ▼
         PersistentVolume
                  │
                  ▼
           Actual Storage
                  │
                  │ volumeMounts
                  ▼
          Container Directory
              /app/data
```

### 8.1. 여러 Pod에서 동일한 PVC 사용하기

동일한 Namespace에 존재하는 여러 Pod가 동일한 PVC를 참조할 수 있습니다.

다음은 두 개의 Pod가 동일한 PVC를 사용하는 예제입니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: application-pod-a
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: application-data
          mountPath: /app/data

  volumes:
    - name: application-data
      persistentVolumeClaim:
        claimName: application-pvc
---
apiVersion: v1
kind: Pod
metadata:
  name: application-pod-b
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: application-data
          mountPath: /app/data

  volumes:
    - name: application-data
      persistentVolumeClaim:
        claimName: application-pvc
```

위 구성에서는 두 Pod가 동일한 `application-pvc`를 참조합니다.

다만 동일한 PVC를 참조한다고 해서 여러 노드에서 동시에 스토리지를 사용할 수 있는 것은 아닙니다.

실제 접근 가능 여부는 PV의 Access Modes와 스토리지 시스템의 기능에 따라 결정됩니다.

예를 들어 `ReadWriteOnce`는 하나의 노드에서 읽기 및 쓰기 접근을 허용하므로, 동일한 노드에서 실행되는 여러 Pod가 함께 사용할 수 있습니다.

반면 서로 다른 노드에서 동시에 읽기 및 쓰기를 수행해야 한다면 `ReadWriteMany`를 지원하는 스토리지가 필요합니다.

## 9. PersistentVolumeClaim 확장

PersistentVolumeClaim은 StorageClass와 스토리지 드라이버가 지원하는 경우 기존 스토리지의 용량을 확장할 수 있습니다.

PVC 확장을 사용하려면 StorageClass의 `allowVolumeExpansion` 필드가 `true`로 설정되어 있어야 합니다.

다음은 스토리지 확장을 허용하는 StorageClass 예제입니다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-storage

provisioner: example.com/csi-driver

allowVolumeExpansion: true
```

해당 StorageClass를 사용하는 PVC는 기존 요청 용량보다 큰 용량으로 수정할 수 있습니다.

예를 들어 기존 PVC가 10Gi를 요청하고 있다면 다음과 같이 20Gi로 변경할 수 있습니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: application-pvc
spec:
  accessModes:
    - ReadWriteOnce

  storageClassName: expandable-storage

  resources:
    requests:
      storage: 20Gi
```

기존 PVC의 용량을 확장하는 경우에도 새로운 PV를 생성하는 것이 아니라 기존 PV와 실제 스토리지의 용량을 확장합니다.

확장 과정은 스토리지 유형과 CSI 드라이버의 지원 기능에 따라 달라집니다.

일부 스토리지는 파일 시스템 확장 과정이 추가로 필요할 수 있으며, 사용 중인 Volume의 종류에 따라 Pod 재시작이나 재마운트가 필요할 수도 있습니다.

또한 PVC의 요청 용량을 기존 값보다 작게 변경하는 방식의 스토리지 축소는 지원하지 않습니다.

PVC 확장 기능은 Kubernetes v1.24부터 Stable 상태입니다.

## 10. PersistentVolumeClaim의 Data Source

PVC는 기존 PVC 또는 VolumeSnapshot을 데이터 소스로 지정하여 새로운 스토리지를 생성할 수 있습니다.

이를 활용하면 기존 스토리지의 데이터를 기반으로 새로운 스토리지를 생성하거나, 스냅샷 시점의 데이터를 복원할 수 있습니다.

주요 필드는 다음과 같습니다.

- `dataSource`
- `dataSourceRef`

### 10.1. 기존 PVC를 이용한 데이터 복제

기존 PVC를 데이터 소스로 지정하면 지원되는 프로비저너를 통해 기존 Volume의 데이터를 기반으로 새로운 Volume을 생성할 수 있습니다.

다음은 기존 PVC를 데이터 소스로 지정하는 예제입니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cloned-pvc
spec:
  accessModes:
    - ReadWriteOnce

  storageClassName: standard

  resources:
    requests:
      storage: 10Gi

  dataSource:
    kind: PersistentVolumeClaim
    name: source-pvc
```

위 구성에서는 `source-pvc`의 데이터를 기반으로 새로운 `cloned-pvc`를 생성하도록 요청합니다.

이 기능은 지원되는 스토리지 프로비저너가 제공하는 Volume Cloning 기능을 사용합니다.

기존 PVC와 새로운 PVC의 스토리지 유형 및 용량 조건 등은 프로비저너의 요구사항을 충족해야 합니다.

### 10.2. VolumeSnapshot을 이용한 데이터 복원

VolumeSnapshot을 데이터 소스로 지정하면 특정 시점에 생성한 스냅샷을 기반으로 새로운 PVC를 생성할 수 있습니다.

다음은 VolumeSnapshot을 데이터 소스로 지정하는 예제입니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
spec:
  accessModes:
    - ReadWriteOnce

  storageClassName: standard

  resources:
    requests:
      storage: 10Gi

  dataSource:
    kind: VolumeSnapshot
    name: application-snapshot
    apiGroup: snapshot.storage.k8s.io
```

위 구성에서는 `application-snapshot`이라는 VolumeSnapshot을 기반으로 새로운 PVC를 생성합니다.

이를 통해 기존 데이터의 특정 시점 상태를 새로운 스토리지에 복원할 수 있습니다.

단, VolumeSnapshot 기능은 CSI 드라이버와 Snapshot Controller 등 관련 구성 요소의 지원이 필요합니다.

VolumeSnapshot의 생성, 상태 관리 및 복원 과정은 별도의 VolumeSnapshot 문서에서 설명합니다.

## 11. PersistentVolumeClaim과 관련 리소스의 관계

PVC는 독립적으로 스토리지를 생성하거나 제공하는 리소스가 아닙니다.

실제 스토리지를 관리하고 제공하는 PV, 동적 프로비저닝 방식을 정의하는 StorageClass, 그리고 스토리지를 사용하는 Pod와 함께 동작합니다.

각 리소스의 관계는 다음과 같습니다.

### 11.1. PVC와 PV

PVC는 스토리지 사용을 요청하고 PV는 해당 요청을 충족하는 실제 스토리지를 제공합니다.

두 리소스는 Binding을 통해 연결됩니다.

### 11.2. PVC와 StorageClass

StorageClass는 동적 프로비저닝에 사용할 스토리지 유형과 생성 방식을 정의합니다.

PVC가 `storageClassName`을 지정하면 해당 StorageClass의 프로비저너를 통해 새로운 스토리지를 생성할 수 있습니다.

### 11.3. PVC와 Pod

Pod는 PVC를 Volume의 소스로 지정하여 영구 스토리지를 사용합니다.

Pod는 실제 스토리지의 종류나 위치를 직접 관리하지 않고 PVC를 통해 스토리지에 접근합니다.

### 11.4. 전체 구조

```text
                  StorageClass
                       │
                       │ Dynamic Provisioning
                       ▼
                  Provisioner
                       │
                       ▼
                 Actual Storage
                       │
                       ▼
              PersistentVolume
                       │
                       │ Binding
                       ▼
             PersistentVolumeClaim
                       │
                       │ Reference
                       ▼
                      Pod
                       │
                       │ volumeMounts
                       ▼
                   Container
```

이 구조를 통해 애플리케이션과 실제 스토리지 구현을 분리할 수 있습니다.

애플리케이션은 PVC를 통해 필요한 스토리지를 요청하고, Kubernetes는 해당 요구사항에 적합한 스토리지를 연결합니다.

## 12. 정리

PersistentVolumeClaim은 Kubernetes에서 애플리케이션이 필요한 영구 스토리지를 선언하고 요청하기 위한 리소스입니다.

핵심 개념은 다음과 같습니다.

- PVC는 스토리지의 용량, 접근 모드 및 Volume Mode 등을 선언합니다.
- PVC는 Namespace에 종속되는 리소스입니다.
- PV와 Binding을 통해 실제 스토리지에 연결됩니다.
- `storageClassName`을 사용하면 특정 StorageClass의 스토리지를 요청할 수 있습니다.
- `volumeName`을 사용하면 특정 PV를 바인딩 대상으로 지정할 수 있습니다.
- `selector`를 사용하면 PV의 Label을 기준으로 바인딩 대상을 제한할 수 있습니다.
- `Immediate`와 `WaitForFirstConsumer`는 StorageClass에서 바인딩 및 프로비저닝 시점을 제어합니다.
- Pod는 `persistentVolumeClaim`을 통해 PVC를 참조하여 스토리지를 사용합니다.
- StorageClass와 스토리지 드라이버가 지원하는 경우 기존 PVC의 용량을 확장할 수 있습니다.
- 기존 PVC 또는 VolumeSnapshot을 데이터 소스로 지정하여 새로운 스토리지를 생성할 수 있습니다.

PVC는 애플리케이션과 실제 스토리지 사이의 추상화 계층으로 동작합니다.

이를 통해 애플리케이션은 특정 스토리지 공급업체나 물리적인 스토리지 구성에 직접 종속되지 않고, 필요한 스토리지의 조건만 선언하여 사용할 수 있습니다.

## 13. 참고자료

- [Kubernetes 공식 문서 - Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes 공식 문서 - Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Kubernetes 공식 문서 - Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Kubernetes 공식 문서 - Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [Kubernetes API Reference - PersistentVolumeClaim](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/persistent-volume-claim-v1/)