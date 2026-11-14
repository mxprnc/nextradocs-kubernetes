# NetworkPolicy

## 1. NetworkPolicy란?

**NetworkPolicy**는 Kubernetes에서 Pod 간 또는 Pod와 외부 네트워크 간의 네트워크 트래픽을 제어하기 위한 리소스입니다.

기본적으로 Kubernetes 클러스터에서는 별도의 네트워크 정책이 적용되지 않은 Pod 간에 네트워크 통신이 허용됩니다. NetworkPolicy를 사용하면 특정 Pod에 대해 Ingress(수신) 및 Egress(송신) 트래픽을 제한하고, 허용할 통신 대상과 포트를 명시적으로 지정할 수 있습니다.

NetworkPolicy는 다음과 같은 목적으로 활용할 수 있습니다.

- 특정 Pod에서만 애플리케이션에 접근하도록 제한합니다.
- 애플리케이션 계층 간 통신을 허용된 범위로 제한합니다.
- 데이터베이스 Pod에 접근할 수 있는 애플리케이션을 제한합니다.
- Pod에서 외부 네트워크로 나가는 트래픽을 제어합니다.
- Namespace 단위로 네트워크 접근 정책을 분리합니다.
- 불필요한 네트워크 통신을 차단하여 보안 수준을 높입니다.

NetworkPolicy는 `networking.k8s.io/v1` API 그룹에 속하며, Namespace-scoped 리소스입니다.

**중요:** NetworkPolicy 리소스를 생성하는 것만으로 모든 Kubernetes 네트워크 환경에서 트래픽이 차단되는 것은 아닙니다. 실제 네트워크 정책을 적용하려면 NetworkPolicy를 지원하는 CNI(Container Network Interface) 플러그인이 설치되어 있어야 합니다.

## 2. NetworkPolicy의 주요 개념

NetworkPolicy를 이해하려면 Ingress, Egress, Pod Selector, Namespace Selector의 개념을 알아야 합니다.

### 2.1 Ingress

**Ingress**는 Pod로 들어오는 네트워크 트래픽을 의미합니다.

NetworkPolicy에서 Ingress 정책을 정의하면 해당 Pod에 접근할 수 있는 출발지와 포트를 제한할 수 있습니다.

예를 들어 다음과 같은 정책을 구현할 수 있습니다.

- 특정 Namespace의 Pod만 접근할 수 있도록 허용합니다.
- 특정 Label을 가진 Pod만 접근할 수 있도록 허용합니다.
- 특정 IP 대역에서 들어오는 트래픽만 허용합니다.
- 특정 TCP 또는 UDP 포트로 들어오는 트래픽만 허용합니다.

### 2.2 Egress

**Egress**는 Pod에서 외부로 나가는 네트워크 트래픽을 의미합니다.

NetworkPolicy에서 Egress 정책을 정의하면 해당 Pod가 접근할 수 있는 목적지와 포트를 제한할 수 있습니다.

예를 들어 다음과 같은 정책을 구현할 수 있습니다.

- 특정 Namespace의 Pod로만 접근할 수 있도록 허용합니다.
- 특정 IP 대역으로만 트래픽을 전송하도록 허용합니다.
- 특정 포트로 나가는 트래픽만 허용합니다.
- 외부 네트워크로 나가는 트래픽을 차단합니다.

### 2.3 Ingress와 Egress의 차이

| 구분 | Ingress | Egress |
|---|---|---|
| 방향 | Pod로 들어오는 트래픽 | Pod에서 나가는 트래픽 |
| 제어 대상 | 출발지(Source) | 목적지(Destination) |
| 주요 조건 | 출발지 Pod, Namespace, IP | 목적지 Pod, Namespace, IP |
| 주요 목적 | Pod 접근 제한 | Pod의 외부 통신 제한 |

하나의 NetworkPolicy에서 Ingress와 Egress를 모두 정의할 수 있습니다.

## 3. NetworkPolicy의 동작 구조

NetworkPolicy는 특정 Pod를 선택하고 해당 Pod에 적용할 네트워크 통신 규칙을 정의하는 구조로 동작합니다.

주요 구성 요소는 다음과 같습니다.

