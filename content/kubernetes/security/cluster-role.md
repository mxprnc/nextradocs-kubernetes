# ClusterRole

## 1. ClusterRole이란?

ClusterRole은 Kubernetes RBAC(Role-Based Access Control)에서 **클러스터 범위의 권한 규칙을 정의하는 리소스**입니다.

Role과 마찬가지로 Kubernetes API 리소스에 대한 접근 권한을 정의하지만, Namespace에 종속되지 않는다는 차이가 있습니다.

Role은 특정 Namespace에 속하는 리소스이며 해당 Namespace 내에서 사용할 권한을 정의합니다. 반면 ClusterRole은 클러스터 전체에서 재사용할 수 있는 권한 집합을 정의합니다.

ClusterRole은 다음과 같은 권한을 정의할 수 있습니다.

- **Namespace 범위 리소스:** Pod, Deployment, Service 등
- **클러스터 범위 리소스:** Node, Namespace, PersistentVolume 등
- **Non-resource URL:** `/healthz`, `/metrics` 등 Kubernetes API의 일반 리소스가 아닌 엔드포인트

ClusterRole은 권한을 정의하는 역할을 수행하며, 실제 사용자나 애플리케이션에 권한을 부여하려면 RoleBinding 또는 ClusterRoleBinding과 연결해야 합니다.

ClusterRole의 주요 특징은 다음과 같습니다.

- Namespace에 속하지 않는 클러스터 범위 리소스입니다.
- `rules` 필드를 통해 Kubernetes API 리소스와 작업에 대한 권한을 정의합니다.
- RoleBinding 또는 ClusterRoleBinding에서 참조할 수 있습니다.
- 동일한 권한 규칙을 여러 Namespace에서 재사용할 수 있습니다.
- AggregationRule을 사용하여 여러 ClusterRole의 권한을 하나로 결합할 수 있습니다.

## 2. ClusterRole의 동작 원리

ClusterRole은 Kubernetes API 리소스에 대한 접근 권한을 정의하지만, 특정 Subject에 권한을 직접 부여하지는 않습니다.

실제 권한 부여는 RoleBinding 또는 ClusterRoleBinding을 통해 이루어집니다.

ClusterRole을 참조하는 Binding의 종류에 따라 권한 적용 범위가 달라집니다.

```text
Kubernetes Cluster
│
├── ClusterRole
│   └── pod-reader
│       └── pods: get, list, watch
│
├── Namespace: backend
│   └── RoleBinding
│       ├── Subject: backend-sa
│       └── RoleRef: pod-reader
│
├── Namespace: frontend
│   └── RoleBinding
│       ├── Subject: frontend-sa
│       └── RoleRef: pod-reader
│
└── ClusterRoleBinding
    ├── Subject: monitoring-user
    └── RoleRef: pod-reader
```

위 구조에서 각 Binding의 권한 적용 범위는 다음과 같습니다.

| Binding | Subject | 권한 적용 범위 |
|---|---|---|
| `backend` Namespace의 RoleBinding | `backend-sa` | `backend` Namespace |
| `frontend` Namespace의 RoleBinding | `frontend-sa` | `frontend` Namespace |
| ClusterRoleBinding | `monitoring-user` | 클러스터 전체 |

동일한 ClusterRole을 참조하더라도 RoleBinding을 사용하는지 ClusterRoleBinding을 사용하는지에 따라 권한 적용 범위가 달라집니다.

따라서 ClusterRole 자체가 권한 적용 범위를 결정하는 것이 아니라, ClusterRole을 참조하는 Binding이 권한 적용 범위를 결정합니다.

## 3. ClusterRole 생성 및 설정

ClusterRole은 `apiVersion: rbac.authorization.k8s.io/v1`, `kind: ClusterRole`을 사용하여 정의합니다.

다음은 Pod 조회 권한을 정의하는 ClusterRole 예제입니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
  - apiGroups: [""]
    resources:
      - pods
    verbs:
      - get
      - list
      - watch
