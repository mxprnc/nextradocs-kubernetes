# Ingress

## 1. Ingress란?

Ingress는 Kubernetes 클러스터 외부에서 내부 Service로 HTTP 및 HTTPS 트래픽을 전달하기 위한 라우팅 규칙을 정의하는 리소스입니다.

일반적으로 Kubernetes 내부의 Pod는 Service를 통해 네트워크 접근 지점을 제공하지만, Service만으로는 도메인 기반 라우팅이나 URL 경로에 따른 트래픽 분기, TLS 종료와 같은 HTTP 계층의 라우팅 기능을 충분히 표현하기 어렵습니다.

Ingress는 이러한 요구사항을 해결하기 위해 외부 HTTP 및 HTTPS 요청을 클러스터 내부의 적절한 Service로 전달하는 규칙을 정의합니다.

예를 들어 하나의 외부 IP 주소를 통해 다음과 같이 여러 애플리케이션에 접근할 수 있습니다.

- `api.example.com` → API Service
- `www.example.com` → Frontend Service
- `example.com/admin` → Admin Service

![Kubernetes Ingress를 이용한 외부 Service 접근 구조](https://images.openai.com/static-rsc-4/uhwEq13_pT5uz2RoOyLYPs7EsAY5FRZ2jxJimDIW_svr5jdeZjptTOWyBkETVttV5dDvdAwWsJgGRUJeoEDtnexKPCsevQIBnl7soSWvadcFrNE33V1pjhoKaqh2JQU7BUwMgH0wbFqxo10vmrG1WpU6l71XaBLVJjEC6bDFxJk?purpose=inline)

Ingress는 `networking.k8s.io/v1` API 그룹에 속하는 Kubernetes 리소스입니다.

**중요:** Ingress는 라우팅 규칙을 정의하는 API 객체이며, 자체적으로 트래픽을 수신하거나 전달하는 네트워크 프록시가 아닙니다. 실제 동작을 위해서는 해당 Ingress 리소스를 처리하는 Ingress Controller가 필요합니다.

## 2. Ingress의 주요 역할

Ingress는 다음과 같은 역할을 수행합니다.

- **외부 HTTP/HTTPS 접근 제공:** 클러스터 외부에서 내부 애플리케이션에 접근할 수 있도록 합니다.
- **Host 기반 라우팅:** 요청의 도메인 이름에 따라 서로 다른 Service로 트래픽을 전달합니다.
- **Path 기반 라우팅:** URL 경로에 따라 서로 다른 Service로 트래픽을 전달합니다.
- **TLS 종료:** HTTPS 요청을 수신하고 TLS 연결을 종료한 후 내부 Service로 HTTP 트래픽을 전달할 수 있습니다.
- **외부 진입점 통합:** 여러 애플리케이션이 하나의 외부 IP 주소 또는 로드 밸런서를 공유할 수 있도록 합니다.
- **라우팅 정책 관리:** 애플리케이션별 외부 접근 규칙을 Kubernetes 리소스로 선언하고 관리할 수 있습니다.

## 3. Ingress의 동작 구조

Ingress는 일반적으로 다음과 같은 구성 요소를 통해 동작합니다.

| 구성 요소 | 설명 |
|---|---|
| Ingress | 외부 HTTP/HTTPS 요청의 라우팅 규칙을 정의합니다. |
| Ingress Controller | Ingress 리소스를 감시하고 실제 트래픽 처리 구성을 관리합니다. |
| LoadBalancer | 외부 클라이언트의 요청을 Ingress Controller로 전달하는 진입점을 제공합니다. |
| Service | Ingress Controller가 선택한 내부 애플리케이션의 네트워크 접근 지점을 제공합니다. |
| EndpointSlice | Service에 연결된 실제 Pod의 IP 주소와 포트 정보를 관리합니다. |
| Pod | 실제 애플리케이션 요청을 처리합니다. |

일반적인 트래픽 처리 과정은 다음과 같습니다.

1. 외부 클라이언트가 도메인 이름을 통해 애플리케이션에 접근합니다.
2. DNS가 도메인 이름을 Ingress Controller의 외부 IP 주소로 해석합니다.
3. 요청이 LoadBalancer 또는 NodePort 등을 거쳐 Ingress Controller에 도달합니다.
4. Ingress Controller가 요청의 Host와 Path를 확인합니다.
5. Ingress에 정의된 라우팅 규칙에 따라 대상 Service를 선택합니다.
6. Service에 연결된 Endpoint를 통해 대상 Pod로 트래픽을 전달합니다.
7. Pod가 요청을 처리하고 응답을 반환합니다.

**참고:** 실제 네트워크 구조는 Ingress Controller의 구현 방식과 배포 환경에 따라 달라질 수 있습니다. Ingress Controller가 LoadBalancer Service를 통해 노출될 수도 있고, NodePort나 호스트 네트워크를 사용할 수도 있습니다.

## 4. Ingress Controller

Ingress Controller는 Kubernetes의 Ingress 리소스를 감시하고, 정의된 라우팅 규칙을 실제 네트워크 프록시에 반영하는 컨트롤러입니다.

Ingress 리소스는 라우팅 규칙만 정의하므로 이를 처리하는 Ingress Controller가 없으면 요청을 전달할 수 없습니다.

대표적인 Ingress Controller 구현체는 다음과 같습니다.

| 구현체 | 설명 |
|---|---|
| NGINX Ingress Controller | NGINX를 기반으로 HTTP/HTTPS 트래픽을 처리하는 구현체입니다. |
| Traefik | 동적 라우팅과 미들웨어 기능을 제공하는 프록시 기반 구현체입니다. |
| HAProxy Ingress | HAProxy를 기반으로 트래픽 라우팅과 로드 밸런싱을 수행합니다. |
| Kong Ingress Controller | Kong Gateway를 기반으로 API Gateway 기능과 연계할 수 있습니다. |
| Cloud Provider Ingress Controller | 클라우드 제공자의 로드 밸런서 및 네트워크 서비스를 활용합니다. |

Ingress Controller는 Kubernetes에 기본적으로 설치되어 있지 않으므로 클러스터 환경에 맞는 구현체를 별도로 설치해야 합니다.

### 4.1 Ingress Controller의 동작 방식

Ingress Controller는 일반적으로 Kubernetes API Server를 감시하면서 Ingress 리소스의 생성, 수정 및 삭제 이벤트를 수신합니다.

Ingress 리소스가 변경되면 다음과 같은 작업을 수행합니다.

1. Ingress 리소스와 연결된 Service 및 EndpointSlice 정보를 조회합니다.
2. Host, Path, TLS 등의 설정을 분석합니다.
3. 실제 프록시에서 사용할 라우팅 설정을 생성하거나 갱신합니다.
4. 프록시가 새로운 설정을 반영하도록 구성합니다.
5. 외부 요청을 정의된 규칙에 따라 내부 Service로 전달합니다.

Ingress Controller는 구현체에 따라 프록시 설정을 동적으로 갱신하거나 별도의 설정 관리 방식을 사용할 수 있습니다.

### 4.2 Ingress Class

하나의 Kubernetes 클러스터에 여러 Ingress Controller가 설치되어 있다면, 각각의 Controller가 어떤 Ingress 리소스를 처리할지 구분할 필요가 있습니다.

이를 위해 IngressClass 리소스를 사용합니다.

Ingress 리소스의 `spec.ingressClassName` 필드에 IngressClass의 이름을 지정하면 해당 Ingress를 처리할 Controller를 선택할 수 있습니다.

```yaml
spec:
  ingressClassName: nginx
```

IngressClass는 Ingress와 Ingress Controller 사이의 연결 관계를 정의하는 리소스입니다.

IngressClass의 자세한 구조와 역할은 별도의 `ingress-class.md` 문서에서 설명합니다.

## 5. Ingress 리소스 구조

Ingress는 `networking.k8s.io/v1` API를 사용하며, 주요 필드는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `spec.ingressClassName` | 사용할 IngressClass를 지정합니다. |
| `spec.rules` | Host 및 Path 기반 라우팅 규칙을 정의합니다. |
| `spec.tls` | HTTPS 통신에 사용할 TLS 설정을 정의합니다. |
| `spec.defaultBackend` | 일치하는 라우팅 규칙이 없을 때 사용할 기본 Backend를 지정합니다. |

### 5.1 기본 Ingress 예제

다음은 `api.example.com`으로 들어오는 요청을 `backend-service`로 전달하는 Ingress 예제입니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80
```

각 필드의 의미는 다음과 같습니다.

- `ingressClassName`: `nginx`라는 IngressClass를 사용합니다.
- `rules`: 외부 요청을 처리할 라우팅 규칙을 정의합니다.
- `host`: `api.example.com` 도메인으로 들어오는 요청을 대상으로 합니다.
- `path`: `/` 경로로 시작하는 요청을 대상으로 합니다.
- `pathType`: 경로를 Prefix 방식으로 비교합니다.
- `backend.service.name`: 요청을 전달할 Service 이름입니다.
- `backend.service.port.number`: 대상 Service의 포트 번호입니다.

위 설정에 따라 다음 요청은 `backend-service`로 전달됩니다.

```text
http://api.example.com/
http://api.example.com/users
http://api.example.com/orders
```

Ingress가 참조하는 Backend Service는 일반적으로 동일한 Namespace에 존재해야 합니다.

## 6. Host 기반 라우팅

Ingress는 HTTP 요청의 Host 헤더를 기준으로 서로 다른 Service에 트래픽을 전달할 수 있습니다.

이를 통해 여러 도메인을 하나의 Ingress Controller에서 처리할 수 있습니다.

예를 들어 다음과 같은 구성을 만들 수 있습니다.

| Host | Backend Service |
|---|---|
| `api.example.com` | api-service |
| `www.example.com` | frontend-service |
| `admin.example.com` | admin-service |

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-routing-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80

    - host: www.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80

    - host: admin.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 80
```

위 예제에서는 하나의 Ingress 리소스가 세 개의 도메인을 처리합니다.

이러한 방식은 여러 애플리케이션이 동일한 외부 진입점을 공유하면서도 독립적인 도메인을 사용할 수 있도록 합니다.

## 7. Path 기반 라우팅

Ingress는 동일한 도메인 내에서도 URL 경로에 따라 서로 다른 Service로 트래픽을 전달할 수 있습니다.

예를 들어 다음과 같은 라우팅 구성을 만들 수 있습니다.

| Host | Path | Backend Service |
|---|---|---|
| `example.com` | `/api` | api-service |
| `example.com` | `/admin` | admin-service |
| `example.com` | `/` | frontend-service |

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-routing-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80

          - path: /admin
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 80

          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

위 설정에서는 URL 경로에 따라 서로 다른 Service가 선택됩니다.

```text
http://example.com/api/users
    → api-service

http://example.com/admin/dashboard
    → admin-service

http://example.com/products
    → frontend-service
```

Ingress는 경로의 일치 방식을 지정하기 위해 `pathType`을 사용합니다.

### 7.1 pathType

`pathType`은 요청 URL의 경로와 Ingress에 정의된 경로를 비교하는 방식을 지정합니다.

Kubernetes Ingress API는 다음 세 가지 방식을 지원합니다.

| 유형 | 설명 |
|---|---|
| `Exact` | URL 경로가 지정된 경로와 정확히 일치해야 합니다. |
| `Prefix` | URL 경로가 지정된 경로의 경로 요소 단위 Prefix와 일치해야 합니다. |
| `ImplementationSpecific` | 실제 경로 매칭 방식은 Ingress Controller 구현체에 따라 달라집니다. |

**Exact**

지정된 경로와 요청 경로가 정확하게 일치해야 합니다.

```yaml
path: /api
pathType: Exact
```

다음과 같이 동작합니다.

| 요청 경로 | 일치 여부 |
|---|---|
| `/api` | O |
| `/api/` | X |
| `/api/users` | X |
| `/apiv2` | X |

**Prefix**

지정된 경로가 요청 경로의 시작 부분과 일치해야 합니다. 단, 단순 문자열 Prefix가 아니라 `/`로 구분되는 경로 요소 단위로 비교합니다.

```yaml
path: /api
pathType: Prefix
```

다음과 같이 동작합니다.

| 요청 경로 | 일치 여부 |
|---|---|
| `/api` | O |
| `/api/` | O |
| `/api/users` | O |
| `/api/v1/users` | O |
| `/apiv2` | X |

**ImplementationSpecific**

경로 매칭 방식이 Ingress Controller의 구현에 따라 달라집니다.

```yaml
path: /api
pathType: ImplementationSpecific
```

특정 Ingress Controller가 제공하는 확장적인 경로 매칭 기능을 사용해야 할 때 활용할 수 있습니다.

다만 Controller를 변경하면 동작이 달라질 수 있으므로 이식성이 중요한 환경에서는 `Exact` 또는 `Prefix`를 사용하는 것이 바람직합니다.

### 7.2 여러 경로가 일치하는 경우

하나의 요청 경로가 여러 Ingress 규칙과 일치할 수 있습니다.

Kubernetes Ingress API는 다음과 같은 우선순위를 정의합니다.

1. 가장 긴 경로가 우선합니다.
2. 길이가 동일한 경우 `Exact`가 `Prefix`보다 우선합니다.

예를 들어 다음과 같은 규칙이 존재한다고 가정하겠습니다.

| Path | Type | Backend |
|---|---|---|
| `/api` | Prefix | api-service |
| `/api/users` | Prefix | user-service |
| `/api/users` | Exact | exact-user-service |

`/api/users` 요청은 `exact-user-service`로 전달됩니다.

`/api/users/123` 요청은 가장 긴 일치 Prefix인 `/api/users`를 기준으로 `user-service`로 전달됩니다.

## 8. TLS 설정

Ingress는 TLS 설정을 통해 외부 클라이언트와 Ingress Controller 사이의 HTTPS 통신을 지원합니다.

일반적으로 TLS 인증서와 개인 키를 Kubernetes Secret에 저장하고, Ingress의 `spec.tls`에서 해당 Secret을 참조합니다.

### 8.1 TLS Secret 생성

먼저 TLS 인증서와 개인 키를 포함하는 Secret을 생성합니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-tls
  namespace: default
type: kubernetes.io/tls
data:
  tls.crt: <BASE64_ENCODED_CERTIFICATE>
  tls.key: <BASE64_ENCODED_PRIVATE_KEY>
```

`kubernetes.io/tls` 유형의 Secret은 다음 두 가지 데이터를 사용합니다.

| 필드 | 설명 |
|---|---|
| `tls.crt` | TLS 인증서입니다. |
| `tls.key` | TLS 개인 키입니다. |

실제 인증서와 개인 키를 Base64로 인코딩하여 입력하거나, 다음 명령어로 Secret을 생성할 수도 있습니다.

```bash
kubectl create secret tls example-tls \
  --cert=./tls.crt \
  --key=./tls.key
```

### 8.2 Ingress TLS 설정

생성한 Secret을 Ingress에서 참조합니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: example-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80
```

위 설정에서는 다음과 같이 동작합니다.

1. 외부 클라이언트가 `https://api.example.com`으로 요청합니다.
2. Ingress Controller가 TLS 인증서를 사용하여 HTTPS 연결을 처리합니다.
3. TLS 연결이 종료된 후 요청이 Backend Service로 전달됩니다.
4. Backend Pod가 HTTP 요청을 처리합니다.

일반적인 Ingress TLS 구성에서는 클라이언트와 Ingress Controller 사이에서 TLS가 종료됩니다. 따라서 Ingress Controller와 Backend Pod 사이의 통신은 별도의 설정이 없다면 평문 HTTP로 이루어집니다.

### 8.3 SNI와 다중 도메인 TLS

하나의 Ingress Controller는 여러 도메인의 HTTPS 요청을 처리할 수 있습니다.

이때 TLS 연결 과정에서 사용되는 SNI(Server Name Indication)를 통해 클라이언트가 요청하는 도메인에 적합한 인증서를 선택할 수 있습니다.

예를 들어 다음과 같이 여러 도메인과 인증서를 설정할 수 있습니다.

```yaml
spec:
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls

    - hosts:
        - www.example.com
      secretName: frontend-tls
```

각 도메인에 해당하는 인증서를 사용하여 HTTPS 연결을 처리할 수 있습니다.

단, 실제 SNI 지원과 인증서 처리 방식은 Ingress Controller 구현에 따라 차이가 있을 수 있습니다.

## 9. Default Backend

Default Backend는 Ingress에 정의된 어떤 Host 또는 Path 규칙에도 일치하지 않는 요청을 처리하는 기본 Backend입니다.

`spec.defaultBackend`를 사용하여 기본 Backend Service를 지정할 수 있습니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: default-backend-ingress
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: default-service
      port:
        number: 80
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

위 예제에서는 다음과 같이 동작합니다.

| 요청 | Backend |
|---|---|
| `api.example.com/` | api-service |
| `api.example.com/users` | api-service |
| `unknown.example.com/` | default-service |
| `unknown.example.com/products` | default-service |

`defaultBackend`를 지정하지 않은 경우 규칙에 일치하지 않는 요청을 어떻게 처리할지는 Ingress Controller의 구현에 따라 달라집니다.

**참고:** `defaultBackend`는 특정 Ingress 리소스에 정의된 규칙의 기본 처리 대상을 지정하는 필드입니다. Ingress Controller 자체에 설정된 기본 Backend와는 구분해야 합니다.

## 10. Wildcard Host

Ingress는 와일드카드 Host를 사용하여 특정 도메인의 하위 도메인에 대한 요청을 처리할 수 있습니다.

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wildcard-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: "*.example.com"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80
```

위 설정에서는 다음과 같은 요청이 매칭됩니다.

| Host | 매칭 여부 |
|---|---|
| `api.example.com` | O |
| `www.example.com` | O |
| `admin.example.com` | O |
| `example.com` | X |
| `v1.api.example.com` | X |

Kubernetes Ingress의 Wildcard Host는 하나의 DNS 레이블만 대체할 수 있습니다.

따라서 `*.example.com`은 `api.example.com`과 일치하지만, `v1.api.example.com`과는 일치하지 않습니다.

또한 Wildcard Host는 루트 도메인인 `example.com` 자체를 포함하지 않으므로, 루트 도메인까지 처리해야 한다면 별도의 Host 규칙을 정의해야 합니다.

## 11. Ingress Annotation

Ingress API는 기본적인 Host, Path, TLS 등의 라우팅 기능을 제공하지만, 모든 프록시 설정을 표준 필드로 표현하지는 않습니다.

이를 보완하기 위해 많은 Ingress Controller는 Annotation을 사용하여 구현체별 추가 기능을 제공합니다.

Annotation은 Kubernetes 리소스의 `metadata.annotations`에 Key-Value 형태로 정의합니다.

예를 들어 NGINX 기반 Ingress Controller에서는 다음과 같은 형태로 설정할 수 있습니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80
```

대표적인 Annotation 활용 영역은 다음과 같습니다.

| 영역 | 설명 |
|---|---|
| Request Body Size | 요청 본문의 최대 크기를 설정합니다. |
| Timeout | Backend 응답 대기 시간 등을 설정합니다. |
| Rewrite | 요청 URL을 다른 경로로 변환합니다. |
| Redirect | HTTP 요청을 HTTPS로 리다이렉트합니다. |
| Authentication | 인증 및 접근 제어 기능을 설정합니다. |
| Rate Limiting | 요청 횟수나 속도를 제한합니다. |
| Load Balancing | Backend 트래픽 분산 방식을 설정합니다. |

**주의사항:** Annotation의 이름과 동작은 Ingress Controller마다 다릅니다. 동일한 Annotation을 모든 Ingress Controller에서 사용할 수 있는 것은 아니며, 지원하지 않는 Annotation은 무시되거나 오류가 발생할 수 있습니다.

따라서 Annotation을 사용할 때는 선택한 Ingress Controller의 공식 문서를 확인해야 합니다.

## 12. Ingress의 한계와 Gateway API

Ingress는 Kubernetes에서 HTTP 및 HTTPS 트래픽을 외부로 노출하기 위한 대표적인 API로 사용되어 왔습니다.

하지만 다양한 네트워크 요구사항을 처리하기에는 몇 가지 한계가 있습니다.

### 12.1 Ingress API의 한계

**1. 제한적인 라우팅 기능**

Ingress API는 기본적으로 Host와 Path 기반 HTTP 라우팅을 제공합니다.

다음과 같은 세부적인 트래픽 제어 기능은 표준 API에서 직접 표현할 수 없습니다.

- HTTP Header 기반 라우팅
- Query Parameter 기반 라우팅
- 트래픽 가중치 기반 분배
- Canary 배포를 위한 정교한 트래픽 분할
- 요청 및 응답 Header 수정
- 다양한 L4 및 L7 프로토콜 처리

이러한 기능을 구현하려면 Ingress Controller별 Annotation이나 별도의 Custom Resource Definition(CRD)에 의존해야 하는 경우가 많습니다.

**2. Controller 종속성**

Ingress API가 표준화되어 있더라도 실제 동작은 Ingress Controller 구현에 따라 차이가 있습니다.

특히 Annotation을 활용한 추가 기능은 특정 Controller에 종속될 수 있으므로 Controller를 변경할 때 설정을 수정해야 할 수 있습니다.

**3. 역할과 정책의 분리 부족**

Ingress는 라우팅 규칙을 정의하는 하나의 리소스에 기능이 집중되어 있습니다.

클러스터 운영자와 애플리케이션 개발자가 네트워크 인프라, Gateway 구성, 라우팅 정책을 독립적으로 관리해야 하는 환경에서는 역할을 분리하기 어렵습니다.

### 12.2 Gateway API와의 관계

Gateway API는 기존 Ingress API의 한계를 보완하고 보다 확장성 있는 네트워크 트래픽 관리를 제공하기 위해 도입된 Kubernetes 네트워킹 API입니다.

Ingress가 비교적 단순한 HTTP/HTTPS 라우팅 규칙을 정의하는 데 초점을 맞춘다면, Gateway API는 네트워크 인프라와 라우팅 정책을 여러 리소스로 분리하여 관리할 수 있도록 설계되었습니다.

| 구분 | Ingress | Gateway API |
|---|---|---|
| 주요 목적 | HTTP/HTTPS 라우팅 | L4/L7 트래픽 관리 |
| 리소스 구조 | Ingress 중심 | GatewayClass, Gateway, Route 등으로 분리 |
| Host 기반 라우팅 | 지원 | 지원 |
| Path 기반 라우팅 | 지원 | 지원 |
| Header 기반 라우팅 | 표준 API에서 미지원 | 지원 |
| 트래픽 가중치 | 표준 API에서 미지원 | 지원 |
| 역할 분리 | 상대적으로 제한적 | 인프라와 애플리케이션 설정 분리 |
| 확장성 | Controller별 확장에 의존 | 표준화된 확장 구조 제공 |

Gateway API는 다음과 같은 리소스를 중심으로 동작합니다.

- GatewayClass: Gateway를 구현하는 Controller와 구현 정책을 정의합니다.
- Gateway: 외부 트래픽을 수신하는 네트워크 진입점을 정의합니다.
- HTTPRoute: HTTP 요청의 라우팅 규칙을 정의합니다.
- GRPCRoute: gRPC 요청의 라우팅 규칙을 정의합니다.
- TCPRoute: TCP 트래픽의 라우팅 규칙을 정의합니다.
- TLSRoute: TLS 트래픽의 라우팅 규칙을 정의합니다.
- UDPRoute: UDP 트래픽의 라우팅 규칙을 정의합니다.

Gateway API의 자세한 구조와 동작 방식은 별도의 `gateway-api.md` 문서에서 설명합니다.

![Kubernetes Gateway API의 리소스 구조와 네트워크 역할 분리](https://images.openai.com/static-rsc-4/YB35_WgSz7PhL1sjgr43V3XdlonHwZSj1VfzUCoog6ennNZB_t7SHYhMINbIj3FfoDsyYwZEYom3Bz6Vj9_6gLFm2LjdDodvTC4XVVGcxtqudfrCxkh8HCcpMdb6aIoVaVT0XvuGoZmvQvaTkM3d5nZe-iFbD97V28Kh_BbGBBY?purpose=inline)

### 12.3 Ingress API의 현재 상태

Kubernetes 공식 프로젝트는 Ingress API를 더 이상 발전시키지 않고 동결(Feature Frozen) 상태로 유지하고 있으며, 신규 기능과 확장은 Gateway API를 중심으로 진행하고 있습니다.

Ingress API는 여전히 Stable API로 지원되며 Kubernetes에서 제거할 계획은 없습니다.

따라서 기존 Ingress 기반 시스템을 즉시 변경할 필요는 없지만, 새로운 네트워크 아키텍처를 설계하거나 복잡한 라우팅 정책이 필요한 경우에는 Gateway API를 우선적으로 검토하는 것이 바람직합니다.

## 13. Ingress 확인 및 관리 명령어

Ingress를 조회하고 관리할 때 사용하는 대표적인 kubectl 명령어는 다음과 같습니다.

### Ingress 목록 조회

```bash
kubectl get ingress
```

또는 다음과 같이 축약할 수 있습니다.

```bash
kubectl get ing
```

### 모든 Namespace의 Ingress 조회

```bash
kubectl get ingress -A
```

### 특정 Ingress 상세 조회

```bash
kubectl describe ingress backend-ingress
```

### Ingress YAML 조회

```bash
kubectl get ingress backend-ingress -o yaml
```

### IngressClass 목록 조회

```bash
kubectl get ingressclass
```

### Ingress Controller Pod 확인

```bash
kubectl get pods -A
```

Ingress Controller가 설치된 Namespace를 알고 있다면 다음과 같이 조회할 수 있습니다.

```bash
kubectl get pods -n ingress-nginx
```

### Ingress Controller Service 확인

```bash
kubectl get svc -n ingress-nginx
```

### Ingress 이벤트 확인

```bash
kubectl describe ingress backend-ingress
```

Ingress Controller가 정상적으로 설정을 반영하지 못하는 경우 Ingress의 Events와 Controller Pod 로그를 함께 확인하는 것이 좋습니다.

```bash
kubectl logs -n ingress-nginx <INGRESS_CONTROLLER_POD>
```

### Ingress 삭제

```bash
kubectl delete ingress backend-ingress
```

Ingress를 삭제하더라도 연결된 Service와 Pod는 삭제되지 않습니다. Ingress는 외부 트래픽의 라우팅 규칙을 관리하는 별도의 리소스입니다.

## 14. Ingress 사용 시 주의사항

Ingress를 운영 환경에서 사용할 때는 다음 사항을 고려해야 합니다.

1. **Ingress Controller 설치:** Ingress 리소스만 생성해서는 트래픽을 처리할 수 없습니다. 반드시 해당 리소스를 처리하는 Ingress Controller가 실행 중이어야 합니다.
2. **IngressClass 지정:** 여러 Ingress Controller를 사용하는 환경에서는 `ingressClassName`을 명시하여 처리 주체를 구분하는 것이 좋습니다.
3. **DNS 설정:** 외부 도메인이 Ingress Controller의 외부 IP 또는 LoadBalancer 주소를 가리키도록 DNS를 구성해야 합니다.
4. **TLS 인증서 관리:** HTTPS를 사용하는 경우 인증서의 유효기간, 갱신 및 개인 키 보안을 관리해야 합니다.
5. **Controller별 Annotation 확인:** Annotation은 구현체마다 지원 여부와 동작이 다르므로 공식 문서를 확인해야 합니다.
6. **Backend Service 확인:** Ingress에서 참조하는 Service와 포트가 실제로 존재하고 정상적으로 Endpoint를 제공하는지 확인해야 합니다.
7. **라우팅 규칙 충돌 방지:** 여러 Ingress 리소스가 동일한 Host와 Path를 정의하는 경우 Controller별 처리 방식에 따라 예상하지 못한 결과가 발생할 수 있으므로 규칙을 명확하게 관리해야 합니다.
8. **네트워크 접근 제어:** Ingress는 외부 트래픽의 라우팅을 담당하지만, 애플리케이션의 세부적인 네트워크 접근 제어는 NetworkPolicy 등 별도의 리소스를 통해 구성해야 합니다.

## 15. 정리

Ingress는 Kubernetes 클러스터 외부에서 내부 Service로 HTTP 및 HTTPS 트래픽을 전달하기 위한 라우팅 규칙을 정의하는 리소스입니다.

Ingress Controller와 함께 동작하며 Host, Path, TLS 등을 기반으로 외부 요청을 적절한 Service에 연결합니다.

Ingress의 주요 특징은 다음과 같습니다.

- 외부 HTTP/HTTPS 트래픽을 클러스터 내부 Service로 전달합니다.
- Host 기반 라우팅을 통해 여러 도메인을 하나의 진입점에서 처리할 수 있습니다.
- Path 기반 라우팅을 통해 URL 경로별로 서로 다른 Service에 요청을 전달할 수 있습니다.
- TLS 설정을 통해 HTTPS 통신과 인증서 관리를 지원합니다.
- Ingress Controller가 실제 라우팅 및 프록시 구성을 담당합니다.
- IngressClass를 통해 여러 Ingress Controller 중 사용할 구현체를 지정할 수 있습니다.
- Annotation을 통해 Controller별 추가 기능을 활용할 수 있습니다.

Ingress와 관련된 주요 리소스의 관계는 다음과 같습니다.

| 리소스 | 역할 |
|---|---|
| Ingress | 외부 HTTP/HTTPS 요청의 라우팅 규칙을 정의합니다. |
| IngressClass | Ingress를 처리할 Controller의 종류와 설정을 정의합니다. |
| Ingress Controller | Ingress 규칙을 실제 네트워크 프록시에 반영합니다. |
| Service | 요청을 전달받을 애플리케이션의 네트워크 접근 지점을 제공합니다. |
| EndpointSlice | Service에 연결된 실제 Pod의 네트워크 정보를 관리합니다. |

Ingress는 Kubernetes에서 외부 HTTP/HTTPS 트래픽을 관리하는 대표적인 리소스이지만, API 자체의 확장과 발전은 동결된 상태입니다.

기존 Ingress 기반 환경은 계속 사용할 수 있으며, 신규 환경이나 보다 복잡한 트래픽 관리가 필요한 경우에는 Gateway API를 검토하는 것이 바람직합니다.

## 공식 문서 및 참고 자료

- [Kubernetes 공식 문서 - Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kubernetes 공식 문서 - Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
- [Kubernetes API Reference - Ingress v1](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/ingress-v1/)
- [Kubernetes 공식 문서 - Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)