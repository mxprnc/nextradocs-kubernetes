# StatefulSet

## 1. StatefulSet이란?

**StatefulSet은 Kubernetes에서 안정적인 식별자와 스토리지를 유지해야 하는 애플리케이션을 배포하고 관리하기 위한 Workload 리소스입니다.**

Deployment는 일반적으로 동일한 구성을 가진 Pod를 여러 개 생성하고 관리하는 데 사용됩니다. 각 Pod는 서로 독립적인 인스턴스로 취급되며, Pod가 삭제되고 새로운 Pod가 생성되면 기존 Pod의 이름이나 IP 주소가 유지되지 않습니다.

반면 StatefulSet은 각 Pod에 고유하고 안정적인 식별자를 부여하며, Pod별로 독립적인 영구 스토리지를 연결할 수 있습니다. Pod가 재생성되더라도 동일한 순서 번호와 이름을 사용하고, 기존 PVC를 다시 연결할 수 있습니다.

StatefulSet은 주로 다음과 같은 애플리케이션에 사용됩니다.

- 데이터베이스(MySQL, PostgreSQL 등)
- 분산 데이터 저장소(Cassandra, MongoDB 등)
- 메시징 시스템(Kafka 등)
- 안정적인 네트워크 식별자가 필요한 분산 애플리케이션
- 각 인스턴스가 독립적인 데이터를 유지해야 하는 애플리케이션

StatefulSet의 주요 특징은 다음과 같습니다.

- Pod마다 고유하고 안정적인 이름을 부여합니다.
- Pod의 생성 및 삭제 순서를 제어할 수 있습니다.
- Headless Service와 연동하여 Pod별 안정적인 네트워크 식별자를 제공합니다.
- `volumeClaimTemplates`를 사용하여 Pod별로 독립적인 PVC를 생성할 수 있습니다.
- Pod가 재생성되더라도 기존 PVC를 다시 연결할 수 있습니다.
- 업데이트 전략을 통해 Pod 교체 순서와 시점을 제어할 수 있습니다.

StatefulSet은 단순히 Pod의 개수를 유지하는 것뿐 아니라 각 Pod의 정체성과 스토리지 상태를 유지해야 하는 애플리케이션에 적합합니다.

다만 StatefulSet 자체가 데이터베이스의 복제, 리더 선출, 데이터 동기화 또는 장애 조치(Failover)를 구현하는 것은 아닙니다. 이러한 기능은 애플리케이션이나 별도의 Operator 등을 통해 구성해야 합니다.

## 2. StatefulSet의 구조

StatefulSet은 Pod Template과 Volume Claim Template 등을 기반으로 Pod를 생성하고 관리합니다.

일반적인 구조는 다음과 같습니다.

```text
Kubernetes Cluster
│
├── Headless Service
│
└── StatefulSet
    │
    ├── Pod: mysql-0
    │   └── Container
    │   └── PVC: data-mysql-0
    │       └── PV
    │
    ├── Pod: mysql-1
    │   └── Container
    │   └── PVC: data-mysql-1
    │       └── PV
    │
    └── Pod: mysql-2
        └── Container
        └── PVC: data-mysql-2
            └── PV
```

예를 들어 StatefulSet의 이름이 `mysql`이고 복제본 개수가 3개라면 다음과 같은 Pod가 생성됩니다.

```text
mysql-0
mysql-1
mysql-2
```

각 Pod는 고유한 순서 번호(Ordinal)를 가지며, 해당 번호를 기반으로 이름이 결정됩니다. 기본적으로 순서 번호는 0부터 시작하지만, `spec.ordinals.start`를 설정하여 시작 번호를 변경할 수도 있습니다.

또한 각 Pod에는 독립적인 PVC가 연결될 수 있습니다.

```text
StatefulSet: mysql
│
├── mysql-0
│   └── data-mysql-0
│
├── mysql-1
│   └── data-mysql-1
│
└── mysql-2
    └── data-mysql-2
```

이러한 구조를 통해 각 Pod는 자신에게 할당된 스토리지를 지속적으로 사용할 수 있습니다.

## 3. StatefulSet YAML 구성

StatefulSet은 YAML Manifest를 통해 정의할 수 있습니다.

다음은 Nginx 컨테이너 3개를 실행하고 각 Pod에 1GiB의 영구 스토리지를 연결하는 StatefulSet 예제입니다.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-statefulset
spec:
  serviceName: nginx-headless
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
          image: nginx:1.27
          ports:
            - containerPort: 80
              name: web

          volumeMounts:
            - name: data
              mountPath: /usr/share/nginx/html

  volumeClaimTemplates:
    - metadata:
        name: data

      spec:
        accessModes:
          - ReadWriteOnce

        resources:
          requests:
            storage: 1Gi
```

이 예제에서는 `nginx-headless`라는 Service가 별도로 생성되어 있어야 합니다. PVC가 실제 PV에 바인딩되려면 클러스터에 적절한 StorageClass가 있거나, 요청 조건에 맞는 PV가 준비되어 있어야 합니다.

각 필드의 역할은 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `apiVersion` | Kubernetes API 버전을 지정합니다. StatefulSet은 `apps/v1`을 사용합니다. |
| `kind` | 생성할 리소스의 종류를 지정합니다. |
| `metadata` | StatefulSet의 이름과 메타데이터를 정의합니다. |
| `spec` | StatefulSet의 원하는 상태를 정의합니다. |
| `serviceName` | StatefulSet의 Pod에 안정적인 네트워크 식별자를 제공하는 Service의 이름을 지정합니다. 일반적으로 Headless Service를 사용합니다. |
| `replicas` | 유지하려는 Pod 복제본의 개수를 지정합니다. |
| `selector` | StatefulSet이 관리할 Pod를 식별하는 조건을 지정합니다. |
| `template` | 생성할 Pod의 구성을 정의합니다. |
| `volumeClaimTemplates` | 각 Pod에 연결할 PVC의 템플릿을 정의합니다. |

### 3.1. serviceName

`serviceName`은 StatefulSet에 속한 Pod의 안정적인 네트워크 식별자를 제공하는 Service의 이름입니다.

일반적으로 Headless Service를 사용합니다.

```yaml
spec:
  serviceName: nginx-headless