```

위 ClusterRole은 Core API Group에 속하는 Pod에 대해 조회 권한을 정의합니다.

ClusterRole은 Namespace에 속하지 않는 리소스이므로 `metadata.namespace`를 지정하지 않습니다.

생성된 ClusterRole은 다음 명령어로 조회할 수 있습니다.

```bash
kubectl apply -f cluster-role.yaml
```

```bash
kubectl get clusterroles
```

특정 ClusterRole의 상세 정보는 다음과 같이 확인합니다.

```bash
kubectl describe clusterrole pod-reader
```

### 3.1. 주요 필드

| 필드 | 설명 |
|---|---|
| `apiVersion` | API 버전을 지정합니다. `rbac.authorization.k8s.io/v1`을 사용합니다. |
| `kind` | 리소스 종류를 지정합니다. `ClusterRole`을 사용합니다. |
| `metadata.name` | ClusterRole의 이름을 지정합니다. |
| `metadata.labels` | ClusterRole을 식별하거나 분류하기 위한 Label을 지정합니다. |
| `metadata.annotations` | ClusterRole에 추가적인 메타데이터를 지정합니다. |
| `rules` | 접근을 허용할 API Group, Resource, Verb 등의 권한 규칙을 정의합니다. |
| `aggregationRule` | 다른 ClusterRole을 선택하여 권한 규칙을 자동으로 결합하는 조건을 정의합니다. |

`aggregationRule`은 선택 사항이며, 여러 ClusterRole의 권한을 결합하는 Aggregated ClusterRole에서 사용합니다.

## 4. ClusterRole의 권한 규칙(Rules)

ClusterRole의 `rules` 필드는 Role과 동일한 PolicyRule 구조를 사용합니다.

각 Rule은 특정 API Group의 리소스에 대해 허용할 작업을 정의합니다.

주요 필드는 다음과 같습니다.

- `apiGroups`: 접근할 Kubernetes API Group을 지정합니다.
- `resources`: 접근할 Kubernetes API 리소스를 지정합니다.
- `verbs`: 허용할 작업을 지정합니다.
- `resourceNames`: 접근을 허용할 특정 리소스 이름을 제한합니다.
- `nonResourceURLs`: 접근을 허용할 Non-resource URL을 지정합니다.

### 4.1. apiGroups

`apiGroups`는 권한을 적용할 Kubernetes API Group을 지정합니다.

Kubernetes 리소스는 API Group에 따라 구분되며, 각 리소스가 속한 API Group을 정확하게 지정해야 합니다.

예를 들어 다음과 같은 API Group이 있습니다.

| API Group | 주요 리소스 |
|---|---|
| `""` | Pod, Service, ConfigMap, Secret, Node |
| `apps` | Deployment, StatefulSet, DaemonSet, ReplicaSet |
| `batch` | Job, CronJob |
| `networking.k8s.io` | Ingress, NetworkPolicy |
| `rbac.authorization.k8s.io` | Role, RoleBinding, ClusterRole, ClusterRoleBinding |
| `storage.k8s.io` | StorageClass, VolumeAttachment |

Core API Group에 속한 리소스를 지정할 때는 빈 문자열 `""`을 사용합니다.

```yaml
rules:
  - apiGroups: [""]
    resources:
      - pods
      - services
    verbs:
      - get
      - list
```

여러 API Group의 리소스에 동일한 권한을 부여할 수도 있습니다.

```yaml
rules:
  - apiGroups:
      - ""
      - apps
    resources:
      - pods
      - deployments
    verbs:
      - get
      - list
      - watch
```

단, 위 예제는 서로 다른 API Group에 속한 리소스를 함께 지정한 것이므로 실제 API Group과 Resource의 조합이 올바른지 확인해야 합니다. 일반적으로는 리소스의 API Group에 따라 Rule을 분리하는 것이 명확합니다.

### 4.2. resources

`resources`는 권한을 적용할 Kubernetes API 리소스를 지정합니다.

다음은 Pod와 Service에 조회 권한을 부여하는 예제입니다.

```yaml
rules:
  - apiGroups: [""]
    resources:
      - pods
      - services
    verbs:
      - get
      - list
      - watch
```

리소스 이름은 Kubernetes API에서 사용하는 복수형 이름을 지정합니다.

| Resource | 설명 |
|---|---|
| `pods` | Pod |
| `services` | Service |
| `configmaps` | ConfigMap |
| `secrets` | Secret |
| `deployments` | Deployment |
| `statefulsets` | StatefulSet |
| `daemonsets` | DaemonSet |
| `jobs` | Job |
| `cronjobs` | CronJob |
| `nodes` | Node |
| `namespaces` | Namespace |
| `persistentvolumes` | PersistentVolume |

모든 리소스를 대상으로 지정하려면 `resources: ["*"]`를 사용할 수 있습니다.

```yaml
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs:
      - get
      - list
      - watch
