기존 문서의 내용은 유지하고, 마지막에 있던 별도의 작성 설명을 문서의 참고자료 및 버전 관련 설명과 자연스럽게 통합한 전체 버전입니다.

# ServiceAccount

## 1. ServiceAccount란?

ServiceAccount는 Kubernetes 클러스터 내부에서 실행되는 애플리케이션이나 프로세스에 **신원(Identity)을 부여하기 위한 리소스**입니다.

Kubernetes에서는 사용자(User)와 서비스 계정(ServiceAccount)을 구분합니다.

- **User:** 클러스터를 관리하거나 리소스를 조작하는 사람 또는 외부 시스템의 사용자 신원입니다.
- **ServiceAccount:** Pod 내부에서 실행되는 애플리케이션이나 프로세스에 부여하는 신원입니다.

예를 들어, Kubernetes 클러스터 내부에서 실행되는 애플리케이션이 Kubernetes API Server에 접근하여 Pod 목록을 조회하거나 ConfigMap을 읽어야 하는 경우 ServiceAccount를 사용할 수 있습니다.

ServiceAccount는 애플리케이션이 Kubernetes API Server에 자신을 인증할 수 있도록 자격 증명(Credential)을 제공합니다.

다만 ServiceAccount 자체가 Kubernetes 리소스에 대한 접근 권한을 정의하는 것은 아닙니다. 실제 접근 권한은 RBAC(Role-Based Access Control)와 같은 권한 부여(Authorization) 메커니즘을 통해 별도로 설정해야 합니다.

ServiceAccount는 Kubernetes의 Namespace 범위에 속하는 리소스입니다. 따라서 동일한 이름의 ServiceAccount를 서로 다른 Namespace에 각각 생성할 수 있습니다.

## 2. ServiceAccount의 역할

ServiceAccount는 크게 다음과 같은 역할을 수행합니다.

### 2.1. Pod의 신원 제공

Pod에 ServiceAccount를 연결하면 해당 Pod에서 실행되는 컨테이너는 연결된 ServiceAccount의 신원을 사용하여 Kubernetes API Server에 인증할 수 있습니다.

예를 들어, `backend` Namespace에 있는 `backend-sa` ServiceAccount를 Pod에 연결하면 해당 Pod는 `backend-sa`의 신원으로 Kubernetes API Server에 접근할 수 있습니다.

### 2.2. API Server 인증(Authentication)

ServiceAccount는 Kubernetes API Server에 접근하기 위한 인증 수단으로 ServiceAccount Token을 사용합니다.

ServiceAccount Token은 일반적으로 JWT(JSON Web Token) 형식의 Bearer Token입니다.

Pod 내부의 애플리케이션은 해당 토큰을 HTTP 요청의 Authorization 헤더에 포함하여 Kubernetes API Server에 전달합니다.

```
GET /api/v1/namespaces/default/pods HTTP/1.1
Host: kubernetes.default.svc
Authorization: Bearer <service-account-token>
```

API Server는 전달받은 토큰의 서명, 유효 기간, 대상(Audience) 등의 정보를 검증하여 요청자의 신원을 확인합니다.

ServiceAccount의 인증된 사용자 이름은 다음과 같은 형식을 사용합니다.

```
system:serviceaccount:<namespace>:<serviceaccount-name>
```

예를 들어, `backend` Namespace의 `backend-sa` ServiceAccount는 다음과 같은 사용자 이름으로 식별됩니다.

```
system:serviceaccount:backend:backend-sa
```

또한 ServiceAccount는 다음과 같은 그룹에 속합니다.

```
system:serviceaccounts
system:serviceaccounts:<namespace>
```

이를 통해 RBAC 정책에서 특정 ServiceAccount뿐만 아니라 Namespace에 속한 ServiceAccount 전체를 대상으로 권한을 부여할 수도 있습니다.

### 2.3. RBAC과 연계한 권한 관리

ServiceAccount는 RBAC과 연계하여 Kubernetes API에 대한 접근 권한을 관리할 수 있습니다.

예를 들어, 특정 애플리케이션이 자신의 Namespace에 있는 Pod 목록만 조회해야 한다면 다음과 같이 구성할 수 있습니다.

1. ServiceAccount를 생성합니다.
2. Pod 조회 권한을 정의하는 Role을 생성합니다.
3. RoleBinding을 통해 Role과 ServiceAccount를 연결합니다.
4. 해당 ServiceAccount를 Pod에 할당합니다.

