# PersistentVolume (PV)

## 1. PersistentVolume이란?

PersistentVolume(PV)은 Kubernetes 클러스터에서 관리하는 영구 스토리지 리소스입니다.

컨테이너의 파일 시스템이나 일반적인 Ephemeral Volume은 컨테이너 또는 Pod의 생명주기에 영향을 받습니다. 반면 데이터베이스, 파일 저장소 등 지속적인 데이터 보존이 필요한 애플리케이션은 Pod가 삭제되거나 재생성되더라도 데이터를 유지할 수 있는 스토리지가 필요합니다.

PersistentVolume은 이러한 요구사항을 충족하기 위해 Kubernetes에서 제공하는 스토리지 관리 리소스입니다.

PV는 관리자가 직접 생성하거나 StorageClass를 통해 동적으로 프로비저닝할 수 있으며, 실제 스토리지의 구현 방식과 관계없이 Kubernetes에서 일관된 방식으로 관리할 수 있습니다.

PV가 나타내는 실제 스토리지는 다음과 같은 다양한 유형으로 구성할 수 있습니다.

- NFS
- iSCSI
- CSI 기반 스토리지
- 클라우드 제공업체의 블록 스토리지
- Local Volume

PersistentVolume의 주요 특징은 다음과 같습니다.

- Pod의 생명주기와 독립적으로 관리됩니다.
- 스토리지의 용량과 접근 모드를 정의할 수 있습니다.
- PersistentVolumeClaim(PVC)을 통해 애플리케이션에 제공됩니다.
- 정적 프로비저닝과 동적 프로비저닝을 지원합니다.
- Reclaim Policy를 통해 PVC 삭제 이후 스토리지의 처리 방식을 지정할 수 있습니다.

PV는 Kubernetes 클러스터에 속하는 리소스이며 특정 Namespace에 종속되지 않는 클러스터 범위(Cluster-scoped)의 리소스입니다.

## 2. PersistentVolume의 기본 구조

PersistentVolume은 `apiVersion: v1`, `kind: PersistentVolume`으로 정의합니다.

다음은 NFS 스토리지를 사용하는 PersistentVolume의 예제입니다.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 10Gi

  volumeMode: Filesystem

  accessModes:
    - ReadWriteMany

  persistentVolumeReclaimPolicy: Retain

  storageClassName: manual

  mountOptions:
    - hard
    - nfsvers=4.1

  nfs:
    path: /data/kubernetes
    server: 192.168.1.100
```

각 설정의 역할은 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `metadata.name` | PersistentVolume의 이름입니다. |
| `capacity.storage` | PV가 제공하는 스토리지 용량입니다. |
| `volumeMode` | 스토리지를 파일 시스템 또는 블록 장치로 제공하는 방식을 지정합니다. |
| `accessModes` | 스토리지의 접근 모드를 지정합니다. |
| `persistentVolumeReclaimPolicy` | PVC가 삭제된 이후 PV 및 실제 스토리지를 처리하는 방식을 지정합니다. |
| `storageClassName` | PV가 속하는 StorageClass를 지정합니다. |
| `mountOptions` | 스토리지를 마운트할 때 사용할 추가 옵션을 지정합니다. |
| `nfs` | 실제 스토리지의 유형과 연결 정보를 지정합니다. |

위 예제에서는 `192.168.1.100`에 위치한 NFS 서버의 `/data/kubernetes` 디렉터리를 10Gi 용량의 PV로 등록합니다.

실제 스토리지의 연결 방식은 Volume 유형에 따라 달라지며, CSI 기반 스토리지를 사용하는 경우에는 CSI 드라이버와 관련된 설정을 사용합니다.

## 3. PersistentVolume 프로비저닝

PersistentVolume은 프로비저닝 방식에 따라 정적 프로비저닝(Static Provisioning)과 동적 프로비저닝(Dynamic Provisioning)으로 구분할 수 있습니다.

### 3.1. 정적 프로비저닝(Static Provisioning)

정적 프로비저닝은 관리자가 실제 스토리지를 준비하고, 이에 대응하는 PersistentVolume 리소스를 직접 생성하는 방식입니다.

스토리지의 생성 및 준비 작업은 Kubernetes 외부에서 수행하거나 스토리지 관리 시스템을 통해 처리할 수 있으며, 준비된 스토리지를 PV로 등록합니다.

일반적인 동작 과정은 다음과 같습니다.

1. 관리자가 NFS, SAN 또는 클라우드 스토리지 등의 실제 스토리지를 준비합니다.
2. 해당 스토리지를 참조하는 PersistentVolume을 생성합니다.
3. 사용자가 필요한 스토리지 조건을 정의한 PersistentVolumeClaim을 생성합니다.
4. Kubernetes가 조건에 적합한 PV와 PVC를 연결합니다.
5. Pod가 PVC를 참조하여 스토리지를 사용합니다.

구조는 다음과 같습니다.

```text
Administrator
      │
      │ Prepare Storage
      ▼