```

Headless Service는 `clusterIP: None`으로 설정된 Service입니다. 일반적인 ClusterIP Service처럼 하나의 가상 IP 주소를 제공하지 않고, 연결 대상 Pod의 개별 IP 주소를 DNS 레코드로 제공합니다.

StatefulSet은 이를 활용하여 각 Pod에 안정적인 DNS 이름을 제공할 수 있습니다.

예를 들어 StatefulSet 이름이 `nginx-statefulset`이고 Service 이름이 `nginx-headless`이며 네임스페이스가 `default`라면 다음과 같은 DNS 이름을 사용할 수 있습니다.

```text
nginx-statefulset-0.nginx-headless.default.svc.cluster.local
nginx-statefulset-1.nginx-headless.default.svc.cluster.local
nginx-statefulset-2.nginx-headless.default.svc.cluster.local
```

Pod가 재생성되어 IP 주소가 변경되더라도 동일한 DNS 이름을 사용하여 해당 Pod를 식별할 수 있습니다.

### 3.2. replicas

`replicas`는 StatefulSet이 유지하려는 Pod 복제본의 개수를 지정합니다.

```yaml
spec:
  replicas: 3
```

위 설정은 기본 Ordinal 설정을 사용하는 경우 다음과 같은 Pod를 생성합니다.

```text
nginx-statefulset-0
nginx-statefulset-1
nginx-statefulset-2
```

StatefulSet은 기본적으로 낮은 순서 번호의 Pod부터 생성하며, 높은 순서 번호의 Pod부터 삭제합니다. 이러한 순서 제어는 `podManagementPolicy`와 업데이트 전략의 영향을 받습니다.

### 3.3. selector

`selector`는 StatefulSet이 관리할 Pod를 식별하는 조건입니다.

```yaml
spec:
  selector:
    matchLabels:
      app: nginx
```

Selector는 Pod Template에 정의된 레이블과 일치해야 합니다.

```yaml
spec:
  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx
```

`apps/v1` API에서 Selector는 필수이며, StatefulSet 생성 이후 변경할 수 없습니다. Selector와 Pod Template의 레이블이 일치하지 않으면 리소스 생성 시 검증 오류가 발생합니다.

### 3.4. template

`template`은 StatefulSet이 생성할 Pod의 구성을 정의합니다.

```yaml
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

Pod Template에는 컨테이너 이미지, 환경 변수, 리소스 요청량, 볼륨 마운트, 프로브 등의 실행 환경을 정의할 수 있습니다.

StatefulSet Controller는 이 Template을 기준으로 각 Ordinal에 해당하는 Pod를 생성하고 관리합니다.

### 3.5. volumeClaimTemplates

`volumeClaimTemplates`는 StatefulSet이 생성하는 각 Pod에 연결할 PVC의 템플릿을 정의합니다.

```yaml
volumeClaimTemplates:
  - metadata:
      name: data

    spec:
      accessModes:
        - ReadWriteOnce

      resources:
        requests:
          storage: 1Gi
```

StatefulSet이 3개의 Pod를 생성하면 다음과 같이 각각 독립적인 PVC가 생성됩니다.

```text
StatefulSet
│
├── Pod: nginx-statefulset-0
│   └── PVC: data-nginx-statefulset-0
│
├── Pod: nginx-statefulset-1
│   └── PVC: data-nginx-statefulset-1
│
└── Pod: nginx-statefulset-2
    └── PVC: data-nginx-statefulset-2
```

PVC 이름은 다음 규칙으로 생성됩니다.

```text
<volumeClaimTemplate 이름>-<StatefulSet 이름>-<Pod Ordinal>
```

각 PVC는 별도의 PV와 연결될 수 있으며, 이를 통해 Pod마다 독립적인 데이터를 저장할 수 있습니다.

PVC와 PV의 관계는 `persistent-volume-claim.md`와 `persistent-volume.md`에서 별도로 설명합니다.

## 4. StatefulSet의 주요 특징

### 4.1. 안정적인 Pod 식별자 (Stable Identity)

StatefulSet은 각 Pod에 고유한 이름을 부여합니다.

Deployment에서 생성되는 Pod는 일반적으로 ReplicaSet이 생성하는 무작위 문자열 기반 이름을 사용합니다.

반면 StatefulSet은 다음과 같이 순서 번호를 기반으로 Pod 이름을 생성합니다.

```text
StatefulSet: mysql

mysql-0
mysql-1
mysql-2
```

Pod가 삭제되더라도 동일한 Ordinal을 가진 새로운 Pod가 생성됩니다.

예를 들어 `mysql-1`이 삭제되면 StatefulSet은 새로운 `mysql-1`을 생성합니다.

```text
Before

mysql-0
mysql-1
mysql-2


mysql-1 삭제


After

mysql-0
mysql-1 (New Pod)
mysql-2
```

새롭게 생성된 Pod는 기존 Pod와 동일한 이름을 사용하지만, 실제 Kubernetes 객체로서는 새로운 Pod입니다.

따라서 Pod의 UID는 변경되며, 기존 Pod의 실행 상태나 메모리 상태가 그대로 복구되는 것은 아닙니다.

### 4.2. 안정적인 네트워크 식별자 (Stable Network Identity)

StatefulSet은 Pod마다 안정적인 네트워크 식별자를 제공합니다.

일반적으로 Headless Service와 함께 사용하여 Pod별 DNS 이름을 생성합니다.

예를 들어 StatefulSet 이름이 `mysql`이고 Service 이름이 `mysql-headless`이며 네임스페이스가 `default`라면 다음과 같은 DNS 이름을 사용할 수 있습니다.

