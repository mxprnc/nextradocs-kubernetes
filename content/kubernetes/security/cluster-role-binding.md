# ClusterRoleBinding

## 1. ClusterRoleBinding이란?

ClusterRoleBinding은 Kubernetes RBAC(Role-Based Access Control)에서 **ClusterRole에 정의된 권한을 사용자, 그룹 또는 ServiceAccount에 클러스터 전체 범위로 부여하는 리소스**입니다.

ClusterRole이 접근 권한을 정의하는 역할을 수행한다면, ClusterRoleBinding은 해당 권한을 누구에게 부여할 것인지 결정합니다.

ClusterRoleBinding의 주요 특징은 다음과 같습니다.

- Namespace에 속하지 않는 클러스터 범위 리소스입니다.
- `subjects` 필드를 통해 권한을 부여할 대상을 지정합니다.
- `roleRef` 필드를 통해 권한을 제공할 ClusterRole을 참조합니다.
- ClusterRole만 참조할 수 있으며 Role은 참조할 수 없습니다.
- 연결된 Subject에 ClusterRole의 권한을 클러스터 전체 범위로 부여합니다.
- `roleRef`는 생성 후 변경할 수 없지만 `subjects`는 수정할 수 있습니다.

예를 들어 `pod-reader`라는 ClusterRole에 Pod 조회 권한이 정의되어 있다면, ClusterRoleBinding을 통해 특정 ServiceAccount에 해당 권한을 부여할 수 있습니다.

이때 ClusterRoleBinding을 사용하면 해당 ServiceAccount는 모든 Namespace의 Pod를 조회할 수 있습니다.

## 2. ClusterRoleBinding의 동작 원리

ClusterRoleBinding은 ClusterRole과 Subject를 연결하여 권한을 부여합니다.

전체적인 구조는 다음과 같습니다.

```text
Kubernetes Cluster
│
├── ClusterRole
│   └── pod-reader
│       └── pods: get, list, watch
│
└── ClusterRoleBinding
    ├── subjects
    │   └── ServiceAccount: monitoring-sa
    │       └── Namespace: monitoring
    │
    └── roleRef
        └── ClusterRole: pod-reader
```

위 구조에서 `monitoring` Namespace에 속한 `monitoring-sa` ServiceAccount는 `pod-reader` ClusterRole에 정의된 권한을 클러스터 전체에서 사용할 수 있습니다.

ClusterRoleBinding은 권한 규칙을 직접 포함하지 않습니다. 대신 ClusterRole을 참조하여 해당 권한을 Subject에 연결합니다.

따라서 권한 규칙을 변경하려면 참조하는 ClusterRole을 수정해야 하며, 권한을 부여받는 대상을 변경하려면 ClusterRoleBinding의 `subjects`를 수정하면 됩니다.

## 3. ClusterRoleBinding 생성 및 설정

ClusterRoleBinding은 `apiVersion: rbac.authorization.k8s.io/v1`, `kind: ClusterRoleBinding`을 사용하여 정의합니다.

