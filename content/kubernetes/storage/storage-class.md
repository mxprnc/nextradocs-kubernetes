# StorageClass

## 1. StorageClass란?

StorageClass는 Kubernetes에서 스토리지를 동적으로 프로비저닝하기 위한 스토리지 클래스 리소스입니다.

PersistentVolume(PV)이 실제 스토리지를 나타내고 PersistentVolumeClaim(PVC)이 필요한 스토리지를 요청한다면, StorageClass는 해당 요청을 어떤 방식으로 처리하고 어떤 유형의 스토리지를 생성할 것인지 정의합니다.

관리자는 StorageClass를 통해 스토리지의 프로비저닝 방식, 스토리지 드라이버, 생성 매개변수 및 스토리지 회수 정책 등을 설정할 수 있습니다.

PVC가 특정 StorageClass를 요청하면 Kubernetes는 해당 StorageClass에 지정된 Provisioner를 사용하여 실제 스토리지를 생성하고, 이에 대응하는 PV를 자동으로 등록합니다.

StorageClass의 주요 특징은 다음과 같습니다.

- 동적 프로비저닝을 통해 PV를 자동으로 생성할 수 있습니다.
- 스토리지 유형과 프로비저닝에 사용할 드라이버를 지정할 수 있습니다.
- 스토리지 공급자별 생성 매개변수를 정의할 수 있습니다.
- PV의 Reclaim Policy를 지정할 수 있습니다.
- Volume Binding Mode를 통해 PV 바인딩 및 프로비저닝 시점을 제어할 수 있습니다.
- 스토리지 확장 허용 여부를 설정할 수 있습니다.
- 기본 StorageClass를 지정하여 PVC에서 StorageClass를 생략할 수 있도록 지원합니다.

StorageClass는 클러스터 범위(Cluster-scoped)의 리소스입니다. 따라서 특정 Namespace에 종속되지 않으며 클러스터 전체의 PVC에서 참조할 수 있습니다.

## 2. StorageClass의 기본 구조

StorageClass는 `apiVersion: storage.k8s.io/v1`, `kind: StorageClass`로 정의합니다.

다음은 CSI 드라이버를 사용하는 StorageClass의 예제입니다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: csi.example.com
parameters:
  type: ssd
  replication: "3"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
mountOptions:
  - discard
```

각 필드의 역할은 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `metadata.name` | StorageClass의 이름입니다. |
| `provisioner` | 스토리지 생성에 사용할 Provisioner를 지정합니다. |
| `parameters` | Provisioner에 전달할 스토리지 생성 매개변수를 지정합니다. |
| `reclaimPolicy` | 동적으로 생성된 PV의 회수 정책을 지정합니다. |
| `allowVolumeExpansion` | PVC를 통한 스토리지 용량 확장을 허용할지 지정합니다. |
| `volumeBindingMode` | PV 바인딩 및 동적 프로비저닝의 수행 시점을 지정합니다. |
| `mountOptions` | 스토리지 마운트 시 사용할 옵션을 지정합니다. |

위 예제에서는 `csi.example.com` Provisioner를 사용하여 SSD 유형의 스토리지를 생성하도록 설정합니다.

또한 생성된 PV의 기본 회수 정책은 `Delete`이며, PVC를 통한 용량 확장을 허용하고 Pod의 스케줄링 조건을 고려한 후 스토리지를 프로비저닝하도록 구성합니다.

실제 `provisioner`와 `parameters`의 값은 사용하는 스토리지 시스템과 CSI 드라이버에 따라 달라집니다.

## 3. Provisioner

Provisioner는 StorageClass에서 정의한 스토리지 생성 요청을 처리하는 구성 요소입니다.

PVC가 특정 StorageClass를 요청하면 Kubernetes는 해당 StorageClass의 Provisioner를 통해 실제 스토리지를 생성합니다.

Provisioner는 스토리지 시스템과 Kubernetes 사이에서 동적 프로비저닝을 수행하는 역할을 합니다.

대표적인 Provisioner 유형은 다음과 같습니다.

| Provisioner | 설명 |
|---|---|
| `ebs.csi.aws.com` | AWS EBS CSI Driver를 사용합니다. |
| `efs.csi.aws.com` | AWS EFS CSI Driver를 사용합니다. |
| `pd.csi.storage.gke.io` | Google Cloud Persistent Disk CSI Driver를 사용합니다. |
| `disk.csi.azure.com` | Azure Disk CSI Driver를 사용합니다. |
| `file.csi.azure.com` | Azure File CSI Driver를 사용합니다. |
| `kubernetes.io/no-provisioner` | 자체적인 동적 프로비저닝을 수행하지 않습니다. |

CSI(Container Storage Interface)는 Kubernetes와 스토리지 시스템 사이의 표준 인터페이스입니다.

CSI 드라이버를 사용하면 Kubernetes는 특정 스토리지 공급자의 내부 구현에 직접 종속되지 않고 표준화된 방식으로 스토리지를 생성하고 연결할 수 있습니다.

### 3.1. CSI Provisioner 동작 구조

CSI 드라이버는 일반적으로 Controller Plugin과 Node Plugin으로 구성됩니다.

- Controller Plugin: 스토리지 생성, 삭제, 확장 등 스토리지의 수명주기를 관리합니다.
- Node Plugin: 실제 노드에서 스토리지를 연결하고 마운트하는 작업을 수행합니다.

일반적인 동작 구조는 다음과 같습니다.

```text
             PersistentVolumeClaim
                       │
                       ▼
                 StorageClass
                       │
                       │ provisioner
                       ▼
                CSI Controller
                       │
                       │ CreateVolume
                       ▼
                 Storage System
                       │
                       │ Create PV
                       ▼
               PersistentVolume
                       │
                       │ Binding
                       ▼
             PersistentVolumeClaim
                       │
                       ▼
                      Pod
                       │
                       ▼
                  CSI Node
                       │
                       │ NodePublishVolume
                       ▼
                 Mounted Volume