```text
mysql-0.mysql-headless.default.svc.cluster.local
mysql-1.mysql-headless.default.svc.cluster.local
mysql-2.mysql-headless.default.svc.cluster.local
```

Pod가 재생성되더라도 동일한 이름과 DNS 식별자를 다시 사용할 수 있습니다.

다만 DNS 이름이 안정적으로 유지된다는 의미이지, Pod의 IP 주소가 영구적으로 유지된다는 의미는 아닙니다. Pod가 재생성되면 IP 주소는 변경될 수 있습니다.

또한 DNS 레코드의 게시 여부와 조회 가능 시점은 Service의 설정 및 DNS 캐시 정책에 영향을 받습니다. 기본적으로 Ready 상태가 아닌 Pod는 일반적인 Service DNS 게시 대상에서 제외될 수 있으며, `publishNotReadyAddresses` 설정을 사용하면 Ready 상태와 관계없이 주소를 게시하도록 구성할 수 있습니다.

### 4.3. 안정적인 스토리지 (Stable Storage)

StatefulSet은 `volumeClaimTemplates`를 사용하여 각 Pod에 독립적인 PVC를 생성할 수 있습니다.

Pod가 삭제되더라도 해당 PVC는 기본적으로 자동 삭제되지 않습니다. 따라서 동일한 Ordinal을 가진 Pod가 다시 생성되면 기존 PVC를 다시 연결하여 데이터를 유지할 수 있습니다.

```text
Before

mysql-0
└── data-mysql-0

mysql-1
└── data-mysql-1

mysql-2
└── data-mysql-2


mysql-1 삭제


After

mysql-0
└── data-mysql-0

mysql-1 (New Pod)
└── data-mysql-1 (Existing PVC)

mysql-2
└── data-mysql-2
```

이러한 특성은 데이터베이스나 분산 저장 시스템처럼 각 인스턴스가 자신의 데이터를 유지해야 하는 애플리케이션에 중요합니다.

단, 실제 데이터 보존 여부는 PVC 및 PV의 상태, 스토리지 시스템의 동작, PersistentVolume의 Reclaim Policy 등에 따라 달라집니다. StatefulSet이 PVC를 유지하더라도 연결된 스토리지의 장애나 데이터 손실까지 방지하는 것은 아닙니다.

## 5. StatefulSet의 Pod 생성 및 삭제 순서

StatefulSet은 Pod의 생성과 삭제 순서를 제어할 수 있습니다.

기본적으로 StatefulSet은 순서가 낮은 Pod부터 생성하고, 순서가 높은 Pod부터 삭제합니다.

### 5.1. Pod 생성 순서

StatefulSet의 기본 Pod Management Policy는 `OrderedReady`입니다.

`OrderedReady`에서는 Pod가 순서대로 생성되며, 이전 Pod가 Ready 상태가 되어야 다음 Pod의 생성이 진행됩니다.

예를 들어 복제본을 3개로 설정하면 다음과 같이 동작합니다.

```text
StatefulSet 생성
      │
      ▼
  mysql-0 생성
      │
      ▼
  mysql-0 Ready
      │
      ▼
  mysql-1 생성
      │
      ▼
  mysql-1 Ready
      │
      ▼
  mysql-2 생성
      │
      ▼
  mysql-2 Ready
```

이러한 순차적 생성 방식은 Pod 간 시작 순서에 의존성이 존재하는 분산 시스템에서 유용합니다.

다만 Pod가 Ready 상태가 되는 것과 애플리케이션 클러스터가 정상적으로 구성되는 것은 별개의 문제입니다. 애플리케이션 수준의 초기화 및 클러스터 참여 상태를 확인하려면 적절한 Readiness Probe를 구성해야 합니다.

### 5.2. Pod 삭제 순서

기본 `OrderedReady` 정책에서는 높은 순서 번호의 Pod부터 삭제합니다.

```text
StatefulSet 삭제 또는 축소
          │
          ▼
       mysql-2 삭제
          │
          ▼
       mysql-1 삭제
          │
          ▼
       mysql-0 삭제
```

높은 순서의 Pod가 종료된 후 다음 Pod의 삭제가 진행됩니다.

이러한 동작은 데이터 복제나 클러스터 구성에 의존성이 있는 애플리케이션에서 유용할 수 있습니다. 다만 Pod 종료 순서가 데이터베이스의 안전한 멤버 제거 또는 복제 구성 변경까지 자동으로 처리하는 것은 아닙니다.

### 5.3. podManagementPolicy

`podManagementPolicy`는 StatefulSet이 Pod를 생성하고 삭제하는 방식을 지정하는 설정입니다.

| 정책 | 설명 |
|---|---|
| `OrderedReady` | 기본값입니다. Pod를 순서대로 생성하며, 이전 Pod가 Ready 상태가 된 후 다음 Pod를 생성합니다. 삭제는 역순으로 진행합니다. |
| `Parallel` | Pod의 생성과 삭제 과정에서 앞선 Pod의 Ready 또는 종료를 순차적으로 기다리지 않고 병렬로 처리할 수 있습니다. |

다음은 Parallel 정책의 예제입니다.

```yaml
spec:
  podManagementPolicy: Parallel
  replicas: 3
```

Parallel 정책을 사용하면 StatefulSet이 Pod를 생성할 때 앞선 Pod의 Ready 상태를 기다리지 않고 여러 Pod를 생성할 수 있습니다.

다만 Parallel 정책을 사용하더라도 각 Pod의 고유한 이름과 Ordinal은 유지됩니다.

또한 `Parallel`은 기본적으로 StatefulSet의 스케일링 과정에서 Pod를 생성하거나 삭제하는 방식을 변경합니다. RollingUpdate에서도 병렬 업데이트가 가능하지만, 이는 `maxUnavailable` 설정과 함께 동작합니다.

