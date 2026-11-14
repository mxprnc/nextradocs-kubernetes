# Volume

## 1. Volume이란?

Kubernetes의 Volume은 Pod 내부에서 실행되는 컨테이너가 파일 시스템을 통해 데이터를 저장하고 공유할 수 있도록 제공하는 스토리지 추상화입니다.

컨테이너의 파일 시스템은 기본적으로 컨테이너의 생명주기에 종속됩니다. 컨테이너가 종료되거나 재시작되면 컨테이너 내부에서 생성하거나 수정한 파일이 초기화될 수 있습니다. 따라서 애플리케이션이 실행 중에 생성한 데이터를 유지하거나 여러 컨테이너가 동일한 데이터를 공유해야 하는 경우 별도의 스토리지 구성이 필요합니다.

Volume은 이러한 문제를 해결하기 위해 Pod에 스토리지를 연결하는 기능을 제공합니다.

Volume을 사용하면 다음과 같은 작업을 수행할 수 있습니다.

- 컨테이너가 재시작되더라도 Pod가 유지되는 동안 데이터를 보존할 수 있습니다.
- 하나의 Pod에 포함된 여러 컨테이너가 동일한 파일을 공유할 수 있습니다.
- ConfigMap이나 Secret에 저장된 설정 및 민감한 정보를 컨테이너의 파일로 제공할 수 있습니다.
- 임시 작업 공간이나 캐시를 위한 스토리지를 제공할 수 있습니다.
- 외부 스토리지 시스템을 연결하여 Pod의 생명주기와 독립적으로 데이터를 유지할 수 있습니다.

Volume은 Pod 단위로 정의되며, 실제 컨테이너에서 사용하려면 해당 Volume을 컨테이너의 파일 시스템에 마운트해야 합니다.

## 2. Volume의 기본 구조

Volume은 Pod의 `.spec.volumes` 필드에 정의하며, 각 컨테이너에서는 `.spec.containers[*].volumeMounts` 필드를 사용하여 Volume을 마운트할 경로를 지정합니다.

두 필드는 다음과 같은 역할을 수행합니다.

| 필드 | 역할 |
|---|---|
| `spec.volumes` | Pod에서 사용할 Volume을 정의합니다. |
| `volumeMounts` | 각 컨테이너에서 Volume을 마운트할 경로와 방식을 지정합니다. |

하나의 Pod에서 여러 Volume을 정의할 수 있으며, 각 컨테이너는 필요한 Volume을 선택하여 사용할 수 있습니다.

### 2.1. Volume 정의 및 마운트 예제

다음은 `emptyDir` Volume을 생성하고 컨테이너의 `/app/data` 경로에 마운트하는 예제입니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-example
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: app-data
          mountPath: /app/data

  volumes:
    - name: app-data
      emptyDir: {}
```

위 구성의 동작 과정은 다음과 같습니다.

1. Pod가 노드에 할당되면 `app-data`라는 이름의 Volume이 생성됩니다.
2. `emptyDir` Volume은 처음 생성될 때 비어 있는 디렉터리로 제공됩니다.
3. 컨테이너의 `volumeMounts` 설정에 따라 해당 Volume이 `/app/data` 경로에 마운트됩니다.
4. 컨테이너는 `/app/data` 경로를 통해 Volume에 데이터를 읽고 쓸 수 있습니다.

여기서 `app-data`는 Pod 내부에서 Volume을 식별하기 위한 이름이며, `mountPath`는 컨테이너 내부에서 해당 Volume에 접근할 경로입니다.

### 2.2. 여러 컨테이너에서 Volume 공유

하나의 Pod에 포함된 여러 컨테이너는 동일한 Volume을 마운트하여 데이터를 공유할 수 있습니다.

다음은 두 개의 컨테이너가 동일한 `emptyDir` Volume을 사용하는 예제입니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-example
spec:
  containers:
    - name: writer
      image: busybox
      command:
        - sh
        - -c
        - while true; do date >> /data/shared.txt; sleep 5; done
      volumeMounts:
        - name: shared-data
          mountPath: /data

    - name: reader
      image: busybox
      command:
        - sh
        - -c
        - while true; do cat /data/shared.txt; sleep 5; done
      volumeMounts:
        - name: shared-data
          mountPath: /data

  volumes:
    - name: shared-data
      emptyDir: {}
```

