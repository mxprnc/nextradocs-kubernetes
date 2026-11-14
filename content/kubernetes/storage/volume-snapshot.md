# VolumeSnapshot

## 1. VolumeSnapshot이란?

VolumeSnapshot은 Kubernetes에서 PersistentVolume(PV)에 저장된 데이터의 특정 시점(Point-in-Time) 상태를 스냅샷으로 생성하고 관리하기 위한 리소스입니다.

스냅샷은 기존 볼륨의 데이터를 특정 시점 기준으로 보존한 복사본입니다. 이를 활용하면 기존 스토리지의 데이터를 별도의 백업 시점으로 보존하거나, 스냅샷을 기반으로 새로운 볼륨을 생성할 수 있습니다.

예를 들어 데이터베이스의 변경 작업을 수행하기 전에 스냅샷을 생성하면, 문제가 발생했을 때 해당 스냅샷을 기반으로 새로운 볼륨을 복원할 수 있습니다.

Kubernetes의 Volume Snapshot 기능은 다음 세 가지 리소스로 구성됩니다.

- **VolumeSnapshot**: 사용자가 볼륨의 스냅샷 생성을 요청하는 리소스입니다.
- **VolumeSnapshotContent**: 실제 스토리지 시스템에 생성된 스냅샷을 나타내는 리소스입니다.
- **VolumeSnapshotClass**: 스냅샷 생성에 사용할 스토리지 드라이버와 생성 매개변수 및 삭제 정책을 정의하는 리소스입니다.

VolumeSnapshot은 CSI(Container Storage Interface)를 지원하는 스토리지 드라이버를 통해 동작합니다. 따라서 CSI 드라이버가 스냅샷 기능을 지원해야 하며, 클러스터에도 Volume Snapshot 관련 CRD와 컨트롤러가 설치되어 있어야 합니다.

VolumeSnapshot의 주요 특징은 다음과 같습니다.

- 기존 PersistentVolumeClaim을 데이터 소스로 사용하여 스냅샷을 생성할 수 있습니다.
- VolumeSnapshotClass를 통해 스냅샷 생성 정책을 정의할 수 있습니다.
- VolumeSnapshotContent와 VolumeSnapshot을 연결하여 스냅샷의 수명주기를 관리합니다.
- 기존 스냅샷을 기반으로 새로운 PersistentVolumeClaim을 생성할 수 있습니다.
- 삭제 정책에 따라 Kubernetes 리소스와 실제 스토리지 스냅샷의 유지 또는 삭제 여부를 결정할 수 있습니다.

단, 스냅샷이 특정 시점의 데이터를 보존한다고 해서 애플리케이션 수준의 데이터 일관성까지 자동으로 보장하는 것은 아닙니다. 데이터베이스와 같이 쓰기 작업이 발생하는 애플리케이션에서는 필요에 따라 쓰기 중단, 체크포인트, 파일 시스템 동결 등의 별도 조치를 고려해야 합니다.

## 2. Volume Snapshot 구성 리소스

Volume Snapshot 기능은 VolumeSnapshot, VolumeSnapshotContent, VolumeSnapshotClass 세 가지 리소스를 중심으로 동작합니다.

### 2.1. VolumeSnapshot

VolumeSnapshot은 사용자가 특정 PersistentVolumeClaim을 대상으로 스냅샷 생성을 요청하는 리소스입니다.

PersistentVolumeClaim이 필요한 스토리지를 요청하는 리소스라면, VolumeSnapshot은 기존 스토리지의 특정 시점 데이터를 스냅샷으로 요청하는 리소스입니다.

VolumeSnapshot은 Namespace 범위(Namespace-scoped)의 리소스입니다.

다음은 VolumeSnapshot의 기본 구조입니다.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: application-snapshot
  namespace: default
spec:
  volumeSnapshotClassName: standard-snapshot
  source:
    persistentVolumeClaimName: application-pvc