```

와일드카드는 향후 Kubernetes에 추가되는 리소스까지 권한 범위에 포함될 수 있으므로 필요한 리소스를 명시하는 것이 좋습니다.

### 4.3. verbs

`verbs`는 지정한 리소스에 대해 허용할 작업을 정의합니다.

| Verb | 설명 |
|---|---|
| `get` | 특정 리소스를 조회합니다. |
| `list` | 리소스 목록을 조회합니다. |
| `watch` | 리소스의 변경 사항을 감시합니다. |
| `create` | 새로운 리소스를 생성합니다. |
| `update` | 기존 리소스를 전체적으로 수정합니다. |
| `patch` | 기존 리소스의 일부를 수정합니다. |
| `delete` | 특정 리소스를 삭제합니다. |
| `deletecollection` | 여러 리소스를 한 번에 삭제합니다. |

예를 들어 Deployment를 조회하고 수정할 수 있는 권한은 다음과 같이 정의합니다.

```yaml
rules:
  - apiGroups:
      - apps
    resources:
      - deployments
    verbs:
      - get
      - list
      - watch
      - update
      - patch
```

모든 작업을 허용하려면 `verbs: ["*"]`를 사용할 수 있지만, 필요한 작업만 명시하는 것이 권장됩니다.

### 4.4. resourceNames

`resourceNames`는 특정 이름을 가진 리소스에 대해서만 권한을 허용할 때 사용합니다.

예를 들어 `backend-config`라는 ConfigMap만 조회할 수 있도록 제한하려면 다음과 같이 정의합니다.

```yaml
rules:
  - apiGroups: [""]
    resources:
      - configmaps
    resourceNames:
      - backend-config
    verbs:
      - get
      - update
```

위 권한은 지정된 이름의 ConfigMap에 대해서만 `get`, `update` 작업을 허용합니다.

`resourceNames`는 리소스 이름을 기준으로 접근 범위를 제한하는 기능이므로 모든 상황에서 사용할 수 있는 것은 아닙니다.

특히 다음 사항에 주의해야 합니다.

- `create` 작업은 생성되는 리소스의 이름을 대상으로 제한할 수 없으므로 `resourceNames`와 함께 사용해도 생성 대상 제한에 적용되지 않습니다.
- `deletecollection` 작업은 여러 리소스를 대상으로 하므로 `resourceNames`로 제한할 수 없습니다.
- `list`와 `watch`에 `resourceNames`를 적용하려면 요청에 일치하는 `metadata.name` Field Selector가 포함되어야 합니다.

예를 들어 특정 Pod만 조회하도록 권한을 제한했다면 다음과 같이 요청해야 합니다.

```bash
kubectl get pods \
  --field-selector=metadata.name=backend-pod
```

### 4.5. Subresource 권한

Kubernetes API는 일부 리소스에 대해 별도의 하위 리소스(Subresource)를 제공합니다.

Subresource는 기존 리소스와 구분되는 API 작업을 수행할 때 사용합니다.

예를 들어 다음과 같은 Subresource가 있습니다.

| Resource | Subresource | 설명 |
|---|---|---|
| `pods` | `pods/log` | Pod 로그 조회 |
| `pods` | `pods/exec` | Pod 컨테이너에서 명령 실행 |
| `pods` | `pods/status` | Pod 상태 조회 및 수정 |
| `deployments` | `deployments/scale` | Deployment의 Scale 관리 |
| `deployments` | `deployments/status` | Deployment 상태 조회 및 수정 |

Subresource에 대한 권한은 `resources`에 슬래시(`/`)를 사용하여 지정합니다.

다음은 Pod 로그를 조회할 수 있는 권한을 정의하는 예제입니다.

```yaml
rules:
  - apiGroups: [""]
    resources:
      - pods
      - pods/log
    verbs:
      - get
```

위 설정은 Pod 자체의 조회와 Pod 로그 조회를 허용합니다.

반면 Pod에 명령을 실행할 수 있는 권한을 부여하려면 다음과 같이 별도로 정의해야 합니다.

```yaml
rules:
  - apiGroups: [""]
    resources:
      - pods/exec
    verbs:
      - create