이렇게 구성하면 Pod에서 실행되는 애플리케이션은 연결된 ServiceAccount의 신원을 사용하고, RoleBinding을 통해 부여받은 권한 범위 내에서 Kubernetes API를 호출할 수 있습니다.

ServiceAccount는 신원을 제공하고, Role과 RoleBinding은 해당 신원에 부여할 권한을 정의한다는 점을 구분해야 합니다.

## 3. 기본 ServiceAccount

Kubernetes는 Namespace가 생성되면 해당 Namespace에 `default`라는 이름의 ServiceAccount를 자동으로 생성합니다.

별도의 ServiceAccount를 지정하지 않은 Pod는 자신이 속한 Namespace의 `default` ServiceAccount를 사용합니다.

예를 들어, `default` Namespace에서 다음과 같이 Pod를 생성하면 별도의 ServiceAccount 지정 없이 `default` ServiceAccount가 사용됩니다.

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
spec:
  containers:
    - name: nginx
      image: nginx:latest
```

위 Pod는 다음과 같이 동작합니다.

- `default` Namespace에 속합니다.
- `spec.serviceAccountName`이 지정되지 않았으므로 `default` ServiceAccount를 사용합니다.
- 별도의 설정이 없다면 ServiceAccount Token이 Pod에 자동으로 마운트됩니다.

기본 ServiceAccount에는 일반적으로 애플리케이션이 Kubernetes API를 관리할 수 있는 별도의 권한이 부여되지 않습니다. 따라서 `default` ServiceAccount를 사용한다고 해서 모든 Kubernetes 리소스에 접근할 수 있는 것은 아닙니다.

애플리케이션별로 필요한 권한을 분리하고 관리하려면 목적에 맞는 ServiceAccount를 별도로 생성하여 사용하는 것이 좋습니다.

## 4. ServiceAccount 생성 및 설정

ServiceAccount는 `apiVersion: v1`, `kind: ServiceAccount`를 사용하여 정의합니다.

다음은 `backend` Namespace에 `backend-sa` ServiceAccount를 생성하는 예제입니다.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend-sa
  namespace: backend
```

위 리소스를 생성하면 `backend` Namespace에서 사용할 수 있는 `backend-sa` ServiceAccount가 생성됩니다.

```
kubectl apply -f service-account.yaml
```

생성된 ServiceAccount는 다음 명령어로 확인할 수 있습니다.

```
kubectl get serviceaccounts -n backend
```

특정 ServiceAccount의 상세 정보는 다음과 같이 조회합니다.

```
kubectl describe serviceaccount backend-sa -n backend
```

### 4.1. 주요 필드

| 필드                             | 설명                                                                   |
| ------------------------------ | -------------------------------------------------------------------- |
| `apiVersion`                   | API 버전을 지정합니다. ServiceAccount는 `v1`을 사용합니다.                          |
| `kind`                         | 리소스 종류를 지정합니다. `ServiceAccount`를 사용합니다.                              |
| `metadata.name`                | ServiceAccount의 이름을 지정합니다.                                           |
| `metadata.namespace`           | ServiceAccount가 속할 Namespace를 지정합니다.                                 |
| `metadata.labels`              | ServiceAccount를 식별하거나 분류하기 위한 Label을 지정합니다.                          |
| `metadata.annotations`         | ServiceAccount에 추가적인 메타데이터를 지정합니다.                                   |
| `automountServiceAccountToken` | 해당 ServiceAccount를 사용하는 Pod에 ServiceAccount Token을 자동으로 마운트할지 지정합니다. |
| `imagePullSecrets`             | 컨테이너 이미지 레지스트리에 인증하기 위해 사용할 Secret을 지정합니다.                           |

### 4.2. imagePullSecrets

`imagePullSecrets`는 비공개 컨테이너 이미지 레지스트리에 접근하기 위한 인증 정보를 지정하는 필드입니다.

예를 들어, 비공개 레지스트리에서 이미지를 가져와야 하는 경우 다음과 같이 설정할 수 있습니다.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend-sa
  namespace: backend
imagePullSecrets:
  - name: registry-secret
