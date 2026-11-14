# RoleBinding

## 1. RoleBinding이란?

RoleBinding은 Kubernetes RBAC(Role-Based Access Control)에서 **Role 또는 ClusterRole에 정의된 권한을 User, Group 또는 ServiceAccount에 부여하기 위한 리소스**입니다.

Role과 ClusterRole은 Kubernetes 리소스에 대한 접근 권한을 정의하지만, 특정 사용자나 애플리케이션에 권한을 직접 부여하지는 않습니다.

RoleBinding은 이러한 권한 정의와 실제 권한을 사용할 주체(Subject)를 연결하는 역할을 수행합니다.

예를 들어, `backend` Namespace에서 실행되는 애플리케이션이 Pod 목록을 조회할 수 있도록 구성하려면 다음과 같은 리소스가 필요합니다.

- **ServiceAccount:** 애플리케이션의 신원을 제공합니다.
- **Role:** Pod에 대한 조회 권한을 정의합니다.
- **RoleBinding:** Role에 정의된 권한을 ServiceAccount에 부여합니다.

RoleBinding의 주요 특징은 다음과 같습니다.

- Namespace 범위의 Kubernetes 리소스입니다.
- Role 또는 ClusterRole을 참조할 수 있습니다.
- User, Group 또는 ServiceAccount에 권한을 부여할 수 있습니다.
- RoleBinding이 생성된 Namespace 내에서만 권한이 적용됩니다.
- 참조하는 Role 또는 ClusterRole은 `roleRef` 필드로 지정합니다.
- 생성된 RoleBinding의 `roleRef`는 변경할 수 없습니다.

RoleBinding은 권한 자체를 정의하는 것이 아니라 기존에 정의된 권한을 특정 주체에 연결한다는 점이 중요합니다.

## 2. RoleBinding의 동작 원리

Kubernetes API Server는 클라이언트의 요청을 처리할 때 인증(Authentication)과 권한 부여(Authorization) 과정을 수행합니다.

RBAC이 활성화된 환경에서는 인증된 요청 주체가 요청한 리소스와 작업에 대해 적절한 권한을 가지고 있는지 확인합니다.

RoleBinding은 RBAC Authorizer가 권한을 확인할 때 참조하는 리소스 중 하나입니다.

예를 들어, `backend` Namespace의 `backend-sa` ServiceAccount에 Pod 조회 권한을 부여하는 경우 다음과 같은 구조로 구성합니다.

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
    ├── subjects
    │   └── backend-sa
    │
    └── roleRef
        └── pod-reader
```

이 구조에서 각 리소스의 역할은 다음과 같습니다.

1. Pod는 `spec.serviceAccountName`을 통해 `backend-sa` ServiceAccount를 사용합니다.
2. 애플리케이션은 ServiceAccount Token을 사용하여 Kubernetes API Server에 요청합니다.
3. API Server는 토큰을 검증하고 요청 주체의 신원을 확인합니다.
4. RBAC Authorizer는 해당 주체와 연결된 RoleBinding을 확인합니다.
5. RoleBinding이 참조하는 Role에서 요청한 리소스와 작업이 허용되는지 확인합니다.
6. 권한이 허용되면 API 요청을 승인합니다.

RoleBinding은 Role에 정의된 권한을 실제 주체에 연결하는 역할을 수행합니다.

## 3. RoleBinding 생성 및 설정

RoleBinding은 `apiVersion: rbac.authorization.k8s.io/v1`, `kind: RoleBinding`을 사용하여 정의합니다.

다음은 `backend` Namespace의 `backend-sa` ServiceAccount에 `pod-reader` Role의 권한을 부여하는 예제입니다.

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
  kind: Role
  name: pod-reader
```

위 RoleBinding은 다음과 같이 동작합니다.

- `subjects`에는 권한을 부여할 `backend-sa` ServiceAccount를 지정합니다.
- `roleRef`에는 권한을 가져올 `pod-reader` Role을 지정합니다.
- `metadata.namespace`에는 RoleBinding이 적용될 Namespace인 `backend`를 지정합니다.

따라서 `backend-sa`는 `backend` Namespace에서 `pod-reader` Role에 정의된 권한을 사용할 수 있습니다.

RoleBinding은 다음 명령어로 생성할 수 있습니다.

```bash
kubectl apply -f role-binding.yaml
```

생성된 RoleBinding은 다음과 같이 조회합니다.

