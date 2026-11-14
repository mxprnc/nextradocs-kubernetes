# Role

## 1. Role이란?

Role은 Kubernetes RBAC(Role-Based Access Control)에서 **특정 Namespace 내 Kubernetes 리소스에 대한 접근 권한을 정의하는 리소스**입니다.

Kubernetes에서는 사용자(User), 그룹(Group), ServiceAccount와 같은 주체(Subject)가 Kubernetes API를 통해 리소스에 접근할 때 적절한 권한이 있는지 확인합니다.

Role은 이러한 접근 권한을 정의하는 역할을 수행합니다.

예를 들어, 특정 애플리케이션이나 사용자가 `backend` Namespace에 있는 Pod를 조회할 수 있도록 허용하려면 Pod에 대한 조회 권한을 Role로 정의할 수 있습니다.

Role은 다음과 같은 정보를 기반으로 접근 권한을 정의합니다.

- **API Group:** 접근할 Kubernetes API 그룹을 지정합니다.
- **Resource:** 접근할 Kubernetes 리소스 종류를 지정합니다.
- **Verb:** 해당 리소스에 대해 허용할 작업을 지정합니다.

Role은 권한을 정의하는 리소스이며, 특정 사용자나 ServiceAccount에 권한을 직접 부여하지는 않습니다.

실제로 권한을 부여하려면 RoleBinding을 사용하여 Role과 User, Group 또는 ServiceAccount를 연결해야 합니다.

Role의 주요 특징은 다음과 같습니다.

- Namespace 범위의 리소스입니다.
- 특정 Namespace 내에서 사용할 권한을 정의합니다.
- 여러 개의 권한 규칙(Rule)을 정의할 수 있습니다.
- 권한은 허용(Allow) 방식으로 추가되며, 명시적인 거부(Deny) 규칙은 지원하지 않습니다.
- RoleBinding을 통해 User, Group 또는 ServiceAccount에 권한을 부여할 수 있습니다.

## 2. Role의 동작 원리

Kubernetes API Server는 요청이 들어오면 인증(Authentication)과 권한 부여(Authorization) 과정을 거쳐 요청을 처리합니다.

RBAC이 활성화된 환경에서는 인증된 요청 주체가 요청한 리소스와 작업에 대해 적절한 권한을 가지고 있는지 확인합니다.

Role은 이 과정에서 참조되는 권한 규칙을 정의합니다.

예를 들어, `backend` Namespace에서 특정 ServiceAccount가 Pod를 조회할 수 있도록 구성하려면 다음과 같은 구조를 사용합니다.

```text
Pod
 └── ServiceAccount
      │
      │ API Server 요청
      ▼
  Authentication
      │
      │ 인증된 신원 확인
      ▼
  Authorization (RBAC)
      │
      │ RoleBinding을 통해 권한 확인
      ▼
  Role
      │
      └── Pods: get, list, watch
```

각 구성 요소의 역할은 다음과 같습니다.

1. Pod 내부의 애플리케이션은 연결된 ServiceAccount의 인증 정보를 사용하여 API Server에 요청합니다.
2. API Server는 전달된 인증 정보를 검증하여 요청 주체의 신원을 확인합니다.
3. RBAC Authorizer는 해당 주체에 연결된 Role 및 ClusterRole의 권한을 확인합니다.
4. 요청한 리소스와 작업이 허용된 권한에 해당하면 요청을 승인합니다.

Role은 권한을 정의하고, RoleBinding은 해당 권한을 사용할 주체를 연결한다는 점을 구분해야 합니다.

## 3. Role 생성 및 설정

Role은 `apiVersion: rbac.authorization.k8s.io/v1`, `kind: Role`을 사용하여 정의합니다.

다음은 `backend` Namespace에서 Pod 조회 권한을 정의하는 Role 예제입니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: backend
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

위 Role은 `backend` Namespace의 Pod에 대해 다음 작업을 허용합니다.