```

주요 필드는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `metadata.name` | VolumeSnapshot의 이름입니다. |
| `metadata.namespace` | VolumeSnapshot이 생성될 Namespace입니다. |
| `spec.volumeSnapshotClassName` | 사용할 VolumeSnapshotClass의 이름입니다. |
| `spec.source.persistentVolumeClaimName` | 스냅샷 생성 대상인 PVC의 이름입니다. |
| `spec.source.volumeSnapshotContentName` | 사전에 생성된 VolumeSnapshotContent를 지정합니다. |

`spec.source`에는 스냅샷 생성 방식에 따라 PVC 또는 기존 VolumeSnapshotContent 중 하나를 지정합니다.

### 2.2. VolumeSnapshotContent

VolumeSnapshotContent는 실제 스토리지 시스템에 생성된 스냅샷을 나타내는 리소스입니다.

VolumeSnapshot이 사용자의 스냅샷 생성 요청을 나타낸다면, VolumeSnapshotContent는 스토리지 시스템에 존재하는 실제 스냅샷을 Kubernetes에서 관리하기 위한 리소스입니다.

VolumeSnapshotContent는 클러스터 범위(Cluster-scoped)의 리소스입니다.

동적 프로비저닝에서는 Snapshot Controller가 VolumeSnapshotContent를 자동으로 생성합니다.

다음은 동적 프로비저닝 과정에서 생성될 수 있는 VolumeSnapshotContent의 구조입니다.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  name: application-snapshot-content
spec:
  deletionPolicy: Delete
  driver: example.csi.driver
  source:
    volumeHandle: volume-123456
  sourceVolumeMode: Filesystem
  volumeSnapshotClassName: standard-snapshot
  volumeSnapshotRef:
    name: application-snapshot
    namespace: default
```

주요 필드는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `spec.deletionPolicy` | 연결된 VolumeSnapshot 삭제 시 스토리지 스냅샷을 처리하는 정책입니다. |
| `spec.driver` | 스냅샷을 생성하고 관리하는 CSI 드라이버입니다. |
| `spec.source.volumeHandle` | 스냅샷 생성 대상이 되는 원본 볼륨의 식별자입니다. |
| `spec.source.snapshotHandle` | 이미 생성된 스토리지 스냅샷의 식별자입니다. |
| `spec.sourceVolumeMode` | 원본 볼륨의 모드입니다. `Filesystem` 또는 `Block`을 사용할 수 있습니다. |
| `spec.volumeSnapshotClassName` | 스냅샷 생성에 사용한 VolumeSnapshotClass의 이름입니다. |
| `spec.volumeSnapshotRef` | 연결된 VolumeSnapshot의 이름과 Namespace 등의 참조 정보입니다. |

`volumeHandle`과 `snapshotHandle`은 서로 다른 용도로 사용됩니다.

- `volumeHandle`: 원본 스토리지 볼륨을 식별합니다. 동적 스냅샷 생성에 사용됩니다.
- `snapshotHandle`: 스토리지 시스템에 이미 생성된 스냅샷을 식별합니다. 사전 프로비저닝된 스냅샷을 등록할 때 사용됩니다.

동적 프로비저닝에서는 Kubernetes가 VolumeSnapshotContent를 생성하므로 사용자가 일반적으로 직접 작성할 필요는 없습니다.

### 2.3. VolumeSnapshotClass

VolumeSnapshotClass는 스냅샷을 생성할 때 사용할 스토리지 드라이버와 생성 매개변수 및 삭제 정책을 정의하는 리소스입니다.

StorageClass가 PersistentVolume을 생성하기 위한 스토리지 유형과 정책을 정의한다면, VolumeSnapshotClass는 스냅샷 생성에 필요한 설정을 정의합니다.

VolumeSnapshotClass는 클러스터 범위(Cluster-scoped)의 리소스입니다.

다음은 VolumeSnapshotClass의 기본 구조입니다.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: standard-snapshot
driver: example.csi.driver
deletionPolicy: Delete
parameters:
  snapshotType: standard