```

위 설정은 `backend-sa`를 사용하는 Pod가 컨테이너 이미지를 가져올 때 `registry-secret` Secret을 사용하도록 합니다.

Pod에 별도의 `imagePullSecrets`가 지정되어 있지 않으면 ServiceAccount에 설정된 `imagePullSecrets`가 Pod에 추가됩니다.

`imagePullSecrets`는 컨테이너 이미지 다운로드를 위한 인증 설정이며, Kubernetes API Server에 접근하기 위한 ServiceAccount Token과는 목적이 다릅니다.

## 5. Pod에 ServiceAccount 연결

ServiceAccount를 생성하는 것만으로 모든 Pod에 해당 ServiceAccount가 자동으로 적용되는 것은 아닙니다.

Pod의 `spec.serviceAccountName` 필드에 사용할 ServiceAccount의 이름을 지정해야 합니다.

다음은 `backend` Namespace의 `backend-sa` ServiceAccount를 사용하는 Pod 예제입니다.

```
apiVersion: v1
kind: Pod
metadata:
  name: backend-pod
  namespace: backend
spec:
  serviceAccountName: backend-sa
  containers:
    - name: backend
      image: nginx:latest
```

위 Pod는 `backend-sa` ServiceAccount를 사용합니다.

Pod가 생성될 때 Kubernetes는 지정된 ServiceAccount가 존재하는지 확인합니다. 해당 ServiceAccount가 존재하지 않으면 Pod 생성이 거부됩니다.

Deployment와 같은 상위 Workload 리소스에서도 동일한 방식으로 지정할 수 있습니다.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      serviceAccountName: backend-sa
      containers:
        - name: backend
          image: nginx:latest
```

이 경우 Deployment가 생성하는 모든 Pod는 `backend-sa` ServiceAccount를 사용합니다.

Pod의 ServiceAccount는 Pod 생성 시 결정되므로 기존 Pod의 ServiceAccount를 변경하려면 일반적으로 새로운 Pod를 생성해야 합니다.

## 6. ServiceAccount Token

ServiceAccount Token은 Kubernetes API Server에 접근할 때 ServiceAccount의 신원을 증명하는 인증 자격 증명입니다.

Kubernetes는 Pod에 ServiceAccount가 연결되면 기본적으로 해당 ServiceAccount의 인증 정보를 Pod에 제공하도록 구성되어 있습니다.

### 6.1. Projected ServiceAccount Token

Kubernetes v1.22부터 Pod에 제공되는 기본 ServiceAccount Token은 TokenRequest API를 통해 발급되는 단기 토큰(Short-lived Token) 방식으로 변경되었습니다.

이 토큰은 Projected Volume을 통해 Pod 내부에 마운트됩니다.

일반적으로 다음과 같은 경로에서 토큰을 확인할 수 있습니다.

```
/var/run/secrets/kubernetes.io/serviceaccount/token
```

Kubernetes는 다음과 같은 정보를 하나의 Projected Volume으로 제공할 수 있습니다.

| 파일          | 설명                                                      |
| ----------- | ------------------------------------------------------- |
| `token`     | Kubernetes API Server 인증에 사용하는 ServiceAccount Token입니다. |
| `ca.crt`    | Kubernetes API Server와 TLS 연결을 수행할 때 사용할 CA 인증서입니다.     |
| `namespace` | Pod가 속한 Namespace의 이름입니다.                               |

이러한 정보는 컨테이너 내부의 애플리케이션이 Kubernetes API Server에 접근하는 데 사용됩니다.

### 6.2. TokenRequest API

TokenRequest API는 ServiceAccount를 위한 단기 인증 토큰을 요청하는 API입니다.

Kubernetes는 TokenRequest API를 사용하여 만료 시간이 지정된 ServiceAccount Token을 발급합니다.

Pod에 기본적으로 제공되는 토큰 역시 kubelet이 TokenRequest API를 통해 요청하고 관리합니다.

필요한 경우 다음과 같이 `kubectl create token` 명령어를 사용하여 특정 ServiceAccount의 토큰을 직접 요청할 수 있습니다.

```
kubectl create token backend-sa -n backend
```

토큰의 유효 기간을 요청하려면 `--duration` 옵션을 사용할 수 있습니다.

```
kubectl create token backend-sa \
  -n backend \
  --duration=30m
```

`--duration`은 요청하는 토큰의 유효 기간이며, 실제 발급되는 토큰의 유효 기간은 API Server 설정과 정책에 따라 달라질 수 있습니다.

### 6.3. Token Rotation