```bash
kubectl get rolebindings -n backend
```

특정 RoleBinding의 상세 정보는 다음 명령어로 확인할 수 있습니다.

```bash
kubectl describe rolebinding backend-pod-reader -n backend
```

### 3.1. 주요 필드

| 필드 | 설명 |
|---|---|
| `apiVersion` | API 버전을 지정합니다. `rbac.authorization.k8s.io/v1`을 사용합니다. |
| `kind` | 리소스 종류를 지정합니다. `RoleBinding`을 사용합니다. |
| `metadata.name` | RoleBinding의 이름을 지정합니다. |
| `metadata.namespace` | RoleBinding이 속하고 권한이 적용될 Namespace를 지정합니다. |
| `metadata.labels` | RoleBinding을 식별하거나 분류하기 위한 Label을 지정합니다. |
| `metadata.annotations` | RoleBinding에 추가적인 메타데이터를 지정합니다. |
| `subjects` | 권한을 부여받을 User, Group 또는 ServiceAccount를 지정합니다. |
| `roleRef` | 권한을 가져올 Role 또는 ClusterRole을 참조합니다. |

## 4. subjects

`subjects`는 Role 또는 ClusterRole에 정의된 권한을 부여받을 대상을 지정하는 필드입니다.

Kubernetes RBAC에서는 다음 세 가지 종류의 주체를 사용할 수 있습니다.

- `User`: 개별 사용자
- `Group`: 사용자 그룹
- `ServiceAccount`: Kubernetes 내부에서 실행되는 애플리케이션이나 프로세스의 신원

하나의 RoleBinding에 여러 개의 Subject를 지정할 수 있습니다.

### 4.1. User

`User`는 Kubernetes API에 접근하는 개별 사용자를 의미합니다.

다음은 `alice` 사용자에게 `pod-reader` Role의 권한을 부여하는 예제입니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-pod-reader
  namespace: backend
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
```

위 RoleBinding은 `alice` 사용자에게 `backend` Namespace의 Pod 조회 권한을 부여합니다.

Kubernetes RBAC은 User를 별도의 Kubernetes 리소스로 관리하지 않습니다.

User의 이름은 인증(Authentication) 과정에서 인증 모듈이 제공하는 문자열을 사용합니다.

따라서 `subjects`의 `name`은 실제 클러스터에서 인증된 사용자 이름과 일치해야 합니다.

### 4.2. Group

`Group`은 여러 사용자를 하나의 그룹으로 묶어 권한을 부여할 때 사용합니다.

예를 들어, `backend-developers` 그룹에 속한 사용자들에게 Pod 조회 권한을 부여하려면 다음과 같이 설정할 수 있습니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-developers-pod-reader
  namespace: backend
subjects:
  - kind: Group
    name: backend-developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
```

위 RoleBinding은 `backend-developers` 그룹에 속한 사용자들에게 `backend` Namespace의 Pod 조회 권한을 부여합니다.

Group 역시 Kubernetes에서 별도의 리소스로 관리하지 않습니다.

사용자가 어떤 그룹에 속하는지는 인증 모듈에서 제공하는 그룹 정보를 기반으로 판단합니다.

### 4.3. ServiceAccount

`ServiceAccount`는 Kubernetes 클러스터 내부에서 실행되는 애플리케이션이나 프로세스에 권한을 부여할 때 사용합니다.

다음은 `backend` Namespace의 `backend-sa` ServiceAccount에 Pod 조회 권한을 부여하는 예제입니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-sa-pod-reader
  namespace: backend
subjects:
  - kind: ServiceAccount
    name: backend-sa
    namespace: backend
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
```

ServiceAccount를 Subject로 지정할 때는 다음 필드를 사용합니다.

| 필드 | 설명 |
|---|---|
| `kind` | `ServiceAccount`를 지정합니다. |
| `name` | ServiceAccount의 이름을 지정합니다. |
| `namespace` | ServiceAccount가 속한 Namespace를 지정합니다. |

User와 Group은 클러스터의 인증 체계에 따라 신원이 결정되지만, ServiceAccount는 Kubernetes 리소스로 관리되므로 Namespace와 이름을 함께 지정합니다.

### 4.4. 여러 Subject 지정

하나의 RoleBinding에는 여러 Subject를 지정할 수 있습니다.

다음은 하나의 RoleBinding으로 User, Group, ServiceAccount에 동일한 Role의 권한을 부여하는 예제입니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: backend
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io

  - kind: Group
    name: backend-developers
    apiGroup: rbac.authorization.k8s.io

  - kind: ServiceAccount
    name: backend-sa
    namespace: backend

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
```