- `get`: 특정 Pod의 정보를 조회합니다.
- `list`: Pod 목록을 조회합니다.
- `watch`: Pod의 변경 사항을 감시합니다.

다만 이 Role을 생성하는 것만으로 특정 사용자나 애플리케이션에 권한이 부여되는 것은 아닙니다.

RoleBinding을 통해 권한을 부여해야 실제로 사용할 수 있습니다.

Role은 다음 명령어로 생성할 수 있습니다.

```bash
kubectl apply -f role.yaml
```

생성된 Role은 다음과 같이 조회합니다.

```bash
kubectl get roles -n backend
```

특정 Role의 상세 정보는 다음 명령어로 확인할 수 있습니다.

```bash
kubectl describe role pod-reader -n backend
```

### 3.1. 주요 필드

| 필드 | 설명 |
|---|---|
| `apiVersion` | API 버전을 지정합니다. `rbac.authorization.k8s.io/v1`을 사용합니다. |
| `kind` | 리소스 종류를 지정합니다. `Role`을 사용합니다. |
| `metadata.name` | Role의 이름을 지정합니다. |
| `metadata.namespace` | Role이 속할 Namespace를 지정합니다. |
| `metadata.labels` | Role을 식별하거나 분류하기 위한 Label을 지정합니다. |
| `metadata.annotations` | Role에 추가적인 메타데이터를 지정합니다. |
| `rules` | 허용할 API Group, Resource, Verb 등의 권한 규칙을 정의합니다. |

## 4. Role의 권한 규칙(Rules)

Role의 핵심은 `rules` 필드입니다.

`rules`는 하나 이상의 권한 규칙을 배열 형태로 정의하며, 각 규칙은 특정 API Group과 Resource에 대해 허용할 작업을 지정합니다.

주요 구성 요소는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `apiGroups` | 접근할 Kubernetes API Group을 지정합니다. |
| `resources` | 접근할 Kubernetes 리소스를 지정합니다. |
| `verbs` | 리소스에 대해 허용할 작업을 지정합니다. |
| `resourceNames` | 접근을 허용할 특정 리소스 이름을 제한적으로 지정합니다. |
| `nonResourceURLs` | Kubernetes API 리소스가 아닌 URL 경로에 대한 접근을 지정합니다. Role에서는 사용할 수 없습니다. |

### 4.1. apiGroups

`apiGroups`는 접근하려는 Kubernetes 리소스가 속한 API Group을 지정합니다.

Kubernetes 리소스는 API Group에 따라 구분되며, Role에서 권한을 정의할 때 해당 리소스가 속한 API Group을 정확하게 지정해야 합니다.

대표적인 API Group은 다음과 같습니다.

| API Group | 대표 리소스 |
|---|---|
| `""` | Pod, Service, ConfigMap, Secret, Namespace |
| `apps` | Deployment, ReplicaSet, StatefulSet, DaemonSet |
| `batch` | Job, CronJob |
| `networking.k8s.io` | Ingress, NetworkPolicy |
| `rbac.authorization.k8s.io` | Role, RoleBinding, ClusterRole, ClusterRoleBinding |
| `autoscaling` | HorizontalPodAutoscaler |
| `storage.k8s.io` | StorageClass |

Core API Group에 속하는 리소스는 빈 문자열(`""`)로 지정합니다.

예를 들어, Pod는 Core API Group에 속하므로 다음과 같이 설정합니다.

```yaml
apiGroups: [""]
```

반면 Deployment는 `apps` API Group에 속하므로 다음과 같이 설정합니다.

```yaml
apiGroups: ["apps"]
```

### 4.2. resources

`resources`는 접근할 Kubernetes 리소스의 종류를 지정합니다.

리소스 이름은 Kubernetes API에서 사용하는 복수형 이름을 기준으로 지정합니다.