```

주요 필드는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `metadata.name` | VolumeSnapshotClass의 이름입니다. |
| `driver` | 스냅샷 생성에 사용할 CSI 드라이버입니다. |
| `deletionPolicy` | 연결된 VolumeSnapshot 삭제 시 실제 스토리지 스냅샷을 처리하는 정책입니다. |
| `parameters` | CSI 드라이버에 전달할 스냅샷 생성 매개변수입니다. |

`parameters`에 사용할 수 있는 항목과 값은 CSI 드라이버마다 다릅니다.

또한 VolumeSnapshotClass는 생성 후 수정할 수 없는 리소스이므로, 생성 매개변수를 변경해야 한다면 새로운 VolumeSnapshotClass를 생성해야 합니다.

## 3. VolumeSnapshot과 관련 리소스의 관계

VolumeSnapshot 기능에서 각 리소스는 다음과 같은 관계를 가집니다.

| 리소스 | 범위 | 역할 |
|---|---|---|
| VolumeSnapshot | Namespace | 사용자의 스냅샷 생성 요청을 나타냅니다. |
| VolumeSnapshotContent | Cluster | 실제 스토리지 스냅샷을 나타냅니다. |
| VolumeSnapshotClass | Cluster | 스냅샷 생성 정책을 정의합니다. |
| PersistentVolumeClaim | Namespace | 스냅샷 생성 대상이 되는 원본 볼륨을 요청합니다. |

VolumeSnapshot과 VolumeSnapshotContent는 서로 일대일로 바인딩됩니다.

VolumeSnapshotClass는 동적 프로비저닝 시 스냅샷 생성에 필요한 설정을 제공합니다.

전체적인 관계는 다음과 같습니다.

```text
                 VolumeSnapshotClass
                          │
                          │ Snapshot Configuration
                          ▼
PersistentVolumeClaim ── VolumeSnapshot
      (Source)                │
                              │ Binding
                              ▼
                     VolumeSnapshotContent
                              │
                              │ CSI Driver
                              ▼
                      Storage System
                              │
                              ▼
                       Actual Snapshot
```

VolumeSnapshot은 원본 PVC를 지정하여 스냅샷을 요청하고, Snapshot Controller는 해당 요청에 대응하는 VolumeSnapshotContent를 생성하거나 기존 VolumeSnapshotContent와 연결합니다.

VolumeSnapshotContent는 CSI 드라이버를 통해 실제 스토리지 시스템의 스냅샷과 연결됩니다.

## 4. Volume Snapshot 동작 구조

Kubernetes의 Volume Snapshot 기능은 CRD, Snapshot Controller, CSI Snapshotter 및 CSI Driver가 함께 동작합니다.

### 4.1. 주요 구성 요소

| 구성 요소 | 역할 |
|---|---|
| Volume Snapshot CRDs | VolumeSnapshot, VolumeSnapshotContent, VolumeSnapshotClass 리소스를 Kubernetes API에 등록합니다. |
| Snapshot Controller | VolumeSnapshot과 VolumeSnapshotContent를 감시하고 두 리소스의 생성 및 바인딩과 삭제를 관리합니다. |
| CSI Snapshotter | VolumeSnapshotContent의 변경을 감시하고 CSI Driver에 스냅샷 생성 및 삭제 요청을 전달합니다. |
| CSI Controller Plugin | 스토리지 시스템에 실제 스냅샷 생성 및 삭제 작업을 수행합니다. |
| CSI Node Plugin | 스토리지 볼륨의 노드 연결 및 마운트 작업을 담당합니다. |

Snapshot Controller와 CSI Snapshotter는 서로 다른 역할을 수행합니다.

Snapshot Controller는 Kubernetes 리소스의 상태와 바인딩을 관리하며, CSI Snapshotter는 CSI Driver에 실제 스냅샷 작업을 요청합니다.

또한 Snapshot 기능을 활성화한 클러스터에는 스냅샷 리소스의 유효성을 검증하는 Validating Webhook이 설치될 수 있습니다.

Volume Snapshot 관련 CRD와 Snapshot Controller 등의 설치 및 관리는 Kubernetes 배포판 또는 클러스터 운영 환경에서 담당합니다. CSI Driver 역시 스냅샷 기능을 지원해야 합니다.

### 4.2. 동작 흐름

사용자가 PVC를 대상으로 VolumeSnapshot을 생성하면 다음과 같은 과정으로 동작합니다.

```text
          User Creates VolumeSnapshot
                       │
                       ▼
               Snapshot Controller
                       │
                       │ Watch VolumeSnapshot
                       ▼
              VolumeSnapshotContent
                       │
                       ▼
                 CSI Snapshotter
                       │
                       │ CreateSnapshot
                       ▼
                 CSI Controller
                       │
                       ▼
                Storage System
                       │
                       │ Snapshot Created
                       ▼
              VolumeSnapshotContent
                       │
                       ▼
                VolumeSnapshot
                       │
                       ▼
                 ReadyToUse