위 RoleBinding은 다음 세 주체에 동일한 권한을 부여합니다.

- `alice` 사용자
- `backend-developers` 그룹
- `backend-sa` ServiceAccount

각 Subject는 동일한 `pod-reader` Role을 참조하므로 동일한 권한 규칙이 적용됩니다.

### 4.5. ServiceAccount Group을 통한 권한 부여

ServiceAccount는 특정 계정뿐만 아니라 Group을 통해서도 권한을 부여할 수 있습니다.

Kubernetes는 ServiceAccount에 대해 다음과 같은 기본 그룹을 제공합니다.

| Group | 설명 |
|---|---|
| `system:serviceaccounts` | 모든 Namespace의 ServiceAccount를 포함합니다. |
| `system:serviceaccounts:<namespace>` | 지정한 Namespace에 속한 모든 ServiceAccount를 포함합니다. |

예를 들어, `backend` Namespace에 속한 모든 ServiceAccount에 Pod 조회 권한을 부여하려면 다음과 같이 설정합니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-serviceaccounts-pod-reader
  namespace: backend
subjects:
  - kind: Group
    name: system:serviceaccounts:backend
    apiGroup: rbac.authorization.k8s.io
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
```

위 설정은 `backend` Namespace에 속한 모든 ServiceAccount에 `pod-reader` Role의 권한을 부여합니다.

특정 애플리케이션에만 권한을 부여하는 것보다 권한 범위가 넓어지므로 실제 사용 목적에 맞게 신중하게 적용해야 합니다.

## 5. roleRef

`roleRef`는 RoleBinding이 참조할 Role 또는 ClusterRole을 지정하는 필드입니다.

RoleBinding은 권한 규칙을 직접 포함하지 않으며, `roleRef`를 통해 기존에 정의된 권한을 참조합니다.

`roleRef`는 다음 세 가지 필드로 구성됩니다.

| 필드 | 설명 |
|---|---|
| `apiGroup` | Role과 ClusterRole이 속한 API Group을 지정합니다. |
| `kind` | 참조할 리소스 종류를 지정합니다. `Role` 또는 `ClusterRole`을 사용합니다. |
| `name` | 참조할 Role 또는 ClusterRole의 이름을 지정합니다. |

예를 들어, `backend` Namespace의 `pod-reader` Role을 참조하려면 다음과 같이 설정합니다.

```yaml
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
```

`roleRef`에서 중요한 점은 참조하는 리소스의 종류와 이름이 실제로 존재하는 리소스와 일치해야 한다는 것입니다.

RoleBinding은 Role 또는 ClusterRole의 권한 규칙을 복사하거나 별도로 저장하지 않습니다. 참조 대상의 권한을 사용합니다.

### 5.1. Role 참조

RoleBinding은 동일한 Namespace에 속한 Role을 참조할 수 있습니다.

다음은 `backend` Namespace의 `pod-reader` Role을 참조하는 예제입니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: backend
subjects:
  - kind: ServiceAccount
    name: backend-sa
    namespace: backend
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
```

이 경우 `backend-sa`는 `backend` Namespace에서 `pod-reader` Role에 정의된 권한을 사용할 수 있습니다.

RoleBinding이 참조하는 Role은 반드시 RoleBinding과 동일한 Namespace에 존재해야 합니다.

### 5.2. ClusterRole 참조

RoleBinding은 Role뿐만 아니라 ClusterRole도 참조할 수 있습니다.

이 경우 ClusterRole에 정의된 권한을 RoleBinding이 생성된 Namespace 범위에서만 적용합니다.

예를 들어, 클러스터에 `pod-reader`라는 ClusterRole이 존재한다고 가정하겠습니다.

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

이 ClusterRole을 `backend` Namespace의 RoleBinding에서 참조하면 다음과 같이 구성할 수 있습니다.

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

위 설정은 `backend-sa`에 `pod-reader` ClusterRole의 권한을 부여하지만, 권한이 적용되는 범위는 `backend` Namespace로 제한됩니다.

즉, ClusterRole을 참조한다고 해서 권한이 클러스터 전체로 확대되는 것은 아닙니다.