다음은 `pod-reader` ClusterRole의 권한을 `monitoring` Namespace에 속한 ServiceAccount에 부여하는 예제입니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-pod-reader
subjects:
  - kind: ServiceAccount
    name: monitoring-sa
    namespace: monitoring
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: pod-reader
```

위 설정은 다음과 같이 동작합니다.

1. `subjects`에서 권한을 부여할 ServiceAccount를 지정합니다.
2. `roleRef`에서 권한을 제공할 ClusterRole을 지정합니다.
3. ClusterRoleBinding을 생성하면 해당 ServiceAccount에 ClusterRole의 권한이 클러스터 전체 범위로 부여됩니다.

ClusterRoleBinding은 Namespace에 속하지 않는 리소스이므로 `metadata.namespace`를 지정하지 않습니다.

생성된 ClusterRoleBinding은 다음 명령어로 조회할 수 있습니다.

```bash
kubectl apply -f cluster-role-binding.yaml
```

```bash
kubectl get clusterrolebindings
```

특정 ClusterRoleBinding의 상세 정보는 다음과 같이 확인합니다.

```bash
kubectl describe clusterrolebinding monitoring-pod-reader
```

### 3.1. 주요 필드

| 필드 | 설명 |
|---|---|
| `apiVersion` | API 버전을 지정합니다. `rbac.authorization.k8s.io/v1`을 사용합니다. |
| `kind` | 리소스 종류를 지정합니다. `ClusterRoleBinding`을 사용합니다. |
| `metadata.name` | ClusterRoleBinding의 이름을 지정합니다. |
| `subjects` | 권한을 부여할 사용자, 그룹 또는 ServiceAccount를 지정합니다. |
| `roleRef` | 권한을 제공할 ClusterRole을 참조합니다. |

ClusterRoleBinding은 `subjects`와 `roleRef`를 통해 권한을 연결하는 구조입니다.

## 4. subjects

`subjects`는 ClusterRole의 권한을 부여받을 대상을 지정하는 필드입니다.

Kubernetes RBAC에서 Subject는 다음 세 가지 종류를 지원합니다.

- `User`: Kubernetes 사용자
- `Group`: Kubernetes 사용자 그룹
- `ServiceAccount`: Kubernetes 내부에서 실행되는 애플리케이션이나 프로세스의 계정

하나의 ClusterRoleBinding에 여러 Subject를 지정할 수도 있습니다.

### 4.1. User

`User`는 Kubernetes API에 인증된 사용자 계정을 의미합니다.

다음은 `admin-user` 사용자에게 ClusterRole의 권한을 부여하는 예제입니다.

```yaml
subjects:
  - kind: User
    name: admin-user
    apiGroup: rbac.authorization.k8s.io
```

Kubernetes RBAC는 사용자 계정을 직접 생성하거나 관리하지 않습니다.

사용자 인증은 X.509 Client Certificate, OIDC, 외부 인증 시스템 등 Kubernetes가 지원하는 인증 방식을 통해 수행하며, 인증 결과로 전달된 사용자 이름을 RBAC의 Subject와 비교합니다.

따라서 `subjects.name`에는 인증 시스템이 Kubernetes API Server에 전달하는 사용자 이름이 지정되어야 합니다.

### 4.2. Group

`Group`은 동일한 권한을 부여할 여러 사용자를 하나의 그룹으로 관리할 때 사용합니다.

다음은 `developers` 그룹에 ClusterRole의 권한을 부여하는 예제입니다.

```yaml
subjects:
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
```

사용자가 인증되면 인증 시스템이 전달한 Group 정보와 ClusterRoleBinding의 Subject를 비교하여 권한을 결정합니다.

하나의 ClusterRoleBinding에 여러 Group을 지정할 수도 있습니다.

```yaml
subjects:
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io

  - kind: Group
    name: operators
    apiGroup: rbac.authorization.k8s.io
```

위 설정은 `developers`와 `operators` 그룹에 동일한 ClusterRole 권한을 부여합니다.

### 4.3. ServiceAccount

`ServiceAccount`는 Kubernetes 클러스터 내부에서 실행되는 애플리케이션이나 프로세스가 Kubernetes API에 접근할 때 사용하는 계정입니다.

다음은 `monitoring` Namespace에 속한 `monitoring-sa` ServiceAccount에 권한을 부여하는 예제입니다.

```yaml
subjects:
  - kind: ServiceAccount
    name: monitoring-sa
    namespace: monitoring
```

ServiceAccount는 Namespace 범위 리소스이므로 `subjects`에 `namespace`를 지정해야 합니다.

ClusterRoleBinding 자체는 클러스터 범위 리소스이지만, 권한을 부여받는 ServiceAccount는 특정 Namespace에 속할 수 있습니다.

예를 들어 다음과 같은 구조가 가능합니다.

```text
Kubernetes Cluster
│
├── Namespace: monitoring
│   └── ServiceAccount: monitoring-sa
│
└── ClusterRoleBinding
    └── monitoring-pod-reader
        ├── Subject:
        │   └── monitoring:monitoring-sa
        │
        └── RoleRef:
            └── ClusterRole: pod-reader