위 구성에서는 다음과 같이 동작합니다.

- `writer` 컨테이너는 `/data/shared.txt` 파일에 현재 시간을 기록합니다.
- `reader` 컨테이너는 동일한 파일을 읽어 내용을 출력합니다.
- 두 컨테이너는 `shared-data`라는 동일한 Volume을 사용하므로 파일을 공유할 수 있습니다.

Volume은 컨테이너마다 동일하거나 서로 다른 경로에 마운트할 수 있습니다. 따라서 하나의 Volume을 여러 컨테이너가 서로 다른 디렉터리 구조로 접근하도록 구성할 수도 있습니다.

## 3. Volume의 생명주기

Volume의 생명주기는 Volume 유형에 따라 달라집니다.

Kubernetes Volume을 이해할 때 중요한 점은 컨테이너의 생명주기와 Pod의 생명주기가 서로 다르다는 것입니다.

### 3.1. 컨테이너 재시작과 Volume

컨테이너가 크래시되거나 재시작되더라도 Pod 자체가 유지된다면 해당 Pod에 연결된 Volume의 데이터는 보존됩니다.

예를 들어 `emptyDir` Volume을 사용하는 Pod에서 컨테이너가 재시작되면 다음과 같이 동작합니다.

1. 컨테이너가 종료됩니다.
2. Kubernetes가 컨테이너를 다시 시작합니다.
3. 기존 Pod에 연결된 `emptyDir` Volume은 유지됩니다.
4. 재시작된 컨테이너는 동일한 Volume을 다시 마운트하여 기존 데이터를 사용할 수 있습니다.

즉, Volume은 컨테이너의 생명주기와 분리되어 있기 때문에 컨테이너 재시작으로 인한 데이터 손실을 방지할 수 있습니다.

### 3.2. Pod 삭제와 Volume

Pod가 삭제되면 Volume 유형에 따라 데이터 처리 방식이 달라집니다.

- **Ephemeral Volume:** Pod의 생명주기에 종속되며, Pod가 삭제되면 함께 제거됩니다.
- **Persistent Volume:** Pod의 생명주기와 독립적으로 관리되며, Pod가 삭제되더라도 스토리지 데이터가 유지됩니다.

예를 들어 `emptyDir`은 Pod가 삭제되면 데이터도 함께 삭제됩니다. 반면 PersistentVolumeClaim을 통해 연결한 PersistentVolume은 Pod가 삭제되더라도 별도의 삭제 정책에 따라 유지될 수 있습니다.

따라서 Volume을 구성할 때는 데이터가 일시적으로 필요한지, Pod가 삭제되거나 교체되더라도 유지되어야 하는지에 따라 적절한 유형을 선택해야 합니다.

## 4. Volume의 주요 유형

Kubernetes는 다양한 목적에 맞는 Volume 유형을 제공합니다.

주요 유형은 다음과 같습니다.

| Volume 유형 | 설명 |
|---|---|
| `emptyDir` | Pod가 노드에 할당될 때 생성되는 임시 디렉터리입니다. |
| `hostPath` | 노드의 파일 시스템에 존재하는 파일이나 디렉터리를 마운트합니다. |
| `persistentVolumeClaim` | PersistentVolumeClaim을 통해 PersistentVolume을 연결합니다. |
| `configMap` | ConfigMap에 저장된 설정 데이터를 파일로 제공합니다. |
| `secret` | Secret에 저장된 민감한 정보를 파일로 제공합니다. |
| `projected` | 여러 Volume 소스를 하나의 디렉터리에 통합하여 제공합니다. |
| `downwardAPI` | Pod 및 컨테이너의 메타데이터를 파일로 제공합니다. |
| `csi` | CSI 드라이버를 통해 외부 스토리지 시스템을 연결합니다. |
| `local` | 특정 노드에 연결된 로컬 디스크나 디렉터리를 사용합니다. |

### 4.1. emptyDir

`emptyDir`은 Pod가 노드에 할당될 때 생성되는 임시 Volume입니다.

