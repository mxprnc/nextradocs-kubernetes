# Gateway API

## 1. Gateway API란?

**Gateway API**는 Kubernetes에서 네트워크 트래픽을 관리하고 클러스터 내부의 Service로 전달하기 위한 확장 가능한 네트워킹 API입니다.

기존 Ingress API는 HTTP 및 HTTPS 트래픽의 Host와 Path 기반 라우팅을 제공하지만, 복잡한 라우팅 정책이나 다양한 프로토콜을 처리하려면 Controller별 Annotation 또는 별도의 확장 기능에 의존해야 하는 한계가 있습니다.

Gateway API는 이러한 한계를 보완하기 위해 설계되었으며, 네트워크 인프라와 애플리케이션의 라우팅 설정을 여러 리소스로 분리하여 관리할 수 있도록 합니다.

특히 클러스터 운영자, 인프라 운영자, 애플리케이션 개발자가 서로 다른 책임 범위에서 네트워크 구성을 관리할 수 있도록 설계되었습니다.

Gateway API를 활용하면 다음과 같은 구성을 구현할 수 있습니다.

- HTTP 및 HTTPS 트래픽 라우팅
- Host, Path, Header, Query Parameter 기반 라우팅
- 가중치 기반 트래픽 분배
- HTTP 요청 및 응답 Header 수정
- 요청 리다이렉트 및 URL 재작성
- gRPC 트래픽 라우팅
- TCP, TLS, UDP 트래픽 관리
- TLS 연결 및 인증서 관리
- Namespace별 Route 연결 범위 제어

Gateway API는 단일 리소스가 아니라 여러 리소스가 협력하여 네트워크 구성을 표현하는 API 집합입니다.

주요 리소스는 다음과 같습니다.

| 리소스 | 역할 |
|---|---|
| GatewayClass | Gateway를 구현하는 Controller의 종류와 공통 설정을 정의합니다. |
| Gateway | 트래픽을 수신하는 네트워크 진입점과 Listener를 정의합니다. |
| HTTPRoute | HTTP 요청의 라우팅 규칙을 정의합니다. |
| GRPCRoute | gRPC 요청의 라우팅 규칙을 정의합니다. |
| TLSRoute | TLS 연결의 라우팅 규칙을 정의합니다. |
| TCPRoute | TCP 트래픽의 라우팅 규칙을 정의합니다. |
| UDPRoute | UDP 트래픽의 라우팅 규칙을 정의합니다. |
| ReferenceGrant | Namespace 간 리소스 참조를 명시적으로 허용합니다. |

Gateway API는 `gateway.networking.k8s.io` API 그룹을 사용합니다.

**중요:** Gateway API는 Kubernetes에 기본적으로 모두 설치되어 있는 리소스가 아닙니다. Gateway API CRD와 이를 처리하는 Gateway Controller를 설치하고 구성해야 실제 트래픽을 처리할 수 있습니다.

## 2. Gateway API의 주요 특징

### 2.1 역할 기반 리소스 분리

Gateway API는 네트워크 인프라를 관리하는 역할과 애플리케이션의 라우팅 규칙을 관리하는 역할을 분리합니다.

| 역할 | 주요 리소스 | 책임 |
|---|---|---|
| 인프라 제공자 | GatewayClass | Gateway의 구현 종류와 공통 설정을 정의합니다. |
| 클러스터 운영자 | Gateway | 트래픽 진입점과 Listener를 구성합니다. |
| 애플리케이션 개발자 | HTTPRoute, GRPCRoute 등 | 애플리케이션의 라우팅 규칙을 정의합니다. |

예를 들어 클러스터 운영자는 공통 Gateway를 구성하고, 애플리케이션 개발자는 자신의 Namespace에서 HTTPRoute를 생성하여 해당 Gateway에 연결할 수 있습니다.

이를 통해 인프라 설정과 애플리케이션 배포 설정을 서로 독립적으로 관리할 수 있습니다.

### 2.2 다양한 프로토콜 지원

Gateway API는 HTTP뿐만 아니라 다양한 네트워크 프로토콜을 처리하기 위한 Route 리소스를 제공합니다.

| Route 리소스 | 프로토콜 | 주요 용도 |
|---|---|---|
| HTTPRoute | HTTP/HTTPS | 웹 애플리케이션 및 API 라우팅 |
| GRPCRoute | gRPC | gRPC 서비스 라우팅 |
| TLSRoute | TLS | SNI 기반 TLS 트래픽 라우팅 |
| TCPRoute | TCP | TCP 연결 전달 |
| UDPRoute | UDP | UDP 트래픽 전달 |

각 Route의 안정성 수준과 지원 여부는 Gateway API 버전 및 Controller 구현에 따라 확인해야 합니다.

### 2.3 확장 가능한 라우팅 정책

Gateway API는 Ingress보다 세밀한 트래픽 제어 기능을 표준화된 리소스 구조로 제공합니다.

HTTPRoute에서는 다음과 같은 조건을 활용할 수 있습니다.

- Hostname
- URL Path
- HTTP Header
- HTTP Method
- Query Parameter

또한 Backend별 가중치를 지정하여 Canary 배포나 점진적인 트래픽 전환을 구현할 수 있습니다.

### 2.4 Namespace 기반 접근 제어

Gateway API는 서로 다른 Namespace에 존재하는 Gateway와 Route를 연결할 수 있도록 설계되었습니다.

Gateway의 `allowedRoutes`와 Route의 `parentRefs`를 통해 연결 허용 범위를 관리할 수 있습니다.

또한 ReferenceGrant를 사용하면 서로 다른 Namespace에 존재하는 Route와 Backend Service 간의 참조를 명시적으로 허용할 수 있습니다.

이를 통해 인프라 운영자는 Gateway에 연결할 수 있는 Route의 종류와 Namespace를 제한할 수 있습니다.

## 3. Gateway API의 전체 구조

Gateway API는 GatewayClass, Gateway, Route 리소스가 서로 연결되는 구조로 동작합니다.