```

위 ServiceAccount는 `monitoring` Namespace에 속하지만, ClusterRoleBinding을 통해 부여받은 권한은 클러스터 전체에 적용됩니다.

### 4.4. 여러 Subject 지정

하나의 ClusterRoleBinding에 서로 다른 종류의 Subject를 함께 지정할 수 있습니다.

```yaml
subjects:
  - kind: User
    name: admin-user
    apiGroup: rbac.authorization.k8s.io

  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io

  - kind: ServiceAccount
    name: monitoring-sa
    namespace: monitoring
```

위 설정은 동일한 ClusterRole의 권한을 세 가지 Subject에 부여합니다.

Subject의 종류에 따라 필요한 필드가 다르므로 정확하게 지정해야 합니다.

| Subject Kind | 필수 식별 정보 | apiGroup |
|---|---|---|
| `User` | `name` | `rbac.authorization.k8s.io` |
| `Group` | `name` | `rbac.authorization.k8s.io` |
| `ServiceAccount` | `name`, `namespace` | 일반적으로 생략 |

## 5. roleRef

`roleRef`는 ClusterRoleBinding이 참조할 권한 정의 리소스를 지정하는 필드입니다.

ClusterRoleBinding은 반드시 ClusterRole을 참조해야 합니다.

```yaml
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: pod-reader
```

각 필드의 의미는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `apiGroup` | 참조 대상의 API Group을 지정합니다. `rbac.authorization.k8s.io`를 사용합니다. |
| `kind` | 참조할 리소스 종류를 지정합니다. 반드시 `ClusterRole`이어야 합니다. |
| `name` | 참조할 ClusterRole의 이름을 지정합니다. |

ClusterRoleBinding은 Role을 참조할 수 없습니다.

Role은 특정 Namespace에 속하는 리소스이기 때문에 클러스터 전체 범위의 권한을 제공하는 ClusterRoleBinding의 참조 대상으로 사용할 수 없습니다.

반면 RoleBinding은 같은 Namespace의 Role 또는 ClusterRole을 참조할 수 있습니다.

### 5.1. roleRef의 Immutable 특성

`roleRef`는 ClusterRoleBinding이 생성된 이후 변경할 수 없는 Immutable 필드입니다.

예를 들어 기존 ClusterRoleBinding이 `pod-reader`를 참조하고 있다면, 이를 `deployment-reader`로 변경하는 작업은 허용되지 않습니다.

```yaml
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: deployment-reader
```

기존 ClusterRoleBinding의 `roleRef`를 변경하려고 하면 Kubernetes API Server에서 검증 오류가 발생합니다.

참조 대상을 변경하려면 다음 순서로 작업해야 합니다.

1. 기존 ClusterRoleBinding을 삭제합니다.
2. 변경할 ClusterRole을 참조하는 새로운 ClusterRoleBinding을 생성합니다.

`roleRef`가 Immutable인 이유는 권한 부여 대상을 관리하는 주체가 기존 Binding의 Subject에 의도하지 않은 권한을 추가로 부여하는 상황을 방지하기 위해서입니다.

예를 들어 특정 ClusterRoleBinding의 `subjects`를 수정할 권한만 가진 사용자에게 `roleRef`까지 변경할 수 있도록 허용하면, 기존 Subject 전체에 더 높은 권한이 부여될 수 있습니다.

따라서 Kubernetes는 `roleRef`를 변경할 수 없도록 제한하고 있습니다.

반면 `subjects`는 수정할 수 있으므로 기존 ClusterRoleBinding에 권한을 부여받는 대상을 추가하거나 제거할 수 있습니다.

## 6. ClusterRoleBinding 예제

### 6.1. 특정 ServiceAccount에 전체 Namespace의 Pod 조회 권한 부여

먼저 Pod 조회 권한을 정의한 ClusterRole을 생성합니다.

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

이후 ClusterRoleBinding을 생성하여 특정 ServiceAccount에 권한을 부여합니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-pod-reader
subjects:
  - kind: ServiceAccount
    name: monitoring-sa
    namespace: monitoring
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: pod-reader
```

위 설정으로 `monitoring` Namespace의 `monitoring-sa` ServiceAccount는 클러스터 전체의 Pod를 조회할 수 있습니다.