| 리소스 | resources 값 |
|---|---|
| Pod | `pods` |
| Service | `services` |
| ConfigMap | `configmaps` |
| Secret | `secrets` |
| Deployment | `deployments` |
| ReplicaSet | `replicasets` |
| StatefulSet | `statefulsets` |
| DaemonSet | `daemonsets` |
| Job | `jobs` |
| CronJob | `cronjobs` |
| Ingress | `ingresses` |
| NetworkPolicy | `networkpolicies` |

예를 들어, Pod와 Service에 대한 권한을 함께 정의하려면 다음과 같이 설정할 수 있습니다.

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

위 규칙은 Pod와 Service에 대해 동일한 Verb를 허용합니다.

### 4.3. verbs

`verbs`는 지정한 리소스에 대해 허용할 작업을 정의합니다.

Kubernetes API에서 사용하는 주요 Verb는 다음과 같습니다.

| Verb | 설명 |
|---|---|
| `get` | 특정 리소스의 정보를 조회합니다. |
| `list` | 리소스 목록을 조회합니다. |
| `watch` | 리소스의 생성, 수정, 삭제 등의 변경 사항을 감시합니다. |
| `create` | 새로운 리소스를 생성합니다. |
| `update` | 기존 리소스를 전체적으로 수정합니다. |
| `patch` | 기존 리소스의 일부 필드를 수정합니다. |
| `delete` | 특정 리소스를 삭제합니다. |
| `deletecollection` | 지정한 조건에 해당하는 여러 리소스를 삭제합니다. |

예를 들어, Pod를 조회하는 권한만 허용하려면 다음과 같이 설정합니다.

```yaml
verbs:
  - get
  - list
  - watch
```

반면 Pod의 생성과 삭제까지 허용하려면 다음과 같이 추가합니다.

```yaml
verbs:
  - get
  - list
  - watch
  - create
  - delete
```

Kubernetes RBAC은 명시적으로 허용한 작업만 수행할 수 있도록 구성하는 것이 좋습니다.

따라서 애플리케이션에 필요한 작업만 `verbs`에 지정하는 최소 권한 원칙(Principle of Least Privilege)을 적용해야 합니다.

### 4.4. resourceNames

`resourceNames`는 특정 이름을 가진 리소스에 대해서만 접근을 허용하도록 제한하는 필드입니다.

예를 들어, `backend-config`라는 ConfigMap만 조회할 수 있도록 제한하려면 다음과 같이 설정합니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: backend
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames:
      - backend-config
    verbs:
      - get
```

위 Role은 `backend` Namespace에 있는 `backend-config` ConfigMap에 대해서만 `get` 작업을 허용합니다.

다른 ConfigMap을 조회하려고 하면 권한이 거부됩니다.

`resourceNames`를 사용할 때는 다음 사항에 주의해야 합니다.

- `get`, `update`, `patch`, `delete` 등의 작업에서 특정 리소스 이름을 제한할 수 있습니다.
- 최상위 리소스에 대한 `create` 요청은 생성 대상의 이름이 권한 확인 시점에 정해져 있지 않으므로 `resourceNames`로 제한할 수 없습니다.
- `deletecollection` 작업은 `resourceNames`로 제한할 수 없습니다.
- `list`와 `watch`에 `resourceNames`를 사용하는 경우 요청에 해당 이름과 일치하는 `metadata.name` 필드 셀렉터가 포함되어야 합니다.

예를 들어, 특정 ConfigMap만 조회하려면 다음과 같이 요청할 수 있습니다.

```bash
kubectl get configmaps \
  -n backend \
  --field-selector=metadata.name=backend-config
