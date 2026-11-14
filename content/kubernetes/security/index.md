# Kubernetes Security Resources

Security 리소스는 Kubernetes API에 접근하는 사용자 및 애플리케이션의 식별과 권한을 관리하기 위한 리소스입니다.

Kubernetes는 RBAC(Role-Based Access Control)를 통해 특정 주체(Subject)가 어떤 리소스에 대해 어떤 작업을 수행할 수 있는지 제어합니다.

## 1. Security 리소스 구성

| 리소스 | 주요 역할 | 주요 특징 |
|---|---|---|
| **ServiceAccount** | Pod 및 애플리케이션의 인증 주체 | Kubernetes API 접근 시 사용하는 계정 |
| **Role** | Namespace 범위의 권한 정의 | 특정 Namespace 내 리소스 접근 권한 |
| **RoleBinding** | Role 권한 연결 | Role을 사용자, 그룹 또는 ServiceAccount에 연결 |
| **ClusterRole** | 클러스터 범위의 권한 정의 | 클러스터 전체 또는 Namespace 범위에서 재사용 가능 |
| **ClusterRoleBinding** | ClusterRole 권한 연결 | ClusterRole을 클러스터 범위로 연결 |

## 2. RBAC 리소스 관계

RBAC은 크게 권한을 정의하는 리소스와 권한을 연결하는 리소스로 구분됩니다.

### 2.1 Role과 RoleBinding

```text
Subject
(User / Group / ServiceAccount)
          │
          ▼
     RoleBinding
          │
          ▼
        Role
          │
          ▼
   Kubernetes Resources
   (Pod, Service, ConfigMap ...)
```

Role은 특정 Namespace에서 허용할 작업을 정의합니다.

RoleBinding은 해당 Role의 권한을 사용자, 그룹 또는 ServiceAccount에 연결합니다.

RoleBinding은 자신이 생성된 Namespace 범위에서 권한을 부여합니다.

### 2.2 ClusterRole과 ClusterRoleBinding

```text
Subject
(User / Group / ServiceAccount)
          │
          ▼
 ClusterRoleBinding
          │
          ▼
     ClusterRole
          │
          ▼
 Cluster-wide Resources
 (Node, Namespace 등)
```

ClusterRole은 클러스터 범위의 리소스에 대한 권한을 정의하거나, Namespace 범위의 리소스에 재사용할 권한을 정의할 수 있습니다.

ClusterRoleBinding은 ClusterRole을 Subject에 연결하여 클러스터 전체 범위에 권한을 부여합니다.

### 2.3 ClusterRole과 RoleBinding의 조합

ClusterRole은 RoleBinding과도 연결할 수 있습니다.

```text
Subject
   │
   ▼
RoleBinding
   │
   ▼
ClusterRole
   │
   ▼
특정 Namespace 내 리소스
```

이 경우 ClusterRole에 정의된 권한이 RoleBinding이 생성된 특정 Namespace 범위에서 적용됩니다.

여러 Namespace에서 동일한 권한 정의를 재사용할 때 유용합니다.

## 3. Role과 ClusterRole 비교

| 구분 | Role | ClusterRole |
|---|---|---|
| 범위 | 특정 Namespace | 클러스터 범위 또는 Namespace 리소스에 대한 공통 권한 정의 |
| 대상 리소스 | Namespace 범위 리소스 | Namespace 범위 및 클러스터 범위 리소스 |
| 권한 연결 | RoleBinding | RoleBinding 또는 ClusterRoleBinding |
| 주요 사용 사례 | 특정 Namespace의 Pod 조회 | Node 조회, 여러 Namespace에서 재사용할 권한 정의 |

## 4. RoleBinding과 ClusterRoleBinding 비교

| 구분 | RoleBinding | ClusterRoleBinding |
|---|---|---|
| 연결 대상 | Role 또는 ClusterRole | ClusterRole |
| 권한 적용 범위 | 특정 Namespace | 클러스터 전체 |
| 주요 사용 사례 | 특정 Namespace의 애플리케이션 관리 권한 | 클러스터 전체 조회 권한 |

## 5. ServiceAccount와 Pod의 관계

ServiceAccount는 Pod 또는 애플리케이션이 Kubernetes API에 접근할 때 사용하는 인증 주체입니다.

Pod의 `serviceAccountName` 필드에 사용할 ServiceAccount를 지정할 수 있습니다.

```text
ServiceAccount
       │
       ▼
      Pod
       │
       ▼
  Container
       │
       ▼
 Kubernetes API
       │
       ▼
 RBAC Authorization
       │
       ▼
 RoleBinding / ClusterRoleBinding
       │
       ▼
 Role / ClusterRole
```

ServiceAccount는 인증 주체를 제공하며, 실제 API 접근 권한은 Role 및 ClusterRole과 해당 Binding을 통해 결정됩니다.

## 6. 주요 개념 구분

Kubernetes의 접근 제어는 다음과 같은 단계로 이해할 수 있습니다.

| 단계 | 관련 리소스 | 역할 |
|---|---|---|
| 인증 주체 | ServiceAccount | API 요청을 수행하는 애플리케이션의 식별 주체 |
| 권한 정의 | Role, ClusterRole | 허용할 리소스와 작업 정의 |
| 권한 연결 | RoleBinding, ClusterRoleBinding | 권한을 사용자, 그룹 또는 ServiceAccount에 연결 |

RBAC에서 권한을 정의할 때는 다음과 같은 항목을 사용합니다.

- apiGroups: 대상 리소스가 속한 API 그룹
- resources: 접근할 Kubernetes 리소스
- verbs: 허용할 작업 (`get`, `list`, `watch`, `create`, `update`, `patch`, `delete` 등)
- resourceNames: 특정 리소스 이름으로 권한 제한

## 7. 핵심 정리

- ServiceAccount는 Pod 및 애플리케이션이 Kubernetes API에 접근할 때 사용하는 인증 주체입니다.
- Role은 특정 Namespace 범위의 권한을 정의합니다.
- RoleBinding은 Role 또는 ClusterRole의 권한을 특정 Namespace 범위에 연결합니다.
- ClusterRole은 클러스터 범위의 리소스에 대한 권한을 정의하거나 공통 권한을 재사용할 때 사용합니다.
- ClusterRoleBinding은 ClusterRole의 권한을 클러스터 전체 범위로 연결합니다.
- RBAC은 권한을 정의하는 리소스와 권한을 연결하는 리소스를 분리하여 접근 제어를 관리합니다.

참고:
- https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- https://kubernetes.io/docs/concepts/security/service-accounts/