여러 Namespace에서 동일한 권한 규칙을 재사용해야 하는 경우 ClusterRole을 정의하고 각 Namespace에서 RoleBinding으로 연결하는 방식이 유용합니다.

## 6. RoleBinding과 ClusterRoleBinding의 차이

RoleBinding과 ClusterRoleBinding은 모두 Role 또는 ClusterRole에 정의된 권한을 특정 Subject에 부여하는 리소스입니다.

가장 큰 차이는 권한이 적용되는 범위입니다.

| 구분 | RoleBinding | ClusterRoleBinding |
|---|---|---|
| API Group | `rbac.authorization.k8s.io` | `rbac.authorization.k8s.io` |
| Resource Kind | `RoleBinding` | `ClusterRoleBinding` |
| Namespace | Namespace 범위 | 클러스터 범위 |
| 참조 대상 | 동일 Namespace의 Role 또는 ClusterRole | ClusterRole |
| 권한 적용 범위 | RoleBinding이 생성된 Namespace | 클러스터 전체 |
| 주요 용도 | 특정 Namespace의 권한 관리 | 클러스터 전체 권한 관리 |

### 6.1. RoleBinding

RoleBinding은 특정 Namespace에 권한을 부여합니다.

참조 대상에 따라 다음과 같이 구성할 수 있습니다.

- 동일 Namespace의 Role을 참조합니다.
- ClusterRole을 참조하여 해당 Namespace에서 권한을 재사용합니다.

예를 들어, `backend` Namespace에서만 Pod 조회 권한을 부여하려면 RoleBinding을 사용합니다.

### 6.2. ClusterRoleBinding

ClusterRoleBinding은 ClusterRole에 정의된 권한을 클러스터 전체 범위에서 부여합니다.

예를 들어, 특정 사용자에게 모든 Namespace의 Pod를 조회할 수 있는 권한을 부여하려면 ClusterRoleBinding을 사용합니다.

ClusterRoleBinding은 Role을 참조할 수 없으며 ClusterRole만 참조할 수 있습니다.

ClusterRoleBinding의 상세한 동작과 설정 방법은 `cluster-role-binding.md`에서 별도로 설명합니다.

## 7. RoleBinding의 Namespace 범위

RoleBinding은 Namespace 범위의 리소스이므로 생성할 때 Namespace를 지정해야 합니다.

RoleBinding이 생성된 Namespace가 권한 적용 범위를 결정합니다.

예를 들어, 동일한 ClusterRole을 서로 다른 Namespace의 RoleBinding에서 참조하면 다음과 같이 구성할 수 있습니다.

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
└── Namespace: monitoring
    └── RoleBinding
        ├── Subject: monitoring-sa
        └── RoleRef: pod-reader
```

위 구조에서는 하나의 ClusterRole을 여러 Namespace에서 재사용합니다.

각 RoleBinding은 자신이 속한 Namespace에 대해서만 권한을 부여합니다.

따라서 다음과 같은 결과가 발생합니다.

| Subject | 권한 적용 범위 |
|---|---|
| `backend-sa` | `backend` Namespace |
| `frontend-sa` | `frontend` Namespace |
| `monitoring-sa` | `monitoring` Namespace |

동일한 ClusterRole을 참조하더라도 각 RoleBinding의 권한 적용 범위는 서로 독립적입니다.

또한 RoleBinding의 Namespace와 Subject의 Namespace는 반드시 같을 필요는 없습니다.

예를 들어, `backend` Namespace의 RoleBinding에서 `monitoring` Namespace에 속한 ServiceAccount를 Subject로 지정할 수 있습니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: monitoring-pod-reader
  namespace: backend
subjects:
  - kind: ServiceAccount
    name: monitoring-sa
    namespace: monitoring
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
```

이 경우 `monitoring` Namespace의 `monitoring-sa` ServiceAccount는 `backend` Namespace의 Pod 조회 권한을 부여받습니다.

즉, Subject가 어느 Namespace에 속하는지와 RoleBinding이 어느 Namespace에 속하는지는 별도로 관리됩니다.

## 8. RoleBinding 변경 및 관리

RoleBinding을 생성한 이후에도 Subject 목록은 변경할 수 있습니다.

다만 `roleRef`는 변경할 수 없는 필드(Immutable Field)입니다.

### 8.1. subjects 변경

기존 RoleBinding에 새로운 Subject를 추가하거나 기존 Subject를 제거할 수 있습니다.