```

이처럼 `resourceNames`는 Namespace 내 모든 리소스가 아닌 특정 리소스 인스턴스에 대해서만 접근을 허용할 때 유용합니다.

## 5. Subresource 권한 설정

Kubernetes API에서는 일부 리소스가 하위 리소스(Subresource)를 제공합니다.

Subresource는 기존 리소스에 종속된 별도의 API 경로를 통해 접근하는 기능입니다.

대표적인 예시는 다음과 같습니다.

| Resource | Subresource | 설명 |
|---|---|---|
| `pods` | `pods/log` | Pod의 로그를 조회합니다. |
| `pods` | `pods/exec` | Pod 컨테이너 내부에서 명령을 실행합니다. |
| `pods` | `pods/attach` | Pod 컨테이너의 프로세스에 연결합니다. |
| `pods` | `pods/portforward` | Pod에 대한 포트 포워딩을 수행합니다. |
| `pods` | `pods/status` | Pod의 상태 정보를 조회하거나 수정합니다. |
| `deployments` | `deployments/scale` | Deployment의 스케일을 조정합니다. |

Role에서는 `resources` 필드에 슬래시(`/`)를 사용하여 Subresource를 지정합니다.

예를 들어, Pod의 로그를 조회할 수 있는 권한을 부여하려면 다음과 같이 설정합니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-log-reader
  namespace: backend
rules:
  - apiGroups: [""]
    resources:
      - pods/log
    verbs:
      - get
```

위 Role은 `backend` Namespace에 있는 Pod의 로그를 조회할 수 있도록 허용합니다.

Pod 자체의 정보를 조회하는 권한과 Pod 로그를 조회하는 권한은 별도로 정의할 수 있습니다.

예를 들어, Pod 정보 조회와 로그 조회를 모두 허용하려면 다음과 같이 설정합니다.

```yaml
rules:
  - apiGroups: [""]
    resources:
      - pods
      - pods/log
    verbs:
      - get
      - list
      - watch
```

다만 위 설정은 `pods/log`에도 `list`와 `watch`를 허용하는 규칙이 됩니다. 실제 필요한 작업만 허용하려면 다음과 같이 규칙을 분리하는 것이 적절합니다.

```yaml
rules:
  - apiGroups: [""]
    resources:
      - pods
    verbs:
      - get
      - list
      - watch

  - apiGroups: [""]
    resources:
      - pods/log
    verbs:
      - get
```

Subresource는 일반 리소스와 별도로 권한을 관리할 수 있으므로 필요한 작업을 정확하게 구분하여 설정하는 것이 중요합니다.

## 6. Role 예제

### 6.1. Pod 조회 전용 Role

다음은 특정 Namespace의 Pod를 조회할 수 있도록 권한을 정의하는 예제입니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: backend
rules:
  - apiGroups: [""]
    resources:
      - pods
    verbs:
      - get
      - list
      - watch
```

이 Role은 `backend` Namespace의 Pod에 대해 조회 작업만 허용합니다.

Pod를 생성하거나 수정하거나 삭제하는 작업은 허용하지 않습니다.

### 6.2. Deployment 관리 Role

다음은 Deployment를 조회하고 생성, 수정, 삭제할 수 있도록 권한을 정의하는 예제입니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-manager
  namespace: backend
rules:
  - apiGroups: ["apps"]
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

이 Role은 `backend` Namespace의 Deployment에 대한 관리 권한을 제공합니다.

Deployment가 속한 `apps` API Group을 지정했다는 점에 주의해야 합니다.

### 6.3. 여러 API Group과 Resource를 사용하는 Role

하나의 Role에는 여러 권한 규칙을 정의할 수 있습니다.

다음은 Pod 조회와 Job 관리를 하나의 Role로 구성한 예제입니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: workload-manager
  namespace: backend
rules:
  - apiGroups: [""]
    resources:
      - pods
    verbs:
      - get
      - list
      - watch

  - apiGroups: ["batch"]
    resources:
      - jobs
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
      - delete
```

이 Role은 다음과 같은 권한을 정의합니다.

- Pod: 조회 권한
- Job: 조회, 생성, 수정, 삭제 권한

각 Rule은 서로 독립적으로 권한을 정의하므로 리소스 종류에 따라 서로 다른 Verb를 지정할 수 있습니다.