```

Pod 조회 권한과 Pod 내부에서 명령을 실행하는 권한은 서로 다른 작업이므로 필요한 권한을 구분하여 설정해야 합니다.

### 4.6. Non-resource URL 권한

ClusterRole은 Kubernetes API 리소스뿐만 아니라 Non-resource URL에 대한 접근 권한도 정의할 수 있습니다.

Non-resource URL은 Kubernetes API의 일반 리소스 경로에 해당하지 않는 엔드포인트입니다.

대표적인 예시는 다음과 같습니다.

| URL | 설명 |
|---|---|
| `/healthz` | API Server의 상태 확인 |
| `/livez` | API Server의 Liveness 확인 |
| `/readyz` | API Server의 Readiness 확인 |
| `/metrics` | 모니터링 지표 조회 |

Non-resource URL에 대한 권한은 `nonResourceURLs`와 `verbs`를 사용하여 정의합니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: health-check-reader
rules:
  - nonResourceURLs:
      - /healthz
      - /readyz
    verbs:
      - get
```

위 ClusterRole은 `/healthz`와 `/readyz` 엔드포인트에 대한 GET 요청을 허용합니다.

Non-resource URL 권한은 클러스터 범위에서 정의해야 하며, Namespace 범위의 Role에서는 정의할 수 없습니다.

또한 Non-resource URL에는 `get`, `post` 등의 HTTP Verb를 사용합니다. `list`, `watch`와 같은 Kubernetes 리소스 작업은 사용할 수 없습니다.

## 5. ClusterRole 예제

### 5.1. Pod 조회 전용 ClusterRole

다음은 클러스터의 모든 Namespace에 존재하는 Pod를 조회할 수 있도록 권한을 정의한 ClusterRole입니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
  - apiGroups: [""]
    resources:
      - pods
    verbs:
      - get
      - list
      - watch
```

이 ClusterRole 자체는 권한을 정의할 뿐이며, 실제로 모든 Namespace의 Pod를 조회하려면 ClusterRoleBinding을 통해 Subject에 권한을 부여해야 합니다.

반면 RoleBinding에서 참조하면 해당 RoleBinding이 생성된 Namespace에서만 Pod 조회 권한이 적용됩니다.

### 5.2. Deployment 관리 ClusterRole

다음은 Deployment를 조회하고 생성, 수정, 삭제할 수 있도록 권한을 정의한 예제입니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployment-manager
rules:
  - apiGroups:
      - apps
    resources:
      - deployments
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
      - delete
```

이 ClusterRole은 Deployment에 대한 관리 권한을 정의합니다.

RoleBinding에서 참조하면 특정 Namespace의 Deployment만 관리할 수 있고, ClusterRoleBinding에서 참조하면 모든 Namespace의 Deployment에 해당 권한이 적용됩니다.

### 5.3. 클러스터 리소스 조회 ClusterRole

다음은 Node와 Namespace를 조회할 수 있도록 권한을 정의한 예제입니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-resource-reader
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - namespaces
      - persistentvolumes
    verbs:
      - get
      - list
      - watch
```

Node, Namespace, PersistentVolume은 클러스터 범위 리소스이므로 Namespace에 속하는 Role에서는 해당 리소스에 대한 권한을 정의할 수 없습니다.

이러한 리소스에 접근할 권한을 정의하려면 ClusterRole을 사용해야 합니다.

### 5.4. 여러 API Group과 Resource를 사용하는 ClusterRole

하나의 ClusterRole에 여러 Rule을 정의할 수 있습니다.

다음은 Pod, Deployment, Service, Ingress를 조회할 수 있도록 권한을 정의한 예제입니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: workload-reader
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - services
    verbs:
      - get
      - list
      - watch

  - apiGroups:
      - apps
    resources:
      - deployments
      - replicasets
      - statefulsets
    verbs:
      - get
      - list
      - watch

  - apiGroups:
      - networking.k8s.io
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
```

각 Rule은 서로 다른 API Group과 Resource에 대한 권한을 정의합니다.

## 6. ClusterRole과 Role의 차이

ClusterRole과 Role은 모두 Kubernetes API 리소스에 대한 권한 규칙을 정의하지만, 리소스 자체의 범위와 사용할 수 있는 권한의 종류에서 차이가 있습니다.

| 구분 | Role | ClusterRole |
|---|---|---|
| API Group | `rbac.authorization.k8s.io` | `rbac.authorization.k8s.io` |
| Resource Kind | `Role` | `ClusterRole` |
| Namespace | Namespace 범위 | 클러스터 범위 |
| 권한 규칙 | `rules` | `rules` |
| Namespace 범위 리소스 | 정의 가능 | 정의 가능 |
| 클러스터 범위 리소스 | 정의 불가 | 정의 가능 |
| Non-resource URL | 정의 불가 | 정의 가능 |
| 참조하는 Binding | RoleBinding | RoleBinding, ClusterRoleBinding |