| 구성 요소 | 역할 |
|---|---|
| `podSelector` | 정책을 적용할 대상 Pod를 선택합니다. |
| `policyTypes` | Ingress 및 Egress 중 정책을 적용할 트래픽 방향을 지정합니다. |
| `ingress` | Pod로 들어오는 트래픽의 허용 규칙을 정의합니다. |
| `egress` | Pod에서 나가는 트래픽의 허용 규칙을 정의합니다. |
| `from` | Ingress 규칙에서 허용할 출발지를 지정합니다. |
| `to` | Egress 규칙에서 허용할 목적지를 지정합니다. |
| `ports` | 허용할 네트워크 프로토콜과 포트를 지정합니다. |

일반적인 NetworkPolicy의 구조는 다음과 같습니다.

```text
NetworkPolicy
      |
      v
  podSelector
      |
      v
  Target Pods
      |
      +-------------------+
      |                   |
      v                   v
   Ingress              Egress
  (수신 정책)          (송신 정책)
      |                   |
      v                   v
   from                 to
  (출발지)             (목적지)
      |                   |
      +---------+---------+
                |
                v
              ports
          (프로토콜 / 포트)
```

NetworkPolicy는 정책에 의해 선택된 Pod의 네트워크 통신을 제어합니다. 정책의 적용 여부와 실제 패킷 차단은 CNI 플러그인이 담당합니다.

## 4. NetworkPolicy 리소스 구조

NetworkPolicy는 다음과 같은 기본 구조를 가집니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Ingress
    - Egress

  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080

  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
```

### 4.1 metadata

NetworkPolicy의 이름과 Namespace를 지정합니다.

```yaml
metadata:
  name: example-policy
  namespace: default
```

NetworkPolicy는 Namespace-scoped 리소스이므로 특정 Namespace에 생성해야 합니다.

### 4.2 podSelector

`podSelector`는 NetworkPolicy를 적용할 대상 Pod를 선택합니다.

```yaml
podSelector:
  matchLabels:
    app: backend
```

위 설정은 `app=backend` Label을 가진 Pod에 정책을 적용합니다.

빈 Selector를 지정하면 해당 Namespace의 모든 Pod가 선택됩니다.

```yaml
podSelector: {}
```

따라서 Namespace 전체에 기본 네트워크 정책을 적용할 때 빈 Selector를 활용할 수 있습니다.

### 4.3 policyTypes

`policyTypes`는 NetworkPolicy가 어떤 방향의 트래픽을 제어할지 지정합니다.

```yaml
policyTypes:
  - Ingress
  - Egress
```

사용할 수 있는 값은 다음과 같습니다.

| 값 | 설명 |
|---|---|
| `Ingress` | Pod로 들어오는 트래픽에 대한 정책을 적용합니다. |
| `Egress` | Pod에서 나가는 트래픽에 대한 정책을 적용합니다. |

`policyTypes`를 생략하면 기본값이 적용됩니다.

- `ingress` 규칙이 존재하면 기본적으로 Ingress가 포함됩니다.
- `egress` 규칙이 존재하면 기본적으로 Egress도 포함됩니다.
- `egress` 규칙이 없으면 기본적으로 Ingress만 적용됩니다.

명시적으로 정책 방향을 지정하면 의도하지 않은 기본 동작을 방지할 수 있으므로 운영 환경에서는 `policyTypes`를 명확하게 작성하는 것이 좋습니다.

### 4.4 ingress

`ingress`는 대상 Pod로 들어오는 트래픽의 허용 규칙을 정의합니다.

```yaml
ingress:
  - from:
      - podSelector:
          matchLabels:
            app: frontend
    ports:
      - protocol: TCP
        port: 8080
```

위 설정은 `app=frontend` Label을 가진 Pod에서 대상 Pod의 TCP 8080 포트로 접근하는 트래픽을 허용합니다.

### 4.5 egress

`egress`는 대상 Pod에서 나가는 트래픽의 허용 규칙을 정의합니다.

```yaml
egress:
  - to:
      - podSelector:
          matchLabels:
            app: database
    ports:
      - protocol: TCP
        port: 5432
```

위 설정은 대상 Pod가 `app=database` Label을 가진 Pod의 TCP 5432 포트로 접근하는 트래픽을 허용합니다.

### 4.6 ports

`ports`는 허용할 네트워크 프로토콜과 포트를 지정합니다.

```yaml
ports:
  - protocol: TCP
    port: 8080