```text
GatewayClass
     |
     v
Gateway Controller
     |
     v
Gateway
  - Listener
  - Port
  - Protocol
  - Hostname
  - TLS
     |
     v
HTTPRoute / GRPCRoute / TLSRoute
  - Hostname
  - Path
  - Header
  - Match
  - Filter
  - Backend
     |
     v
Service
     |
     v
EndpointSlice
     |
     v
Pod
```

각 리소스의 관계는 다음과 같습니다.

1. GatewayClass가 Gateway를 구현할 Controller를 지정합니다.
2. Gateway가 트래픽을 수신할 네트워크 진입점과 Listener를 정의합니다.
3. HTTPRoute 등의 Route 리소스가 `parentRefs`를 통해 Gateway에 연결됩니다.
4. Gateway Controller가 Gateway와 Route 설정을 감시하고 해석합니다.
5. Controller가 실제 네트워크 프록시 또는 LoadBalancer의 구성을 생성합니다.
6. 외부 요청이 Listener에 도달하면 Route 규칙에 따라 Backend Service로 전달됩니다.
7. Service가 연결된 EndpointSlice를 통해 실제 Pod로 트래픽을 전달합니다.

일반적인 HTTP 트래픽 처리 흐름은 다음과 같습니다.

```text
External Client
      |
      v
DNS Resolution
      |
      v
Gateway Address
      |
      v
Gateway Listener
      |
      v
HTTPRoute
      |
      v
Backend Service
      |
      v
EndpointSlice
      |
      v
Pod
```

Gateway API는 리소스 간의 참조 관계를 통해 네트워크 구성을 표현하며, 실제 데이터 플레인의 구현 방식은 Gateway Controller에 따라 달라집니다.

## 4. GatewayClass

**GatewayClass**는 동일한 종류의 설정과 동작 방식을 사용하는 Gateway의 그룹을 정의하는 리소스입니다.

Ingress API의 IngressClass와 유사한 역할을 수행하지만, Gateway API에서는 Controller가 관리하는 Gateway 인프라의 종류와 공통 설정을 정의하는 데 사용됩니다.

GatewayClass는 다음과 같은 정보를 포함합니다.

| 필드 | 설명 |
|---|---|
| `metadata.name` | GatewayClass의 이름입니다. |
| `spec.controllerName` | GatewayClass를 관리하는 Controller의 식별자입니다. |
| `spec.parametersRef` | Controller가 사용할 추가 설정 리소스를 참조합니다. |
| `spec.description` | GatewayClass에 대한 설명입니다. |

GatewayClass는 Cluster-scoped 리소스입니다.

### 4.1 GatewayClass 생성 예제

다음은 GatewayClass를 생성하는 예제입니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: example-gateway-class
spec:
  controllerName: example.com/gateway-controller
  description: Example Gateway Controller
```

각 필드의 의미는 다음과 같습니다.

- `metadata.name`: Gateway에서 참조할 GatewayClass의 이름입니다.
- `spec.controllerName`: 해당 GatewayClass를 관리하는 Gateway Controller의 식별자입니다.
- `spec.description`: GatewayClass에 대한 설명입니다.

`controllerName`은 실제 설치된 Gateway Controller가 사용하는 식별자와 일치해야 합니다.

Gateway Controller는 자신이 관리하도록 구성된 GatewayClass를 감시하고 해당 Class를 참조하는 Gateway를 처리합니다.

**참고:** `example.com/gateway-controller`는 예시 식별자입니다. 실제 환경에서는 선택한 Gateway Controller의 공식 문서에서 정확한 Controller 이름을 확인해야 합니다.

### 4.2 GatewayClass와 Controller의 관계

하나의 GatewayClass는 하나의 Controller에 의해 관리됩니다. 반면 하나의 Controller가 여러 GatewayClass를 관리하는 것은 가능합니다.

예를 들어 다음과 같이 외부용과 내부용 GatewayClass를 분리할 수 있습니다.

```text
Gateway Controller
       |
       +---------------------+
       |                     |
       v                     v
GatewayClass             GatewayClass
  internet                  internal
       |                     |
       v                     v
Internet Gateway        Internal Gateway
```

이러한 구성을 사용하면 동일한 Controller를 사용하면서도 서로 다른 네트워크 정책이나 인프라 구성을 갖는 Gateway를 분리할 수 있습니다.

## 5. Gateway

**Gateway**는 실제 네트워크 트래픽을 수신하고 처리하는 인프라의 인스턴스를 정의하는 리소스입니다.

GatewayClass가 Gateway의 구현 종류를 정의한다면, Gateway는 실제로 사용할 네트워크 진입점과 Listener 구성을 정의합니다.

Gateway는 다음과 같은 인프라를 나타낼 수 있습니다.

- 클라우드 LoadBalancer
- Reverse Proxy
- Gateway Controller가 관리하는 네트워크 진입점
- 클러스터 내부에서 사용하는 서비스 메시 Gateway

Gateway의 주요 필드는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `spec.gatewayClassName` | 사용할 GatewayClass를 지정합니다. |
| `spec.addresses` | Gateway에 할당할 네트워크 주소를 지정합니다. |
| `spec.listeners` | 트래픽을 수신할 포트와 프로토콜을 정의합니다. |
| `spec.listeners[].name` | Listener의 이름입니다. |
| `spec.listeners[].protocol` | Listener가 처리할 프로토콜입니다. |
| `spec.listeners[].port` | 트래픽을 수신할 포트입니다. |
| `spec.listeners[].hostname` | Listener가 처리할 Hostname입니다. |
| `spec.listeners[].tls` | TLS 관련 설정을 지정합니다. |
| `spec.listeners[].allowedRoutes` | Listener에 연결할 수 있는 Route의 종류와 Namespace를 제한합니다. |

### 5.1 기본 Gateway 생성

다음은 HTTP 트래픽을 80번 포트에서 수신하는 Gateway 예제입니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
  namespace: default
spec:
  gatewayClassName: example-gateway-class
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      hostname: example.com
      allowedRoutes:
        namespaces:
          from: Same
```

각 필드의 의미는 다음과 같습니다.