Projected ServiceAccount Token은 영구적으로 유지되는 인증 정보가 아닙니다.

토큰이 만료되기 전에 kubelet이 새로운 토큰을 요청하여 기존 토큰을 교체합니다.

이러한 과정을 Token Rotation이라고 합니다.

기본적으로 TokenRequest API를 통해 발급되는 토큰의 유효 기간은 1시간이며, kubelet은 토큰이 만료되기 전에 자동으로 갱신합니다.

따라서 애플리케이션은 토큰이 한 번 마운트되었다고 가정하고 해당 파일의 내용을 메모리에 영구적으로 보관해서는 안 됩니다. 토큰이 갱신될 수 있으므로 필요한 시점에 파일에서 최신 토큰을 읽도록 구현하는 것이 좋습니다.

### 6.4. Token Projection 직접 설정

기본 ServiceAccount Token 외에 특정 API나 외부 서비스에 사용할 토큰이 필요한 경우 Projected Volume의 `serviceAccountToken`을 직접 설정할 수 있습니다.

다음은 `audience`와 `expirationSeconds`를 지정하는 예제입니다.

```
apiVersion: v1
kind: Pod
metadata:
  name: token-projection-pod
  namespace: backend
spec:
  serviceAccountName: backend-sa
  containers:
    - name: backend
      image: nginx:latest
      volumeMounts:
        - name: custom-token
          mountPath: /var/run/secrets/custom
          readOnly: true
  volumes:
    - name: custom-token
      projected:
        sources:
          - serviceAccountToken:
              audience: my-external-service
              expirationSeconds: 3600
              path: token
```

주요 설정은 다음과 같습니다.

| 필드                  | 설명                                                                                          |
| ------------------- | ------------------------------------------------------------------------------------------- |
| `audience`          | 토큰을 사용할 대상(Audience)을 지정합니다. 토큰을 검증하는 서비스는 자신이 허용하는 Audience와 토큰의 Audience가 일치하는지 확인해야 합니다. |
| `expirationSeconds` | 토큰의 요청 유효 기간을 초 단위로 지정합니다. 기본값은 3600초이며 최소값은 600초입니다.                                       |
| `path`              | Projected Volume 내부에서 토큰이 저장될 파일 경로를 지정합니다.                                                 |

`audience`를 지정하면 해당 토큰을 사용할 수 있는 대상을 제한할 수 있습니다. 따라서 Kubernetes API Server가 아닌 외부 서비스에 인증해야 하는 경우에도 활용할 수 있습니다.

## 7. ServiceAccount Token 자동 마운트 제어

Pod에 ServiceAccount가 연결되어 있더라도 애플리케이션이 Kubernetes API Server에 접근하지 않는다면 인증 토큰이 필요하지 않을 수 있습니다.

이러한 경우 `automountServiceAccountToken` 필드를 사용하여 토큰의 자동 마운트를 비활성화할 수 있습니다.

### 7.1. Pod 단위 설정

Pod의 `spec.automountServiceAccountToken`을 `false`로 지정하면 해당 Pod에 ServiceAccount Token이 자동으로 마운트되지 않습니다.

```
apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
spec:
  automountServiceAccountToken: false
  containers:
    - name: frontend
      image: nginx:latest
```

이 설정은 해당 Pod에만 적용됩니다.

### 7.2. ServiceAccount 단위 설정

ServiceAccount의 `automountServiceAccountToken`을 `false`로 지정하면 해당 ServiceAccount를 사용하는 Pod에 인증 토큰이 자동으로 마운트되지 않도록 설정할 수 있습니다.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: frontend-sa
  namespace: frontend