Actual Storage
      │
      │ Create PV
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
      ▼
  Application
```

정적 프로비저닝은 관리자가 스토리지의 생성과 구성을 직접 제어할 수 있다는 장점이 있습니다.

반면 애플리케이션의 요구사항에 맞는 스토리지를 미리 준비해야 하므로, 사용량이 많거나 스토리지 요구사항이 다양한 환경에서는 관리 부담이 증가할 수 있습니다.

### 3.2. 동적 프로비저닝(Dynamic Provisioning)

동적 프로비저닝은 PersistentVolumeClaim이 요청한 스토리지 조건에 따라 Kubernetes가 필요한 스토리지를 자동으로 생성하는 방식입니다.

동적 프로비저닝을 사용하려면 StorageClass가 필요합니다.

StorageClass는 스토리지의 유형, 프로비저닝에 사용할 드라이버 및 관련 설정을 정의합니다.

일반적인 동작 과정은 다음과 같습니다.

1. 관리자가 스토리지 프로비저너와 StorageClass를 구성합니다.
2. 사용자가 특정 StorageClass를 참조하는 PVC를 생성합니다.
3. Kubernetes의 스토리지 컨트롤러가 PVC의 요구사항을 확인합니다.
4. 해당 StorageClass에 지정된 프로비저너가 실제 스토리지를 생성합니다.
5. 생성된 스토리지에 대응하는 PV가 Kubernetes에 등록됩니다.
6. 생성된 PV와 PVC가 자동으로 바인딩됩니다.
7. Pod가 PVC를 참조하여 스토리지를 사용합니다.

구조는 다음과 같습니다.

```text
PersistentVolumeClaim
          │
          │ storageClassName
          ▼
     StorageClass
          │
          │ Provisioning Request
          ▼
  Provisioner / CSI Driver
          │
          │ Create Storage
          ▼
     Actual Storage
          │
          │ Register PV
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
```

동적 프로비저닝은 애플리케이션이 필요한 스토리지를 요청하면 자동으로 생성하고 연결할 수 있으므로 운영 효율성이 높습니다.

특히 클라우드 환경에서는 CSI 드라이버와 StorageClass를 조합하여 스토리지 생성 및 관리를 자동화하는 방식이 널리 사용됩니다.

### 3.3. 정적 프로비저닝과 동적 프로비저닝 비교

| 구분 | 정적 프로비저닝 | 동적 프로비저닝 |
|---|---|---|
| 스토리지 준비 | 관리자가 사전에 준비합니다. | PVC 요청에 따라 자동으로 생성합니다. |
| PV 생성 | 관리자가 직접 생성합니다. | 프로비저너가 자동으로 생성합니다. |
| StorageClass | 선택적으로 사용할 수 있습니다. | 동적 프로비저닝에 필요합니다. |
| 관리 방식 | 관리자의 직접적인 개입이 필요합니다. | 자동화된 스토리지 관리가 가능합니다. |
| 주요 용도 | 사전에 준비된 스토리지의 활용 | 애플리케이션 요구사항에 따른 자동 스토리지 생성 |

## 4. PersistentVolume과 PersistentVolumeClaim의 바인딩

PersistentVolume과 PersistentVolumeClaim은 Binding 과정을 통해 연결됩니다.

PV는 클러스터에서 사용할 수 있는 스토리지를 나타내며, PVC는 애플리케이션이 필요한 스토리지의 조건을 요청하는 리소스입니다.

Kubernetes는 PVC의 요구사항과 PV의 속성을 비교하여 적합한 PV를 선택합니다.

주요 바인딩 조건은 다음과 같습니다.

- 스토리지 용량
- 접근 모드(Access Modes)
- 볼륨 모드(Volume Mode)
- StorageClass

예를 들어 PVC가 10Gi의 스토리지를 요청하면 Kubernetes는 해당 요구사항을 충족하는 PV를 탐색합니다.

여러 PV가 조건을 만족하는 경우에는 적합한 PV 중 하나를 선택하여 바인딩합니다.

### 4.1. Binding 예제

다음은 PV와 PVC를 정의하는 예제입니다.

**PersistentVolume**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: application-pv
spec:
  capacity:
    storage: 10Gi

  volumeMode: Filesystem

  accessModes:
    - ReadWriteOnce

  persistentVolumeReclaimPolicy: Retain

  storageClassName: manual

  hostPath:
    path: /mnt/data/application
    type: DirectoryOrCreate
```