예를 들어, 다음 RoleBinding에 새로운 ServiceAccount를 추가할 수 있습니다.

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

  - kind: ServiceAccount
    name: monitoring-sa
    namespace: monitoring

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
```

위와 같이 변경하면 두 ServiceAccount 모두 `backend` Namespace의 `pod-reader` Role에 정의된 권한을 사용할 수 있습니다.

### 8.2. roleRef 변경 제한

RoleBinding이 생성된 이후에는 참조 대상인 `roleRef`를 변경할 수 없습니다.

예를 들어, 기존 RoleBinding이 `pod-reader` Role을 참조하고 있다면 이를 `deployment-manager` Role로 직접 변경할 수 없습니다.

이러한 제한은 기존 Subject에 의도하지 않은 권한이 부여되는 상황을 방지하기 위한 것입니다.

참조 대상을 변경하려면 기존 RoleBinding을 삭제하고 새로운 RoleBinding을 생성해야 합니다.

```bash
kubectl delete rolebinding backend-pod-reader -n backend
```

이후 변경된 `roleRef`를 포함하는 새로운 RoleBinding을 생성합니다.

```bash
kubectl apply -f role-binding.yaml
```

RoleBinding을 삭제하고 다시 생성하는 과정에서는 기존 권한이 일시적으로 해제될 수 있으므로 운영 환경에서는 변경 시점을 고려해야 합니다.

## 9. RoleBinding 생성 및 확인 명령어

Kubernetes에서는 `kubectl`을 사용하여 RoleBinding을 생성하고 조회할 수 있습니다.

### 9.1. RoleBinding 조회

특정 Namespace의 RoleBinding 목록을 조회합니다.

```bash
kubectl get rolebindings -n backend
```

전체 Namespace의 RoleBinding을 조회하려면 다음 명령어를 사용합니다.

```bash
kubectl get rolebindings -A
```

### 9.2. RoleBinding 상세 정보 확인

특정 RoleBinding의 Subject와 RoleRef를 확인합니다.

```bash
kubectl describe rolebinding backend-pod-reader -n backend
```

### 9.3. RoleBinding 생성

`kubectl create rolebinding` 명령어를 사용하면 Role 또는 ClusterRole을 참조하는 RoleBinding을 생성할 수 있습니다.

다음은 `backend` Namespace의 `backend-sa` ServiceAccount에 `pod-reader` Role의 권한을 부여하는 예제입니다.

```bash
kubectl create rolebinding backend-pod-reader \
  --role=pod-reader \
  --serviceaccount=backend:backend-sa \
  -n backend
```

ClusterRole을 참조하는 RoleBinding을 생성하려면 `--clusterrole` 옵션을 사용할 수 있습니다.

```bash
kubectl create rolebinding backend-pod-reader \
  --clusterrole=pod-reader \
  --serviceaccount=backend:backend-sa \
  -n backend