- `gatewayClassName`: Gateway를 관리할 GatewayClass를 지정합니다.
- `listeners`: 트래픽을 수신할 Listener 목록입니다.
- `name`: Listener의 식별 이름입니다.
- `protocol`: HTTP 프로토콜을 사용합니다.
- `port`: 80번 포트에서 요청을 수신합니다.
- `hostname`: `example.com` 도메인의 요청을 처리합니다.
- `allowedRoutes`: 동일한 Namespace에 존재하는 Route만 연결할 수 있도록 제한합니다.

위 설정에서는 `default` Namespace에 존재하는 HTTPRoute가 해당 Gateway에 연결될 수 있습니다.

### 5.2 Listener

Listener는 Gateway가 네트워크 트래픽을 수신하는 구체적인 조건을 정의합니다.

하나의 Gateway에는 여러 Listener를 정의할 수 있습니다.

| 필드 | 설명 |
|---|---|
| `name` | Listener를 식별하는 이름입니다. |
| `protocol` | HTTP, HTTPS, TLS, TCP 등을 지정합니다. |
| `port` | 트래픽을 수신할 포트입니다. |
| `hostname` | 처리할 도메인 이름을 지정합니다. |
| `tls` | TLS 설정을 정의합니다. |
| `allowedRoutes` | 연결을 허용할 Route의 범위를 지정합니다. |

예를 들어 하나의 Gateway에서 HTTP와 HTTPS를 동시에 수신하도록 구성할 수 있습니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: default
spec:
  gatewayClassName: example-gateway-class
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      hostname: example.com
      allowedRoutes:
        namespaces:
          from: Same

    - name: https
      protocol: HTTPS
      port: 443
      hostname: example.com
      tls:
        mode: Terminate
        certificateRefs:
          - name: example-tls
      allowedRoutes:
        namespaces:
          from: Same
```

이 Gateway는 다음과 같이 동작합니다.

- HTTP Listener는 80번 포트에서 HTTP 요청을 수신합니다.
- HTTPS Listener는 443번 포트에서 HTTPS 요청을 수신합니다.
- HTTPS Listener는 TLS 연결을 종료하는 `Terminate` 모드를 사용합니다.
- TLS 인증서는 `example-tls` Secret을 참조합니다.

위 예제에서 Secret은 Gateway와 동일한 Namespace에 존재해야 합니다.

### 5.3 TLS 설정

Gateway API에서는 Listener의 `tls` 필드를 통해 TLS 연결을 설정합니다.

대표적인 TLS 모드는 다음과 같습니다.

| 모드 | 설명 |
|---|---|
| `Terminate` | Gateway에서 TLS 연결을 종료하고 복호화된 트래픽을 Backend로 전달합니다. |
| `Passthrough` | Gateway에서 TLS를 종료하지 않고 암호화된 트래픽을 Backend로 전달합니다. |

HTTPS Listener에서는 일반적으로 `Terminate` 모드를 사용합니다.

```yaml
listeners:
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
        - kind: Secret
          name: example-tls
```

TLS 인증서는 Kubernetes Secret으로 관리할 수 있습니다.

예를 들어 다음 명령어로 TLS Secret을 생성할 수 있습니다.

```bash
kubectl create secret tls example-tls \
  --cert=tls.crt \
  --key=tls.key \
  -n default
```

Gateway Controller는 Listener의 TLS 설정을 확인하고 해당 인증서를 실제 데이터 플레인에 반영합니다.

### 5.4 allowedRoutes

`allowedRoutes`는 Gateway Listener에 연결할 수 있는 Route의 종류와 Namespace 범위를 제한하는 설정입니다.

이를 통해 Gateway 운영자가 어떤 애플리케이션의 Route를 허용할지 제어할 수 있습니다.

대표적인 Namespace 설정은 다음과 같습니다.

| 값 | 설명 |
|---|---|
| `Same` | Gateway와 동일한 Namespace의 Route만 허용합니다. |
| `All` | 모든 Namespace의 Route를 허용합니다. |
| `Selector` | 지정한 Label Selector에 일치하는 Namespace의 Route를 허용합니다. |

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
allowedRoutes:
  namespaces:
    from: Selector
    selector:
      matchLabels:
        environment: production
```

위 설정에서는 `environment=production` Label을 가진 Namespace의 Route만 연결할 수 있습니다.

Route 종류도 별도로 제한할 수 있습니다.

```yaml
allowedRoutes:
  kinds:
    - kind: HTTPRoute
    - kind: GRPCRoute
  namespaces:
    from: All
```

이 경우 모든 Namespace에서 생성된 HTTPRoute와 GRPCRoute만 해당 Listener에 연결할 수 있습니다.

### 5.5 Gateway와 Route의 연결

Gateway는 Route 리소스가 자신의 Listener에 연결될 수 있도록 허용 범위를 정의합니다.

반대로 Route는 `parentRefs`를 통해 자신이 연결될 Gateway를 지정합니다.

이러한 구조는 Gateway 운영자와 애플리케이션 개발자 사이의 역할을 분리하면서도 명시적인 연결 관계를 유지할 수 있도록 합니다.

## 6. HTTPRoute

**HTTPRoute**는 HTTP 요청을 어떤 Backend로 전달할지 정의하는 리소스입니다.

Ingress의 Host 및 Path 기반 라우팅 기능을 포함하며, Header, Query Parameter, HTTP Method 등 다양한 조건을 활용할 수 있습니다.

HTTPRoute는 다음과 같은 주요 필드로 구성됩니다.

| 필드 | 설명 |
|---|---|
| `spec.parentRefs` | 연결할 Gateway 또는 지원되는 다른 Parent 리소스를 지정합니다. |
| `spec.hostnames` | 라우팅 대상 Hostname을 지정합니다. |
| `spec.rules` | HTTP 요청의 매칭 및 전달 규칙을 정의합니다. |
| `spec.rules[].matches` | 요청을 구분할 조건을 정의합니다. |
| `spec.rules[].filters` | 요청 또는 응답에 적용할 추가 처리 규칙을 정의합니다. |
| `spec.rules[].backendRefs` | 요청을 전달할 Backend를 지정합니다. |

### 6.1 기본 HTTPRoute 생성

다음은 `example-gateway`에 연결하여 `example.com`의 HTTP 요청을 `frontend-service`로 전달하는 HTTPRoute 예제입니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend-route
  namespace: default
