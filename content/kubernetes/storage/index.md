# Kubernetes Storage Resources

Storage 리소스는 Kubernetes에서 Pod와 컨테이너에 스토리지를 연결하고 데이터를 저장 및 관리하기 위한 리소스입니다.

Pod의 생명주기와 스토리지의 생명주기를 분리하여 애플리케이션이 재시작되거나 Pod가 재생성되더라도 데이터를 유지할 수 있도록 지원합니다.

## 1. Storage 리소스 구성

| 리소스 | 주요 역할 | 주요 특징 |
|---|---|---|
| **Volume** | Pod에 스토리지 제공 | 컨테이너 간 스토리지 공유 및 데이터 저장 |
| **PersistentVolume (PV)** | 영속 스토리지 제공 | Pod의 생명주기와 독립적인 스토리지 리소스 |
| **PersistentVolumeClaim (PVC)** | 스토리지 요청 | 필요한 용량 및 접근 모드 등을 지정 |
| **StorageClass** | 스토리지 프로비저닝 정의 | 스토리지 유형 및 동적 프로비저닝 설정 |
| **VolumeSnapshot** | 볼륨 스냅샷 생성 | 특정 시점의 스토리지 상태 보존 |

## 2. 리소스 관계

### 2.1 Volume 기반 스토리지 연결

```text
Pod
 │
 ├── Container 1
 │       │
 │       └── Volume
 │
 └── Container 2
         │
         └── Volume
```

Volume은 Pod에 연결되며, 동일한 Pod 내 여러 컨테이너가 하나의 Volume을 공유할 수 있습니다.

### 2.2 PersistentVolume 및 PersistentVolumeClaim

```text
Pod
 │
 ▼
PVC (PersistentVolumeClaim)
 │
 ▼
PV (PersistentVolume)
 │
 ▼
실제 스토리지
```

PVC는 애플리케이션이 필요한 스토리지를 요청하는 리소스이며, PV는 해당 요청에 연결되는 실제 영속 스토리지 리소스입니다.

### 2.3 StorageClass 기반 동적 프로비저닝

```text
Pod
 │
 ▼
PVC
 │
 ├── StorageClass
 │       │
 │       ▼
 │    Provisioner
 │       │
 │       ▼
 │    실제 스토리지 생성
 │       │
 │       ▼
 │       PV
 │
 ▼
스토리지 사용
```

StorageClass를 지정한 PVC가 생성되면 Provisioner가 필요한 스토리지를 동적으로 생성하고, 이에 대응하는 PV를 프로비저닝할 수 있습니다.

### 2.4 VolumeSnapshot 기반 스토리지 백업

```text
PersistentVolume
       │
       ▼
VolumeSnapshot
       │
       ▼
Snapshot 데이터
       │
       ▼
VolumeSnapshotContent
```

VolumeSnapshot은 기존 볼륨의 특정 시점 상태를 스냅샷으로 생성합니다.

VolumeSnapshotContent는 실제 스냅샷 데이터를 나타내는 클러스터 리소스이며, VolumeSnapshot과 연결되어 관리됩니다.

스냅샷을 기반으로 새로운 PVC를 생성하여 데이터를 복원할 수도 있습니다.

## 3. 기능에 따른 분류

| 기능 | 관련 리소스 |
|---|---|
| Pod에 스토리지 연결 | Volume |
| 영속 스토리지 관리 | PersistentVolume |
| 스토리지 사용 요청 | PersistentVolumeClaim |
| 스토리지 유형 및 동적 프로비저닝 | StorageClass |
| 스토리지 스냅샷 및 복구 | VolumeSnapshot, VolumeSnapshotContent |

## 4. 주요 스토리지 접근 모드

PersistentVolume 및 PersistentVolumeClaim에서 사용하는 주요 접근 모드는 다음과 같습니다.

| 접근 모드 | 설명 |
|---|---|
| ReadWriteOnce (RWO) | 하나의 Node에서 읽기 및 쓰기 가능 |
| ReadOnlyMany (ROX) | 여러 Node에서 읽기 전용으로 사용 가능 |
| ReadWriteMany (RWX) | 여러 Node에서 읽기 및 쓰기 가능 |
| ReadWriteOncePod (RWOP) | 하나의 Pod에서만 읽기 및 쓰기 가능 |

실제 지원 여부는 스토리지 구현체 및 CSI 드라이버에 따라 달라집니다.

## 5. StatefulSet과 Storage 리소스의 관계

StatefulSet은 `volumeClaimTemplates`를 사용하여 각 Pod에 독립적인 PVC를 생성할 수 있습니다.

```text
StatefulSet
    │
    ├── Pod: mysql-0
    │      └── PVC: data-mysql-0
    │              └── PV
    │
    ├── Pod: mysql-1
    │      └── PVC: data-mysql-1
    │              └── PV
    │
    └── Pod: mysql-2
           └── PVC: data-mysql-2
                   └── PV
```

이를 통해 각 Pod가 독립적인 스토리지를 사용할 수 있으며, Pod가 재생성되더라도 기존 PVC와 연결된 데이터를 유지할 수 있습니다.

## 6. 핵심 정리

- Volume은 Pod에 스토리지를 연결하는 기본 단위입니다.
- PersistentVolume은 클러스터에서 관리하는 영속 스토리지입니다.
- PersistentVolumeClaim은 애플리케이션이 필요한 스토리지를 요청하는 리소스입니다.
- StorageClass는 스토리지 유형과 동적 프로비저닝 방식을 정의합니다.
- VolumeSnapshot은 기존 볼륨의 특정 시점 상태를 보존하고 복구하는 데 사용합니다.
- StatefulSet은 Pod별 PVC를 생성하여 안정적인 스토리지 구성을 지원합니다.

참고:
- https://kubernetes.io/docs/concepts/storage/
- https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- https://kubernetes.io/docs/concepts/storage/volume-snapshots/