두 리소스는 동일한 PolicyRule 구조를 사용하므로 `apiGroups`, `resources`, `verbs`, `resourceNames` 등의 권한 규칙을 동일한 방식으로 정의합니다.

가장 중요한 차이는 다음과 같습니다.

**Role은 특정 Namespace에서 사용할 권한을 정의하며, ClusterRole은 Namespace에 종속되지 않는 권한 집합을 정의합니다.**

ClusterRole은 Namespace 범위 리소스에 대한 권한도 정의할 수 있으므로, 반드시 클러스터 전체에 권한을 부여해야 하는 상황에서만 사용하는 것은 아닙니다.

예를 들어 여러 Namespace에서 동일한 Pod 조회 권한을 재사용하려면 ClusterRole을 정의한 뒤 각 Namespace에 RoleBinding을 생성할 수 있습니다.

반대로 Node와 같이 클러스터 범위에 속하는 리소스에 대한 권한은 ClusterRole을 통해서만 정의할 수 있습니다.

## 7. Aggregated ClusterRole

Aggregated ClusterRole은 여러 ClusterRole에 정의된 권한 규칙을 하나의 ClusterRole로 자동 결합하는 기능입니다.

일반적으로 공통 권한을 여러 ClusterRole로 분리하여 관리하면서, 이를 하나의 ClusterRole로 통합하여 사용할 때 유용합니다.

AggregationRule을 사용하면 Label Selector에 일치하는 ClusterRole을 찾아 해당 ClusterRole의 `rules`를 대상 ClusterRole에 자동으로 결합합니다.

### 7.1. AggregationRule

`aggregationRule`은 Aggregated ClusterRole을 구성하기 위한 필드입니다.

다음은 `monitoring`이라는 이름의 Aggregated ClusterRole 예제입니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
    - matchLabels:
        rbac.example.com/aggregate-to-monitoring: "true"
```

위 설정은 다음 조건에 해당하는 ClusterRole을 선택합니다.

```yaml
metadata:
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
```

선택된 ClusterRole의 권한 규칙은 `monitoring` ClusterRole에 자동으로 추가됩니다.

`clusterRoleSelectors`는 여러 Label Selector를 지정할 수 있으며, 각 Selector 중 하나라도 일치하면 해당 ClusterRole의 권한 규칙이 결합 대상에 포함됩니다.

### 7.2. Aggregated ClusterRole에 권한 추가

다음은 `monitoring` ClusterRole에 Pod와 Service 조회 권한을 추가하는 예제입니다.

먼저 해당 권한을 정의한 ClusterRole을 생성합니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-workload-reader
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
rules:
  - apiGroups: [""]
    resources:
      - pods
      - services
    verbs:
      - get
      - list
      - watch
```

이 ClusterRole은 `monitoring` ClusterRole의 Label Selector와 일치하므로, 해당 권한 규칙이 자동으로 결합됩니다.

전체 구조는 다음과 같습니다.

```text
ClusterRole: monitoring
│
└── aggregationRule
    └── Label Selector
        └── rbac.example.com/aggregate-to-monitoring: "true"

ClusterRole: monitoring-workload-reader
│
├── labels
│   └── rbac.example.com/aggregate-to-monitoring: "true"
│
└── rules
    ├── pods: get, list, watch
    └── services: get, list, watch

              │
              ▼
    Aggregated ClusterRole
    └── monitoring
        ├── pods: get, list, watch
        └── services: get, list, watch
```

Aggregation Controller가 Label Selector를 기준으로 대상 ClusterRole을 감시하고 권한 규칙을 결합합니다.

따라서 새로운 ClusterRole을 생성하거나 기존 ClusterRole의 Label을 변경하면 Aggregation 결과도 이에 따라 갱신됩니다.

### 7.3. Aggregated ClusterRole의 rules 관리 주의사항

`aggregationRule`이 설정된 ClusterRole의 `rules` 필드는 Controller가 관리합니다.

따라서 Aggregated ClusterRole의 `rules`에 권한을 직접 추가하거나 수정하더라도 Controller가 다시 계산한 결과로 덮어쓰게 됩니다.

권한을 추가하거나 변경하려면 Aggregation 대상이 되는 별도의 ClusterRole을 수정해야 합니다.

또한 Aggregated ClusterRole의 Manifest를 작성할 때는 `rules` 필드를 직접 지정하지 않는 것이 좋습니다.