spec:
  parentRefs:
    - name: example-gateway
  hostnames:
    - example.com
  rules:
    - backendRefs:
        - name: frontend-service
          port: 8080
```

각 필드의 의미는 다음과 같습니다.

- `parentRefs`: 연결할 Gateway를 지정합니다.
- `hostnames`: `example.com` 요청을 대상으로 합니다.
- `rules`: HTTP 요청의 처리 규칙을 정의합니다.
- `backendRefs`: 요청을 전달할 Service와 포트를 지정합니다.

위 예제에서는 `example.com`으로 들어오는 HTTP 요청이 `frontend-service`의 8080번 포트로 전달됩니다.

Gateway와 HTTPRoute는 동일한 Namespace에 있으므로 별도의 ReferenceGrant 없이 연결할 수 있습니다.

### 6.2 Path 기반 라우팅

HTTPRoute는 URL Path를 기준으로 요청을 구분할 수 있습니다.

다음은 `/api` 경로로 들어오는 요청을 `api-service`로 전달하는 예제입니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
  namespace: default
spec:
  parentRefs:
    - name: example-gateway
  hostnames:
    - example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: api-service
          port: 8080
```

HTTPRoute에서 사용하는 Path 매칭 유형은 다음과 같습니다.

| 유형 | 설명 |
|---|---|
| `Exact` | 요청 경로가 지정된 경로와 정확하게 일치해야 합니다. |
| `PathPrefix` | `/`로 구분되는 경로 요소 단위로 Prefix를 비교합니다. |

예를 들어 `PathPrefix: /api`는 `/api`, `/api/users`와 같은 경로에 매칭되지만 `/apiv2`에는 매칭되지 않습니다.

### 6.3 Header 기반 라우팅

HTTPRoute는 HTTP Header 값을 기준으로 요청을 구분할 수 있습니다.

예를 들어 `x-version: v2` Header가 포함된 요청을 새로운 버전의 Backend로 전달할 수 있습니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: header-routing
  namespace: default
spec:
  parentRefs:
    - name: example-gateway
  hostnames:
    - example.com
  rules:
    - matches:
        - headers:
            - name: x-version
              value: v2
      backendRefs:
        - name: backend-v2
          port: 8080

    - backendRefs:
        - name: backend-v1
          port: 8080
```

위 설정에서는 다음과 같이 동작합니다.

| 요청 Header | Backend |
|---|---|
| `x-version: v2` | backend-v2 |
| Header 없음 | backend-v1 |
| `x-version: v1` | backend-v1 |

이를 활용하면 특정 Header를 가진 요청만 새로운 애플리케이션 버전으로 전달할 수 있습니다.

### 6.4 Query Parameter 기반 라우팅

HTTPRoute는 URL의 Query Parameter를 기준으로 요청을 구분할 수도 있습니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: query-routing
  namespace: default
spec:
  parentRefs:
    - name: example-gateway
  hostnames:
    - example.com
  rules:
    - matches:
        - queryParams:
            - type: Exact
              name: version
              value: v2
      backendRefs:
        - name: backend-v2
          port: 8080
```

다음 요청은 `backend-v2`로 전달됩니다.

```text
http://example.com/products?version=v2
```

Query Parameter 매칭은 API 요청의 특정 조건에 따라 Backend를 분리해야 할 때 활용할 수 있습니다.

### 6.5 HTTP Method 기반 라우팅

HTTPRoute는 HTTP Method를 기준으로 요청을 구분할 수도 있습니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: method-routing
  namespace: default
spec:
  parentRefs:
    - name: example-gateway
  hostnames:
    - example.com
  rules:
    - matches:
        - method: GET
      backendRefs:
        - name: read-service
          port: 8080

    - matches:
        - method: POST
      backendRefs:
        - name: write-service
          port: 8080
```

위 설정에서는 다음과 같이 동작합니다.

| HTTP Method | Backend |
|---|---|
| GET | read-service |
| POST | write-service |

이를 활용하면 읽기 요청과 쓰기 요청을 서로 다른 Backend로 전달할 수 있습니다.

### 6.6 가중치 기반 트래픽 분배

HTTPRoute는 Backend별 가중치를 지정하여 요청을 분배할 수 있습니다.

이를 활용하면 Canary 배포나 점진적인 서비스 버전 전환을 구현할 수 있습니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: weighted-routing
  namespace: default
spec:
  parentRefs:
    - name: example-gateway
  hostnames:
    - example.com
  rules:
    - backendRefs:
        - name: backend-v1
          port: 8080
          weight: 90
        - name: backend-v2
          port: 8080
          weight: 10
```

위 설정에서는 전체 가중치 100을 기준으로 다음과 같이 트래픽을 분배합니다.

| Backend | 가중치 | 예상 트래픽 비율 |
|---|---:|---:|
| backend-v1 | 90 | 90% |
| backend-v2 | 10 | 10% |

이 방식은 새로운 버전의 애플리케이션에 일부 트래픽만 전달하여 안정성을 확인할 때 유용합니다.

단, 가중치는 요청 분배 비율을 나타내는 설정이며 실제 트래픽 비율이 항상 정확하게 일치하는 것은 아닙니다. 분배 방식과 세부 동작은 Controller 구현에 따라 달라질 수 있습니다.

### 6.7 HTTPRoute Filters

HTTPRoute의 `filters`는 요청 또는 응답을 처리하는 추가 기능을 정의합니다.

대표적인 Filter 유형은 다음과 같습니다.

| Filter | 설명 |
|---|---|
| `RequestHeaderModifier` | 요청 Header를 추가, 수정 또는 삭제합니다. |
| `ResponseHeaderModifier` | 응답 Header를 추가, 수정 또는 삭제합니다. |
| `RequestRedirect` | 요청을 다른 URL로 리다이렉트합니다. |
| `URLRewrite` | 요청 URL의 Host 또는 Path를 변경합니다. |
| `RequestMirror` | 원본 요청을 유지하면서 별도 Backend에 요청을 복제합니다. |
| `ExtensionRef` | Controller 또는 확장 구현체가 제공하는 추가 기능을 참조합니다. |

예를 들어 요청 Header를 추가하는 설정은 다음과 같습니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: header-modifier
  namespace: default