```

실제 CSI 동작 과정에서는 Kubernetes의 PV Controller와 CSI External Provisioner 등의 구성 요소가 함께 동작합니다.

CSI External Provisioner는 PVC와 StorageClass를 확인하고 CSI Controller에 스토리지 생성 요청을 전달합니다.

이후 생성된 스토리지 정보를 기반으로 PV가 등록되고 PVC와 바인딩됩니다.

Pod가 해당 PVC를 사용하게 되면 CSI Node Plugin이 노드에서 스토리지를 연결하고 마운트합니다.

## 4. Parameters

`parameters`는 Provisioner에 전달할 스토리지 생성 매개변수를 정의하는 필드입니다.

동일한 Provisioner를 사용하더라도 Parameters에 따라 서로 다른 유형의 스토리지를 생성할 수 있습니다.

예를 들어 클라우드 스토리지에서는 디스크 유형, 암호화 여부, 성능 등급 및 복제 정책 등을 지정할 수 있습니다.

다음은 AWS EBS CSI Driver를 사용하는 StorageClass 예제입니다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
  fsType: ext4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

주요 Parameters의 역할은 다음과 같습니다.

| Parameter | 설명 |
|---|---|
| `type` | 생성할 EBS 볼륨의 유형을 지정합니다. |
| `encrypted` | 스토리지 암호화 여부를 지정합니다. |
| `fsType` | 파일 시스템 유형을 지정합니다. |

위 예제는 암호화가 활성화된 GP3 유형의 EBS 볼륨을 생성하도록 구성합니다.

단, `parameters`에 사용할 수 있는 항목과 값은 Provisioner마다 다릅니다.

따라서 실제 운영 환경에서는 사용하는 CSI 드라이버의 공식 문서를 확인해야 합니다.

## 5. Reclaim Policy

Reclaim Policy는 PVC가 삭제된 이후 해당 PVC와 바인딩되었던 PV 및 실제 스토리지를 어떻게 처리할 것인지 정의하는 정책입니다.

StorageClass에서는 `reclaimPolicy` 필드를 통해 동적으로 생성되는 PV의 기본 회수 정책을 지정할 수 있습니다.

주요 정책은 다음과 같습니다.

| 정책 | 설명 |
|---|---|
| `Retain` | 실제 스토리지와 데이터를 유지합니다. |
| `Delete` | PV와 연결된 실제 스토리지 자산을 삭제합니다. |

다음은 Retain 정책을 지정하는 예제입니다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: retained-storage
provisioner: ebs.csi.aws.com
reclaimPolicy: Retain
```

위 StorageClass를 통해 생성된 PV는 기본적으로 `Retain` 정책을 사용합니다.

PVC가 삭제되더라도 실제 스토리지는 유지되며, PV는 `Released` 상태가 될 수 있습니다.

반면 `Delete` 정책을 사용하는 경우 PVC 삭제 이후 PV와 실제 스토리지 자산의 삭제가 진행됩니다.

StorageClass에서 `reclaimPolicy`를 생략하면 기본값은 `Delete`입니다.