특히 Server-Side Apply를 사용하는 경우 `rules` 필드를 빈 배열로 지정하거나 직접 관리하면 Controller와 Field Ownership 충돌이 발생할 수 있습니다.

Aggregated ClusterRole은 다음과 같이 `aggregationRule`만 정의하는 방식으로 구성하는 것이 적절합니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
    - matchLabels:
        rbac.example.com/aggregate-to-monitoring: "true"
```

## 8. Kubernetes 기본 제공 ClusterRole

Kubernetes는 클러스터 관리와 사용자 권한 관리를 위해 여러 기본 ClusterRole을 제공합니다.

대표적인 기본 ClusterRole은 다음과 같습니다.

| ClusterRole | 설명 |
|---|---|
| `cluster-admin` | 클러스터의 모든 리소스와 작업에 대한 전체 관리자 권한을 제공합니다. |
| `admin` | Namespace 범위의 대부분의 리소스를 관리할 수 있는 권한을 제공합니다. |
| `edit` | Namespace 범위의 대부분의 리소스를 조회하고 수정할 수 있는 권한을 제공합니다. |
| `view` | Namespace 범위의 리소스를 조회할 수 있는 읽기 전용 권한을 제공합니다. |

이 중 `admin`, `edit`, `view`는 주로 RoleBinding을 통해 특정 Namespace에서 사용하도록 설계되어 있습니다.

반면 `cluster-admin`은 클러스터 전체를 관리하는 권한을 제공하며, ClusterRoleBinding으로 연결하면 클러스터 전체에 대한 관리자 권한이 부여됩니다.

### 8.1. cluster-admin

`cluster-admin`은 Kubernetes에서 제공하는 최고 수준의 관리자 권한입니다.

모든 API Group과 Resource에 대해 모든 작업을 허용하며 Non-resource URL에 대한 접근도 허용합니다.

개념적으로 다음과 같은 권한 규칙으로 구성됩니다.

```yaml
rules:
  - apiGroups:
      - "*"
    resources:
      - "*"
    verbs:
      - "*"

  - nonResourceURLs:
      - "*"
    verbs:
      - "*"
```

`cluster-admin`은 클러스터의 모든 리소스를 관리할 수 있는 권한을 제공하므로 필요한 사용자나 시스템에만 제한적으로 부여해야 합니다.

### 8.2. admin

`admin`은 특정 Namespace의 리소스를 관리하기 위한 권한입니다.

일반적으로 해당 Namespace의 대부분의 리소스를 조회하고 수정할 수 있으며, Role과 RoleBinding을 관리하는 권한도 포함합니다.

다만 Namespace 범위의 리소스에 대한 관리 권한과 클러스터 전체에 대한 관리자 권한은 구분됩니다.

`admin` ClusterRole을 RoleBinding으로 연결하면 해당 Namespace에서만 권한이 적용됩니다.

### 8.3. edit

`edit`는 특정 Namespace의 애플리케이션 리소스를 생성하고 수정할 수 있는 권한을 제공합니다.

Deployment, Pod, Service 등의 리소스를 관리할 수 있지만 Role이나 RoleBinding을 직접 관리하는 권한은 일반적으로 포함하지 않습니다.

또한 Secret을 읽을 수 있는 권한 등이 포함될 수 있으므로, 민감한 인증 정보를 다루는 환경에서는 주의해야 합니다.

### 8.4. view

`view`는 특정 Namespace의 리소스를 조회할 수 있는 읽기 전용 권한을 제공합니다.

일반적으로 애플리케이션 리소스의 상태를 확인하거나 모니터링하는 용도로 사용할 수 있습니다.

다만 Secret은 민감한 정보이므로 기본 `view` 권한에는 Secret 조회 권한이 포함되지 않습니다.

### 8.5. 기본 ClusterRole의 Aggregation

Kubernetes의 기본 `admin`, `edit`, `view` ClusterRole은 Aggregated ClusterRole을 사용하여 추가 권한을 결합할 수 있도록 구성되어 있습니다.

예를 들어 Custom Resource Definition(CRD)을 통해 새로운 리소스를 추가한 경우, 해당 리소스에 대한 권한을 기본 `view` ClusterRole에 포함하려면 다음과 같이 별도의 ClusterRole을 생성할 수 있습니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: aggregate-custom-resource-view
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
  - apiGroups:
      - stable.example.com
    resources:
      - crontabs
    verbs:
      - get
      - list
      - watch
```