### 5.4. Parallel 정책과 RollingUpdate의 관계

`Parallel` 정책은 RollingUpdate의 업데이트 동작과 완전히 무관한 설정은 아닙니다.

기본적으로 RollingUpdate는 높은 Ordinal의 Pod부터 낮은 Ordinal 방향으로 순차적으로 업데이트합니다. 그러나 `podManagementPolicy: Parallel`과 `rollingUpdate.maxUnavailable`을 함께 사용하면 여러 Pod를 동시에 업데이트할 수 있습니다.

예를 들어 복제본이 5개이고 `maxUnavailable: 2`로 설정하면 업데이트 과정에서 최대 2개의 Pod가 동시에 unavailable 상태가 되는 것을 허용합니다.

```text
StatefulSet
│
├── mysql-0
├── mysql-1
├── mysql-2
├── mysql-3
└── mysql-4

maxUnavailable: 2

Rolling Update

mysql-4, mysql-3 업데이트
          │
          ▼
mysql-4, mysql-3 Ready
          │
          ▼
mysql-2, mysql-1 업데이트
          │
          ▼
mysql-2, mysql-1 Ready
          │
          ▼
mysql-0 업데이트
```

실제 업데이트 대상과 진행 시점은 Pod의 가용 상태 및 Controller의 조정 결과에 따라 달라질 수 있습니다. 여러 Pod가 동시에 교체되면 Ready 상태가 되는 순서도 Ordinal 순서와 달라질 수 있습니다.

따라서 애플리케이션이 엄격한 시작 순서나 종료 순서를 요구한다면 `OrderedReady` 정책을 사용하는 것이 적절합니다. `Parallel`과 `maxUnavailable`을 함께 사용하는 경우에는 여러 인스턴스가 동시에 unavailable 상태가 되어도 애플리케이션이 정상적으로 동작할 수 있는지 확인해야 합니다.

## 6. StatefulSet의 업데이트 전략

StatefulSet은 Pod Template이 변경되었을 때 기존 Pod를 새로운 구성으로 교체할 수 있습니다.

대표적인 업데이트 전략은 다음과 같습니다.

- `RollingUpdate`
- `OnDelete`

### 6.1. RollingUpdate

RollingUpdate는 StatefulSet의 기본 업데이트 전략입니다.

새로운 Pod Template이 적용되면 기존 Pod를 새로운 Template을 사용하도록 교체합니다.

기본적으로 업데이트 대상 중 순서 번호가 가장 높은 Pod부터 교체하며, `OrderedReady` 정책에서는 해당 Pod가 Running 및 Ready 상태가 된 후 다음 Pod를 업데이트합니다.

예를 들어 Nginx 1.27을 Nginx 1.28로 업데이트하는 경우 다음과 같이 동작합니다.

```text
Before Update

mysql-0 (nginx:1.27)
mysql-1 (nginx:1.27)
mysql-2 (nginx:1.27)


Rolling Update

mysql-0 (nginx:1.27)
mysql-1 (nginx:1.27)
mysql-2 (nginx:1.28)

          │
          ▼

mysql-0 (nginx:1.27)
mysql-1 (nginx:1.28)
mysql-2 (nginx:1.28)

          │
          ▼

mysql-0 (nginx:1.28)
mysql-1 (nginx:1.28)
mysql-2 (nginx:1.28)
```

다음과 같이 설정할 수 있습니다.

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
```

#### partition

`partition`은 RollingUpdate 과정에서 업데이트할 Pod의 Ordinal 기준을 지정합니다.

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2
```

Pod가 `mysql-0`, `mysql-1`, `mysql-2`로 구성되어 있다면 다음과 같이 동작합니다.

- `mysql-0`: 기존 버전 유지
- `mysql-1`: 기존 버전 유지
- `mysql-2`: 새로운 버전 적용

즉, `partition` 값 이상의 Ordinal을 가진 Pod만 새로운 Template으로 업데이트됩니다.

이를 활용하면 일부 Pod에만 새로운 버전을 먼저 배포하는 단계적 업데이트를 구성할 수 있습니다.

예를 들어 특정 Pod에서 새로운 애플리케이션 버전을 먼저 검증한 후 나머지 Pod를 업데이트하는 방식으로 활용할 수 있습니다.

`partition`은 업데이트를 자동으로 완료하는 기능이 아니라 업데이트 대상의 Ordinal 범위를 제한하는 설정입니다. 검증이 끝난 후 partition 값을 낮추어 나머지 Pod의 업데이트를 진행할 수 있습니다.

#### maxUnavailable

`maxUnavailable`은 RollingUpdate 과정에서 동시에 unavailable 상태가 되는 Pod의 최대 개수를 지정하는 설정입니다.

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2
```

값은 정수 또는 백분율로 지정할 수 있습니다.

```yaml
maxUnavailable: 2
```

또는 다음과 같이 설정할 수 있습니다.

```yaml
maxUnavailable: 40%
```

백분율을 사용하는 경우 계산된 값은 올림 처리됩니다. 기본값은 1이며, 0으로 설정할 수 없습니다.

`maxUnavailable`은 업데이트 과정에서 허용할 수 있는 Pod의 가용성 저하 범위를 제어합니다. 다만 이 설정만으로 모든 StatefulSet에서 병렬 업데이트가 수행되는 것은 아닙니다.

- `OrderedReady`에서는 기본적으로 Pod를 하나씩 업데이트하며, `maxUnavailable`을 1보다 크게 설정하더라도 순차적인 Ready 대기 동작이 유지됩니다.
- `Parallel`에서는 `maxUnavailable`을 1보다 크게 설정하면 여러 Pod를 동시에 교체할 수 있습니다.
- 여러 Pod가 동시에 업데이트되면 Pod의 Ready 순서가 Ordinal 순서와 달라질 수 있습니다.

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
spec:
  podManagementPolicy: Parallel
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2
```