처음 생성될 때 비어 있는 디렉터리로 제공되며, 동일한 Pod에 포함된 여러 컨테이너가 공유할 수 있습니다.

주요 특징은 다음과 같습니다.

- Pod가 노드에 할당되면 생성됩니다.
- Pod가 실행되는 동안 데이터가 유지됩니다.
- 컨테이너가 재시작되더라도 데이터가 유지됩니다.
- Pod가 삭제되면 Volume과 데이터도 함께 삭제됩니다.
- 기본적으로 노드의 스토리지를 사용하지만 메모리 기반 스토리지도 사용할 수 있습니다.

다음은 `emptyDir`의 기본 구성 예제입니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-example
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: cache
          mountPath: /cache

  volumes:
    - name: cache
      emptyDir: {}
```

`emptyDir`은 다음과 같은 용도로 활용할 수 있습니다.

- 애플리케이션의 임시 파일 저장
- 캐시 데이터 저장
- 여러 컨테이너 간 파일 공유
- 애플리케이션 실행 중 발생하는 임시 작업 데이터 저장

#### 메모리 기반 emptyDir

`emptyDir`은 `medium` 필드를 사용하여 메모리 기반 스토리지를 사용할 수 있습니다.

`medium: Memory`를 설정하면 Kubernetes는 tmpfs를 사용하여 데이터를 메모리에 저장합니다.

```yaml
volumes:
  - name: memory-cache
    emptyDir:
      medium: Memory
      sizeLimit: 256Mi
```

메모리 기반 `emptyDir`의 특징은 다음과 같습니다.

- 디스크 대신 메모리를 사용합니다.
- 일반적인 디스크 기반 `emptyDir`보다 빠른 데이터 접근이 가능합니다.
- Pod가 삭제되면 데이터가 제거됩니다.
- 해당 Volume에 기록된 데이터는 메모리 사용량으로 계산되며, 메모리 기반 Volume의 사용량은 컨테이너 메모리 제한에도 영향을 줍니다.

메모리 기반 Volume은 캐시나 임시 데이터를 저장하는 용도로 적합하지만, 메모리 사용량을 고려하여 적절한 용량 제한을 설정해야 합니다.

### 4.2. hostPath

`hostPath`는 Kubernetes 노드의 파일 시스템에 존재하는 파일이나 디렉터리를 Pod 내부에 마운트하는 Volume 유형입니다.

예를 들어 노드의 `/var/log` 디렉터리를 컨테이너의 `/host-logs` 경로에 연결할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-example
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: host-logs
          mountPath: /host-logs

  volumes:
    - name: host-logs
      hostPath:
        path: /var/log
        type: Directory
```

위 구성에서는 컨테이너가 `/host-logs` 경로를 통해 해당 노드의 `/var/log` 디렉터리에 접근할 수 있습니다.

`hostPath.type`은 마운트할 경로의 유형과 존재 여부를 지정하는 데 사용됩니다.

| 유형 | 설명 |
|---|---|
| `""` | 경로의 존재 여부나 유형을 검사하지 않습니다. |
| `DirectoryOrCreate` | 디렉터리가 없으면 생성합니다. |
| `Directory` | 기존 디렉터리만 허용합니다. |
| `FileOrCreate` | 파일이 없으면 생성합니다. |
| `File` | 기존 파일만 허용합니다. |
| `Socket` | 기존 Unix 소켓만 허용합니다. |
| `CharDevice` | 기존 문자 장치만 허용합니다. |
| `BlockDevice` | 기존 블록 장치만 허용합니다. |

`hostPath`는 노드의 파일 시스템에 직접 접근할 수 있다는 특징이 있지만, 다음과 같은 주의사항이 있습니다.

- 특정 노드의 파일 시스템에 종속됩니다.
- 동일한 Pod라도 실행되는 노드에 따라 접근하는 데이터가 달라질 수 있습니다.
- 노드의 중요한 파일이나 디렉터리에 접근할 경우 보안 문제가 발생할 수 있습니다.
- 노드의 파일 시스템을 직접 사용하므로 스토리지 격리와 용량 관리에 주의해야 합니다.