```

1. 사용자가 원본 PVC를 지정한 VolumeSnapshot을 생성합니다.
2. Snapshot Controller가 VolumeSnapshot을 감지합니다.
3. 동적 프로비저닝에 필요한 VolumeSnapshotContent가 생성됩니다.
4. CSI Snapshotter가 VolumeSnapshotContent를 감지하고 CSI Driver에 스냅샷 생성을 요청합니다.
5. CSI Controller가 스토리지 시스템에 실제 스냅샷을 생성합니다.
6. 생성된 스냅샷 정보가 VolumeSnapshotContent에 반영됩니다.
7. VolumeSnapshot의 상태가 갱신되고 스냅샷을 사용할 수 있는 상태가 됩니다.

실제 작업의 세부 순서와 상태 갱신은 Snapshot Controller 및 CSI Driver의 구현에 따라 달라질 수 있습니다.

## 5. Volume Snapshot 프로비저닝 방식

Volume Snapshot은 크게 동적 프로비저닝(Dynamic Provisioning)과 정적 프로비저닝(Pre-provisioned) 방식으로 생성할 수 있습니다.

### 5.1. Dynamic Provisioning

동적 프로비저닝은 사용자가 VolumeSnapshot을 생성하면 Kubernetes가 CSI Driver를 통해 실제 스냅샷을 생성하는 방식입니다.

일반적으로 사용자가 직접 VolumeSnapshotContent를 생성할 필요가 없습니다.

다음은 동적 프로비저닝을 위한 VolumeSnapshotClass 예제입니다.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: standard-snapshot
driver: example.csi.driver
deletionPolicy: Delete
parameters:
  snapshotType: standard
```

이후 VolumeSnapshot을 생성합니다.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: application-snapshot
  namespace: default
spec:
  volumeSnapshotClassName: standard-snapshot
  source:
    persistentVolumeClaimName: application-pvc
```

위 예제에서는 `application-pvc`를 대상으로 스냅샷을 생성합니다.

Kubernetes는 지정된 VolumeSnapshotClass를 확인하고 CSI Driver를 통해 실제 스냅샷을 생성합니다.

생성된 VolumeSnapshotContent는 VolumeSnapshot과 자동으로 바인딩됩니다.

### 5.2. Pre-provisioned

정적 프로비저닝은 스토리지 시스템에 이미 존재하는 스냅샷을 Kubernetes 리소스로 등록하여 사용하는 방식입니다.

관리자가 실제 스토리지 스냅샷을 준비하고 해당 스냅샷을 나타내는 VolumeSnapshotContent를 직접 생성해야 합니다.

먼저 VolumeSnapshotContent를 생성합니다.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  name: existing-snapshot-content
spec:
  deletionPolicy: Retain
  driver: example.csi.driver
  source:
    snapshotHandle: snapshot-123456
  sourceVolumeMode: Filesystem
  volumeSnapshotRef:
    name: existing-snapshot
    namespace: default
```

`source.snapshotHandle`에는 스토리지 시스템에 이미 존재하는 스냅샷의 고유 식별자를 지정합니다.

이후 해당 VolumeSnapshotContent를 참조하는 VolumeSnapshot을 생성합니다.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: existing-snapshot
  namespace: default
spec:
  source:
    volumeSnapshotContentName: existing-snapshot-content