이 설정은 최대 2개의 Pod가 동시에 unavailable 상태가 되는 것을 허용하여 업데이트를 병렬로 진행할 수 있도록 합니다.

다만 `maxUnavailable`은 가용성 저하를 허용하는 범위이지, 애플리케이션의 데이터 정합성이나 클러스터 쿼럼을 보장하는 설정은 아닙니다. 데이터베이스나 분산 시스템에서는 동시에 교체할 수 있는 노드 수와 복제본 수, 쿼럼 조건 등을 함께 고려해야 합니다.

#### minReadySeconds

`minReadySeconds`는 새롭게 생성된 Pod가 Ready 상태가 된 후 Available 상태로 판단되기 위해 Ready 상태를 유지해야 하는 최소 시간을 지정합니다.

```yaml
spec:
  minReadySeconds: 10
```

예를 들어 `minReadySeconds: 10`으로 설정하면 Pod가 Ready 상태가 된 이후 10초 동안 해당 상태를 유지해야 Available로 판단됩니다.

이를 통해 Pod가 일시적으로 Ready 상태가 되었다가 곧바로 비정상 상태로 전환되는 상황에서 업데이트가 너무 빠르게 진행되는 것을 방지할 수 있습니다.

`minReadySeconds`는 업데이트 과정에서 Pod의 안정성을 확인하는 데 활용할 수 있습니다.

### 6.2. OnDelete

OnDelete는 StatefulSet의 Pod Template이 변경되더라도 기존 Pod를 자동으로 교체하지 않는 업데이트 전략입니다.

기존 Pod를 직접 삭제해야 새로운 Template을 적용한 Pod가 생성됩니다.

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

예를 들어 컨테이너 이미지를 변경하더라도 기존 Pod는 이전 이미지를 계속 사용합니다.

```text
Before

mysql-0 (nginx:1.27)
mysql-1 (nginx:1.27)
mysql-2 (nginx:1.27)


Template 변경

nginx:1.27 → nginx:1.28


기존 Pod 유지

mysql-0 (nginx:1.27)
mysql-1 (nginx:1.27)
mysql-2 (nginx:1.27)


mysql-2 수동 삭제


새로운 Pod 생성

mysql-0 (nginx:1.27)
mysql-1 (nginx:1.27)
mysql-2 (nginx:1.28)
```

OnDelete는 업데이트 시점을 운영자가 직접 제어해야 하는 경우에 사용할 수 있습니다.

| 구분 | RollingUpdate | OnDelete |
|---|---|---|
| 업데이트 방식 | Pod를 자동으로 교체합니다. | Pod를 직접 삭제해야 교체됩니다. |
| 업데이트 순서 | 기본적으로 높은 Ordinal부터 진행합니다. 병렬 업데이트 설정을 사용할 수도 있습니다. | 운영자가 삭제하는 순서에 따라 진행됩니다. |
| 자동 업데이트 | 지원합니다. | 지원하지 않습니다. |
| 주요 사용 사례 | 일반적인 StatefulSet 업데이트 | 업데이트 시점을 수동으로 제어해야 하는 경우 |

## 7. StatefulSet의 스케일링 (Scaling)

StatefulSet은 `replicas` 값을 변경하여 Pod 복제본 개수를 조절할 수 있습니다.

### 7.1. 수동 스케일링

`kubectl scale` 명령어를 사용하여 Pod 개수를 변경할 수 있습니다.

```bash
kubectl scale statefulset mysql --replicas=5
```

또는 YAML Manifest에서 `replicas` 값을 변경할 수 있습니다.

```yaml
spec:
  replicas: 5
```

기존에 3개의 Pod가 존재했다면 다음과 같이 확장됩니다.

```text
Before

mysql-0
mysql-1
mysql-2


Scale Up

mysql-0
mysql-1
mysql-2
mysql-3
mysql-4
```

새롭게 생성되는 Pod는 기존 Pod와 동일한 규칙에 따라 이름이 지정됩니다. `volumeClaimTemplates`가 설정되어 있다면 각 Pod에 대응하는 새로운 PVC도 생성됩니다.

기본 `OrderedReady` 정책에서는 기존 Pod가 Ready 상태가 된 후 다음 Pod가 생성됩니다. `Parallel` 정책에서는 이러한 순차적인 Ready 대기를 적용하지 않습니다.

### 7.2. 스케일 다운

복제본 개수를 5개에서 3개로 줄이면 기본 `OrderedReady` 정책에서는 높은 Ordinal의 Pod부터 삭제됩니다.

```text
Before

mysql-0
mysql-1
mysql-2
mysql-3
mysql-4


Scale Down

mysql-0
mysql-1
mysql-2
```

Pod가 삭제되더라도 해당 Pod에 연결된 PVC는 기본적으로 유지됩니다. 따라서 나중에 복제본 개수를 다시 늘려 동일한 Ordinal의 Pod가 생성되면 기존 PVC를 다시 사용할 수 있습니다.

단, 스케일 다운으로 Pod가 종료되었다고 해서 데이터베이스 클러스터 내부의 복제 구성이나 데이터 복제 관계까지 자동으로 정리되는 것은 아닙니다. 애플리케이션 자체의 클러스터 관리 정책도 함께 고려해야 합니다.

### 7.3. HorizontalPodAutoscaler 연동

StatefulSet도 HorizontalPodAutoscaler(HPA)와 연동하여 Pod 개수를 자동으로 조절할 수 있습니다.

HPA는 CPU 사용률, 메모리 사용률 또는 사용자 정의 메트릭 등을 기준으로 StatefulSet의 복제본 개수를 조정합니다.

다만 데이터베이스나 분산 시스템은 단순히 Pod 개수를 늘리는 것만으로 정상적인 확장이 완료되지 않을 수 있습니다. 새로운 인스턴스가 기존 클러스터에 참여하고 데이터를 동기화하는 과정이 필요할 수 있기 때문입니다.