### 6.4. 여러 리소스에 동일한 권한 부여

동일한 API Group에 속하는 여러 리소스에 같은 권한을 부여할 수도 있습니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: workload-reader
  namespace: backend
rules:
  - apiGroups: ["apps"]
    resources:
      - deployments
      - replicasets
      - statefulsets
      - daemonsets
    verbs:
      - get
      - list
      - watch
```

위 Role은 `apps` API Group에 속하는 Deployment, ReplicaSet, StatefulSet, DaemonSet을 조회할 수 있도록 허용합니다.

## 7. Role의 Namespace 범위

Role은 Namespace 범위의 리소스이므로 생성할 때 반드시 Namespace를 지정해야 합니다.

예를 들어, 다음 Role은 `backend` Namespace에 속합니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: backend
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

이 Role이 정의하는 권한은 `backend` Namespace에 한정됩니다.

동일한 이름의 Role이 다른 Namespace에 존재하더라도 서로 다른 리소스로 취급됩니다.

예를 들어 다음과 같은 구조가 가능합니다.

```text
Kubernetes Cluster
│
├── Namespace: backend
│   └── Role: pod-reader
│
├── Namespace: frontend
│   └── Role: pod-reader
│
└── Namespace: monitoring
    └── Role: pod-reader
```

각 Namespace의 `pod-reader`는 독립적으로 관리되며, 서로의 권한 설정에 영향을 주지 않습니다.

또한 Role은 Namespace 범위의 리소스만 대상으로 권한을 정의할 수 있습니다.

Node, PersistentVolume과 같은 클러스터 범위(Cluster-scoped) 리소스에 대한 권한은 Role로 정의할 수 없습니다.

클러스터 범위의 리소스나 여러 Namespace에 걸친 권한을 정의하려면 ClusterRole을 사용해야 합니다.

## 8. Role과 ClusterRole의 차이

Role과 ClusterRole은 모두 RBAC에서 접근 권한을 정의하는 리소스입니다.

두 리소스는 권한 규칙을 정의하는 방식은 동일하지만 리소스의 범위와 활용 방식에 차이가 있습니다.

| 구분 | Role | ClusterRole |
|---|---|---|
| API Version | `rbac.authorization.k8s.io/v1` | `rbac.authorization.k8s.io/v1` |
| Resource Kind | `Role` | `ClusterRole` |
| Namespace | Namespace 범위 | 클러스터 범위 |
| 권한 정의 | Namespace 범위의 리소스 | Namespace 및 클러스터 범위의 리소스 |
| Cluster-scoped Resource 접근 | 불가능 | 가능 |
| 여러 Namespace에 동일한 권한 적용 | Namespace마다 별도 Role 필요 | 하나의 ClusterRole을 여러 RoleBinding에서 재사용 가능 |
| 권한 부여 | RoleBinding | RoleBinding 또는 ClusterRoleBinding |

두 리소스의 주요 차이는 다음과 같습니다.

**Role**

- 특정 Namespace에서 사용할 권한을 정의합니다.
- 해당 Namespace의 리소스에 대한 접근 권한을 관리합니다.
- RoleBinding을 통해 권한을 부여합니다.

**ClusterRole**

- Namespace 범위에 종속되지 않는 리소스입니다.
- Namespace 범위의 리소스뿐만 아니라 Node와 같은 클러스터 범위 리소스에 대한 권한도 정의할 수 있습니다.
- RoleBinding을 통해 특정 Namespace에서 권한을 부여하거나 ClusterRoleBinding을 통해 클러스터 전체에 권한을 부여할 수 있습니다.

예를 들어, 여러 Namespace에서 동일한 Pod 조회 권한을 사용해야 한다면 각 Namespace에 동일한 Role을 반복해서 생성하는 대신 하나의 ClusterRole을 생성하고 각 Namespace에서 RoleBinding으로 연결할 수 있습니다.

## 9. Role과 RoleBinding의 관계

Role은 접근 권한을 정의하지만, 특정 사용자나 ServiceAccount에 권한을 직접 부여하지 않습니다.

실제로 권한을 부여하려면 RoleBinding을 사용해야 합니다.

예를 들어, `backend` Namespace에서 `backend-sa` ServiceAccount에 `pod-reader` Role의 권한을 부여하려면 다음과 같이 구성합니다.

```text
Namespace: backend
│
├── ServiceAccount
│   └── backend-sa
│
├── Role
│   └── pod-reader
│       └── pods: get, list, watch
│
└── RoleBinding
    ├── Subject: backend-sa
    └── RoleRef: pod-reader