위 ClusterRole은 `rbac.authorization.k8s.io/aggregate-to-view: "true"` Label을 가지고 있으므로 기본 `view` ClusterRole의 Aggregation 대상이 됩니다.

이러한 방식을 사용하면 Kubernetes 기본 ClusterRole을 직접 수정하지 않고도 사용자 정의 리소스에 대한 권한을 확장할 수 있습니다.

## 9. ClusterRole 권한 부여 방식

ClusterRole은 권한을 정의하는 리소스이므로 실제 권한을 사용하려면 Binding이 필요합니다.

ClusterRole은 다음 두 가지 Binding에서 참조할 수 있습니다.

- **RoleBinding:** 특정 Namespace에서 ClusterRole의 권한을 부여합니다.
- **ClusterRoleBinding:** 클러스터 전체에서 ClusterRole의 권한을 부여합니다.

예를 들어 `pod-reader` ClusterRole을 `backend` Namespace에서만 사용하려면 RoleBinding을 생성합니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-pod-reader
  namespace: backend
subjects:
  - kind: ServiceAccount
    name: backend-sa
    namespace: backend
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: pod-reader
```

반면 모든 Namespace에서 Pod를 조회할 수 있도록 하려면 ClusterRoleBinding을 사용합니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pod-reader-global
subjects:
  - kind: ServiceAccount
    name: monitoring-sa
    namespace: monitoring
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: pod-reader
```

두 Binding의 차이는 참조하는 ClusterRole이 아니라 권한이 적용되는 범위입니다.

Binding의 상세한 동작과 설정 방법은 `role-binding.md`와 `cluster-role-binding.md`에서 별도로 설명합니다.

## 10. 권한 확인 및 관리 명령어

### 10.1. ClusterRole 조회

전체 ClusterRole 목록을 조회합니다.

```bash
kubectl get clusterroles
```

특정 ClusterRole의 상세 정보를 확인합니다.

```bash
kubectl describe clusterrole pod-reader
```

ClusterRole의 YAML 정의를 확인하려면 다음 명령어를 사용합니다.

```bash
kubectl get clusterrole pod-reader -o yaml
```

### 10.2. ClusterRole 생성

`kubectl create clusterrole` 명령어를 사용하면 YAML 파일 없이 ClusterRole을 생성할 수 있습니다.

다음은 Pod 조회 권한을 정의하는 예제입니다.

```bash
kubectl create clusterrole pod-reader \
  --verb=get,list,watch \
  --resource=pods
```

특정 API Group에 속한 리소스를 지정할 수도 있습니다.

```bash
kubectl create clusterrole deployment-reader \
  --verb=get,list,watch \
  --resource=deployments.apps
```

Subresource에 대한 권한도 지정할 수 있습니다.

```bash
kubectl create clusterrole pod-log-reader \
  --verb=get \
  --resource=pods,pods/log
```

### 10.3. 권한 확인

`kubectl auth can-i` 명령어를 사용하면 현재 사용자가 특정 리소스에 대해 작업할 수 있는지 확인할 수 있습니다.

현재 사용자가 모든 Namespace의 Pod를 조회할 수 있는지 확인하려면 다음과 같이 실행합니다.

```bash
kubectl auth can-i list pods --all-namespaces
```

특정 Namespace에서만 권한을 확인하려면 다음과 같이 실행합니다.

```bash
kubectl auth can-i list pods -n backend
```

특정 ServiceAccount를 가장(Impersonation)하여 권한을 확인할 수도 있습니다.

```bash
kubectl auth can-i list pods \
  --all-namespaces \
  --as=system:serviceaccount:monitoring:monitoring-sa
```

이 명령어를 실행하려면 현재 사용자에게 해당 ServiceAccount를 가장할 수 있는 권한이 있어야 합니다.

## 11. 보안 고려사항

ClusterRole은 여러 Namespace에서 재사용할 수 있고 클러스터 범위 리소스에 대한 권한도 정의할 수 있으므로 권한 설계에 주의해야 합니다.

### 11.1. 최소 권한 원칙

애플리케이션이나 사용자에게 필요한 최소한의 권한만 정의해야 합니다.

예를 들어 Pod 조회만 필요한 경우 `get`, `list`, `watch`만 허용하고, `create`, `update`, `delete` 등의 권한은 부여하지 않는 것이 좋습니다.

### 11.2. Binding 범위 확인