```

주요 필드는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `protocol` | TCP, UDP 또는 SCTP를 지정합니다. |
| `port` | 허용할 포트 번호 또는 이름을 지정합니다. |
| `endPort` | 포트 범위의 마지막 포트를 지정합니다. |

`protocol`을 생략하면 기본적으로 TCP가 적용됩니다.

포트 범위를 지정하려면 `endPort`를 사용할 수 있습니다.

```yaml
ports:
  - protocol: TCP
    port: 8000
    endPort: 8010
```

위 설정은 TCP 8000번부터 8010번까지의 포트를 허용합니다.

`endPort`를 사용하는 경우 `port`는 숫자로 지정해야 하며, `endPort`는 `port`보다 크거나 같아야 합니다.

## 5. Ingress 정책 설정

Ingress 정책은 특정 Pod로 들어오는 트래픽을 제한하는 데 사용됩니다.

### 5.1 특정 Pod에서만 접근 허용

다음은 `app=backend` Label을 가진 Pod에 대해 `app=frontend` Label을 가진 Pod의 접근만 허용하는 예제입니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-ingress-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Ingress

  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
```

위 정책의 동작은 다음과 같습니다.

- `app=backend` Pod에 정책이 적용됩니다.
- `app=frontend` Pod에서 들어오는 트래픽을 허용합니다.
- 그 외 출발지에서 들어오는 트래픽은 차단됩니다.

포트를 별도로 지정하지 않았으므로 허용 규칙에 매칭되는 출발지의 모든 포트 트래픽이 허용됩니다.

### 5.2 특정 포트만 허용

다음은 `app=frontend` Pod에서 `app=backend` Pod의 TCP 8080 포트로 접근하는 트래픽만 허용하는 예제입니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-port-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Ingress

  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

위 정책은 출발지와 목적지 포트를 모두 제한합니다.

### 5.3 특정 Namespace에서만 접근 허용

`namespaceSelector`를 사용하면 특정 Namespace에 속한 Pod에서 들어오는 트래픽만 허용할 수 있습니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-namespace-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Ingress

  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              environment: production
```

위 설정은 `environment=production` Label을 가진 Namespace에 속한 Pod의 접근을 허용합니다.

Namespace Selector는 Namespace 자체의 Label을 기준으로 대상을 선택합니다.

### 5.4 특정 Namespace의 특정 Pod만 허용

`namespaceSelector`와 `podSelector`를 함께 사용하면 특정 Namespace에 속한 특정 Pod만 허용할 수 있습니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-specific-source-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Ingress

  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              environment: production
          podSelector:
            matchLabels:
              app: frontend
```

위 설정은 다음 두 조건을 모두 만족하는 Pod의 접근을 허용합니다.

1. `environment=production` Label을 가진 Namespace에 속해야 합니다.
2. 해당 Namespace에서 `app=frontend` Label을 가진 Pod여야 합니다.

**중요:** 하나의 `from` 항목 안에 `namespaceSelector`와 `podSelector`를 함께 작성하면 두 조건은 AND로 결합됩니다.

반면 다음과 같이 별도의 `from` 항목으로 작성하면 두 조건은 OR로 결합됩니다.

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            environment: production
      - podSelector:
          matchLabels:
            app: frontend
```

이 경우 다음 중 하나만 만족해도 접근이 허용됩니다.

- `environment=production` Namespace에 속한 모든 Pod
- 정책과 동일한 Namespace에서 `app=frontend` Label을 가진 Pod

### 5.5 특정 IP 대역에서만 접근 허용

`ipBlock`을 사용하면 IP 주소 또는 CIDR 범위를 기준으로 트래픽을 허용할 수 있습니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: external-ip-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Ingress

  ingress:
    - from:
        - ipBlock:
            cidr: 10.0.0.0/8
            except:
              - 10.1.0.0/16
```

위 설정은 다음과 같이 동작합니다.

- `10.0.0.0/8` 범위에서 들어오는 트래픽을 허용합니다.
- `10.1.0.0/16` 범위는 허용 대상에서 제외합니다.