automountServiceAccountToken: false
```

해당 ServiceAccount를 사용하는 Pod는 기본적으로 ServiceAccount Token을 자동 마운트하지 않습니다.

### 7.3. Pod와 ServiceAccount 설정 우선순위

Pod와 ServiceAccount 모두 `automountServiceAccountToken` 필드를 제공하지만, 두 설정이 서로 다른 경우 Pod의 설정이 우선합니다.

| Pod 설정  | ServiceAccount 설정 | 결과           |
| ------- | ----------------- | ------------ |
| `true`  | `true`            | 자동 마운트       |
| `true`  | `false`           | 자동 마운트       |
| `false` | `true`            | 자동 마운트하지 않음  |
| `false` | `false`           | 자동 마운트하지 않음  |
| 미설정     | `true`            | 자동 마운트       |
| 미설정     | `false`           | 자동 마운트하지 않음  |
| 미설정     | 미설정               | 기본적으로 자동 마운트 |

자동 마운트를 비활성화하더라도 ServiceAccount 자체가 Pod에서 제거되는 것은 아닙니다.

또한 `automountServiceAccountToken: false`는 기본적으로 주입되는 인증 토큰을 비활성화하는 설정입니다. 별도로 정의한 Projected Volume이나 다른 인증 수단까지 차단하는 설정은 아닙니다.

## 8. Legacy ServiceAccount Token

Kubernetes v1.24 이전에는 ServiceAccount가 생성될 때 해당 ServiceAccount에 연결되는 Token Secret이 자동으로 생성되는 방식이 사용되었습니다.

이 방식에서는 `kubernetes.io/service-account-token` 타입의 Secret에 토큰이 저장되며, 해당 Secret을 Pod에 마운트하여 인증에 사용했습니다.

그러나 Secret 기반 토큰은 기본적으로 만료되지 않고 자동으로 갱신되지 않는 장기 인증 정보(Long-lived Credential)라는 문제가 있습니다.

토큰이 외부로 유출되면 해당 토큰을 무효화하기 전까지 악용될 가능성이 있으므로 보안상 주의가 필요합니다.

Kubernetes v1.24부터는 ServiceAccount를 생성해도 장기 토큰 Secret이 자동으로 생성되지 않습니다.

현재는 TokenRequest API와 Projected Volume을 사용하는 단기 토큰 방식이 권장됩니다.

다만 기존 시스템과의 호환성 등을 위해 장기 토큰이 반드시 필요한 경우에는 다음과 같이 Secret을 직접 생성할 수 있습니다.

```
apiVersion: v1
kind: Secret
metadata:
  name: backend-sa-token
  namespace: backend
  annotations:
    kubernetes.io/service-account.name: backend-sa
type: kubernetes.io/service-account-token
```

위 Secret을 생성하면 Kubernetes의 ServiceAccount Token Controller가 해당 ServiceAccount를 확인하고 Secret에 토큰을 채웁니다.

이 방식은 장기 토큰을 생성하는 방법이므로 특별한 이유가 없는 한 사용하지 않는 것이 좋습니다.

## 9. ServiceAccount와 RBAC의 관계

ServiceAccount와 RBAC은 Kubernetes의 인증(Authentication)과 권한 부여(Authorization)를 구성하는 서로 다른 요소입니다.

두 요소의 역할은 다음과 같이 구분됩니다.

| 구분                 | 역할                                                                      |
| ------------------ | ----------------------------------------------------------------------- |
| ServiceAccount     | Pod 또는 애플리케이션의 신원을 제공합니다.                                               |
| Role               | 특정 Namespace에서 허용할 리소스와 작업을 정의합니다.                                      |
| RoleBinding        | Role에 정의된 권한을 User, Group 또는 ServiceAccount에 부여합니다.                     |
| ClusterRole        | 클러스터 범위 또는 여러 Namespace에서 사용할 수 있는 권한을 정의합니다.                           |
| ClusterRoleBinding | ClusterRole에 정의된 권한을 클러스터 전체 범위에서 User, Group 또는 ServiceAccount에 부여합니다. |

예를 들어, 특정 Pod가 자신의 Namespace에 있는 Pod 목록을 조회해야 한다면 다음과 같은 구조로 구성할 수 있습니다.

```
Pod
 └── ServiceAccount
      └── RoleBinding
           └── Role
                └── Pods: list, get, watch