spec:
  parentRefs:
    - name: example-gateway
  rules:
    - filters:
        - type: RequestHeaderModifier
          requestHeaderModifier:
            add:
              - name: x-gateway
                value: kubernetes
      backendRefs:
        - name: backend-service
          port: 8080
```

이 설정에서는 Backend로 요청을 전달하기 전에 `x-gateway: kubernetes` Header를 추가합니다.

Filter의 종류와 지원 여부는 Gateway API 버전 및 Controller 구현의 지원 범위를 확인해야 합니다.

## 7. 기타 Route 리소스

Gateway API는 HTTPRoute 외에도 다양한 프로토콜을 처리하기 위한 Route 리소스를 제공합니다.

각 리소스는 지원하는 프로토콜과 라우팅 기준이 다르며, 안정성 수준도 구분됩니다.

### 7.1 GRPCRoute

**GRPCRoute**는 gRPC 요청을 Backend Service로 전달하기 위한 라우팅 규칙을 정의합니다.

gRPC는 HTTP/2를 기반으로 동작하며, 일반 HTTP 요청과 달리 서비스 및 메서드 정보를 이용하여 요청을 구분할 수 있습니다.

다음은 특정 gRPC 서비스와 메서드를 기준으로 라우팅하는 예제입니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: grpc-route
  namespace: default
spec:
  parentRefs:
    - name: example-gateway
  hostnames:
    - grpc.example.com
  rules:
    - matches:
        - method:
            service: example.UserService
            method: GetUser
      backendRefs:
        - name: user-grpc-service
          port: 50051
```

위 설정에서는 `grpc.example.com`으로 들어오는 `example.UserService/GetUser` 요청을 `user-grpc-service`의 50051번 포트로 전달합니다.

GRPCRoute는 gRPC 서비스의 메서드 단위 라우팅을 구현할 때 활용할 수 있습니다.

### 7.2 TLSRoute

**TLSRoute**는 TLS 연결의 SNI(Server Name Indication) 정보를 기반으로 트래픽을 전달하기 위한 라우팅 규칙을 정의합니다.

TLSRoute는 HTTP Header나 URL Path를 검사하지 않고 TLS 연결의 정보를 기준으로 라우팅합니다.

다음은 TLS Listener와 TLSRoute를 연결하는 개념적인 예제입니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: tls-gateway
  namespace: default
spec:
  gatewayClassName: example-gateway-class
  listeners:
    - name: tls-passthrough
      protocol: TLS
      port: 443
      hostname: secure.example.com
      tls:
        mode: Passthrough
      allowedRoutes:
        kinds:
          - kind: TLSRoute
```

TLSRoute는 다음과 같이 정의할 수 있습니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: TLSRoute
metadata:
  name: secure-route
  namespace: default
spec:
  parentRefs:
    - name: tls-gateway
      sectionName: tls-passthrough
  hostnames:
    - secure.example.com
  rules:
    - backendRefs:
        - name: secure-backend
          port: 443
```

위 구성에서는 Gateway가 TLS 연결을 종료하지 않고 암호화된 트래픽을 Backend로 전달합니다.

TLSRoute는 다음과 같은 경우에 활용할 수 있습니다.

- SNI를 기준으로 TLS 트래픽을 분리해야 하는 경우
- Gateway에서 TLS를 종료하지 않고 Backend에서 처리해야 하는 경우
- HTTP 수준의 요청 내용을 검사하지 않고 TLS 연결을 전달해야 하는 경우

### 7.3 TCPRoute

**TCPRoute**는 TCP 연결을 Backend로 전달하기 위한 라우팅 규칙을 정의합니다.

HTTP와 달리 TCP는 애플리케이션 계층의 URL이나 HTTP Header를 기준으로 요청을 구분하지 않습니다.

따라서 TCPRoute는 주로 Listener와 연결된 TCP 트래픽을 Backend로 전달하는 데 사용됩니다.

다음은 TCPRoute의 개념적인 예제입니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
metadata:
  name: tcp-route
  namespace: default
spec:
  parentRefs:
    - name: tcp-gateway
  rules:
    - backendRefs:
        - name: tcp-backend
          port: 9000
```

TCPRoute는 현재 Experimental Channel에 포함되는 리소스이므로 설치한 Gateway API CRD와 Controller가 해당 API 버전을 지원하는지 확인해야 합니다.

### 7.4 UDPRoute

**UDPRoute**는 UDP 트래픽을 Backend로 전달하기 위한 라우팅 규칙을 정의합니다.

UDP는 TCP와 달리 연결 설정 및 연결 상태 유지가 없는 비연결형 프로토콜입니다.

따라서 UDPRoute는 UDP 기반 서비스에 대한 트래픽 전달을 구성할 때 활용할 수 있습니다.

다음은 UDPRoute의 개념적인 예제입니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: UDPRoute
metadata:
  name: udp-route
  namespace: default
spec:
  parentRefs:
    - name: udp-gateway
  rules:
    - backendRefs:
        - name: udp-backend
          port: 5353
```

UDPRoute 역시 Experimental Channel에 포함되는 리소스이므로 실제 사용 전에 CRD와 Controller의 지원 여부를 확인해야 합니다.

### 7.5 Route 리소스별 안정성

Gateway API는 리소스의 안정성에 따라 Standard Channel과 Experimental Channel을 구분합니다.

| Route 리소스 | Channel | 설명 |
|---|---|---|
| HTTPRoute | Standard | HTTP/HTTPS 요청 라우팅 |
| GRPCRoute | Standard | gRPC 요청 라우팅 |
| TLSRoute | Standard | TLS SNI 기반 라우팅 |
| TCPRoute | Experimental | TCP 트래픽 전달 |
| UDPRoute | Experimental | UDP 트래픽 전달 |

Experimental Channel의 리소스와 필드는 향후 버전에서 호환성이 깨지는 변경이 발생하거나 제거될 수 있으므로 운영 환경에서는 사용에 주의해야 합니다.

## 8. ReferenceGrant

**ReferenceGrant**는 서로 다른 Namespace에 존재하는 Kubernetes 리소스 간의 참조를 허용하기 위한 리소스입니다.