참고로 Reclaim Policy는 StorageClass를 통해 생성되는 PV에 적용되는 기본 정책입니다. 이미 생성된 PV의 Reclaim Policy는 해당 PV의 `persistentVolumeReclaimPolicy` 필드에서 확인할 수 있습니다.

## 6. Volume Binding Mode

Volume Binding Mode는 PVC 생성 이후 PV 바인딩 및 동적 프로비저닝이 수행되는 시점을 제어하는 설정입니다.

StorageClass의 `volumeBindingMode` 필드에서 지정할 수 있습니다.

지원하는 주요 설정값은 다음과 같습니다.

| 설정값 | 설명 |
|---|---|
| `Immediate` | PVC가 생성되면 즉시 PV 바인딩 및 동적 프로비저닝을 수행합니다. |
| `WaitForFirstConsumer` | PVC를 사용하는 Pod가 생성될 때까지 바인딩 및 동적 프로비저닝을 지연합니다. |

### 6.1. Immediate

`Immediate`는 PVC가 생성되는 즉시 PV 바인딩 및 동적 프로비저닝을 수행하는 방식입니다.

별도로 지정하지 않으면 기본값은 `Immediate`입니다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: immediate-storage
provisioner: ebs.csi.aws.com
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

이 방식은 스토리지가 특정 노드나 가용 영역에 종속되지 않고 여러 노드에서 접근 가능한 경우에 적합합니다.

반면 특정 노드나 가용 영역에 종속된 스토리지에서는 Pod의 스케줄링 조건을 고려하기 전에 스토리지가 먼저 생성될 수 있다는 문제가 있습니다.

예를 들어 특정 가용 영역에 스토리지가 생성된 이후 Pod가 해당 스토리지에 접근할 수 없는 다른 가용 영역으로 스케줄링되는 상황이 발생할 수 있습니다.

### 6.2. WaitForFirstConsumer

`WaitForFirstConsumer`는 PVC를 사용하는 Pod가 생성될 때까지 PV 바인딩 및 동적 프로비저닝을 지연하는 방식입니다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: topology-aware-storage
provisioner: ebs.csi.aws.com
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
    ├── Node Affinity
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

이를 통해 스토리지가 특정 가용 영역이나 노드에 종속되어 있더라도 Pod가 해당 스토리지를 사용할 수 있는 위치에 배치되도록 할 수 있습니다.

특히 AWS EBS와 같은 가용 영역 종속 스토리지나 Local Volume을 사용하는 환경에서 유용합니다.

### 6.3. Immediate와 WaitForFirstConsumer 비교

| 구분 | Immediate | WaitForFirstConsumer |
|---|---|---|
| 바인딩 시점 | PVC 생성 직후 | PVC를 사용하는 Pod가 생성된 이후 |
| Pod 스케줄링 조건 | 바인딩 시 고려하지 않습니다. | 함께 고려합니다. |
| 스토리지 토폴로지 | 제약이 있는 환경에서 배치 문제가 발생할 수 있습니다. | 스토리지 위치와 Pod 배치를 함께 고려합니다. |
| 주요 용도 | 여러 노드에서 접근 가능한 스토리지 | 노드 또는 가용 영역에 종속된 스토리지 |

## 7. AllowVolumeExpansion

`allowVolumeExpansion`은 PVC를 통해 기존 스토리지의 용량을 확장할 수 있도록 허용하는 설정입니다.

기본값은 `false`이며, 스토리지 확장을 허용하려면 `true`로 지정해야 합니다.

다음은 스토리지 확장을 허용하는 StorageClass 예제입니다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-storage
provisioner: ebs.csi.aws.com
allowVolumeExpansion: true
```

위 StorageClass를 사용하는 PVC는 기존 요청 용량보다 큰 용량으로 수정할 수 있습니다.

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

Kubernetes는 PVC의 변경된 용량을 확인하고 스토리지 드라이버를 통해 기존 Volume의 확장을 요청합니다.

확장 과정은 스토리지 유형과 CSI 드라이버의 지원 기능에 따라 달라집니다.

일부 스토리지는 블록 장치 확장 이후 파일 시스템 확장 과정이 추가로 필요할 수 있습니다.

또한 PVC의 요청 용량을 기존 값보다 작게 변경하는 방식의 스토리지 축소는 지원하지 않습니다.

PVC 확장 기능은 Kubernetes v1.24부터 Stable 상태입니다.

## 8. Mount Options

`mountOptions`는 PV를 노드에 마운트할 때 사용할 추가 옵션을 지정하는 필드입니다.

다음은 NFS 스토리지를 사용하는 StorageClass 예제입니다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: example.com/nfs
mountOptions:
  - hard
  - nfsvers=4.1
```