**PersistentVolumeClaim**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: application-pvc
spec:
  accessModes:
    - ReadWriteOnce

  volumeMode: Filesystem

  storageClassName: manual

  resources:
    requests:
      storage: 5Gi
```

위 구성에서는 다음과 같은 조건으로 바인딩이 수행됩니다.

- PV는 10Gi의 스토리지를 제공합니다.
- PVC는 5Gi의 스토리지를 요청합니다.
- 두 리소스 모두 `ReadWriteOnce` 접근 모드를 사용합니다.
- 두 리소스 모두 `Filesystem` 볼륨 모드를 사용합니다.
- 두 리소스 모두 `manual` StorageClass를 사용합니다.

PV가 PVC의 요구사항을 충족하므로 바인딩할 수 있습니다.

PVC는 PV가 제공하는 용량보다 작은 용량을 요청할 수 있지만, PVC가 요청한 용량보다 작은 PV에는 바인딩할 수 없습니다.

참고로 위 예제의 `hostPath`는 로컬 노드의 파일 시스템을 사용하는 예시입니다. 실제 운영 환경에서는 노드 종속성과 보안 문제를 고려해야 하며, 일반적인 영구 스토리지 구성에는 적절한 외부 스토리지나 CSI 기반 스토리지를 사용하는 것이 좋습니다.

### 4.2. 특정 PV 지정하기

PVC의 `volumeName` 필드를 사용하면 바인딩할 PersistentVolume을 명시적으로 지정할 수 있습니다.

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
      storage: 5Gi
```

위 구성에서는 `application-pv`라는 이름의 PV를 대상으로 바인딩을 시도합니다.

다만 `volumeName`을 지정하더라도 PV와 PVC의 용량, 접근 모드, 볼륨 모드 등의 호환 조건은 충족해야 합니다.

### 4.3. Binding 상태

PersistentVolume은 다음과 같은 상태(Phase)를 가질 수 있습니다.

| 상태 | 설명 |
|---|---|
| `Available` | PVC와 바인딩되지 않은 상태입니다. |
| `Bound` | PVC와 정상적으로 바인딩된 상태입니다. |
| `Released` | 이전에 바인딩되었던 PVC가 삭제된 상태입니다. |
| `Failed` | 자동 회수 등의 작업이 실패한 상태입니다. |

각 상태의 의미는 다음과 같습니다.

**Available**

PV가 생성되었으며 아직 어떤 PVC에도 바인딩되지 않은 상태입니다.

**Bound**

PV와 PVC가 정상적으로 연결된 상태입니다. 하나의 PV는 동시에 하나의 PVC에만 바인딩될 수 있습니다.

**Released**