따라서 StatefulSet에 HPA를 적용할 때는 애플리케이션의 클러스터 구성과 확장 정책을 함께 고려해야 합니다. 특히 복제본 수의 증가와 감소가 데이터 복제, 멤버 제거 및 쿼럼 구성에 미치는 영향을 검토해야 합니다.

## 8. StatefulSet과 Headless Service의 관계

StatefulSet은 각 Pod에 안정적인 네트워크 식별자를 제공하기 위해 일반적으로 Headless Service와 함께 사용됩니다.

Headless Service는 `clusterIP: None`으로 설정하는 Service입니다.

일반적인 ClusterIP Service는 하나의 가상 IP 주소를 제공하고 해당 Service에 연결된 Pod로 트래픽을 전달합니다.

반면 Headless Service는 가상 IP 주소를 생성하지 않고, 연결 대상 Pod의 개별 IP 주소를 DNS 레코드로 제공합니다.

### 8.1. Headless Service YAML

다음은 StatefulSet과 함께 사용할 Headless Service 예제입니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None

  selector:
    app: mysql

  ports:
    - name: mysql
      port: 3306
      targetPort: 3306
```

여기서 중요한 설정은 다음과 같습니다.

```yaml
spec:
  clusterIP: None
```

이 설정을 통해 일반적인 ClusterIP를 할당하지 않는 Headless Service를 생성합니다.

### 8.2. StatefulSet과 연결

StatefulSet에서는 `serviceName`을 통해 Headless Service를 지정합니다.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless
  replicas: 3

  selector:
    matchLabels:
      app: mysql
```

Service의 이름과 StatefulSet의 `serviceName`이 일치해야 합니다. 또한 Service의 Selector가 StatefulSet이 생성하는 Pod의 레이블과 일치해야 합니다.

두 리소스가 연결되면 각 Pod에 대해 다음과 같은 DNS 이름을 사용할 수 있습니다.

```text
mysql-0.mysql-headless.default.svc.cluster.local
mysql-1.mysql-headless.default.svc.cluster.local
mysql-2.mysql-headless.default.svc.cluster.local
```

### 8.3. 일반 Service와 Headless Service 비교

| 구분 | 일반 ClusterIP Service | Headless Service |
|---|---|---|
| ClusterIP | 가상 IP 주소를 할당합니다. | 가상 IP 주소를 할당하지 않습니다. |
| DNS 응답 | 일반적으로 Service의 ClusterIP를 반환합니다. | 조건에 따라 대상 Pod의 개별 IP 주소를 반환합니다. |
| 트래픽 전달 | Service의 가상 IP를 통해 Pod로 전달합니다. | 클라이언트가 개별 Pod 주소를 조회하여 직접 접근할 수 있습니다. |
| 주요 사용 사례 | 일반적인 애플리케이션 접근 | StatefulSet의 개별 Pod 식별 및 접근 |

Headless Service는 StatefulSet의 Pod를 개별적으로 식별하고 접근해야 하는 경우에 유용합니다.

특히 데이터베이스 복제 구성이나 분산 시스템에서 특정 노드를 직접 참조해야 할 때 활용할 수 있습니다.

## 9. StatefulSet과 Deployment의 차이

StatefulSet과 Deployment는 모두 Pod를 생성하고 관리하는 Workload 리소스입니다.

그러나 관리 대상 애플리케이션의 특성과 Pod의 식별 및 스토리지 관리 방식에 차이가 있습니다.

| 구분 | Deployment | StatefulSet |
|---|---|---|
| 주요 목적 | 일반적으로 Stateless 애플리케이션 배포 | 안정적인 식별자와 스토리지가 필요한 애플리케이션 배포 |
| Pod 이름 | 무작위 문자열 기반 이름 | Ordinal 기반의 고정된 이름 |
| Pod 식별자 | 재생성 시 변경될 수 있습니다. | 동일한 Ordinal의 이름을 유지합니다. |
| 네트워크 식별 | 일반적으로 Service를 통한 접근 | Headless Service를 통한 개별 Pod DNS 제공 |
| 스토리지 | Pod 간 공유 또는 별도 구성 가능 | PVC 템플릿을 통해 Pod별 독립 스토리지 구성 가능 |
| 생성 순서 | 특정 순서를 보장하지 않습니다. | 기본적으로 순차적 생성 |
| 삭제 순서 | 특정 순서를 보장하지 않습니다. | 기본적으로 역순 삭제 |
| 업데이트 | RollingUpdate, Recreate | RollingUpdate, OnDelete |
| 주요 사용 사례 | 웹 애플리케이션, API 서버 | 데이터베이스, 분산 저장 시스템 |

### 9.1. Deployment의 Pod 관리

Deployment는 동일한 구성을 가진 Pod를 여러 개 실행하는 데 적합합니다.

각 Pod는 독립적인 인스턴스로 취급되며, 특정 Pod가 삭제되면 새로운 Pod가 생성됩니다.

```text
Deployment
│
└── ReplicaSet
    │
    ├── nginx-7d8f9c-abcde
    ├── nginx-7d8f9c-fghij
    └── nginx-7d8f9c-klmno
```

각 Pod는 고유한 이름을 가지지만, 특정 Pod의 이름이나 식별자가 유지된다는 보장은 없습니다.

### 9.2. StatefulSet의 Pod 관리

StatefulSet은 각 Pod에 안정적인 식별자를 부여합니다.

```text
StatefulSet
│
├── mysql-0
├── mysql-1
└── mysql-2
```

Pod가 삭제되고 다시 생성되더라도 동일한 Ordinal을 가진 이름이 사용됩니다.

또한 각 Pod에 독립적인 PVC를 연결하여 데이터를 유지할 수 있습니다.

따라서 각 인스턴스가 고유한 식별자와 데이터를 유지해야 하는 애플리케이션에는 StatefulSet이 적합합니다.