따라서 일반적인 애플리케이션 데이터 저장 용도로 사용하는 것은 권장되지 않습니다. 주로 노드 수준의 로그 수집, 모니터링 에이전트 등 호스트 파일 시스템에 접근해야 하는 특수한 용도에 사용됩니다.

### 4.3. persistentVolumeClaim

`persistentVolumeClaim`은 Pod에서 PersistentVolumeClaim(PVC)을 참조하여 PersistentVolume(PV)을 연결하는 Volume 유형입니다.

일반적인 애플리케이션에서는 Pod가 실제 스토리지의 종류나 위치를 직접 지정하는 대신 PVC를 통해 필요한 스토리지를 요청합니다.

다음은 PVC를 사용하는 Pod의 예제입니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-example
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
        claimName: app-data-pvc
```

위 구성에서는 `app-data-pvc`라는 PVC를 참조하여 스토리지를 연결합니다.

PVC와 연결되는 PV는 Kubernetes의 스토리지 관리 체계를 통해 결정됩니다.

이 방식은 애플리케이션과 실제 스토리지 구현을 분리할 수 있다는 장점이 있습니다.

PersistentVolume, PersistentVolumeClaim 및 StorageClass에 대한 자세한 내용은 각각의 문서에서 설명합니다.

### 4.4. configMap

`configMap` Volume은 ConfigMap에 저장된 설정 데이터를 컨테이너의 파일 시스템에 읽기 전용 파일로 제공합니다.

애플리케이션의 설정 파일을 컨테이너 이미지에 포함하지 않고 Kubernetes 리소스로 분리하여 관리할 수 있습니다.

다음은 ConfigMap Volume을 사용하는 예제입니다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  application.properties: |
    server.port=8080
    application.name=my-app
---
apiVersion: v1
kind: Pod
metadata:
  name: configmap-volume-example
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: config
          mountPath: /etc/app/config

  volumes:
    - name: config
      configMap:
        name: app-config
```

위 구성에서는 ConfigMap의 `application.properties` 항목이 `/etc/app/config/application.properties` 파일로 제공됩니다.

ConfigMap Volume은 설정 데이터를 컨테이너 이미지와 분리하여 관리할 수 있으며, ConfigMap이 변경되면 마운트된 파일에도 변경 사항이 반영될 수 있습니다.

다만 `subPath`를 사용하여 ConfigMap을 마운트한 경우에는 ConfigMap의 변경 사항이 해당 컨테이너에 자동으로 반영되지 않습니다.

### 4.5. secret

`secret` Volume은 Secret에 저장된 비밀번호, 인증서, API 키 등의 민감한 정보를 컨테이너의 파일 시스템에 읽기 전용 파일로 제공합니다.

다음은 Secret Volume을 사용하는 예제입니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  username: admin
  password: password123
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-example
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: secret-data
          mountPath: /etc/app/secrets
          readOnly: true

  volumes:
    - name: secret-data
      secret:
        secretName: app-secret
```

위 구성에서는 Secret에 저장된 데이터가 다음과 같은 파일로 제공됩니다.

```text
/etc/app/secrets/
├── username
└── password
```

각 파일의 내용은 해당 Secret 데이터의 값입니다.

Secret Volume의 주요 특징은 다음과 같습니다.

- Secret 데이터를 컨테이너 내부의 파일로 제공합니다.
- 기본적으로 읽기 전용으로 마운트됩니다.
- Linux 환경에서는 메모리 기반 파일 시스템을 사용하여 노드의 비휘발성 스토리지에 Secret 데이터를 직접 기록하지 않습니다.
- Secret이 변경되면 마운트된 파일에도 변경 사항이 반영될 수 있습니다.
- `subPath`를 사용한 마운트에서는 Secret 변경 사항이 자동으로 반영되지 않습니다.

Secret은 민감한 정보를 관리하기 위한 리소스이지만, Secret을 사용한다는 사실만으로 데이터가 자동으로 암호화되는 것은 아닙니다. 실제 보안 수준을 높이려면 API 서버의 암호화 설정, RBAC 권한 관리 등 추가적인 보안 구성이 필요합니다.

### 4.6. projected

`projected` Volume은 여러 Volume 소스를 하나의 디렉터리에 통합하여 제공하는 Volume 유형입니다.

여러 설정 정보나 메타데이터를 각각 별도의 경로에 마운트하는 대신 하나의 Volume으로 통합하여 관리할 수 있습니다.

지원하는 주요 소스는 다음과 같습니다.

- `secret`
- `configMap`
- `downwardAPI`
- `serviceAccountToken`

다음은 ConfigMap과 Secret을 하나의 디렉터리에 통합하는 예제입니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: projected-example
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: application-config
          mountPath: /etc/app/config

  volumes:
    - name: application-config
      projected:
        sources:
          - configMap:
              name: app-config
          - secret:
              name: app-secret
```