### 6.2. 특정 사용자에게 클러스터 관리자 권한 부여

Kubernetes는 기본적으로 `cluster-admin`이라는 ClusterRole을 제공합니다.

해당 ClusterRole은 클러스터의 모든 리소스와 작업에 대한 관리자 권한을 정의합니다.

다음은 `admin-user` 사용자에게 `cluster-admin` 권한을 부여하는 예제입니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user-cluster-admin
subjects:
  - kind: User
    name: admin-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```

위 ClusterRoleBinding이 생성되면 `admin-user`는 클러스터 전체에 대한 관리자 권한을 갖게 됩니다.

`cluster-admin`은 매우 높은 수준의 권한을 제공하므로 필요한 사용자에게만 제한적으로 부여해야 합니다.

### 6.3. 특정 그룹에 전체 Namespace의 리소스 조회 권한 부여

Kubernetes는 기본적으로 `view`라는 ClusterRole을 제공합니다.

`view`는 Namespace 범위의 리소스를 조회할 수 있도록 설계된 읽기 전용 권한입니다.

다음은 `developers` 그룹에 `view` 권한을 클러스터 전체 범위로 부여하는 예제입니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: developers-global-view
subjects:
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
```

위 설정은 `developers` 그룹에 속한 사용자들이 모든 Namespace의 조회 대상 리소스를 확인할 수 있도록 합니다.

단, `view`는 모든 클러스터 리소스를 조회할 수 있는 권한이 아닙니다. Node와 같은 클러스터 범위 리소스에 대한 권한은 별도로 확인해야 합니다.

또한 Secret과 같은 민감한 정보는 기본 `view` 권한에 포함되지 않습니다.

## 7. ClusterRoleBinding과 RoleBinding의 차이

ClusterRoleBinding과 RoleBinding은 모두 Role 또는 ClusterRole에 정의된 권한을 Subject에 부여하는 Binding 리소스입니다.

두 리소스의 가장 중요한 차이는 권한이 적용되는 범위입니다.

| 구분 | RoleBinding | ClusterRoleBinding |
|---|---|---|
| Namespace | Namespace 범위 | 클러스터 범위 |
| 권한 적용 범위 | 특정 Namespace | 클러스터 전체 |
| 참조 대상 | Role 또는 ClusterRole | ClusterRole만 가능 |
| Subject | User, Group, ServiceAccount | User, Group, ServiceAccount |
| `roleRef` | 생성 후 변경 불가 | 생성 후 변경 불가 |
| `subjects` | 수정 가능 | 수정 가능 |

### 7.1. RoleBinding으로 ClusterRole 참조

RoleBinding은 ClusterRole을 참조할 수 있습니다.

이 경우 ClusterRole에 정의된 권한은 RoleBinding이 생성된 Namespace에만 적용됩니다.

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

위 설정은 `backend` Namespace에만 Pod 조회 권한을 부여합니다.

### 7.2. ClusterRoleBinding으로 ClusterRole 참조

동일한 ClusterRole을 ClusterRoleBinding으로 참조하면 권한이 클러스터 전체에 적용됩니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: backend-pod-reader-global
subjects:
  - kind: ServiceAccount
    name: backend-sa
    namespace: backend
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: pod-reader
```

위 설정은 `backend` Namespace의 `backend-sa` ServiceAccount에 모든 Namespace의 Pod 조회 권한을 부여합니다.

### 7.3. 권한 적용 범위 비교

동일한 ClusterRole을 참조하더라도 Binding 종류에 따라 권한 적용 범위가 달라집니다.

```text
ClusterRole: pod-reader
│
└── pods: get, list, watch
    │
    ├── RoleBinding
    │   └── Namespace: backend
    │       └── backend-sa
    │           └── backend Namespace의 Pod 조회
    │
    └── ClusterRoleBinding
        └── monitoring-sa
            └── 모든 Namespace의 Pod 조회