기존에 바인딩되어 있던 PVC가 삭제된 상태입니다.

PV는 더 이상 해당 PVC에 바인딩되어 있지 않지만, 실제 스토리지에는 기존 데이터가 남아 있을 수 있습니다.

특히 `Retain` 정책을 사용하는 경우 기존 데이터가 유지되므로 해당 PV를 다른 PVC에 자동으로 재사용할 수 없습니다.

**Failed**

스토리지의 자동 회수 작업 등에서 오류가 발생한 상태입니다.

### 4.4. PV와 PVC의 상태 확인

다음 명령어를 사용하여 PV와 PVC의 상태를 확인할 수 있습니다.

```bash
kubectl get pv
kubectl get pvc
```

특정 PV의 상세 정보를 확인하려면 다음 명령어를 사용합니다.

```bash
kubectl describe pv application-pv
```

특정 PVC의 상세 정보를 확인하려면 다음 명령어를 사용합니다.

```bash
kubectl describe pvc application-pvc
```

## 5. PersistentVolume의 Access Modes

Access Modes는 PersistentVolume이 어떤 방식으로 접근될 수 있는지를 나타내는 설정입니다.

Kubernetes는 Access Modes를 사용하여 PVC와 PV의 호환성을 판단합니다.

주요 접근 모드는 다음과 같습니다.

| 접근 모드 | 약어 | 설명 |
|---|---|---|
| `ReadWriteOnce` | RWO | 하나의 노드에서 읽기 및 쓰기 접근을 허용합니다. |
| `ReadOnlyMany` | ROX | 여러 노드에서 읽기 전용 접근을 허용합니다. |
| `ReadWriteMany` | RWX | 여러 노드에서 읽기 및 쓰기 접근을 허용합니다. |
| `ReadWriteOncePod` | RWOP | 클러스터 전체에서 하나의 Pod만 읽기 및 쓰기 접근을 허용합니다. |

### 5.1. ReadWriteOnce (RWO)

`ReadWriteOnce`는 하나의 노드에서 해당 Volume을 읽기 및 쓰기 방식으로 마운트할 수 있도록 합니다.

여기서 중요한 점은 하나의 노드라는 조건이 하나의 Pod를 의미하지는 않는다는 것입니다.

동일한 노드에서 실행되는 여러 Pod가 같은 RWO Volume에 접근할 수 있습니다.

예를 들어 다음과 같은 구성이 가능합니다.

```text
             Node A
        ┌──────────────┐
        │   Pod A      │
        │   Pod B      │
        │      │       │
        │      ▼       │
        │   RWO PV     │
        └──────────────┘
```

반면 서로 다른 노드에서 동시에 동일한 RWO Volume을 읽기 및 쓰기 방식으로 마운트하는 것은 지원하지 않습니다.

### 5.2. ReadOnlyMany (ROX)

`ReadOnlyMany`는 여러 노드에서 동일한 Volume을 읽기 전용으로 마운트할 수 있도록 합니다.

```text
    Node A             Node B
┌────────────┐     ┌────────────┐
│   Pod A    │     │   Pod B    │
│     │      │     │     │      │
│     └──────┼─────┼─────┘      │
│            ▼     ▼            │
└────────────┘     └────────────┘
           ReadOnlyMany PV
```

여러 노드에서 동일한 데이터를 읽어야 하는 애플리케이션에 활용할 수 있습니다.

### 5.3. ReadWriteMany (RWX)

`ReadWriteMany`는 여러 노드에서 동일한 Volume을 읽기 및 쓰기 방식으로 마운트할 수 있도록 합니다.

```text
    Node A             Node B
┌────────────┐     ┌────────────┐
│   Pod A    │     │   Pod B    │
│     │      │     │     │      │
│     └──────┼─────┼─────┘      │
│            ▼     ▼            │
└────────────┘     └────────────┘
           ReadWriteMany PV
```

여러 노드에서 실행되는 애플리케이션이 동일한 파일을 공유해야 하는 경우에 활용할 수 있습니다.