`except`는 지정한 CIDR 범위 안에서 추가로 제외할 IP 대역을 정의합니다.

`ipBlock`은 일반적으로 클러스터 외부 IP 주소를 기준으로 접근을 제어할 때 활용합니다. 다만 트래픽이 Pod에 도달하기 전에 SNAT 또는 DNAT 등의 주소 변환을 거치는 경우 실제 정책 평가에 사용되는 IP가 예상과 다를 수 있으므로 CNI와 네트워크 환경을 함께 확인해야 합니다.

## 6. Egress 정책 설정

Egress 정책은 특정 Pod에서 나가는 트래픽을 제한하는 데 사용됩니다.

### 6.1 특정 Pod로만 접근 허용

다음은 `app=backend` Pod가 `app=database` Pod로만 접근할 수 있도록 허용하는 예제입니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Egress

  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
```

위 정책은 다음과 같이 동작합니다.

- `app=backend` Pod에 정책이 적용됩니다.
- `app=database` Pod로 나가는 트래픽을 허용합니다.
- 그 외 목적지로 나가는 트래픽은 차단됩니다.

### 6.2 특정 포트로만 접근 허용

다음은 `app=backend` Pod에서 `app=database` Pod의 TCP 5432 포트로 나가는 트래픽만 허용하는 예제입니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-database-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Egress

  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
```

이러한 정책은 애플리케이션이 데이터베이스에 접근하는 데 필요한 통신만 허용할 때 활용할 수 있습니다.

### 6.3 특정 외부 IP 대역으로만 접근 허용

다음은 특정 Pod가 지정한 외부 IP 대역으로만 트래픽을 전송하도록 허용하는 예제입니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: external-egress-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Egress

  egress:
    - to:
        - ipBlock:
            cidr: 203.0.113.0/24
      ports:
        - protocol: TCP
          port: 443
```

위 정책은 `app=backend` Pod에서 `203.0.113.0/24` IP 대역의 TCP 443 포트로 나가는 트래픽만 허용합니다.

`203.0.113.0/24`는 문서 예제를 위한 예약된 IP 대역입니다.

### 6.4 DNS 통신 허용

Egress 정책을 적용하면 DNS 요청도 차단될 수 있습니다.

Pod가 DNS 이름을 IP 주소로 변환하려면 클러스터 DNS 서버로 통신할 수 있어야 합니다.

일반적인 Kubernetes 환경에서는 CoreDNS가 kube-system Namespace에서 실행됩니다.

다음은 `app=backend` Pod가 kube-system Namespace의 DNS Pod로 UDP 및 TCP 53 포트 통신을 허용하는 예제입니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-dns-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Egress

  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

위 설정은 다음과 같이 동작합니다.

- `app=backend` Pod에 Egress 정책을 적용합니다.
- `kube-system` Namespace의 `k8s-app=kube-dns` Label을 가진 Pod를 DNS 대상으로 선택합니다.
- UDP 53 및 TCP 53 포트 통신을 허용합니다.

DNS Pod의 Label은 클러스터 구성에 따라 다를 수 있으므로 실제 환경에서 다음 명령어로 확인해야 합니다.

```bash
kubectl get pods -n kube-system --show-labels
```

또한 일부 클러스터에서는 NodeLocal DNSCache를 사용하거나 DNS 요청이 별도의 네트워크 경로를 통과할 수 있습니다. 이 경우 실제 DNS 통신 경로에 맞게 정책을 구성해야 합니다.

## 7. Default Deny 정책

NetworkPolicy에서 중요한 개념 중 하나는 **Default Deny**입니다.

특정 Pod에 Ingress 또는 Egress 정책이 적용되면 해당 방향의 트래픽은 기본적으로 차단되고, 명시적으로 허용한 규칙만 통과할 수 있습니다.

이를 통해 허용 목록(Allowlist) 기반의 네트워크 접근 제어를 구현할 수 있습니다.

### 7.1 Default Deny Ingress

다음은 특정 Namespace의 모든 Pod에 대해 들어오는 트래픽을 기본적으로 차단하는 정책입니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}

  policyTypes:
    - Ingress
```

위 설정은 다음과 같이 동작합니다.