```

이 구조에서 각 리소스의 역할은 다음과 같습니다.

1. Pod는 `spec.serviceAccountName`을 통해 ServiceAccount를 사용합니다.
2. ServiceAccount는 API Server에 접근할 때 자신의 신원을 증명합니다.
3. RoleBinding은 ServiceAccount에 Role을 연결합니다.
4. Role은 허용된 리소스와 작업을 정의합니다.
5. API Server는 인증된 신원과 RBAC 권한을 확인하여 요청의 허용 여부를 결정합니다.

따라서 ServiceAccount를 생성하는 것만으로 Kubernetes API에 대한 모든 권한이 부여되는 것은 아닙니다.

애플리케이션에 필요한 최소한의 권한만 부여하는 최소 권한 원칙(Principle of Least Privilege)을 적용하는 것이 중요합니다.

## 10. 보안 고려사항

ServiceAccount는 Pod에서 실행되는 애플리케이션의 신원과 인증 정보를 관리하므로 다음과 같은 보안 사항을 고려해야 합니다.

### 10.1. 최소 권한 원칙 적용

모든 애플리케이션에 동일한 ServiceAccount를 사용하는 것보다 애플리케이션의 역할에 따라 ServiceAccount를 분리하는 것이 좋습니다.

각 ServiceAccount에는 실제로 필요한 Kubernetes API 권한만 부여해야 합니다.

### 10.2. default ServiceAccount 사용 최소화

애플리케이션별로 ServiceAccount를 분리하면 권한 관리와 보안 감사가 쉬워집니다.

또한 특정 애플리케이션의 인증 정보가 유출되더라도 다른 애플리케이션의 권한에 미치는 영향을 제한할 수 있습니다.

### 10.3. 불필요한 토큰 마운트 비활성화

Kubernetes API Server에 접근할 필요가 없는 애플리케이션이라면 `automountServiceAccountToken: false`를 설정하는 것이 좋습니다.

이를 통해 컨테이너 내부에 불필요한 인증 정보가 노출되는 것을 줄일 수 있습니다.

### 10.4. 단기 토큰 사용

가능하면 TokenRequest API와 Projected Volume을 사용하는 단기 토큰을 사용해야 합니다.

Secret에 저장하는 장기 토큰은 만료되지 않고 자동으로 갱신되지 않으므로 유출 시 보안 위험이 커질 수 있습니다.

### 10.5. Secret 접근 권한 관리

ServiceAccount Token이 저장된 Secret을 읽을 수 있는 사용자는 해당 토큰을 이용하여 ServiceAccount의 신원으로 인증할 수 있습니다.

따라서 Secret에 대한 읽기 및 쓰기 권한을 엄격하게 제한해야 합니다.

특히 Secret을 조회할 수 있는 권한과 ServiceAccount의 권한은 별개로 보이지만, Secret에 저장된 인증 정보가 유출되면 해당 ServiceAccount의 권한을 악용할 수 있다는 점에 주의해야 합니다.

## 11. 정리

ServiceAccount는 Kubernetes 클러스터 내부에서 실행되는 애플리케이션과 프로세스에 신원을 제공하는 리소스입니다.

핵심 내용을 정리하면 다음과 같습니다.

- ServiceAccount는 Namespace 범위의 Kubernetes 리소스입니다.
- Pod는 `spec.serviceAccountName`을 통해 사용할 ServiceAccount를 지정합니다.
- 별도의 ServiceAccount를 지정하지 않으면 해당 Namespace의 `default` ServiceAccount를 사용합니다.
- ServiceAccount Token은 Kubernetes API Server에 접근할 때 신원을 증명하는 인증 자격 증명입니다.
- Kubernetes v1.22부터 TokenRequest API와 Projected Volume을 이용한 단기 토큰 방식이 기본적으로 사용됩니다.
- `automountServiceAccountToken`을 사용하여 인증 토큰의 자동 마운트를 제어할 수 있습니다.
- ServiceAccount 자체는 리소스 접근 권한을 정의하지 않으며, 실제 권한은 RBAC 등을 통해 별도로 부여해야 합니다.
- 애플리케이션별 ServiceAccount 분리, 최소 권한 원칙, 단기 토큰 사용 등을 통해 보안을 강화할 수 있습니다.

---

## 참고자료

ServiceAccount의 인증 및 토큰 관리 방식은 Kubernetes 버전에 따라 변경된 부분이 있으므로 관련 버전의 공식 문서를 함께 참고하는 것이 좋습니다.

특히 Kubernetes v1.22에서는 TokenRequest API와 Projected Volume을 활용한 단기 토큰 방식이 기본적으로 적용되었으며, v1.24에서는 ServiceAccount 생성 시 장기 토큰 Secret을 자동으로 생성하던 기존 동작이 제거되었습니다.

- [Kubernetes Documentation - Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
- [Kubernetes Documentation - Managing Service Accounts](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)
- [Kubernetes Documentation - Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [Kubernetes Documentation - Projected Volumes](https://kubernetes.io/docs/concepts/storage/projected-volumes/)
- [Kubernetes Documentation - RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)