## 10. StatefulSet과 다른 Workload 리소스의 관계

Kubernetes의 Workload 리소스는 애플리케이션의 실행 목적에 따라 구분됩니다.

| 리소스 | 주요 역할 | 주요 사용 사례 |
|---|---|---|
| Pod | 컨테이너 실행 단위 | 단일 애플리케이션 실행 |
| Deployment | 일반적으로 Stateless 애플리케이션 배포 및 관리 | 웹 애플리케이션, API 서버 |
| ReplicaSet | 지정된 Pod 복제본 개수 유지 | Pod 복제본 관리 |
| StatefulSet | 안정적인 식별자와 스토리지를 가진 Pod 관리 | 데이터베이스, 분산 시스템 |
| DaemonSet | 각 노드 또는 지정된 노드에 Pod 배치 | 로그 수집, 모니터링 에이전트 |
| Job | 작업이 완료될 때까지 Pod 실행 | 배치 작업 |
| CronJob | 일정에 따라 Job 실행 | 정기적인 배치 작업 |

Deployment와 StatefulSet은 모두 여러 Pod를 관리하지만, Pod의 식별자와 데이터 유지 방식에 차이가 있습니다.

ReplicaSet은 Deployment의 하위 리소스로 Pod 복제본을 관리하지만, StatefulSet은 자체 Controller를 통해 Pod를 직접 관리합니다.

## 11. StatefulSet 사용 시 고려사항

### 11.1. StatefulSet은 데이터베이스 자체를 관리하지 않습니다.

StatefulSet은 Pod의 식별자와 스토리지를 관리하는 Kubernetes 리소스입니다.

데이터베이스의 복제, 리더 선출, 데이터 동기화, 장애 조치(Failover) 등의 기능을 자동으로 제공하지는 않습니다.

예를 들어 StatefulSet으로 MySQL Pod 3개를 생성했다고 해서 자동으로 MySQL 복제 클러스터가 구성되는 것은 아닙니다.

데이터베이스 자체의 복제 설정이나 별도의 Operator 등을 함께 사용해야 합니다.

### 11.2. PVC와 PV의 생명주기를 고려해야 합니다.

StatefulSet의 Pod가 삭제되더라도 연결된 PVC는 기본적으로 유지됩니다.

하지만 PVC가 유지된다고 해서 실제 데이터까지 반드시 보존되는 것은 아닙니다.

PVC가 삭제되면 PV의 Reclaim Policy가 `Delete`인 경우 실제 스토리지 자원도 삭제될 수 있습니다. 반대로 `Retain`인 경우 PV와 실제 스토리지는 관리자가 별도로 정리해야 할 수 있습니다.

따라서 StatefulSet을 운영할 때는 PVC, PV, StorageClass 및 Reclaim Policy를 함께 고려해야 합니다.

### 11.3. Headless Service를 함께 구성해야 합니다.

StatefulSet에서 Pod별 안정적인 네트워크 식별자를 제공하려면 일반적으로 Headless Service를 함께 사용합니다.

StatefulSet의 `serviceName`에 지정된 Service가 존재해야 하며, Service의 Selector도 StatefulSet의 Pod 레이블과 일치해야 합니다.

Headless Service는 StatefulSet과 별도로 생성하고 관리하는 리소스입니다. StatefulSet이 해당 Service를 자동으로 생성하지는 않습니다.

### 11.4. Pod 생성 순서로 인한 교착 상태에 주의해야 합니다.

기본 `OrderedReady` 정책에서는 이전 Pod가 Ready 상태가 되어야 다음 Pod가 생성됩니다.

따라서 애플리케이션이 모든 Pod가 생성되어야 Ready 상태가 되는 구조라면 문제가 발생할 수 있습니다.

예를 들어 첫 번째 Pod가 다른 Pod의 실행을 기다리는 구조라면 다음 Pod가 생성되지 않아 전체 StatefulSet이 정상적으로 시작되지 못할 수 있습니다.

이러한 경우 `podManagementPolicy: Parallel`을 고려할 수 있습니다. 다만 병렬 생성이 애플리케이션의 초기화 순서나 클러스터 구성 요구사항과 충돌하지 않는지 확인해야 합니다.

### 11.5. StatefulSet 삭제와 PVC 삭제는 별도로 고려해야 합니다.

StatefulSet을 삭제하더라도 연결된 PVC는 기본적으로 유지됩니다.

다만 StatefulSet에는 `persistentVolumeClaimRetentionPolicy` 설정이 있으며, 이를 통해 StatefulSet 삭제 또는 스케일 다운 시 PVC의 보존 및 삭제 동작을 제어할 수 있습니다.

주요 설정은 다음과 같습니다.

| 설정 | 설명 |
|---|---|
| `whenDeleted` | StatefulSet이 삭제될 때 해당 StatefulSet의 PVC를 어떻게 처리할지 지정합니다. |
| `whenScaled` | StatefulSet의 복제본 수를 줄일 때 제거 대상 Pod의 PVC를 어떻게 처리할지 지정합니다. |
| `Retain` | 해당 이벤트가 발생해도 PVC를 유지합니다. |
| `Delete` | 해당 이벤트에 따라 관련 PVC를 삭제하도록 설정합니다. |

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain
    whenScaled: Delete