```

이 구조에서 RoleBinding은 Role과 ServiceAccount를 연결하는 역할을 수행합니다.

RoleBinding을 생성하는 방법과 `subjects`, `roleRef` 필드의 상세 내용은 `role-binding.md`에서 별도로 설명합니다.

## 10. 권한 확인 및 관리 명령어

Kubernetes에서는 `kubectl`을 사용하여 Role을 생성하고 조회하거나 권한을 확인할 수 있습니다.

### 10.1. Role 조회

특정 Namespace의 Role 목록을 조회합니다.

```bash
kubectl get roles -n backend
```

전체 Namespace의 Role 목록을 조회하려면 다음 명령어를 사용합니다.

```bash
kubectl get roles -A
```

### 10.2. Role 상세 정보 확인

특정 Role에 정의된 권한 규칙을 확인합니다.

```bash
kubectl describe role pod-reader -n backend
```

### 10.3. Role 생성

`kubectl create role` 명령어를 사용하여 Role을 생성할 수도 있습니다.

```bash
kubectl create role pod-reader \
  --verb=get \
  --verb=list \
  --verb=watch \
  --resource=pods \
  -n backend
```

위 명령어는 `backend` Namespace에 `pod-reader` Role을 생성합니다.

### 10.4. 권한 확인

`kubectl auth can-i` 명령어를 사용하면 특정 주체가 Kubernetes API에서 특정 작업을 수행할 수 있는지 확인할 수 있습니다.

현재 사용자의 Pod 조회 권한을 확인하려면 다음과 같이 실행합니다.

```bash
kubectl auth can-i list pods -n backend
```

특정 ServiceAccount의 권한을 확인하려면 `--as` 옵션을 사용할 수 있습니다.

```bash
kubectl auth can-i list pods \
  -n backend \
  --as=system:serviceaccount:backend:backend-sa
```

위 명령어는 `backend` Namespace의 `backend-sa` ServiceAccount가 Pod 목록을 조회할 수 있는지 확인합니다.

이 명령어를 사용하려면 현재 실행 중인 사용자에게 해당 주체를 가장(Impersonation)할 수 있는 권한이 있어야 합니다.

## 11. 보안 고려사항

Role은 Kubernetes API에 대한 접근 권한을 정의하므로 권한을 설계할 때 다음과 같은 보안 사항을 고려해야 합니다.

### 11.1. 최소 권한 원칙 적용

애플리케이션이 실제로 사용하는 리소스와 작업만 허용해야 합니다.

예를 들어, Pod 정보를 조회하는 애플리케이션에 Pod 생성이나 삭제 권한까지 부여할 필요는 없습니다.

필요한 `apiGroups`, `resources`, `verbs`만 명시적으로 지정하는 것이 좋습니다.

### 11.2. 와일드카드 사용 주의

Kubernetes RBAC에서는 `*`를 사용하여 API Group, Resource 또는 Verb 전체를 지정할 수 있습니다.

예를 들어 다음과 같이 설정하면 Core API Group의 모든 리소스에 대해 모든 작업을 허용하게 됩니다.

```yaml
rules:
  - apiGroups: [""]
    resources: ["*"]
    verbs: ["*"]
