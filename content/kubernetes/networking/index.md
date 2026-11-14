# Kubernetes Networking Resources

Networking 리소스는 Kubernetes 클러스터 내부의 Pod 간 통신, Service Discovery 및 외부 트래픽 유입을 관리합니다.

## 1. Networking 리소스 구성

| 리소스 | 주요 역할 | 주요 특징 |
|---|---|---|
| **Service** | Pod 네트워크 접근점 제공 | ClusterIP, NodePort, LoadBalancer, ExternalName |
| **EndpointSlice** | Service의 실제 Endpoint 관리 | Pod IP 및 Port 정보 관리 |
| **Ingress** | HTTP/HTTPS 외부 트래픽 라우팅 | Host 및 Path 기반 라우팅 |
| **IngressClass** | Ingress Controller 지정 | Ingress 처리 주체 정의 |
| **GatewayClass** | Gateway Controller 유형 정의 | Gateway 구현체 지정 |
| **Gateway** | 네트워크 진입점 정의 | Listener 및 연결 정책 구성 |
| **HTTPRoute** | HTTP 트래픽 라우팅 | Host 및 Path 기반 라우팅 |
| **GRPCRoute** | gRPC 트래픽 라우팅 | gRPC 서비스 및 메서드 기반 라우팅 |
| **TCPRoute** | TCP 트래픽 라우팅 | TCP 연결 라우팅 |
| **NetworkPolicy** | Pod 네트워크 접근 제어 | Ingress 및 Egress 트래픽 제어 |

## 2. 리소스 관계

### 2.1 Service 기반 통신

```text
Client Pod
    │
    ▼
  Service
    │
    ▼
EndpointSlice
    │
    ├── Pod 1
    ├── Pod 2
    └── Pod 3
```

Service는 안정적인 네트워크 접근점을 제공하며, EndpointSlice는 실제 연결 대상 Endpoint 정보를 관리합니다.

### 2.2 Ingress 기반 외부 트래픽 유입

```text
External Client
      │
      ▼
Ingress Controller
      │
      ▼
   Ingress
      │
      ▼
   Service
      │
      ▼
     Pod
```

Ingress는 HTTP/HTTPS 트래픽의 라우팅 규칙을 정의하고, Ingress Controller가 해당 규칙을 실제로 처리합니다.

### 2.3 Gateway API 기반 외부 트래픽 유입

```text
GatewayClass
     │
     ▼
 Gateway
     │
     ▼
 HTTPRoute / GRPCRoute / TCPRoute
     │
     ▼
  Service
     │
     ▼
    Pod
```

Gateway API는 Gateway와 Route 리소스를 통해 트래픽 진입점과 라우팅 정책을 보다 세분화하여 관리합니다.

### 2.4 NetworkPolicy 기반 접근 제어

```text
Pod A ──────── Pod B
  │              ▲
  │              │
  └── NetworkPolicy
       접근 허용 및 제한
```

NetworkPolicy는 Pod를 대상으로 네트워크 Ingress 및 Egress 트래픽을 제어합니다.

## 3. 기능에 따른 분류

| 기능 | 관련 리소스 |
|---|---|
| Pod 접근 및 Service Discovery | Service, EndpointSlice |
| 외부 HTTP/HTTPS 트래픽 유입 | Ingress, IngressClass |
| 확장 가능한 트래픽 라우팅 | GatewayClass, Gateway, HTTPRoute, GRPCRoute, TCPRoute |
| Pod 네트워크 접근 제어 | NetworkPolicy |

## 4. 핵심 정리

- Service는 Pod에 안정적인 네트워크 접근점을 제공합니다.
- EndpointSlice는 Service가 연결할 실제 Endpoint를 관리합니다.
- Ingress는 HTTP/HTTPS 외부 트래픽 라우팅 규칙을 정의합니다.
- Gateway API는 Ingress보다 확장된 트래픽 관리 모델을 제공합니다.
- NetworkPolicy는 Pod 간 및 외부 네트워크 통신을 제어합니다.

참고:
- https://kubernetes.io/docs/concepts/services-networking/
- https://kubernetes.io/docs/concepts/services-networking/gateway/