```

따라서 Namespace 범위에서 권한을 제한하려면 RoleBinding을 사용하고, 클러스터 전체에 권한을 부여하려면 ClusterRoleBinding을 사용해야 합니다.

## 8. 권한 상승 방지(Privilege Escalation Prevention)

Kubernetes는 RBAC 리소스의 생성 및 수정 과정에서 권한 상승을 방지하기 위한 검증을 수행합니다.

사용자가 자신이 보유하지 않은 권한을 ClusterRoleBinding을 통해 다른 Subject에게 부여하는 상황을 방지하기 위한 기능입니다.

예를 들어 특정 사용자에게 ClusterRoleBinding 생성 권한만 부여했다고 해서 모든 ClusterRole을 자유롭게 다른 사용자에게 연결할 수 있는 것은 아닙니다.

사용자가 특정 ClusterRole을 Binding에 연결하려면 일반적으로 다음 조건 중 하나를 충족해야 합니다.

1. 참조하려는 ClusterRole에 정의된 모든 권한을 이미 보유하고 있어야 합니다.
2. 해당 ClusterRole에 대한 `bind` Verb 권한을 명시적으로 부여받아야 합니다.

`bind` 권한은 다음과 같이 정의할 수 있습니다.

```yaml
rules:
  - apiGroups:
      - rbac.authorization.k8s.io
    resources:
      - clusterroles
    resourceNames:
      - pod-reader
    verbs:
      - bind
```

위 설정은 `pod-reader` ClusterRole에 대한 `bind` 권한을 부여합니다.

`resourceNames`를 지정하면 해당 ClusterRole에 대해서만 Binding을 생성할 수 있도록 제한할 수 있습니다.

반면 `resourceNames`를 생략하면 해당 API Resource에 대한 `bind` 권한의 대상이 더 넓어지므로 주의해야 합니다.

또한 ClusterRole 자체를 생성하거나 수정하는 작업에도 권한 상승 방지 검증이 적용됩니다.

ClusterRole에 정의된 권한을 생성하거나 수정하려면 해당 권한을 이미 보유하고 있거나 `escalate` Verb 권한을 가지고 있어야 합니다.

이러한 검증은 Kubernetes API Server에서 수행되므로 RBAC Authorizer의 사용 여부와 관계없이 적용됩니다.

## 9. 기본 제공 ClusterRoleBinding

Kubernetes API Server는 클러스터 운영에 필요한 기본 ClusterRoleBinding을 생성하고 관리합니다.

대표적인 기본 ClusterRoleBinding은 다음과 같습니다.

| ClusterRoleBinding | 연결된 Subject | 참조 ClusterRole | 설명 |
|---|---|---|---|
| `cluster-admin` | `system:masters` Group | `cluster-admin` | 클러스터 전체 관리자 권한을 부여합니다. |
| `system:discovery` | `system:authenticated` Group | `system:discovery` | 인증된 사용자에게 API Discovery 정보를 조회할 수 있는 권한을 제공합니다. |
| `system:basic-user` | `system:authenticated` Group | `system:basic-user` | 인증된 사용자에게 기본적인 자기 정보 조회 권한을 제공합니다. |
| `system:public-info-viewer` | `system:authenticated`, `system:unauthenticated` Group | `system:public-info-viewer` | 민감하지 않은 일부 클러스터 정보를 조회할 수 있도록 합니다. |

기본 ClusterRoleBinding은 Kubernetes 클러스터의 정상적인 동작과 API 접근을 위해 사용됩니다.

특히 `system:` 접두사가 붙은 기본 ClusterRoleBinding은 Kubernetes 시스템 구성 요소와 밀접하게 연결되어 있으므로 임의로 수정하거나 삭제하지 않는 것이 좋습니다.

### 9.1. Auto-reconciliation

Kubernetes API Server는 시작 과정에서 기본 ClusterRole과 ClusterRoleBinding의 상태를 자동으로 확인하고 필요한 권한이나 Subject를 복구하는 Auto-reconciliation 기능을 제공합니다.

예를 들어 기본 ClusterRoleBinding에서 Subject를 임의로 제거하더라도 API Server가 다시 시작되면 누락된 Subject가 복구될 수 있습니다.

기본 ClusterRoleBinding의 자동 복구를 비활성화하려면 다음 Annotation을 사용할 수 있습니다.

```yaml
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "false"
```

다만 Auto-reconciliation을 비활성화하면 Kubernetes 기본 권한과 Subject가 최신 버전의 기본 구성에 맞게 갱신되지 않을 수 있으므로 신중하게 사용해야 합니다.

## 10. 권한 확인 및 관리 명령어

### 10.1. ClusterRoleBinding 조회

전체 ClusterRoleBinding 목록을 조회합니다.

```bash
kubectl get clusterrolebindings
```

특정 ClusterRoleBinding의 상세 정보를 확인합니다.

```bash
kubectl describe clusterrolebinding monitoring-pod-reader
```

ClusterRoleBinding의 YAML 정의를 확인하려면 다음 명령어를 사용합니다.

```bash
kubectl get clusterrolebinding monitoring-pod-reader -o yaml
```

### 10.2. ClusterRoleBinding 생성

`kubectl create clusterrolebinding` 명령어를 사용하면 YAML 파일 없이 ClusterRoleBinding을 생성할 수 있습니다.

다음은 `pod-reader` ClusterRole을 `monitoring-sa` ServiceAccount에 연결하는 예제입니다.

```bash
kubectl create clusterrolebinding monitoring-pod-reader \
  --clusterrole=pod-reader \
  --serviceaccount=monitoring:monitoring-sa