대표적인 예로 NFS 기반 스토리지를 사용할 수 있습니다.

단, RWX 지원 여부는 실제 스토리지 시스템과 CSI 드라이버의 기능에 따라 달라집니다.

### 5.4. ReadWriteOncePod (RWOP)

`ReadWriteOncePod`는 클러스터 전체에서 하나의 Pod만 해당 Volume을 읽기 및 쓰기 방식으로 마운트할 수 있도록 제한하는 접근 모드입니다.

RWO와 달리 노드가 아니라 Pod를 기준으로 접근을 제한합니다.

따라서 동일한 노드에 여러 Pod가 실행되더라도 해당 Volume은 하나의 Pod에서만 사용할 수 있습니다.

이 기능은 단일 Pod만 스토리지에 접근해야 하는 워크로드에 유용합니다.

`ReadWriteOncePod`는 CSI Volume에서 지원되며, Kubernetes v1.29부터 Stable 상태입니다.

### 5.5. Access Modes 사용 시 주의사항

Access Modes는 PV와 PVC의 바인딩 조건을 결정하고, 스토리지의 접근 가능 범위를 표현하는 데 사용됩니다.

다만 모든 접근 모드가 실제 파일 시스템 수준의 읽기 및 쓰기 권한을 강제하는 것은 아닙니다.

특히 다음 사항을 구분해야 합니다.

- `ReadWriteOnce`는 하나의 노드에서 읽기 및 쓰기 마운트를 허용한다는 의미입니다.
- `ReadOnlyMany`는 여러 노드에서 읽기 전용으로 마운트할 수 있다는 의미이지만, 접근 모드 자체가 모든 쓰기 작업을 차단하는 보안 정책은 아닙니다.
- `ReadWriteMany`는 여러 노드에서 읽기 및 쓰기 접근이 가능하다는 의미입니다.
- `ReadWriteOncePod`는 하나의 Pod만 해당 Volume을 마운트하도록 제한합니다.

또한 실제 지원 여부는 스토리지 유형과 CSI 드라이버에 따라 달라집니다.

하나의 PV가 여러 접근 모드를 지원하도록 선언되어 있더라도, 실제로는 한 번에 하나의 접근 모드로 마운트됩니다.

## 6. PersistentVolume의 Volume Modes

Volume Mode는 PersistentVolume을 파일 시스템 또는 원시 블록 장치 형태로 제공하는 방식을 지정합니다.

Kubernetes는 다음 두 가지 Volume Mode를 지원합니다.

- `Filesystem`
- `Block`

### 6.1. Filesystem

`Filesystem`은 Volume을 일반적인 파일 시스템 형태로 제공하는 모드입니다.

PV를 Pod에 마운트하면 컨테이너 내부의 특정 디렉터리에서 파일을 읽고 쓸 수 있습니다.

별도로 지정하지 않으면 기본적으로 `Filesystem`이 사용됩니다.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: filesystem-pv
spec:
  capacity:
    storage: 10Gi

  volumeMode: Filesystem

  accessModes:
    - ReadWriteOnce

  persistentVolumeReclaimPolicy: Retain

  hostPath:
    path: /mnt/data/filesystem
    type: DirectoryOrCreate
```

Pod에서는 `volumeMounts`를 사용하여 파일 시스템을 마운트합니다.

```yaml
volumeMounts:
  - name: application-data
    mountPath: /app/data
```

일반적인 애플리케이션의 데이터 저장에는 Filesystem 모드가 사용됩니다.

### 6.2. Block

`Block`은 Volume을 파일 시스템으로 마운트하지 않고 원시 블록 장치(Raw Block Device) 형태로 제공하는 모드입니다.

파일 시스템 계층을 거치지 않고 블록 장치에 직접 접근할 수 있으므로, 원시 블록 장치를 직접 처리하는 애플리케이션에서 사용할 수 있습니다.

다음은 Block Volume을 정의하는 PV 예제입니다.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: block-pv
spec:
  capacity:
    storage: 10Gi

  volumeMode: Block

  accessModes:
    - ReadWriteOnce

  persistentVolumeReclaimPolicy: Retain

  # 실제 운영 환경에서는 CSI 또는
  # 블록 스토리지를 지원하는 Volume Source를 사용합니다.
```