```

이 방식에서는 새로운 스냅샷을 생성하지 않습니다.

기존 스토리지 스냅샷을 Kubernetes에서 관리할 수 있도록 VolumeSnapshotContent와 VolumeSnapshot을 연결합니다.

### 5.3. Dynamic Provisioning과 Pre-provisioned 비교

| 구분 | Dynamic Provisioning | Pre-provisioned |
|---|---|---|
| 스냅샷 생성 | CSI Driver를 통해 자동 생성합니다. | 기존 스토리지 스냅샷을 사용합니다. |
| VolumeSnapshotContent | Snapshot Controller가 생성합니다. | 관리자가 직접 생성합니다. |
| 스냅샷 식별자 | 원본 볼륨의 `volumeHandle`을 사용합니다. | 기존 스냅샷의 `snapshotHandle`을 사용합니다. |
| 주요 용도 | 새로운 스냅샷을 생성할 때 사용합니다. | 외부에서 생성한 스냅샷을 Kubernetes에 등록할 때 사용합니다. |

## 6. VolumeSnapshot 상태

VolumeSnapshot은 `status` 필드를 통해 스냅샷의 현재 상태를 확인할 수 있습니다.

주요 상태 필드는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `status.boundVolumeSnapshotContentName` | 바인딩된 VolumeSnapshotContent의 이름입니다. |
| `status.readyToUse` | 스냅샷을 PVC 복원 등의 작업에 사용할 수 있는지 나타냅니다. |
| `status.restoreSize` | 스냅샷으로부터 복원할 때 필요한 데이터 용량을 나타냅니다. |

`readyToUse`가 `true`이면 스냅샷이 준비되어 복원 작업에 사용할 수 있는 상태입니다.

다음 명령어로 VolumeSnapshot의 상태를 확인할 수 있습니다.

```bash
kubectl get volumesnapshot
```

또는 다음과 같이 축약된 명령어를 사용할 수 있습니다.

```bash
kubectl get vs
```

상세한 상태를 확인하려면 다음 명령어를 사용합니다.

```bash
kubectl describe volumesnapshot application-snapshot
```

VolumeSnapshotContent의 상태는 다음 명령어로 확인할 수 있습니다.

```bash
kubectl get volumesnapshotcontent
```

VolumeSnapshotContent는 클러스터 범위의 리소스이므로 Namespace를 지정하지 않고 조회합니다.

## 7. Deletion Policy

Deletion Policy는 VolumeSnapshot이 삭제되었을 때 연결된 VolumeSnapshotContent와 실제 스토리지 스냅샷을 어떻게 처리할 것인지 정의하는 정책입니다.

VolumeSnapshotClass의 `deletionPolicy` 필드에서 지정하며, 생성된 VolumeSnapshotContent에도 해당 정책이 반영됩니다.

지원하는 정책은 다음과 같습니다.

| 정책 | 설명 |
|---|---|
| `Delete` | VolumeSnapshot 삭제 시 VolumeSnapshotContent와 실제 스토리지 스냅샷을 삭제합니다. |
| `Retain` | VolumeSnapshot이 삭제되어도 VolumeSnapshotContent와 실제 스토리지 스냅샷을 유지합니다. |

### 7.1. Delete

`Delete`는 VolumeSnapshot 삭제 시 연결된 VolumeSnapshotContent와 실제 스토리지 스냅샷까지 삭제하는 정책입니다.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: delete-snapshot
driver: example.csi.driver
deletionPolicy: Delete
```

이 VolumeSnapshotClass를 사용하는 VolumeSnapshot이 삭제되면 Snapshot Controller는 연결된 VolumeSnapshotContent의 삭제를 처리하고, CSI Snapshotter를 통해 실제 스토리지 스냅샷 삭제 작업을 요청합니다.

### 7.2. Retain

`Retain`은 VolumeSnapshot이 삭제되더라도 VolumeSnapshotContent와 실제 스토리지 스냅샷을 유지하는 정책입니다.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: retain-snapshot
driver: example.csi.driver
deletionPolicy: Retain
```

이 정책을 사용하면 Kubernetes의 VolumeSnapshot 리소스가 삭제되어도 실제 스토리지 스냅샷을 보존할 수 있습니다.

보존된 VolumeSnapshotContent와 스토리지 스냅샷은 관리자가 추후 다시 사용하거나 별도로 정리할 수 있습니다.

운영 환경에서는 데이터 보존 요구사항과 스토리지 비용을 고려하여 적절한 정책을 선택해야 합니다.

## 8. Default VolumeSnapshotClass

Default VolumeSnapshotClass는 VolumeSnapshot에서 `volumeSnapshotClassName`을 지정하지 않았을 때 기본적으로 사용할 VolumeSnapshotClass입니다.

기본 VolumeSnapshotClass는 다음 Annotation을 통해 지정할 수 있습니다.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: default-snapshot
  annotations:
    snapshot.storage.kubernetes.io/is-default-class: "true"
driver: example.csi.driver
deletionPolicy: Delete
```