Gateway API에서는 Route가 다른 Namespace의 Service를 Backend로 참조하거나, Gateway가 다른 Namespace의 Secret을 참조하는 구성이 필요할 수 있습니다.

이러한 교차 Namespace 참조(Cross-Namespace Reference)는 보안상의 이유로 기본적으로 제한되며, 참조 대상 Namespace에서 ReferenceGrant를 생성하여 명시적으로 허용해야 합니다.

### 8.1 ReferenceGrant의 주요 필드

| 필드 | 설명 |
|---|---|
| `metadata.namespace` | 참조를 허용할 대상 리소스가 존재하는 Namespace입니다. |
| `spec.from` | 참조를 허용할 출발지 리소스를 지정합니다. |
| `spec.to` | 참조를 허용할 대상 리소스의 종류를 지정합니다. |

ReferenceGrant는 참조를 허용하는 대상 리소스가 존재하는 Namespace에 생성해야 합니다.

### 8.2 교차 Namespace Service 참조

다음과 같은 구성을 가정하겠습니다.

- `default` Namespace: HTTPRoute가 존재합니다.
- `backend` Namespace: Backend Service가 존재합니다.
- HTTPRoute가 `backend` Namespace의 Service를 참조합니다.

먼저 `default` Namespace의 HTTPRoute에서 다른 Namespace의 Service를 참조합니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: cross-namespace-route
  namespace: default
spec:
  parentRefs:
    - name: example-gateway
  hostnames:
    - example.com
  rules:
    - backendRefs:
        - name: backend-service
          namespace: backend
          port: 8080
```

다음으로 Service가 존재하는 `backend` Namespace에 ReferenceGrant를 생성합니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-frontend-route
  namespace: backend
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: default
  to:
    - group: ""
      kind: Service
      name: backend-service
```

각 필드의 의미는 다음과 같습니다.

- `from`: `default` Namespace의 HTTPRoute가 참조할 수 있도록 허용합니다.
- `to`: `backend` Namespace의 `backend-service` Service에 대한 참조를 허용합니다.
- `group: ""`: Kubernetes Core API 그룹의 리소스를 의미합니다.
- `kind: Service`: 참조 대상이 Service임을 지정합니다.

위 예제는 Gateway API v1.7.0 기준 ReferenceGrant의 API 버전인 `v1beta1`을 사용합니다.

ReferenceGrant가 생성되면 해당 HTTPRoute는 다른 Namespace에 존재하는 Backend Service를 참조할 수 있습니다.

**주의사항:** ReferenceGrant는 참조를 허용하는 대상 Namespace에서 관리해야 합니다. 출발지 Namespace에 생성하는 것이 아닙니다.

## 9. Gateway API 설치

Gateway API는 Kubernetes 기본 API에 포함되어 있지 않으므로 CRD와 Gateway Controller를 별도로 설치해야 합니다.

설치 과정은 크게 다음 두 단계로 구분됩니다.

1. Gateway API CRD 설치
2. Gateway Controller 설치 및 구성

### 9.1 Gateway API CRD 설치

Gateway API는 기능의 안정성에 따라 Standard Channel과 Experimental Channel을 제공합니다.

| Channel | 설명 |
|---|---|
| Standard | GA 또는 Beta 단계로 안정화된 리소스와 필드를 포함합니다. |
| Experimental | Standard Channel의 모든 기능과 아직 안정화되지 않은 리소스 및 필드를 포함합니다. |

Gateway API v1.7.0 기준 설치 명령은 다음과 같습니다.

**Standard Channel**

```bash
kubectl apply --server-side -f \
  https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.7.0/standard-install.yaml
```

**Experimental Channel**

```bash
kubectl apply --server-side -f \
  https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.7.0/experimental-install.yaml
```

Standard Channel에는 GatewayClass, Gateway, HTTPRoute, GRPCRoute, TLSRoute, ReferenceGrant 등의 안정화된 리소스가 포함됩니다.

Experimental Channel에는 Standard Channel의 리소스에 더해 TCPRoute, UDPRoute 등의 실험적 리소스와 필드가 포함됩니다.

Experimental Channel은 향후 버전에서 호환성이 깨지는 변경이 발생하거나 리소스가 제거될 수 있으므로 운영 환경에서는 Standard Channel을 우선 사용하는 것이 좋습니다.

### 9.2 Gateway Controller 설치

CRD를 설치한 후에는 실제 Gateway API를 구현하는 Controller를 설치해야 합니다.

대표적인 구현체는 다음과 같습니다.

| 구현체 | 설명 |
|---|---|
| Envoy Gateway | Envoy Proxy를 기반으로 Gateway API를 구현합니다. |
| Istio | Service Mesh 기능과 함께 Gateway API를 지원합니다. |
| NGINX Gateway Fabric | NGINX 기반으로 Gateway API를 구현합니다. |
| Cilium | eBPF 기반 네트워킹과 Gateway API 기능을 제공합니다. |
| Kong | Kong Gateway를 기반으로 Gateway API를 구현합니다. |

Controller마다 지원하는 Gateway API 버전과 기능 범위가 다르므로 설치 전에 공식 호환성 문서를 확인해야 합니다.

## 10. Gateway API 상태 확인

Gateway API는 리소스 생성 여부뿐만 아니라 Controller가 실제로 설정을 수락하고 적용했는지 확인하는 것이 중요합니다.

이를 위해 GatewayClass, Gateway, Route 리소스의 Status와 Conditions를 확인합니다.

### 10.1 GatewayClass 상태

GatewayClass는 Controller가 해당 Class를 정상적으로 관리할 수 있는지 나타내는 상태를 제공합니다.

대표적인 Condition은 다음과 같습니다.

| Condition | 설명 |
|---|---|
| `Accepted` | GatewayClass가 Controller에 의해 수락되었는지 나타냅니다. |

### 10.2 Gateway 상태

Gateway는 Listener 및 네트워크 인프라 구성의 적용 상태를 제공합니다.

대표적인 Condition은 다음과 같습니다.

| Condition | 설명 |
|---|---|
| `Accepted` | Gateway 설정이 정상적으로 수락되었는지 나타냅니다. |
| `Programmed` | Gateway의 설정이 실제 데이터 플레인에 반영되었는지 나타냅니다. |