PVC에서도 동일한 Volume Mode를 요청해야 합니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  accessModes:
    - ReadWriteOnce

  volumeMode: Block

  resources:
    requests:
      storage: 10Gi
```

Pod에서는 `volumeMounts` 대신 `volumeDevices`를 사용하여 블록 장치를 연결합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: block-volume-example
spec:
  containers:
    - name: app
      image: busybox
      command:
        - sh
        - -c
        - sleep 3600
      volumeDevices:
        - name: block-data
          devicePath: /dev/xvda

  volumes:
    - name: block-data
      persistentVolumeClaim:
        claimName: block-pvc
```

위 구성에서는 PVC를 통해 연결된 블록 장치가 컨테이너 내부의 `/dev/xvda` 경로로 제공됩니다.

Block Volume을 사용하는 애플리케이션은 파일 시스템이 제공되지 않는 원시 블록 장치를 직접 처리할 수 있어야 합니다.

### 6.3. Filesystem과 Block 비교

| 구분 | Filesystem | Block |
|---|---|---|
| 제공 형태 | 파일 시스템 | 원시 블록 장치 |
| Pod 연결 방식 | `volumeMounts` | `volumeDevices` |
| 접근 방식 | 디렉터리 및 파일 | 블록 장치 |
| 파일 시스템 | 사용 가능 | 제공되지 않음 |
| 주요 용도 | 일반적인 파일 저장 및 애플리케이션 데이터 | 블록 장치를 직접 처리하는 애플리케이션 |

## 7. PersistentVolume의 Reclaim Policy

Reclaim Policy는 PersistentVolumeClaim이 삭제되어 PV가 기존 PVC와의 연결을 해제한 이후, 해당 PV와 실제 스토리지를 어떻게 처리할지 결정하는 정책입니다.

Kubernetes에서 정의하는 주요 정책은 다음과 같습니다.

| 정책 | 설명 |
|---|---|
| `Retain` | 실제 스토리지와 데이터를 유지합니다. |
| `Delete` | PV와 연결된 실제 스토리지 자산을 삭제합니다. |
| `Recycle` | 기존 데이터를 삭제하고 Volume을 재사용할 수 있도록 합니다. |

현재 일반적으로 사용하는 정책은 `Retain`과 `Delete`입니다.

`Recycle`은 Deprecated 상태이며, 새로운 환경에서는 동적 프로비저닝을 사용하는 것이 권장됩니다.

### 7.1. Retain

`Retain`은 PVC가 삭제되더라도 실제 스토리지와 데이터를 유지하는 정책입니다.

PVC가 삭제되면 PV는 `Released` 상태가 됩니다.

하지만 기존 PVC의 데이터가 남아 있으므로 해당 PV는 자동으로 다른 PVC에 재사용되지 않습니다.

다음은 Retain 정책의 동작 과정입니다.

```text
PVC
 │
 │ Bound
 ▼
PV
 │
 ▼
Actual Storage
 │
 │ Delete PVC
 ▼
PV: Released
 │
 │ Data remains
 ▼
Actual Storage: Preserved
```

Retain 정책에서는 관리자가 직접 스토리지를 정리하고 재사용 여부를 결정해야 합니다.

일반적인 처리 과정은 다음과 같습니다.

1. 기존 PVC를 삭제합니다.
2. PV가 `Released` 상태로 변경되었는지 확인합니다.
3. 실제 스토리지에 저장된 데이터의 백업 및 보존 여부를 확인합니다.
4. 필요한 경우 데이터를 별도로 백업하거나 정리합니다.
5. PV를 삭제하고 동일한 스토리지를 참조하는 새로운 PV를 생성하여 재사용하거나, 스토리지를 별도로 정리합니다.

Retain은 중요한 데이터를 보존해야 하는 데이터베이스나 영구 저장소에 적합합니다.