- `default` Namespace의 모든 Pod가 정책 대상입니다.
- Ingress 정책을 적용합니다.
- 허용 규칙이 없으므로 외부에서 들어오는 Ingress 트래픽을 차단합니다.

### 7.2 Default Deny Egress

다음은 특정 Namespace의 모든 Pod에서 나가는 트래픽을 기본적으로 차단하는 정책입니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: default
spec:
  podSelector: {}

  policyTypes:
    - Egress
```

위 설정은 다음과 같이 동작합니다.

- `default` Namespace의 모든 Pod가 정책 대상입니다.
- Egress 정책을 적용합니다.
- 허용 규칙이 없으므로 외부로 나가는 Egress 트래픽을 차단합니다.

### 7.3 Default Deny Ingress와 Egress

두 방향을 모두 차단하려면 다음과 같이 구성할 수 있습니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}

  policyTypes:
    - Ingress
    - Egress
```

위 정책은 해당 Namespace의 모든 Pod에 대해 Ingress와 Egress를 모두 차단합니다.

단, DNS 조회, 외부 API 호출, 다른 애플리케이션과의 통신 등 필요한 연결을 별도의 허용 정책으로 추가해야 합니다.

## 8. 여러 NetworkPolicy의 동작 방식

하나의 Pod에는 여러 NetworkPolicy가 동시에 적용될 수 있습니다.

이때 NetworkPolicy는 서로 우선순위를 갖는 방식으로 동작하지 않습니다.

**동일한 방향에 적용되는 여러 NetworkPolicy의 허용 규칙은 서로 합산됩니다.**

즉, 하나의 정책에서 트래픽을 허용하고 다른 정책에서 해당 트래픽을 허용하지 않더라도, 허용 규칙 중 하나라도 일치하면 해당 트래픽은 허용됩니다.

### 8.1 여러 정책의 허용 규칙 합산

예를 들어 다음 두 정책이 있다고 가정하겠습니다.

**정책 A**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Ingress

  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
```

**정책 B**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Ingress

  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: monitoring
```

두 정책 모두 `app=backend` Pod에 적용됩니다.

이 경우 다음과 같이 동작합니다.

| 출발지 Pod | 접근 결과 |
|---|---|
| app=frontend | 허용 |
| app=monitoring | 허용 |
| app=other | 차단 |

각 정책이 허용하는 규칙이 합산되기 때문입니다.

### 8.2 정책 간 우선순위가 없는 이유

Kubernetes의 기본 NetworkPolicy API는 명시적인 Allow 및 Deny 우선순위 체계를 제공하지 않습니다.

따라서 특정 NetworkPolicy에서 허용한 트래픽을 다른 NetworkPolicy로 다시 차단할 수는 없습니다.

여러 정책을 함께 사용할 때는 다음 사항을 고려해야 합니다.

- 정책의 대상 Pod가 중복되는지 확인합니다.
- Ingress와 Egress 방향을 구분하여 정책을 설계합니다.
- 필요한 통신을 허용하는 정책이 누락되지 않았는지 확인합니다.
- 하나의 정책에서 차단한 트래픽을 다른 정책으로 다시 차단할 수 있다고 가정하지 않습니다.

명시적인 Deny 우선순위나 계층적인 정책 관리가 필요한 경우에는 CNI가 제공하는 별도의 Network Policy 확장 기능을 검토할 수 있습니다.

## 9. NetworkPolicy와 Service의 관계

NetworkPolicy는 Service 자체에 적용되는 리소스가 아니라 Pod를 대상으로 적용되는 리소스입니다.

Service는 클라이언트가 Pod에 접근할 수 있도록 안정적인 네트워크 접근점을 제공하고, NetworkPolicy는 Pod 간의 통신 가능 여부를 제어합니다.

일반적인 트래픽 흐름은 다음과 같습니다.

```text
Client Pod
    |
    v
Service
    |
    v
EndpointSlice
    |
    v
Target Pod
    |
    v
NetworkPolicy
```

NetworkPolicy는 실제 Pod의 네트워크 통신을 제어합니다. 따라서 Service를 통해 접근하더라도 대상 Pod에 적용된 NetworkPolicy의 허용 규칙을 만족해야 합니다.

다음과 같은 구성을 생각할 수 있습니다.