Gateway의 Listener별 상태도 별도로 확인할 수 있습니다.

### 10.3 HTTPRoute 상태

HTTPRoute는 Gateway에 정상적으로 연결되었는지, 라우팅 설정이 수락되었는지 등의 정보를 제공합니다.

대표적인 Condition은 다음과 같습니다.

| Condition | 설명 |
|---|---|
| `Accepted` | Route의 설정이 정상적으로 수락되었는지 나타냅니다. |
| `ResolvedRefs` | 참조하는 Gateway 및 Backend 등의 리소스가 정상적으로 확인되었는지 나타냅니다. |

예를 들어 존재하지 않는 Service를 Backend로 지정하거나 교차 Namespace 참조에 필요한 ReferenceGrant가 없는 경우 `ResolvedRefs`가 False 상태가 될 수 있습니다.

### 10.4 상태 조회 명령어

GatewayClass 조회:

```bash
kubectl get gatewayclass
kubectl describe gatewayclass example-gateway-class
```

Gateway 조회:

```bash
kubectl get gateway -A
kubectl describe gateway example-gateway
```

HTTPRoute 조회:

```bash
kubectl get httproute -A
kubectl describe httproute frontend-route
```

Gateway의 상세 Status 확인:

```bash
kubectl get gateway example-gateway -o yaml
```

HTTPRoute의 상세 Status 확인:

```bash
kubectl get httproute frontend-route -o yaml
```

Controller가 Gateway 설정을 정상적으로 반영하지 못하는 경우 Gateway와 Route의 Status Conditions를 확인하고 Controller 로그를 함께 살펴보는 것이 좋습니다.

## 11. Ingress와 Gateway API 비교

Gateway API는 기존 Ingress API의 한계를 보완하기 위해 설계된 확장 가능한 네트워킹 API입니다.

두 API는 모두 외부 트래픽을 Kubernetes 내부 Service로 전달하는 데 사용할 수 있지만, 리소스 구조와 제공하는 기능에서 차이가 있습니다.

| 구분 | Ingress | Gateway API |
|---|---|---|
| 주요 목적 | HTTP/HTTPS 라우팅 | L4/L7 트래픽 관리 |
| 리소스 구조 | Ingress 중심 | GatewayClass, Gateway, Route 등으로 분리 |
| Controller 연결 | IngressClass | GatewayClass |
| 트래픽 진입점 | Ingress Controller가 구현 | Gateway |
| 라우팅 규칙 | Ingress에 포함 | 별도의 Route 리소스 |
| Host 기반 라우팅 | 지원 | 지원 |
| Path 기반 라우팅 | 지원 | 지원 |
| Header 기반 매칭 | 표준 API에서 미지원 | HTTPRoute에서 지원 |
| Query Parameter 매칭 | 표준 API에서 미지원 | HTTPRoute에서 지원 |
| 가중치 기반 라우팅 | Controller별 확장 기능 | HTTPRoute에서 지원 |
| gRPC 라우팅 | Controller별 확장 기능 | GRPCRoute |
| TCP/UDP 라우팅 | Controller별 확장 기능 | 관련 Route 리소스 제공 |
| 역할 분리 | 상대적으로 제한적 | 인프라와 애플리케이션 설정 분리 |
| 확장 방식 | Annotation 및 CRD 중심 | 표준화된 확장 구조 제공 |

### 11.1 리소스 구조 비교

Ingress는 하나의 리소스에서 외부 요청의 라우팅 규칙을 정의합니다.

```text
IngressClass
     |
     v
Ingress Controller
     |
     v
Ingress
  - Host
  - Path
  - TLS
     |
     v
Service
     |
     v
Pod
```

Gateway API는 네트워크 인프라와 라우팅 규칙을 여러 리소스로 분리합니다.

```text
GatewayClass
     |
     v
Gateway Controller
     |
     v
Gateway
  - Listener
  - Port
  - Protocol
  - TLS
     |
     v
HTTPRoute / GRPCRoute
  - Hostname
  - Path
  - Header
  - Backend
     |
     v
Service
     |
     v
Pod
```

### 11.2 Gateway API를 사용하는 것이 적합한 경우

다음과 같은 환경에서는 Gateway API를 우선적으로 검토할 수 있습니다.

- 복잡한 HTTP 라우팅 정책이 필요한 경우
- Canary 배포 및 가중치 기반 트래픽 분배가 필요한 경우
- HTTP와 gRPC 등 여러 프로토콜을 함께 관리해야 하는 경우
- 네트워크 인프라 운영자와 애플리케이션 개발자의 역할을 분리해야 하는 경우
- 여러 Namespace의 애플리케이션이 공통 Gateway를 공유하는 경우
- Controller별 Annotation 의존성을 줄이고 표준화된 네트워크 구성을 원하는 경우

기존 Ingress 기반 환경은 계속 사용할 수 있으므로 Gateway API 도입을 위해 반드시 모든 Ingress를 즉시 마이그레이션할 필요는 없습니다.

## 12. Gateway API 확인 및 관리 명령어

Gateway API 리소스를 조회하고 관리할 때 사용하는 대표적인 kubectl 명령어는 다음과 같습니다.

### 12.1 GatewayClass 목록 조회

```bash
kubectl get gatewayclass
```

### 12.2 Gateway 목록 조회

```bash
kubectl get gateway -A
```

### 12.3 특정 Gateway 상세 조회

```bash
kubectl describe gateway example-gateway
```

### 12.4 HTTPRoute 목록 조회

```bash
kubectl get httproute -A
```

### 12.5 GRPCRoute 목록 조회

```bash
kubectl get grpcroute -A
```

### 12.6 TLSRoute 목록 조회

```bash
kubectl get tlsroute -A
```

### 12.7 ReferenceGrant 목록 조회

```bash
kubectl get referencegrant -A
```

### 12.8 Gateway API CRD 확인

```bash
kubectl get crds | grep gateway.networking.k8s.io
```

### 12.9 Gateway Controller Pod 확인

```bash
kubectl get pods -A
```

### 12.10 Gateway API 리소스 YAML 조회