```

이러한 설정은 새로운 리소스나 작업이 추가되었을 때도 자동으로 권한 범위에 포함될 수 있으므로 과도한 권한 부여로 이어질 수 있습니다.

따라서 필요한 리소스와 작업을 명시적으로 지정하는 것이 좋습니다.

### 11.3. Secret 접근 권한 제한

Secret에는 데이터베이스 비밀번호, API Key, 인증서 등 민감한 정보가 포함될 수 있습니다.

Secret에 대한 `get`, `list`, `watch` 권한을 가진 주체는 민감한 정보에 접근할 수 있으므로 필요한 애플리케이션에만 제한적으로 권한을 부여해야 합니다.

### 11.4. 권한 범위 최소화

특정 Namespace에서만 필요한 권한이라면 ClusterRole보다 Role을 사용하는 것이 적절합니다.

Namespace 범위로 권한을 제한하면 불필요한 클러스터 전체 접근을 방지할 수 있습니다.

### 11.5. 권한 상승 방지

Kubernetes RBAC은 사용자가 자신이 가진 권한보다 더 높은 권한을 다른 사용자에게 부여하는 행위를 방지하기 위한 제한을 제공합니다.

예를 들어, 특정 사용자가 Secret을 조회할 권한이 없는데 Secret 조회 권한을 포함하는 Role을 임의로 생성하려고 하면 API Server에서 요청이 거부될 수 있습니다.

Role을 생성하거나 수정할 때는 해당 Role에 포함된 권한을 이미 보유하고 있거나 `escalate` 권한을 별도로 부여받아야 합니다.

또한 RoleBinding을 생성하거나 수정할 때도 참조하는 Role의 권한을 부여할 수 있는 권한이 필요합니다. 이미 해당 권한을 가지고 있거나 대상 Role에 대한 `bind` 권한을 별도로 부여받아야 합니다.

이러한 제한은 RBAC을 이용한 권한 상승(Privilege Escalation)을 방지하기 위한 보호 장치입니다.

## 12. 정리

Role은 Kubernetes RBAC에서 특정 Namespace의 리소스에 대한 접근 권한을 정의하는 리소스입니다.

핵심 내용을 정리하면 다음과 같습니다.

- Role은 Namespace 범위의 Kubernetes 리소스입니다.
- `rules` 필드를 통해 API Group, Resource, Verb 등의 권한을 정의합니다.
- `apiGroups`는 접근할 API 그룹을 지정하며 Core API Group은 빈 문자열(`""`)을 사용합니다.
- `resources`는 접근할 리소스를 지정하며 Subresource도 함께 지정할 수 있습니다.
- `verbs`는 허용할 작업을 정의합니다.
- `resourceNames`를 사용하면 특정 이름을 가진 리소스에 대해서만 접근을 허용할 수 있습니다.
- Role은 권한을 정의할 뿐 특정 주체에 권한을 직접 부여하지 않습니다.
- 실제 권한 부여에는 RoleBinding이 필요합니다.
- 클러스터 범위의 리소스나 여러 Namespace에 걸친 권한을 정의하려면 ClusterRole을 사용합니다.
- 최소 권한 원칙을 적용하고 불필요한 와일드카드 사용을 피하는 것이 좋습니다.

---

## 참고자료

Role과 RBAC의 동작 방식 및 권한 규칙은 Kubernetes 공식 문서를 기준으로 정리했습니다.

특히 API Group, Resource, Verb의 관계와 Subresource 접근 제어, `resourceNames`를 이용한 리소스별 권한 제한, Role과 ClusterRole의 범위 차이를 참고할 수 있습니다.

- [Kubernetes Documentation - Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes Documentation - Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [Kubernetes Documentation - API Concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/)
- [Kubernetes API Reference - Role](https://kubernetes.io/docs/reference/kubernetes-api/authorization-resources/role-v1/)
- [Kubernetes Documentation - kubectl auth can-i](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_auth/kubectl_auth_can-i/)