- frontend-service는 frontend Pod를 대상으로 합니다.
- backend-service는 backend Pod를 대상으로 합니다.
- backend Pod에는 frontend Pod의 접근만 허용하는 NetworkPolicy가 적용됩니다.

이 경우 frontend Pod는 backend-service를 통해 backend Pod에 접근할 수 있지만, 허용되지 않은 다른 Pod는 접근할 수 없습니다.

**주의사항:** Service의 ClusterIP, NodePort, LoadBalancer 트래픽이 실제로 어떤 주소와 인터페이스를 거쳐 Pod에 도달하는지는 CNI 및 네트워크 구현에 따라 달라질 수 있습니다. 특히 외부 트래픽이나 NodePort 트래픽을 제어할 때는 주소 변환 및 정책 적용 시점을 확인해야 합니다.

## 10. NetworkPolicy와 CNI

NetworkPolicy는 Kubernetes API 리소스만으로 네트워크 패킷을 차단하지 않습니다.

실제 네트워크 정책의 적용은 CNI 플러그인이 담당합니다.

### 10.1 CNI의 역할

CNI(Container Network Interface)는 Kubernetes Pod에 네트워크 연결을 제공하기 위한 인터페이스와 플러그인 구조입니다.

CNI 구현체에 따라 Pod의 네트워크 연결뿐만 아니라 NetworkPolicy를 처리하는 기능도 제공할 수 있습니다.

NetworkPolicy를 지원하는 CNI는 Kubernetes API에서 생성된 NetworkPolicy를 감시하고 실제 네트워크 장비, 커널 네트워크 기능 또는 프록시 설정을 통해 정책을 적용합니다.

### 10.2 NetworkPolicy를 지원하는 대표적인 CNI

| CNI | 설명 |
|---|---|
| Calico | Kubernetes NetworkPolicy와 자체 확장 정책을 지원합니다. |
| Cilium | eBPF 기반 네트워킹과 Kubernetes NetworkPolicy를 지원합니다. |
| Antrea | Kubernetes NetworkPolicy와 추가 네트워크 정책 기능을 제공합니다. |
| Kube-router | Kubernetes NetworkPolicy를 지원합니다. |

### 10.3 NetworkPolicy를 지원하지 않는 환경

NetworkPolicy를 지원하지 않는 CNI 환경에서 NetworkPolicy 리소스를 생성하더라도 실제 트래픽이 차단되지 않을 수 있습니다.

예를 들어 다음 명령어로 NetworkPolicy 리소스를 생성할 수는 있습니다.

```bash
kubectl apply -f network-policy.yaml
```

하지만 해당 리소스를 처리하는 네트워크 구현체가 없다면 정책이 실제 통신에 영향을 주지 않습니다.

따라서 NetworkPolicy를 사용하기 전에 현재 클러스터의 CNI가 NetworkPolicy를 지원하는지 확인해야 합니다.

## 11. NetworkPolicy 확인 및 관리 명령어

NetworkPolicy를 조회하고 관리할 때 사용하는 대표적인 kubectl 명령어는 다음과 같습니다.

### 11.1 NetworkPolicy 목록 조회

```bash
kubectl get networkpolicy
```

특정 Namespace의 NetworkPolicy를 조회합니다.

```bash
kubectl get networkpolicy -n default
```

모든 Namespace의 NetworkPolicy를 조회합니다.

```bash
kubectl get networkpolicy -A
```

### 11.2 특정 NetworkPolicy 상세 조회

```bash
kubectl describe networkpolicy backend-ingress-policy -n default
```

정책의 대상 Pod, Ingress, Egress 규칙 등을 확인할 수 있습니다.

### 11.3 NetworkPolicy YAML 조회

```bash
kubectl get networkpolicy backend-ingress-policy \
  -n default -o yaml
```

### 11.4 NetworkPolicy 생성 및 수정

```bash
kubectl apply -f network-policy.yaml
```

YAML 파일의 변경 사항을 클러스터에 적용합니다.

### 11.5 NetworkPolicy 삭제

```bash
kubectl delete networkpolicy backend-ingress-policy -n default
```

### 11.6 Pod Label 확인

NetworkPolicy의 `podSelector`가 올바른 Pod를 선택하는지 확인할 때 사용합니다.