### 7.2. Delete

`Delete`는 PVC가 삭제되어 PV가 해제되면 Kubernetes가 PV와 연결된 실제 스토리지 자산을 삭제하도록 하는 정책입니다.

다음은 Delete 정책의 동작 과정입니다.

```text
PVC
 │
 │ Bound
 ▼
PV
 │
 ▼
Actual Storage
 │
 │ Delete PVC
 ▼
PV Reclaiming
 │
 │ Delete Storage
 ▼
PV Deleted
 │
 ▼
Actual Storage Deleted
```

Delete 정책은 동적으로 생성된 스토리지를 자동으로 정리할 수 있다는 장점이 있습니다.

다만 실제 스토리지 자산까지 삭제될 수 있으므로 데이터 보존이 필요한 환경에서는 주의해야 합니다.

Delete 정책의 실제 동작은 해당 스토리지 유형과 프로비저너가 삭제 기능을 지원하는지에 따라 달라집니다.

동적 프로비저닝을 사용하는 경우 PV는 해당 StorageClass의 Reclaim Policy를 상속합니다.

StorageClass의 Reclaim Policy 기본값은 `Delete`입니다.

### 7.3. Recycle

`Recycle`은 PVC가 삭제된 이후 Volume 내부의 데이터를 삭제하고 해당 Volume을 다시 사용할 수 있도록 하는 정책입니다.

기본적인 동작은 Volume 내부의 파일을 삭제하는 방식으로 수행됩니다.

다만 `Recycle`은 Deprecated 상태이며, 현재는 사용이 권장되지 않습니다.

새로운 스토리지 환경에서는 `Recycle` 대신 동적 프로비저닝을 사용하는 것이 권장됩니다.

### 7.4. Reclaim Policy 비교

| 구분 | Retain | Delete | Recycle |
|---|---|---|---|
| PVC 삭제 후 PV | Released 상태로 유지 | 삭제 | 재사용 가능 상태로 전환 |
| 실제 스토리지 | 유지 | 지원되는 경우 삭제 | 유지 |
| 기존 데이터 | 유지 | 삭제 | 삭제 |
| 자동 처리 | 관리자의 수동 작업 필요 | 자동 삭제 | 데이터 정리 |
| 권장 여부 | 사용 가능 | 사용 가능 | Deprecated |

## 8. Storage Object in Use Protection

Storage Object in Use Protection은 사용 중인 PVC나 PV가 삭제되어 데이터 손실이 발생하는 것을 방지하기 위한 보호 기능입니다.

Kubernetes는 PVC와 PV에 Finalizer를 사용하여 삭제 시점을 제어합니다.

주요 Finalizer는 다음과 같습니다.

| Finalizer | 설명 |
|---|---|
| `kubernetes.io/pvc-protection` | Pod에서 사용 중인 PVC가 즉시 삭제되지 않도록 보호합니다. |
| `kubernetes.io/pv-protection` | PVC와 바인딩된 PV가 즉시 삭제되지 않도록 보호합니다. |

예를 들어 Pod가 PVC를 사용하고 있는 상황에서 해당 PVC를 삭제하면 Kubernetes는 PVC를 즉시 제거하지 않고 삭제를 지연합니다.

Pod가 더 이상 해당 PVC를 사용하지 않게 되면 보호 조건이 해제되고 PVC 삭제가 진행됩니다.

PV 역시 PVC와 바인딩되어 있는 동안에는 삭제가 지연될 수 있습니다.

이 기능은 사용 중인 스토리지 리소스가 실수로 삭제되는 상황을 방지하는 데 도움이 됩니다.

다만 데이터 백업이나 스토리지 자체의 장애까지 방지하는 기능은 아니므로 별도의 백업 및 복구 전략이 필요합니다.

## 9. Node Affinity

Node Affinity는 PersistentVolume이 특정 노드에서만 접근할 수 있도록 노드 배치 조건을 지정하는 설정입니다.

대부분의 네트워크 스토리지에서는 특정 노드에 종속되지 않으므로 별도의 Node Affinity 설정이 필요하지 않습니다.