```bash
kubectl get gateway example-gateway -o yaml

kubectl get httproute frontend-route -o yaml
```

### 12.11 Gateway API 리소스 삭제

```bash
kubectl delete httproute frontend-route

kubectl delete gateway example-gateway
```

Gateway API 리소스를 삭제하더라도 참조하는 Service와 Pod가 자동으로 삭제되지는 않습니다.

## 13. Gateway API 사용 시 주의사항

Gateway API를 운영 환경에서 사용할 때는 다음 사항을 고려해야 합니다.

1. **CRD 설치:** Gateway API는 기본 Kubernetes API에 포함되어 있지 않으므로 별도로 CRD를 설치해야 합니다.
2. **Controller 설치:** CRD만 설치해서는 트래픽을 처리할 수 없습니다. Gateway API를 구현하는 Controller가 필요합니다.
3. **Controller별 기능 지원:** Gateway Controller마다 지원하는 API 리소스와 기능이 다르므로 호환성 및 Conformance 정보를 확인해야 합니다.
4. **API Channel 구분:** Standard와 Experimental Channel의 안정성 수준이 다르므로 운영 환경에서는 Standard Channel을 우선 사용하는 것이 바람직합니다.
5. **Namespace 접근 제어:** `allowedRoutes`와 ReferenceGrant를 활용하여 서로 다른 Namespace 간의 리소스 연결을 적절하게 제한해야 합니다.
6. **Gateway 상태 확인:** Gateway가 생성되었다고 해서 실제 네트워크 구성이 완료된 것은 아닙니다. `Accepted`와 `Programmed` 등의 상태를 확인해야 합니다.
7. **Route 연결 상태 확인:** Route의 `Accepted`와 `ResolvedRefs` Condition을 확인하여 Gateway 연결 및 Backend 참조가 정상적인지 검증해야 합니다.
8. **TLS 설정 관리:** HTTPS Listener를 사용하는 경우 인증서와 개인 키를 안전하게 관리해야 합니다.
9. **Controller별 구현 차이:** 동일한 Gateway API 설정이라도 Controller의 지원 기능과 동작 방식에 따라 결과가 달라질 수 있습니다.
10. **교차 Namespace 참조:** 다른 Namespace의 Service나 Secret을 참조할 때는 대상 Namespace에 ReferenceGrant가 필요한지 확인해야 합니다.

## 14. 정리

Gateway API는 Kubernetes에서 네트워크 트래픽을 관리하기 위한 확장 가능하고 역할 지향적인 네트워킹 API입니다.

기존 Ingress API의 제한적인 라우팅 기능을 보완하며, 네트워크 인프라와 애플리케이션의 라우팅 정책을 별도의 리소스로 분리하여 관리할 수 있도록 합니다.

Gateway API의 주요 특징은 다음과 같습니다.

- GatewayClass를 통해 Gateway를 구현하는 Controller를 지정합니다.
- Gateway를 통해 실제 네트워크 트래픽을 수신할 진입점을 정의합니다.
- Listener를 통해 포트, 프로토콜, Hostname 및 TLS 설정을 관리합니다.
- HTTPRoute를 통해 Host, Path, Header, Query Parameter 기반 라우팅을 수행합니다.
- GRPCRoute를 통해 gRPC 서비스의 메서드 단위 라우팅을 지원합니다.
- TLSRoute를 통해 TLS 연결의 SNI 기반 라우팅을 지원합니다.
- TCPRoute와 UDPRoute를 통해 다양한 네트워크 프로토콜을 처리할 수 있습니다.
- 가중치 기반 트래픽 분배를 통해 Canary 배포와 같은 트래픽 제어를 구현할 수 있습니다.
- ReferenceGrant를 통해 Namespace 간 리소스 참조를 명시적으로 허용할 수 있습니다.
- Gateway Controller가 리소스의 설정을 실제 네트워크 인프라에 반영합니다.

Gateway API의 주요 리소스 관계는 다음과 같습니다.

| 리소스 | 역할 |
|---|---|
| GatewayClass | Gateway를 구현하는 Controller와 공통 설정을 정의합니다. |
| Gateway | 트래픽을 수신하는 네트워크 진입점과 Listener를 정의합니다. |
| HTTPRoute | HTTP 요청의 라우팅 규칙을 정의합니다. |
| GRPCRoute | gRPC 요청의 라우팅 규칙을 정의합니다. |
| TLSRoute | TLS 트래픽의 라우팅 규칙을 정의합니다. |
| TCPRoute | TCP 트래픽의 라우팅 규칙을 정의합니다. |
| UDPRoute | UDP 트래픽의 라우팅 규칙을 정의합니다. |
| ReferenceGrant | Namespace 간 리소스 참조를 허용합니다. |

Gateway API는 네트워크 인프라 운영자와 애플리케이션 개발자가 서로 다른 책임 범위에서 네트워크 구성을 관리할 수 있도록 설계되었습니다.

Ingress가 HTTP/HTTPS 라우팅 규칙을 정의하는 데 초점을 맞춘다면, Gateway API는 다양한 프로토콜과 세밀한 트래픽 정책을 보다 체계적으로 관리하기 위한 확장된 네트워킹 모델을 제공합니다.

---

## 공식 문서 및 참고 자료

- [Kubernetes 공식 문서 - Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)
- [Gateway API 공식 문서](https://gateway-api.sigs.k8s.io/)
- [Gateway API - API Overview](https://gateway-api.sigs.k8s.io/docs/concepts/api-overview/)
- [Gateway API - GatewayClass](https://gateway-api.sigs.k8s.io/reference/api-types/gatewayclass/)
- [Gateway API - Gateway](https://gateway-api.sigs.k8s.io/reference/api-types/gateway/)
- [Gateway API - HTTPRoute](https://gateway-api.sigs.k8s.io/reference/api-types/httproute/)
- [Gateway API - ReferenceGrant](https://gateway-api.sigs.k8s.io/reference/api-types/referencegrant/)
- [Gateway API - Versioning](https://gateway-api.sigs.k8s.io/docs/concepts/versioning/)
- [Gateway API - Conformance](https://gateway-api.sigs.k8s.io/docs/concepts/conformance/)