위 예제에서는 `default-snapshot`을 기본 VolumeSnapshotClass로 지정합니다.

이후 다음과 같이 VolumeSnapshot을 생성할 수 있습니다.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: application-snapshot
spec:
  source:
    persistentVolumeClaimName: application-pvc
```

`volumeSnapshotClassName`을 생략했으므로 기본 VolumeSnapshotClass가 적용됩니다.

기본 VolumeSnapshotClass가 설정되어 있지 않다면 스냅샷 요청 시 VolumeSnapshotClass를 명시해야 합니다.

또한 기본 VolumeSnapshotClass가 여러 개 지정된 경우 기본 클래스를 선택하는 과정에서 예상하지 못한 동작이 발생할 수 있으므로, 운영 환경에서는 기본 클래스를 하나만 지정하는 것이 좋습니다.

## 9. Snapshot을 이용한 PVC 복원

VolumeSnapshot으로부터 새로운 PersistentVolumeClaim을 생성하여 스냅샷 시점의 데이터를 복원할 수 있습니다.

기존 PVC를 직접 되돌리는 방식이 아니라, 스냅샷을 데이터 소스로 사용하는 새로운 PVC를 생성하는 방식입니다.

PVC의 `spec.dataSource` 필드에 VolumeSnapshot을 지정하면 됩니다.

다음은 스냅샷을 기반으로 새로운 PVC를 생성하는 예제입니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 10Gi
  dataSource:
    name: application-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

주요 설정은 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `spec.storageClassName` | 복원된 볼륨을 생성할 때 사용할 StorageClass입니다. |
| `spec.resources.requests.storage` | 복원할 PVC의 요청 용량입니다. |
| `spec.dataSource.name` | 데이터 복원에 사용할 VolumeSnapshot의 이름입니다. |
| `spec.dataSource.kind` | 데이터 소스의 종류입니다. 여기서는 `VolumeSnapshot`을 지정합니다. |
| `spec.dataSource.apiGroup` | 데이터 소스 리소스의 API 그룹입니다. |

위 PVC가 생성되면 Kubernetes는 지정된 VolumeSnapshot을 데이터 소스로 사용하여 새로운 볼륨을 프로비저닝합니다.

이때 StorageClass와 CSI Driver는 스냅샷으로부터 볼륨을 복원하는 기능을 지원해야 합니다.

또한 요청하는 스토리지 용량은 스냅샷의 `restoreSize`보다 작지 않아야 합니다.

복원된 PVC가 Bound 상태가 되면 일반적인 PVC와 동일한 방식으로 Pod에서 사용할 수 있습니다.

다음은 복원된 PVC를 Pod에서 사용하는 예제입니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restored-pod
spec:
  containers:
    - name: application
      image: nginx:latest
      volumeMounts:
        - name: restored-volume
          mountPath: /data
  volumes:
    - name: restored-volume
      persistentVolumeClaim:
        claimName: restored-pvc
```

위 Pod는 `restored-pvc`를 `/data` 경로에 마운트합니다.

이처럼 VolumeSnapshot을 사용하면 기존 데이터의 특정 시점을 보존하고, 해당 시점의 데이터를 기반으로 새로운 스토리지 볼륨을 생성할 수 있습니다.

## 10. Snapshot Source PVC Protection

Snapshot Source PVC Protection은 스냅샷 생성 과정에서 원본 PVC가 삭제되어 데이터 손실이 발생하는 것을 방지하기 위한 보호 기능입니다.

Kubernetes는 PVC를 스냅샷 생성에 사용하고 있는 동안 해당 PVC에 `snapshot.storage.kubernetes.io/pvc-as-source-protection` Finalizer를 적용합니다.

이 상태에서 원본 PVC를 삭제하더라도 스냅샷 생성 작업이 완료되거나 중단되기 전까지 PVC의 실제 삭제가 지연됩니다.

스냅샷이 준비되거나 작업이 중단되면 보호 Finalizer가 제거되고 PVC 삭제가 진행될 수 있습니다.

이 기능은 스냅샷 생성에 사용 중인 원본 PVC가 먼저 삭제되어 발생할 수 있는 데이터 손실을 방지합니다.