하지만 Local Volume은 특정 노드에 연결된 로컬 디스크를 사용하기 때문에 해당 스토리지가 위치한 노드를 지정해야 합니다.

다음은 Local Volume을 사용하는 PV의 예제입니다.

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

위 구성에서는 `/mnt/disks/ssd1` 경로에 위치한 로컬 스토리지를 PV로 등록하고, 해당 스토리지가 `worker-node-1` 노드에 존재한다는 조건을 지정합니다.

이 PV를 사용하는 Pod는 Node Affinity 조건을 충족하는 노드에 스케줄링되어야 합니다.

Local Volume은 특정 노드의 스토리지에 종속되므로 해당 노드에 장애가 발생하면 스토리지에 접근하지 못할 수 있습니다.

따라서 Local Volume을 사용하는 경우 노드의 가용성과 데이터 복구 전략을 함께 고려해야 합니다.

## 10. PersistentVolume 관련 주요 설정

PersistentVolume에서 자주 사용하는 주요 필드를 정리하면 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `capacity` | PV가 제공하는 스토리지 용량을 지정합니다. |
| `accessModes` | 스토리지의 접근 모드를 지정합니다. |
| `volumeMode` | 파일 시스템 또는 블록 장치 형태를 지정합니다. |
| `persistentVolumeReclaimPolicy` | PVC 삭제 이후 스토리지의 회수 정책을 지정합니다. |
| `storageClassName` | PV가 속하는 StorageClass를 지정합니다. |
| `mountOptions` | 스토리지 마운트 시 사용할 옵션을 지정합니다. |
| `claimRef` | PV가 특정 PVC에 바인딩되도록 연결 정보를 지정합니다. |
| `nodeAffinity` | PV에 접근할 수 있는 노드의 조건을 지정합니다. |

이외에도 PV의 실제 스토리지 유형에 따라 다음과 같은 설정이 사용됩니다.

- `nfs`
- `csi`
- `local`
- `iscsi`
- `hostPath`

실제 운영 환경에서는 스토리지 유형과 CSI 드라이버가 제공하는 기능에 따라 사용할 수 있는 필드가 달라집니다.

## 11. 정리

PersistentVolume은 Kubernetes 클러스터에서 영구 스토리지를 관리하기 위한 리소스입니다.

핵심 개념은 다음과 같습니다.

- PV는 Pod의 생명주기와 독립적으로 관리되는 스토리지 리소스입니다.
- 관리자가 직접 생성하는 정적 프로비저닝과 StorageClass를 활용하는 동적 프로비저닝을 지원합니다.
- PVC와 Binding 과정을 통해 애플리케이션에 스토리지를 제공합니다.
- Access Modes를 통해 스토리지의 접근 가능 범위를 정의합니다.
- Volume Mode를 통해 파일 시스템 또는 원시 블록 장치 형태로 스토리지를 제공합니다.
- Reclaim Policy를 통해 PVC 삭제 이후 실제 스토리지의 처리 방식을 결정합니다.
- Node Affinity를 통해 특정 노드에 종속된 Local Volume의 배치 조건을 지정할 수 있습니다.
- Storage Object in Use Protection을 통해 사용 중인 스토리지 리소스의 실수로 인한 삭제를 방지합니다.

PV는 실제 스토리지의 구현 방식과 애플리케이션의 스토리지 사용 방식을 분리하는 핵심 리소스입니다.

이를 통해 애플리케이션은 특정 스토리지 공급업체나 물리적인 스토리지 구성에 직접 종속되지 않고, PVC를 통해 필요한 스토리지를 사용할 수 있습니다.

## 12. 참고자료

- [Kubernetes 공식 문서 - Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes 공식 문서 - Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Kubernetes 공식 문서 - Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Kubernetes 공식 문서 - Storage Object in Use Protection](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#storage-object-in-use-protection)
- [Kubernetes 공식 문서 - Local Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#local)
- [Kubernetes API Reference - PersistentVolume](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/persistent-volume-v1/)