위 구성에서는 ConfigMap과 Secret의 데이터가 `/etc/app/config` 디렉터리에 통합되어 제공됩니다.

projected Volume은 여러 설정 소스를 하나의 디렉터리 구조로 관리해야 하는 경우 유용합니다.

### 4.7. downwardAPI

`downwardAPI` Volume은 Pod 및 컨테이너의 메타데이터를 컨테이너 내부의 파일로 제공합니다.

애플리케이션이 Kubernetes API 서버에 직접 접근하지 않고도 자신의 실행 환경에 대한 정보를 확인할 수 있도록 지원합니다.

제공할 수 있는 정보의 예시는 다음과 같습니다.

- Pod 이름
- Pod 네임스페이스
- Pod IP 주소
- 노드 이름
- Pod에 설정된 리소스 요청 및 제한

다음은 Pod 이름과 네임스페이스를 파일로 제공하는 예제입니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downwardapi-example
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: pod-info
          mountPath: /etc/pod-info

  volumes:
    - name: pod-info
      downwardAPI:
        items:
          - path: pod-name
            fieldRef:
              fieldPath: metadata.name
          - path: namespace
            fieldRef:
              fieldPath: metadata.namespace
```

위 구성에서는 다음과 같은 파일이 생성됩니다.

```text
/etc/pod-info/
├── pod-name
└── namespace
```

각 파일에는 해당 Pod의 메타데이터 값이 저장됩니다.

### 4.8. CSI

CSI(Container Storage Interface)는 Kubernetes와 외부 스토리지 시스템 사이의 표준 인터페이스입니다.

CSI를 통해 스토리지 공급업체는 Kubernetes의 핵심 코드와 독립적으로 스토리지 드라이버를 개발하고 배포할 수 있습니다.

Kubernetes는 CSI 드라이버를 통해 다음과 같은 스토리지 작업을 수행할 수 있습니다.

- 스토리지 볼륨 생성 및 삭제
- 스토리지 볼륨 연결 및 해제
- 스토리지 마운트 및 마운트 해제
- 스토리지 용량 확장
- 스토리지 스냅샷 생성 및 관리

CSI 드라이버는 Kubernetes 클러스터에 별도로 설치해야 하며, 실제 지원 기능은 사용하는 드라이버에 따라 달라집니다.

CSI 스토리지는 다음과 같은 방식으로 사용할 수 있습니다.

- PersistentVolume을 통해 연결하는 방식
- Generic Ephemeral Volume을 통해 Pod에 임시 스토리지를 제공하는 방식
- CSI Ephemeral Volume을 통해 Pod에 직접 임시 스토리지를 제공하는 방식

일반적인 애플리케이션에서는 PVC를 통해 CSI 기반 스토리지를 사용하는 방식이 널리 활용됩니다.

### 4.9. local

`local` Volume은 특정 노드에 연결된 디스크, 파티션 또는 디렉터리를 사용하는 Volume 유형입니다.

`hostPath`와 마찬가지로 노드의 로컬 스토리지를 사용하지만, PersistentVolume의 `nodeAffinity`를 통해 해당 스토리지가 위치한 노드를 명시적으로 관리할 수 있습니다.

주요 특징은 다음과 같습니다.

- 특정 노드에 연결된 로컬 스토리지를 사용합니다.
- PersistentVolume을 통해 정적으로 구성해야 합니다.
- 동적 프로비저닝은 지원하지 않습니다.
- PersistentVolume의 Node Affinity를 통해 스토리지가 위치한 노드를 지정합니다.
- 해당 노드의 장애가 발생하면 스토리지에 접근할 수 없을 수 있습니다.

`local` Volume은 네트워크 스토리지보다 낮은 지연 시간이 중요한 워크로드 등에 활용할 수 있습니다.

다만 특정 노드에 종속되므로 노드 장애에 따른 가용성 저하와 데이터 손실 가능성을 고려해야 합니다.

## 5. VolumeMount 주요 설정

`volumeMounts`는 Pod에 정의된 Volume을 개별 컨테이너의 파일 시스템에 연결하는 설정입니다.

하나의 Pod에서 Volume을 정의하더라도 실제 컨테이너에서 사용하려면 `volumeMounts`를 통해 마운트해야 합니다.

주요 설정은 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `name` | Pod에 정의된 Volume의 이름을 지정합니다. |
| `mountPath` | 컨테이너 내부에서 Volume을 마운트할 경로를 지정합니다. |
| `readOnly` | 해당 컨테이너에서 Volume을 읽기 전용으로 마운트할지 지정합니다. |
| `subPath` | Volume 전체가 아닌 특정 하위 디렉터리나 파일을 마운트합니다. |
| `subPathExpr` | 환경 변수를 사용하여 마운트할 하위 경로를 동적으로 지정합니다. |
| `mountPropagation` | 마운트 전파 방식을 지정합니다. |

### 5.1. readOnly

`readOnly`는 특정 컨테이너에서 Volume을 읽기 전용으로 마운트하도록 설정합니다.

```yaml
volumeMounts:
  - name: shared-data
    mountPath: /app/data
    readOnly: true