## 11. VolumeSnapshot과 Backup의 차이

VolumeSnapshot은 특정 시점의 스토리지 데이터를 보존하는 기능이지만, 그 자체가 완전한 백업 전략을 의미하지는 않습니다.

VolumeSnapshot과 백업은 다음과 같은 차이가 있습니다.

| 구분 | VolumeSnapshot | Backup |
|---|---|---|
| 목적 | 특정 시점의 볼륨 데이터를 보존합니다. | 장애 및 데이터 손실에 대비하여 데이터를 복구할 수 있도록 보존합니다. |
| 저장 위치 | 일반적으로 원본 스토리지 시스템의 스냅샷 기능을 사용합니다. | 별도의 스토리지, 백업 서버 또는 원격 저장소 등을 사용할 수 있습니다. |
| 복구 방식 | 스냅샷을 기반으로 새로운 볼륨을 생성합니다. | 백업 데이터와 복구 도구를 사용하여 데이터를 복원합니다. |
| 장애 대응 | 원본 스토리지 시스템에 문제가 발생하면 스냅샷도 영향을 받을 수 있습니다. | 별도 장애 영역에 보관하면 원본 스토리지 장애에 대응할 수 있습니다. |

VolumeSnapshot은 빠른 데이터 복원이나 특정 시점으로의 복구에 유용합니다.

하지만 스토리지 시스템 자체의 장애나 리전 단위 장애 등에 대비하려면 별도 백업 저장소를 활용하는 전략이 필요합니다.

또한 데이터베이스와 같은 애플리케이션에서는 스토리지 스냅샷만으로 애플리케이션 수준의 일관성이 보장되지 않을 수 있으므로 별도의 백업 및 복구 절차를 고려해야 합니다.

## 12. 정리

VolumeSnapshot은 Kubernetes에서 PersistentVolume의 특정 시점 데이터를 스냅샷으로 생성하고 관리하기 위한 기능입니다.

핵심 개념은 다음과 같습니다.

- VolumeSnapshot은 사용자가 스냅샷 생성을 요청하는 Namespace 범위의 리소스입니다.
- VolumeSnapshotContent는 실제 스토리지 스냅샷을 나타내는 클러스터 범위의 리소스입니다.
- VolumeSnapshotClass는 스냅샷 생성에 사용할 CSI Driver, Parameters 및 Deletion Policy를 정의하는 클러스터 범위의 리소스입니다.
- VolumeSnapshot과 VolumeSnapshotContent는 일대일로 바인딩됩니다.
- Dynamic Provisioning은 CSI Driver를 통해 새로운 스냅샷을 자동으로 생성합니다.
- Pre-provisioned 방식은 기존 스토리지 스냅샷을 VolumeSnapshotContent로 등록하여 사용합니다.
- Deletion Policy는 VolumeSnapshot 삭제 이후 실제 스토리지 스냅샷의 유지 또는 삭제 여부를 결정합니다.
- VolumeSnapshot을 데이터 소스로 지정한 새로운 PVC를 생성하여 스냅샷 시점의 데이터를 복원할 수 있습니다.
- VolumeSnapshot 기능을 사용하려면 CSI Driver의 스냅샷 지원과 관련 CRD 및 컨트롤러 설치가 필요합니다.

VolumeSnapshot을 활용하면 Kubernetes에서 스토리지 데이터의 특정 시점을 보존하고, 필요할 때 해당 데이터를 기반으로 새로운 볼륨을 생성할 수 있습니다.

이를 통해 데이터 복구와 스토리지 운영을 Kubernetes 리소스 기반으로 일관되게 관리할 수 있습니다.

## 13. 참고자료

- [Kubernetes 공식 문서 - Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [Kubernetes 공식 문서 - Volume Snapshot Classes](https://kubernetes.io/docs/concepts/storage/volume-snapshot-classes/)
- [Kubernetes 공식 문서 - Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes 공식 문서 - Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Kubernetes CSI Developer Documentation - Volume Snapshot](https://kubernetes-csi.github.io/docs/api/volume-snapshot.html)
- [Kubernetes CSI Developer Documentation - External Snapshotter](https://github.com/kubernetes-csi/external-snapshotter)