위 구성에서는 마운트 시 `hard`, `nfsvers=4.1` 옵션을 사용하도록 지정합니다.

Mount Options는 파일 시스템 유형이나 스토리지 드라이버에 따라 사용할 수 있는 값이 다릅니다.

또한 Kubernetes는 지정한 Mount Options가 실제 스토리지에서 유효한지 검증하지 않습니다.

잘못된 옵션을 지정하면 마운트 과정에서 오류가 발생할 수 있으므로 스토리지 드라이버의 공식 문서를 확인해야 합니다.

## 9. Default StorageClass

Default StorageClass는 PVC에서 `storageClassName`을 명시하지 않았을 때 기본적으로 사용할 StorageClass입니다.

클러스터에 기본 StorageClass를 지정하면 사용자는 PVC에서 StorageClass를 직접 지정하지 않고도 동적 프로비저닝을 사용할 수 있습니다.

기본 StorageClass는 다음과 같이 지정할 수 있습니다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: example.com/csi-driver
```

`storageclass.kubernetes.io/is-default-class` Annotation을 `"true"`로 설정하면 해당 StorageClass가 기본 StorageClass로 지정됩니다.

이후 다음과 같이 `storageClassName`을 생략한 PVC를 생성할 수 있습니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: application-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Kubernetes는 기본 StorageClass를 적용하여 PVC를 처리합니다.

### 9.1. 기본 StorageClass 확인

다음 명령어로 클러스터에 등록된 StorageClass를 확인할 수 있습니다.

```bash
kubectl get storageclass
```

또는 다음과 같이 축약된 명령어를 사용할 수 있습니다.

```bash
kubectl get sc
```

기본 StorageClass는 출력 결과에서 `(default)`로 표시됩니다.

### 9.2. 기본 StorageClass 변경

기존 StorageClass의 기본 설정을 해제하고 다른 StorageClass를 기본값으로 지정할 수 있습니다.

기존 StorageClass의 기본 설정을 해제합니다.

```bash
kubectl patch storageclass old-storage \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

새로운 StorageClass를 기본값으로 지정합니다.

```bash
kubectl patch storageclass new-storage \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

클러스터에 기본 StorageClass가 여러 개 지정되어 있다면 PVC에서 `storageClassName`을 생략했을 때 가장 최근에 생성된 기본 StorageClass가 선택됩니다.

운영 환경에서는 혼란을 방지하기 위해 기본 StorageClass를 하나만 지정하는 것이 일반적입니다.

### 9.3. 기본 StorageClass와 빈 문자열의 차이

PVC에서 `storageClassName`을 생략하는 것과 빈 문자열을 지정하는 것은 서로 다른 의미입니다.

| PVC 설정 | 동작 |
|---|---|
| `storageClassName` 생략 | 기본 StorageClass가 적용될 수 있습니다. |
| `storageClassName: ""` | StorageClass가 없는 PV를 요청합니다. |
| `storageClassName: standard` | `standard` StorageClass를 요청합니다. |

특히 `storageClassName: ""`를 명시하면 기본 StorageClass가 적용되지 않습니다.

정적 프로비저닝 환경에서 StorageClass가 없는 PV를 사용해야 하는 경우 활용할 수 있습니다.

## 10. No-Provisioner StorageClass

모든 StorageClass가 동적 프로비저닝을 수행하는 것은 아닙니다.

`kubernetes.io/no-provisioner`를 사용하면 StorageClass를 정의하면서도 동적 스토리지 생성을 수행하지 않도록 구성할 수 있습니다.

대표적인 활용 사례는 Local Volume입니다.

Local Volume은 특정 노드에 연결된 로컬 디스크를 사용하기 때문에 일반적인 네트워크 스토리지처럼 필요할 때마다 새로운 스토리지를 자동으로 생성할 수 없습니다.

따라서 관리자가 실제 로컬 디스크를 준비하고 PV를 직접 생성하는 정적 프로비저닝 방식이 필요합니다.

다음은 Local Volume을 위한 StorageClass 예제입니다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

위 구성에서는 동적 프로비저닝을 수행하지 않으며, PVC를 사용하는 Pod의 스케줄링 조건을 고려한 후 PV 바인딩을 진행합니다.

Local Volume을 사용하는 PV는 다음과 같이 구성할 수 있습니다.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 10Gi

  volumeMode: Filesystem

  accessModes:
    - ReadWriteOnce

  persistentVolumeReclaimPolicy: Retain

  storageClassName: local-storage

  local:
    path: /mnt/disks/ssd1

  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - worker-node-1
```