```

특정 사용자에게 권한을 부여하려면 다음과 같이 실행합니다.

```bash
kubectl create clusterrolebinding admin-user-binding \
  --clusterrole=cluster-admin \
  --user=admin-user
```

특정 그룹에 권한을 부여하려면 다음과 같이 실행합니다.

```bash
kubectl create clusterrolebinding developers-view \
  --clusterrole=view \
  --group=developers
```

### 10.3. 권한 확인

`kubectl auth can-i` 명령어를 사용하면 특정 Subject가 원하는 작업을 수행할 수 있는지 확인할 수 있습니다.

현재 사용자가 모든 Namespace의 Pod를 조회할 수 있는지 확인하려면 다음과 같이 실행합니다.

```bash
kubectl auth can-i list pods --all-namespaces
```

특정 ServiceAccount가 클러스터 전체의 Pod를 조회할 수 있는지 확인하려면 다음과 같이 실행합니다.

```bash
kubectl auth can-i list pods \
  --all-namespaces \
  --as=system:serviceaccount:monitoring:monitoring-sa
```

특정 Namespace에서 권한을 확인하려면 다음과 같이 실행합니다.

```bash
kubectl auth can-i list pods \
  -n backend \
  --as=system:serviceaccount:monitoring:monitoring-sa
```

`--as` 옵션은 Impersonation을 사용하므로 현재 사용자에게 해당 ServiceAccount를 가장할 수 있는 권한이 있어야 합니다.

### 10.4. ClusterRoleBinding 삭제

ClusterRoleBinding을 삭제하려면 다음 명령어를 사용합니다.

```bash
kubectl delete clusterrolebinding monitoring-pod-reader
```

ClusterRoleBinding을 삭제하면 해당 Binding을 통해 부여된 권한은 더 이상 적용되지 않습니다.

다만 참조 대상인 ClusterRole 자체는 삭제되지 않으며, 다른 Binding에서 동일한 ClusterRole을 사용하고 있다면 해당 권한에는 영향을 주지 않습니다.

## 11. 보안 고려사항

ClusterRoleBinding은 Subject에 클러스터 전체 범위의 권한을 부여하므로 권한 설정에 각별한 주의가 필요합니다.

### 11.1. 최소 권한 원칙

Subject에 필요한 최소한의 권한만 부여해야 합니다.

특정 Namespace의 Pod 조회만 필요한 경우 ClusterRoleBinding을 사용하기보다는 ClusterRole을 참조하는 RoleBinding을 사용하는 것이 적절합니다.

불필요하게 클러스터 전체 범위의 권한을 부여하면 보안 위험이 증가할 수 있습니다.

### 11.2. cluster-admin 권한 제한

`cluster-admin`은 클러스터 전체의 리소스와 작업을 관리할 수 있는 최고 수준의 권한입니다.

해당 권한을 가진 Subject는 Secret 조회, RBAC 설정 변경, 워크로드 생성 및 삭제 등 클러스터 운영에 영향을 미치는 작업을 수행할 수 있습니다.

따라서 `cluster-admin`을 ClusterRoleBinding으로 부여할 때는 반드시 대상과 목적을 확인해야 합니다.

### 11.3. ServiceAccount의 Namespace 확인

ServiceAccount는 Namespace 범위 리소스이므로 동일한 이름이라도 Namespace가 다르면 서로 다른 계정입니다.

예를 들어 다음 두 ServiceAccount는 서로 다른 Subject입니다.

- `monitoring:monitoring-sa`
- `backend:monitoring-sa`

따라서 ClusterRoleBinding을 생성할 때는 `subjects.namespace`를 정확하게 지정해야 합니다.

### 11.4. Secret 조회 권한 주의

Secret 조회 권한은 인증 정보와 같은 민감한 데이터에 접근할 수 있도록 합니다.

ClusterRoleBinding을 통해 Secret 조회 권한을 부여하면 해당 Subject가 클러스터 전체 Namespace의 Secret에 접근할 수 있으므로 반드시 필요한 경우에만 허용해야 합니다.

### 11.5. 기본 ClusterRoleBinding 수정 주의

`system:` 접두사가 붙은 기본 ClusterRoleBinding은 Kubernetes 내부 구성 요소의 정상적인 동작에 영향을 미칠 수 있습니다.

또한 기본 ClusterRoleBinding은 Auto-reconciliation에 의해 변경 사항이 복구될 수 있으므로, 기본 리소스를 직접 수정하기보다는 별도의 ClusterRole과 ClusterRoleBinding을 생성하여 권한을 관리하는 것이 좋습니다.

## 12. 정리

ClusterRoleBinding은 ClusterRole에 정의된 권한을 사용자, 그룹 또는 ServiceAccount에 클러스터 전체 범위로 부여하는 리소스입니다.

핵심 내용을 정리하면 다음과 같습니다.

- ClusterRoleBinding은 Namespace에 속하지 않는 클러스터 범위 리소스입니다.
- `subjects`를 통해 권한을 부여할 User, Group, ServiceAccount를 지정합니다.
- `roleRef`를 통해 권한을 제공할 ClusterRole을 참조합니다.
- ClusterRoleBinding은 Role을 참조할 수 없으며 ClusterRole만 참조할 수 있습니다.
- `roleRef`는 생성 후 변경할 수 없으며, 참조 대상을 변경하려면 기존 Binding을 삭제하고 새로 생성해야 합니다.
- `subjects`는 수정할 수 있습니다.
- ClusterRoleBinding을 사용하면 Namespace 범위 리소스에 대한 권한도 모든 Namespace에 적용할 수 있습니다.
- RoleBinding에서 동일한 ClusterRole을 참조하면 해당 RoleBinding이 생성된 Namespace에만 권한이 적용됩니다.
- Kubernetes는 기본 ClusterRoleBinding과 Auto-reconciliation 기능을 통해 시스템 권한을 관리합니다.
- 클러스터 전체에 권한을 부여하는 리소스이므로 최소 권한 원칙을 준수해야 합니다.

---

## 참고자료

ClusterRoleBinding의 리소스 구조와 Subject 및 RoleRef 설정, 권한 적용 범위, 권한 상승 방지 및 기본 RBAC 리소스의 동작 방식은 Kubernetes 공식 문서를 기준으로 정리했습니다.

- [Kubernetes Documentation - Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes API Reference - ClusterRoleBinding](https://kubernetes.io/docs/reference/kubernetes-api/authorization-resources/cluster-role-binding-v1/)
- [Kubernetes API Reference - RBAC](https://kubernetes.io/docs/reference/kubernetes-api/rbac/)
- [Kubernetes Documentation - Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [Kubernetes Documentation - kubectl create clusterrolebinding](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_clusterrolebinding/)
- [Kubernetes Documentation - kubectl auth can-i](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_auth/kubectl_auth_can-i/)