ClusterRole을 참조한다고 해서 반드시 클러스터 전체에 권한이 부여되는 것은 아닙니다.

RoleBinding을 통해 연결하면 해당 Namespace에만 권한이 적용되지만, ClusterRoleBinding을 사용하면 클러스터 전체에 권한이 적용됩니다.

따라서 ClusterRole을 생성하거나 Binding을 구성할 때 권한 적용 범위를 명확하게 확인해야 합니다.

### 11.3. 와일드카드 사용 주의

`apiGroups`, `resources`, `verbs`에 `*`를 사용하면 현재 존재하는 리소스뿐만 아니라 향후 추가되는 리소스나 작업까지 권한 범위에 포함될 수 있습니다.

따라서 특별한 목적이 없다면 필요한 API Group, Resource, Verb를 구체적으로 지정하는 것이 좋습니다.

### 11.4. 민감한 리소스 접근 제한

Secret에는 비밀번호, API Token, 인증서 등의 민감한 정보가 포함될 수 있습니다.

Secret 조회 권한을 가진 Subject는 해당 Namespace의 민감한 정보에 접근할 수 있으므로, 필요한 경우에만 권한을 부여해야 합니다.

특히 `view`와 같은 읽기 전용 역할을 사용하더라도 실제 포함된 권한을 확인하는 것이 좋습니다.

### 11.5. Aggregated ClusterRole 관리 주의

AggregationRule이 설정된 ClusterRole의 `rules`는 Controller가 관리합니다.

따라서 Aggregated ClusterRole에 권한을 직접 추가하는 대신 Label Selector에 일치하는 별도의 ClusterRole을 생성하거나 수정해야 합니다.

또한 Aggregation 대상 ClusterRole에 잘못된 Label을 지정하면 의도하지 않은 권한이 결합될 수 있으므로 Label 관리에도 주의해야 합니다.

### 11.6. 권한 상승 방지

Kubernetes는 RBAC 리소스 생성 및 수정 과정에서 권한 상승(Privilege Escalation)을 방지하기 위한 검증을 수행합니다.

ClusterRole을 생성하거나 수정하려면 일반적으로 해당 권한을 이미 보유하고 있어야 합니다.

또는 대상 ClusterRole에 대해 `escalate` Verb를 수행할 수 있는 권한이 별도로 부여되어 있어야 합니다.

이러한 검증은 사용자가 자신이 보유하지 않은 권한을 임의로 정의하여 다른 주체에게 부여하는 상황을 방지합니다.

## 12. 정리

ClusterRole은 Kubernetes RBAC에서 클러스터 범위의 권한 규칙을 정의하는 리소스입니다.

핵심 내용을 정리하면 다음과 같습니다.

- ClusterRole은 Namespace에 속하지 않는 클러스터 범위 리소스입니다.
- `rules` 필드를 사용하여 API Group, Resource, Verb 등의 권한을 정의합니다.
- Namespace 범위 리소스뿐만 아니라 Node와 같은 클러스터 범위 리소스에 대한 권한도 정의할 수 있습니다.
- `/healthz`와 같은 Non-resource URL에 대한 접근 권한도 정의할 수 있습니다.
- RoleBinding에서 참조하면 해당 Namespace에만 권한이 적용됩니다.
- ClusterRoleBinding에서 참조하면 클러스터 전체에 권한이 적용됩니다.
- AggregationRule을 사용하면 여러 ClusterRole의 권한 규칙을 하나로 결합할 수 있습니다.
- Kubernetes는 `cluster-admin`, `admin`, `edit`, `view` 등의 기본 ClusterRole을 제공합니다.
- 권한을 설계할 때는 최소 권한 원칙과 Binding의 권한 적용 범위를 고려해야 합니다.

---

## 참고자료

ClusterRole의 리소스 구조와 권한 규칙, AggregationRule 및 기본 ClusterRole의 동작 방식은 Kubernetes 공식 문서를 기준으로 정리했습니다.

- [Kubernetes Documentation - Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes API Reference - ClusterRole](https://kubernetes.io/docs/reference/kubernetes-api/authorization-resources/cluster-role-v1/)
- [Kubernetes API Reference - RBAC](https://kubernetes.io/docs/reference/kubernetes-api/rbac/)
- [Kubernetes Documentation - Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [Kubernetes Documentation - kubectl create clusterrole](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_clusterrole/)
- [Kubernetes Documentation - kubectl auth can-i](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_auth/kubectl_auth_can-i/)