```bash
kubectl get pods -n default --show-labels
```

### 11.7 Namespace Label 확인

`namespaceSelector`가 올바른 Namespace를 선택하는지 확인할 때 사용합니다.

```bash
kubectl get namespaces --show-labels
```

### 11.8 정책 적용 대상 Pod 확인

NetworkPolicy의 Selector와 일치하는 Pod를 확인할 수 있습니다.

```bash
kubectl get pods -n default -l app=backend
```

## 12. NetworkPolicy 사용 시 주의사항

NetworkPolicy를 운영 환경에 적용할 때는 다음 사항을 고려해야 합니다.

1. **CNI 지원 여부:** NetworkPolicy를 실제로 적용하려면 해당 기능을 지원하는 CNI가 필요합니다.
2. **Namespace 범위:** NetworkPolicy는 Namespace-scoped 리소스이며, 기본적으로 자신이 생성된 Namespace의 Pod만 대상으로 선택할 수 있습니다.
3. **기본 통신 허용:** 별도의 정책이 적용되지 않은 Pod는 해당 방향의 네트워크 통신이 기본적으로 허용됩니다.
4. **Default Deny 영향:** Ingress 또는 Egress 정책을 적용하면 해당 방향의 통신이 제한되므로 필요한 연결을 별도로 허용해야 합니다.
5. **DNS 통신:** Egress를 제한하는 경우 DNS 서버로의 통신도 차단될 수 있으므로 DNS 허용 규칙을 고려해야 합니다.
6. **정책 합산:** 동일한 방향에 적용되는 여러 NetworkPolicy의 허용 규칙은 합산됩니다. 기본 NetworkPolicy API에는 Deny 우선순위가 없습니다.
7. **Label 관리:** Pod와 Namespace의 Label이 변경되면 NetworkPolicy의 적용 대상도 달라질 수 있습니다.
8. **주소 변환:** SNAT 및 DNAT 등의 네트워크 주소 변환에 따라 정책에서 평가되는 출발지 또는 목적지 IP가 예상과 다를 수 있습니다.
9. **외부 트래픽:** 외부 IP, NodePort, LoadBalancer 등의 트래픽은 CNI 구현에 따라 정책 적용 방식이 달라질 수 있습니다.
10. **기존 연결:** 정책 변경이 기존 연결에 반영되는 시점과 방식은 CNI 구현에 따라 차이가 있을 수 있습니다.

## 13. 정리

NetworkPolicy는 Kubernetes에서 Pod 간 및 Pod와 외부 네트워크 간의 통신을 제어하기 위한 리소스입니다.

Pod Selector를 통해 정책을 적용할 대상을 선택하고, Ingress 및 Egress 규칙을 통해 허용할 네트워크 트래픽을 정의합니다.

NetworkPolicy의 주요 특징은 다음과 같습니다.

- `podSelector`를 통해 정책을 적용할 Pod를 선택합니다.
- `policyTypes`를 통해 Ingress와 Egress 중 제어할 트래픽 방향을 지정합니다.
- Ingress 정책으로 Pod에 들어오는 트래픽을 제한합니다.
- Egress 정책으로 Pod에서 나가는 트래픽을 제한합니다.
- `podSelector`와 `namespaceSelector`를 통해 통신 대상을 선택합니다.
- `ipBlock`을 통해 IP 및 CIDR 기반 접근 제어를 수행합니다.
- `ports`를 통해 허용할 프로토콜과 포트를 지정합니다.
- 여러 NetworkPolicy의 허용 규칙은 합산됩니다.
- NetworkPolicy를 실제로 적용하려면 이를 지원하는 CNI가 필요합니다.

NetworkPolicy는 Kubernetes의 네트워크 보안을 구성하는 핵심 리소스입니다.

특히 애플리케이션을 여러 계층으로 분리하여 운영하는 환경에서 각 계층 간 통신을 필요한 범위로 제한하고, 불필요한 네트워크 접근을 차단하는 데 활용할 수 있습니다.

---

**공식 문서 및 참고 자료**

- [Kubernetes 공식 문서 - Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes API Reference - NetworkPolicy](https://kubernetes.io/docs/reference/kubernetes-api/networking/network-policy-v1/)
- [Kubernetes 공식 문서 - Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)