이처럼 No-Provisioner StorageClass는 동적 프로비저닝 없이 StorageClass의 바인딩 정책을 활용해야 하는 환경에서 사용할 수 있습니다.

## 11. StorageClass와 관련 리소스의 관계

StorageClass는 PersistentVolume 및 PersistentVolumeClaim과 함께 동작합니다.

각 리소스의 역할은 다음과 같습니다.

| 리소스 | 역할 |
|---|---|
| StorageClass | 스토리지 생성 방식과 정책을 정의합니다. |
| PersistentVolume (PV) | 클러스터에서 사용할 수 있는 실제 스토리지를 나타냅니다. |
| PersistentVolumeClaim (PVC) | 애플리케이션이 필요한 스토리지의 조건을 요청합니다. |
| Pod | PVC를 참조하여 실제 스토리지를 사용합니다. |
| CSI Driver | 실제 스토리지의 생성, 삭제, 연결 및 마운트를 처리합니다. |

전체적인 동작 구조는 다음과 같습니다.

```text
                 Administrator
                       │
                       │ Define
                       ▼
                  StorageClass
                       │
                       │ Provisioner
                       ▼
                  CSI Controller
                       │
                       │ CreateVolume
                       ▼
                 Actual Storage
                       │
                       │ Register
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
                   CSI Node
                       │
                       │ Mount
                       ▼
                  Container
```

StorageClass는 스토리지 생성 정책을 정의하고, PVC는 해당 정책에 따라 필요한 스토리지를 요청합니다.

Kubernetes는 CSI 드라이버 등의 스토리지 프로비저너를 통해 실제 스토리지를 생성하고 PV를 등록합니다.

이후 PVC와 PV가 바인딩되면 Pod는 PVC를 참조하여 스토리지를 사용할 수 있습니다.

## 12. 주요 설정 비교

StorageClass에서 자주 사용하는 설정을 정리하면 다음과 같습니다.

| 필드 | 기본값 | 설명 |
|---|---|---|
| `provisioner` | 필수 | 스토리지 생성에 사용할 Provisioner입니다. |
| `parameters` | 없음 | Provisioner에 전달할 스토리지 생성 매개변수입니다. |
| `reclaimPolicy` | `Delete` | 동적으로 생성된 PV의 기본 회수 정책입니다. |
| `allowVolumeExpansion` | `false` | PVC를 통한 스토리지 확장 허용 여부입니다. |
| `volumeBindingMode` | `Immediate` | PV 바인딩 및 프로비저닝 시점입니다. |
| `mountOptions` | 없음 | 스토리지 마운트 시 사용할 추가 옵션입니다. |

## 13. 정리

StorageClass는 Kubernetes에서 동적 스토리지 프로비저닝 방식과 스토리지 관리 정책을 정의하는 리소스입니다.

핵심 개념은 다음과 같습니다.

- StorageClass는 클러스터 범위의 리소스입니다.
- Provisioner를 통해 실제 스토리지를 생성할 드라이버를 지정합니다.
- Parameters를 통해 스토리지 유형과 생성 옵션을 정의합니다.
- Reclaim Policy를 통해 PVC 삭제 이후 PV와 실제 스토리지의 처리 방식을 지정합니다.
- Volume Binding Mode를 통해 스토리지의 바인딩 및 프로비저닝 시점을 제어합니다.
- AllowVolumeExpansion을 통해 PVC 기반 스토리지 확장을 허용할 수 있습니다.
- Default StorageClass를 지정하면 PVC에서 StorageClass를 생략할 수 있습니다.
- No-Provisioner StorageClass를 사용하면 동적 프로비저닝 없이 정적 PV의 바인딩 정책을 활용할 수 있습니다.

StorageClass를 사용하면 애플리케이션이 실제 스토리지의 생성 방식이나 공급자별 설정을 직접 관리하지 않고도 필요한 스토리지를 요청할 수 있습니다.

이를 통해 Kubernetes 클러스터의 스토리지 관리 정책을 표준화하고, 다양한 스토리지 시스템을 일관된 방식으로 활용할 수 있습니다.

## 14. 참고자료

- [Kubernetes 공식 문서 - Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Kubernetes 공식 문서 - Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes 공식 문서 - Volume Binding Mode](https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode)
- [Kubernetes 공식 문서 - Dynamic Volume Provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
- [Kubernetes 공식 문서 - Container Storage Interface (CSI)](https://kubernetes.io/docs/concepts/storage/volumes/#csi)
- [Kubernetes API Reference - StorageClass](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/storage-class-v1/)