```

두 명령어의 차이는 참조하는 리소스의 종류입니다.

- `--role`: 동일 Namespace에 존재하는 Role을 참조합니다.
- `--clusterrole`: 클러스터 범위의 ClusterRole을 참조합니다.

두 경우 모두 권한은 `backend` Namespace에만 적용됩니다.

### 9.4. 권한 확인

`kubectl auth can-i` 명령어를 사용하면 특정 주체가 특정 작업을 수행할 수 있는지 확인할 수 있습니다.

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

위 명령어는 `backend-sa` ServiceAccount가 `backend` Namespace에서 Pod 목록을 조회할 수 있는지 확인합니다.

이 명령어를 사용하려면 현재 실행 중인 사용자에게 해당 주체를 가장(Impersonation)할 수 있는 권한이 있어야 합니다.

## 10. 보안 고려사항

RoleBinding은 권한을 실제 주체에 부여하는 리소스이므로 다음과 같은 보안 사항을 고려해야 합니다.

### 10.1. 애플리케이션별 ServiceAccount 사용

가능하면 Namespace의 모든 애플리케이션이 사용하는 `default` ServiceAccount보다는 애플리케이션별 ServiceAccount를 생성하여 사용하는 것이 좋습니다.

특정 애플리케이션에 필요한 권한만 개별적으로 부여하면 인증 정보 유출이나 애플리케이션 침해가 발생했을 때 영향 범위를 줄일 수 있습니다.

### 10.2. 권한 부여 범위 제한

특정 Namespace에서만 필요한 권한이라면 ClusterRoleBinding보다 RoleBinding을 사용하는 것이 적절합니다.

ClusterRole을 참조하는 RoleBinding을 사용하면 공통 권한 정의를 재사용하면서도 Namespace별로 권한 적용 범위를 제한할 수 있습니다.

### 10.3. 불필요한 Subject 공유 방지

하나의 RoleBinding에 여러 Subject를 지정하면 모든 Subject에 동일한 권한이 부여됩니다.

따라서 권한이 필요한 주체만 명시적으로 추가해야 합니다.

특히 Group을 Subject로 지정할 때는 해당 Group에 포함된 모든 사용자가 동일한 권한을 부여받는다는 점에 주의해야 합니다.

### 10.4. default ServiceAccount에 대한 권한 부여 주의

Namespace의 `default` ServiceAccount에 권한을 부여하면 별도의 ServiceAccount를 지정하지 않은 모든 Pod가 해당 권한을 사용할 수 있습니다.

따라서 애플리케이션별로 권한을 분리하고 필요한 ServiceAccount에만 RoleBinding을 적용하는 것이 좋습니다.

### 10.5. roleRef 변경 제한 이해

`roleRef`는 변경할 수 없는 필드이므로 기존 RoleBinding의 권한 참조 대상을 변경하려면 RoleBinding을 삭제하고 다시 생성해야 합니다.

RoleBinding을 수정할 때는 Subject 목록이 의도한 대상과 일치하는지 확인하고, 새로운 권한이 잘못 부여되지 않도록 주의해야 합니다.

### 10.6. 권한 상승 방지

Kubernetes는 RoleBinding 생성 및 수정 과정에서 권한 상승(Privilege Escalation)을 방지하기 위한 검증을 수행합니다.

일반적으로 RoleBinding을 생성하거나 수정하려면 참조하는 Role 또는 ClusterRole에 포함된 권한을 이미 보유하고 있어야 합니다.

또는 해당 Role이나 ClusterRole에 대해 `bind` Verb를 수행할 수 있는 권한이 별도로 부여되어 있어야 합니다.

이러한 제한을 통해 사용자가 자신에게 허용되지 않은 권한을 다른 사용자에게 임의로 부여하는 상황을 방지합니다.

## 11. 정리

RoleBinding은 Kubernetes RBAC에서 Role 또는 ClusterRole에 정의된 권한을 User, Group 또는 ServiceAccount에 부여하는 리소스입니다.

핵심 내용을 정리하면 다음과 같습니다.

- RoleBinding은 Namespace 범위의 Kubernetes 리소스입니다.
- `subjects` 필드를 통해 권한을 부여받을 User, Group 또는 ServiceAccount를 지정합니다.
- `roleRef` 필드를 통해 권한을 정의한 Role 또는 ClusterRole을 참조합니다.
- RoleBinding은 동일 Namespace의 Role 또는 ClusterRole을 참조할 수 있습니다.
- ClusterRole을 참조하더라도 권한은 RoleBinding이 생성된 Namespace에만 적용됩니다.
- Subject의 Namespace와 RoleBinding의 Namespace는 서로 다를 수 있습니다.
- 하나의 RoleBinding에 여러 Subject를 지정할 수 있습니다.
- 생성된 RoleBinding의 `subjects`는 변경할 수 있지만 `roleRef`는 변경할 수 없습니다.
- 참조하는 권한을 변경하려면 기존 RoleBinding을 삭제하고 새로운 RoleBinding을 생성해야 합니다.
- 클러스터 전체에 권한을 부여하려면 ClusterRoleBinding을 사용해야 합니다.

---

## 참고자료

RoleBinding의 동작 방식과 Subject 및 RoleRef의 필드 구성은 Kubernetes 공식 RBAC 문서를 기준으로 정리했습니다.

특히 RoleBinding과 ClusterRoleBinding의 권한 적용 범위, ClusterRole을 RoleBinding에서 참조하는 방식, `roleRef`의 변경 불가 특성 및 권한 상승 방지 정책을 참고할 수 있습니다.

- [Kubernetes Documentation - Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes API Reference - RoleBinding](https://kubernetes.io/docs/reference/kubernetes-api/authorization-resources/role-binding-v1/)
- [Kubernetes Documentation - Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [Kubernetes Documentation - Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
- [Kubernetes Documentation - kubectl create rolebinding](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_rolebinding/)
- [Kubernetes Documentation - kubectl auth can-i](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_auth/kubectl_auth_can-i/)