```

위 구성에서는 컨테이너가 `/app/data` 경로를 통해 데이터를 읽을 수 있지만 수정하거나 삭제할 수는 없습니다.

`readOnly`는 해당 컨테이너의 마운트에 적용되는 설정입니다. 동일한 Volume을 사용하는 다른 컨테이너에서는 별도로 읽기 및 쓰기 권한을 설정할 수 있습니다.

또한 읽기 전용 마운트는 Volume 자체를 읽기 전용으로 변경하는 것이 아니라 해당 컨테이너의 마운트 접근 방식을 제한합니다.

### 5.2. subPath

`subPath`는 Volume의 특정 하위 디렉터리나 파일만 컨테이너에 마운트할 때 사용합니다.

하나의 Volume을 여러 컨테이너에서 서로 다른 용도로 사용하거나, 컨테이너의 특정 경로에 기존 디렉터리 구조를 유지하면서 데이터를 연결할 때 유용합니다.

다음은 하나의 Volume을 서로 다른 경로에 나누어 마운트하는 예제입니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: subpath-example
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: application-data
          mountPath: /app/config
          subPath: config

        - name: application-data
          mountPath: /app/logs
          subPath: logs

  volumes:
    - name: application-data
      emptyDir: {}
```

위 구성에서는 하나의 `application-data` Volume을 다음과 같이 나누어 사용합니다.

```text
application-data
├── config  → /app/config
└── logs    → /app/logs
```

각 마운트는 동일한 Volume을 사용하지만 서로 다른 하위 경로를 대상으로 합니다.

단, ConfigMap, Secret, DownwardAPI 등을 `subPath`로 마운트한 경우 원본 데이터가 변경되더라도 마운트된 파일에는 변경 사항이 자동으로 반영되지 않습니다.

### 5.3. subPathExpr

`subPathExpr`는 환경 변수를 사용하여 Volume 내부의 하위 경로를 동적으로 지정할 때 사용합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: subpath-expr-example
spec:
  containers:
    - name: app
      image: busybox
      command:
        - sh
        - -c
        - sleep 3600
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
      volumeMounts:
        - name: application-data
          mountPath: /app/data
          subPathExpr: $(POD_NAME)

  volumes:
    - name: application-data
      emptyDir: {}