```

이 설정은 StatefulSet 자체가 삭제될 때 PVC를 유지하고, 스케일 다운으로 제거되는 Pod의 PVC는 삭제하도록 구성합니다.

PVC가 삭제되는 경우에도 실제 스토리지의 삭제 여부는 PV의 Reclaim Policy 등에 따라 달라집니다. 따라서 StatefulSet, PVC, PV 및 실제 스토리지의 삭제 정책을 함께 확인해야 합니다.

### 11.6. StatefulSet의 업데이트는 신중하게 진행해야 합니다.

데이터베이스나 분산 시스템은 애플리케이션 버전이 변경될 때 데이터 구조나 통신 프로토콜이 달라질 수 있습니다.

RollingUpdate를 사용하면 기본적으로 높은 Ordinal의 Pod부터 순차적으로 교체되므로 애플리케이션의 버전 호환성을 고려해야 합니다.

필요하다면 `partition`을 사용하여 일부 Pod에 먼저 업데이트를 적용하고 검증하는 방식으로 운영할 수 있습니다.

`Parallel` 정책과 `maxUnavailable`을 함께 사용하면 여러 Pod가 동시에 교체될 수 있으므로, 데이터베이스의 쿼럼 및 복제 상태를 고려하여 적절한 unavailable 허용 범위를 설정해야 합니다.

### 11.7. 스토리지 접근 모드와 노드 배치 조건을 고려해야 합니다.

StatefulSet의 PVC는 각 Pod에 연결되지만, 실제로 해당 PVC를 사용할 수 있는 노드는 스토리지 유형과 접근 모드에 따라 달라집니다.

예를 들어 `ReadWriteOnce`는 일반적으로 하나의 노드에서 해당 볼륨을 읽기 및 쓰기 모드로 마운트할 수 있다는 의미입니다. 반드시 하나의 Pod만 접근할 수 있다는 의미는 아닙니다.

또한 특정 노드에 종속된 로컬 스토리지나 특정 영역(Zone)에 생성된 PV를 사용하는 경우, Pod가 해당 스토리지에 접근할 수 있는 노드에 배치되어야 합니다.

따라서 StatefulSet을 운영할 때는 StorageClass의 동적 프로비저닝 정책, PV의 노드 및 영역 제약, Pod의 스케줄링 조건 등을 함께 고려해야 합니다.

## 12. StatefulSet 관련 kubectl 명령어

| 명령어 | 설명 |
|---|---|
| `kubectl get statefulsets` | StatefulSet 목록을 조회합니다. |
| `kubectl get sts` | StatefulSet 목록을 축약된 리소스 이름으로 조회합니다. |
| `kubectl describe statefulset mysql` | StatefulSet의 상세 정보와 이벤트를 확인합니다. |
| `kubectl get pods -l app=mysql` | 지정한 레이블을 가진 Pod를 조회합니다. |
| `kubectl get pvc` | PVC 목록을 조회합니다. |
| `kubectl get pv` | PV 목록을 조회합니다. |
| `kubectl scale statefulset mysql --replicas=5` | StatefulSet의 복제본 개수를 변경합니다. |
| `kubectl rollout status statefulset/mysql` | StatefulSet의 롤아웃 진행 상태를 확인합니다. |
| `kubectl rollout history statefulset/mysql` | StatefulSet의 업데이트 이력을 확인합니다. |
| `kubectl rollout undo statefulset/mysql` | 이전 revision으로 롤백을 요청합니다. |
| `kubectl delete statefulset mysql` | StatefulSet을 삭제합니다. |

`kubectl rollout undo`를 사용하려면 이전 revision 정보가 남아 있어야 합니다. StatefulSet의 `revisionHistoryLimit` 설정에 따라 보관되는 ControllerRevision의 수가 제한될 수 있습니다.

StatefulSet을 삭제할 때는 Pod와 PVC의 보존 정책을 확인해야 합니다. 특히 PVC 및 실제 스토리지 데이터가 유지되는지 별도로 확인하는 것이 중요합니다.

## 13. 핵심 정리

StatefulSet은 Kubernetes에서 안정적인 식별자와 스토리지를 유지해야 하는 애플리케이션을 배포하고 관리하기 위한 Workload 리소스입니다.

- 각 Pod에 Ordinal 기반의 고유한 이름을 부여합니다.
- Pod가 재생성되더라도 동일한 Ordinal과 이름을 유지합니다.
- Headless Service와 연동하여 안정적인 Pod별 DNS 이름을 제공합니다.
- `volumeClaimTemplates`를 통해 각 Pod에 독립적인 PVC를 생성할 수 있습니다.
- Pod가 재생성되더라도 기존 PVC를 다시 연결하여 데이터를 유지할 수 있습니다.
- 기본적으로 Pod를 순차적으로 생성하고 역순으로 삭제합니다.
- `podManagementPolicy`를 통해 Pod 생성 및 삭제 방식을 제어할 수 있습니다.
- RollingUpdate와 OnDelete 전략을 통해 Pod 업데이트 방식을 제어할 수 있습니다.
- `partition`을 사용하여 특정 Ordinal 이상의 Pod에만 업데이트를 적용할 수 있습니다.
- `maxUnavailable`과 `Parallel` 정책을 함께 활용하면 여러 Pod를 동시에 업데이트할 수 있습니다.
- `persistentVolumeClaimRetentionPolicy`를 통해 StatefulSet 삭제 및 스케일 다운 시 PVC 보존 정책을 지정할 수 있습니다.
- 데이터베이스 복제나 클러스터 구성 자체를 자동으로 제공하지는 않습니다.

**StatefulSet은 Pod의 안정적인 식별자, 네트워크 접근 정보, 영구 스토리지를 유지해야 하는 애플리케이션을 관리하기 위한 Workload 리소스입니다.**

Deployment가 일반적으로 동일한 애플리케이션 인스턴스를 독립적으로 실행하는 데 적합하다면, StatefulSet은 각 인스턴스가 고유한 정체성과 데이터를 유지해야 하는 애플리케이션에 적합합니다.

## 참고자료

- [StatefulSets - Kubernetes 공식 문서](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [StatefulSet API Reference](https://kubernetes.io/docs/reference/kubernetes-api/apps/stateful-set-v1/)
- [Headless Services - Kubernetes 공식 문서](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)
- [DNS for Services and Pods - Kubernetes 공식 문서](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Persistent Volumes - Kubernetes 공식 문서](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Storage Classes - Kubernetes 공식 문서](https://kubernetes.io/docs/concepts/storage/storage-classes/)