```

위 구성에서는 Pod의 이름이 `subpath-expr-example`이므로 Volume 내부의 해당 디렉터리가 `/app/data`에 마운트됩니다.

`subPath`와 `subPathExpr`는 동시에 사용할 수 없으며 둘 중 하나만 지정해야 합니다.

### 5.4. mountPropagation

`mountPropagation`은 컨테이너 내부에서 발생한 마운트 작업이나 호스트에서 발생한 마운트 작업이 다른 마운트 영역으로 전파되는 방식을 제어합니다.

주요 설정값은 다음과 같습니다.

| 설정값 | 설명 |
|---|---|
| `None` | 추가적인 마운트 전파를 허용하지 않습니다. 기본값입니다. |
| `HostToContainer` | 호스트에서 발생한 마운트 변경 사항을 컨테이너에서 확인할 수 있도록 합니다. |
| `Bidirectional` | 호스트에서 컨테이너로의 전파뿐 아니라 컨테이너에서 발생한 마운트 변경 사항도 호스트로 전파합니다. |

`Bidirectional`은 호스트 시스템의 마운트 구조에 영향을 줄 수 있으므로 주의해야 합니다. 이 설정은 특권 컨테이너에서만 사용할 수 있으며, 호스트 시스템에 대한 충분한 이해가 필요합니다.

일반적인 애플리케이션에서는 기본값인 `None`을 사용하는 것이 적절합니다.

## 6. Ephemeral Volume과 Persistent Volume

Kubernetes Volume은 데이터의 유지 기간과 관리 방식에 따라 Ephemeral Volume과 Persistent Volume으로 구분할 수 있습니다.

### 6.1. Ephemeral Volume

Ephemeral Volume은 Pod의 생명주기에 종속되는 임시 스토리지입니다.

Pod가 실행되는 동안 데이터를 저장하고 공유할 수 있지만, Pod가 삭제되면 해당 Volume도 함께 제거됩니다.

주요 유형은 다음과 같습니다.

- `emptyDir`
- `configMap`
- `secret`
- `downwardAPI`
- `projected`
- CSI Ephemeral Volume
- Generic Ephemeral Volume

주로 임시 파일, 캐시, 설정 파일, 인증 정보 등을 제공하는 용도로 사용합니다.

### 6.2. Persistent Volume

Persistent Volume은 Pod의 생명주기와 독립적으로 데이터를 유지하기 위한 스토리지입니다.

Kubernetes에서는 PersistentVolume(PV)과 PersistentVolumeClaim(PVC)을 사용하여 영구 스토리지를 관리합니다.

Pod는 PVC를 참조하여 스토리지를 사용하며, Pod가 삭제되더라도 실제 스토리지 데이터는 PV의 생명주기와 회수 정책에 따라 유지될 수 있습니다.

Persistent Volume은 데이터베이스, 파일 저장소 등 지속적인 데이터 보존이 필요한 애플리케이션에 적합합니다.

### 6.3. 비교

| 구분 | Ephemeral Volume | Persistent Volume |
|---|---|---|
| 데이터 유지 | Pod의 생명주기에 종속됩니다. | Pod의 생명주기와 독립적으로 관리됩니다. |
| 컨테이너 재시작 | Pod가 유지되면 데이터가 보존됩니다. | 데이터가 유지됩니다. |
| Pod 삭제 | 일반적으로 Volume과 데이터가 제거됩니다. | Pod 삭제와 별개로 스토리지가 유지될 수 있습니다. |
| 주요 용도 | 캐시, 임시 파일, 설정 데이터 | 데이터베이스, 파일 저장소, 영구 데이터 |
| 대표 유형 | `emptyDir`, `configMap`, `secret` | PVC를 통한 PV 연결 |

Ephemeral Volume과 Persistent Volume의 차이는 단순히 디스크와 메모리의 차이가 아닙니다. 핵심은 스토리지의 생명주기가 Pod에 종속되는지, Pod와 독립적으로 관리되는지에 있습니다.

## 7. Volume과 관련 리소스의 관계

Volume은 Pod에서 스토리지를 사용하기 위한 기본적인 추상화이며, 영구 스토리지를 관리하기 위해서는 여러 Kubernetes 리소스와 함께 사용됩니다.

주요 리소스의 관계는 다음과 같습니다.

### 7.1. Volume과 PersistentVolume

PersistentVolume(PV)은 Kubernetes 클러스터에서 관리하는 실제 영구 스토리지 리소스입니다.

Volume이 Pod 내부에서 스토리지를 사용하는 방법을 정의한다면, PV는 클러스터에서 사용할 수 있는 영구 스토리지 자체를 나타냅니다.

PV는 관리자가 직접 생성하거나 StorageClass와 스토리지 프로비저너를 통해 동적으로 생성할 수 있습니다.

### 7.2. Volume과 PersistentVolumeClaim

PersistentVolumeClaim(PVC)은 애플리케이션이 필요한 스토리지 용량과 접근 모드를 요청하는 리소스입니다.

Pod는 PVC를 직접 Volume의 소스로 지정하여 영구 스토리지를 사용합니다.

일반적인 연결 구조는 다음과 같습니다.

```text
Pod
 │
 │ volumes
 ▼
PersistentVolumeClaim (PVC)
 │
 │ Binding
 ▼
PersistentVolume (PV)
 │
 ▼
Actual Storage
```

Pod는 PVC를 통해 스토리지를 요청하며, Kubernetes는 해당 PVC에 적합한 PV를 연결합니다.

### 7.3. Volume과 StorageClass

StorageClass는 스토리지를 동적으로 생성할 때 사용할 프로비저닝 방식과 관련 설정을 정의하는 리소스입니다.

예를 들어 클라우드 환경에서는 StorageClass를 통해 특정 스토리지 유형이나 디스크 성능 등을 지정할 수 있습니다.

PVC가 StorageClass를 참조하면 Kubernetes의 동적 프로비저닝을 통해 필요한 PV를 생성할 수 있습니다.

전체적인 관계는 다음과 같습니다.

```text
Pod
 │
 ▼
Volume
 │
 ▼
PVC
 │
 ▼
StorageClass
 │
 ▼
Provisioner / CSI Driver
 │
 ▼
Storage System
 │
 ▼
PV
```

위 구조에서 StorageClass는 스토리지 프로비저닝 방식을 정의하고, 실제 PV 생성 및 연결은 스토리지 프로비저너와 Kubernetes의 볼륨 관리 과정에 의해 수행됩니다.

StorageClass, PersistentVolume 및 PersistentVolumeClaim의 상세 동작은 각각의 문서에서 설명합니다.

## 8. 정리

Kubernetes Volume은 컨테이너의 파일 시스템과 별도로 스토리지를 제공하여 데이터 저장 및 컨테이너 간 데이터 공유를 지원하는 기능입니다.

핵심 개념은 다음과 같습니다.

- Volume은 Pod 단위로 정의되며, 컨테이너에서는 `volumeMounts`를 통해 사용합니다.
- 하나의 Pod에 포함된 여러 컨테이너는 동일한 Volume을 공유할 수 있습니다.
- 컨테이너가 재시작되더라도 Pod가 유지되는 동안 Volume의 데이터는 보존됩니다.
- Volume 유형에 따라 데이터의 저장 위치와 생명주기가 달라집니다.
- `emptyDir`은 Pod의 생명주기에 종속되는 임시 스토리지입니다.
- `hostPath`는 노드의 파일 시스템을 직접 마운트하는 방식입니다.
- PVC를 사용하면 Pod의 생명주기와 독립적인 영구 스토리지를 연결할 수 있습니다.
- CSI는 외부 스토리지 시스템과 Kubernetes를 연결하기 위한 표준 인터페이스입니다.

따라서 Volume을 설계할 때는 데이터의 보존 기간, 공유 범위, 스토리지 유형, 노드 종속성 및 접근 권한 등을 고려해야 합니다.

## 9. 참고자료

- [Kubernetes 공식 문서 - Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Kubernetes 공식 문서 - Ephemeral Volumes](https://kubernetes.io/docs/concepts/storage/ephemeral-volumes/)
- [Kubernetes 공식 문서 - Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes 공식 문서 - ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes 공식 문서 - Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Kubernetes 공식 문서 - Container Storage Interface (CSI)](https://kubernetes.io/docs/concepts/storage/volumes/#csi)