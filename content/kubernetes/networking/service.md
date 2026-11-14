
# Service

## 1. Service란?

Service는 Kubernetes 클러스터 내부에서 실행되는 하나 이상의 Pod에 네트워크 접근을 제공하는 리소스입니다.

Pod는 생성되고 삭제되는 과정에서 IP 주소가 변경될 수 있으며, Deployment나 ReplicaSet에 의해 관리되는 Pod의 수도 상황에 따라 달라질 수 있습니다. 이러한 특성 때문에 특정 Pod의 IP 주소를 직접 참조하는 방식은 안정적인 서비스 통신을 보장하기 어렵습니다.

Service는 이러한 문제를 해결하기 위해 Pod 집합에 대한 안정적인 네트워크 접근 지점을 제공합니다.

Service는 일반적으로 Label Selector를 사용하여 대상 Pod를 식별하며, 해당 Pod들의 IP 주소와 포트 정보를 기반으로 트래픽을 전달합니다.

예를 들어 동일한 애플리케이션을 실행하는 Pod가 3개 존재한다면 Service는 이 Pod들을 하나의 논리적인 서비스로 묶어 클라이언트가 개별 Pod의 위치를 알 필요 없이 접근할 수 있도록 합니다.

![Kubernetes Service 개념과 Pod IP 주소의 추상화](https://images.openai.com/static-rsc-4/20oClK_fQK1M0Ot-zH-W6EAMJJ-iSuPb6Zyg3Hrzi69c1DdDCZ4iKAivVzOFImoqzrY5F04P7aTR4OEPbjy-U6kDxuHeCPnQFvNrLspPz6Kebdh0tdLdXZxEo1wRad2S42lPzngQmE26hxNhwNrAL5dofR-QHp7Mtap9E2GU7D8?purpose=inline)

참고: [KAI Road of Kubernetes 03 — What a Service is, and why Pod IPs should not be your interface | LemonKAI](https://lemonkai.github.io/blog/en/kai-k8s-road-03-service/)

## 2. Service의 주요 역할

Service는 다음과 같은 역할을 수행합니다.

- **안정적인 네트워크 접근 제공**: Pod의 IP 주소가 변경되더라도 동일한 Service 주소를 통해 접근할 수 있습니다.
- **Pod 집합 추상화**: 여러 Pod를 하나의 논리적인 서비스로 묶어 관리할 수 있습니다.
- **트래픽 분산**: Service에 연결된 여러 Endpoint로 트래픽을 전달할 수 있습니다.
- **서비스 디스커버리**: DNS를 통해 클러스터 내부에서 Service를 이름으로 검색하고 접근할 수 있습니다.
- **외부 접근 제공**: Service 유형에 따라 클러스터 외부에서도 애플리케이션에 접근할 수 있습니다.

## 3. Service의 동작 구조

Service는 일반적으로 다음과 같은 구성 요소를 통해 동작합니다.

| 구성 요소 | 설명 |
|---|---|
| Service | 클라이언트가 접근하는 논리적인 네트워크 엔드포인트입니다. |
| Label Selector | Service가 연결할 Pod를 식별하는 조건입니다. |
| EndpointSlice | Service에 연결된 Pod의 IP 주소와 포트 등 Endpoint 정보를 관리합니다. |
| kube-proxy | Service의 네트워크 규칙을 구성하여 트래픽을 적절한 Endpoint로 전달합니다. |
| CoreDNS | Service 이름을 클러스터 내부 DNS 주소로 해석합니다. |

일반적인 Service의 트래픽 처리 과정은 다음과 같습니다.

1. 클라이언트가 Service의 DNS 이름 또는 ClusterIP로 요청을 전송합니다.
2. CoreDNS가 Service 이름을 ClusterIP로 해석합니다. 이미 IP 주소를 알고 있다면 DNS 조회 과정은 생략됩니다.
3. kube-proxy가 구성한 네트워크 규칙에 따라 요청이 적절한 Endpoint로 전달됩니다.
4. 대상 Pod가 요청을 처리하고 응답을 반환합니다.

참고: kube-proxy는 일반적인 Kubernetes 클러스터에서 Service 트래픽 전달을 담당하지만, CNI 또는 eBPF 기반 네트워크 구현에서는 kube-proxy를 사용하지 않고 동일한 기능을 구현할 수도 있습니다.

## 4. Service와 EndpointSlice의 관계

Service는 Label Selector를 통해 대상 Pod를 식별하며, Kubernetes 컨트롤 플레인은 해당 조건에 부합하는 Pod를 기반으로 EndpointSlice를 관리합니다.

EndpointSlice는 Service가 트래픽을 전달할 수 있는 실제 네트워크 대상 정보를 저장합니다.

예를 들어 다음과 같은 Pod가 존재한다고 가정하겠습니다.

| Pod | Label | IP |
|---|---|---|
| backend-1 | `app: backend` | `10.0.1.10` |
| backend-2 | `app: backend` | `10.0.1.11` |
| backend-3 | `app: frontend` | `10.0.1.12` |

다음 Service가 생성되면 `app: backend` Label을 가진 Pod만 대상으로 선택됩니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - name: http
      port: 80
      targetPort: 8080
  type: ClusterIP
```

이 경우 EndpointSlice에는 다음과 같은 Endpoint 정보가 등록됩니다.

| Endpoint | IP | Port |
|---|---|---|
| backend-1 | `10.0.1.10` | `8080` |
| backend-2 | `10.0.1.11` | `8080` |

`backend-3`은 Label Selector 조건에 부합하지 않으므로 포함되지 않습니다.

Pod가 추가되거나 삭제되거나 IP 주소가 변경되면 Kubernetes 컨트롤 플레인이 EndpointSlice 정보를 갱신합니다.

이를 통해 Service는 개별 Pod의 변경 사항을 클라이언트에 노출하지 않고 안정적인 접근 지점을 유지합니다.

참고: EndpointSlice는 대량의 Endpoint를 효율적으로 관리하기 위해 도입된 리소스입니다. 하나의 Service에 여러 EndpointSlice가 연결될 수 있으며, Service와 EndpointSlice는 별개의 Kubernetes 리소스입니다.

## 5. Service 유형(Type)

Kubernetes Service는 `spec.type`을 통해 네트워크 접근 범위를 지정할 수 있습니다.

기본값은 `ClusterIP`이며, 대표적인 유형은 다음과 같습니다.

| 유형 | 설명 |
|---|---|
| ClusterIP | 클러스터 내부에서만 접근할 수 있는 가상 IP를 제공합니다. |
| NodePort | 각 Node의 특정 포트를 통해 외부에서 접근할 수 있도록 합니다. |
| LoadBalancer | 외부 로드 밸런서를 프로비저닝하여 클러스터 외부에서 접근할 수 있도록 합니다. |
| ExternalName | 외부 DNS 이름을 Service 이름에 매핑합니다. |

### 5.1. ClusterIP

ClusterIP는 Kubernetes Service의 기본 유형으로, 클러스터 내부에서 접근할 수 있는 가상 IP 주소를 제공합니다.

클러스터 내부의 다른 Pod는 Service의 DNS 이름 또는 ClusterIP를 사용하여 대상 애플리케이션에 접근할 수 있습니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
```

위 예제에서 다음과 같은 네트워크 관계가 형성됩니다.

- Service IP: 클러스터 내부에서 접근하는 가상 IP입니다.
- Service Port: `80`으로 클라이언트 요청을 수신합니다.
- Target Port: 실제 대상 Pod의 `8080` 포트로 트래픽을 전달합니다.

클러스터 내부의 클라이언트는 다음과 같이 접근할 수 있습니다.

```bash
curl http://backend-service
```

동일한 Namespace가 아닌 경우 다음과 같이 접근할 수 있습니다.

```bash
curl http://backend-service.default.svc.cluster.local
```

ClusterIP는 외부에서 직접 접근할 수 없으며, 일반적으로 내부 마이크로서비스 간 통신에 사용됩니다.

### 5.2. NodePort

NodePort는 클러스터의 각 Node에 특정 포트를 개방하여 외부에서 Service에 접근할 수 있도록 하는 유형입니다.

NodePort는 기본적으로 `30000~32767` 범위에서 포트를 할당하며, `nodePort` 필드를 지정하면 해당 범위 내에서 특정 포트를 사용할 수 있습니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: NodePort
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
      protocol: TCP
```

위 예제에서는 각 Node의 `30080` 포트를 통해 Service에 접근할 수 있습니다.

```bash
curl http://<NODE_IP>:30080
```

NodePort의 트래픽 처리 구조는 다음과 같습니다.

1. 외부 클라이언트가 Node의 IP 주소와 NodePort로 요청을 전송합니다.
2. Node의 네트워크 규칙이 요청을 해당 Service로 전달합니다.
3. Service에 연결된 Endpoint 중 하나로 트래픽이 전달됩니다.
4. 대상 Pod가 요청을 처리합니다.

NodePort는 별도의 외부 로드 밸런서 없이도 외부 접근 경로를 제공할 수 있지만, 일반적인 운영 환경에서는 LoadBalancer나 Ingress, Gateway API와 함께 사용하는 경우가 많습니다.

### 5.3. LoadBalancer

LoadBalancer는 클라우드 환경에서 외부 로드 밸런서를 프로비저닝하여 Service에 외부 접근 경로를 제공하는 유형입니다.

일반적으로 클라우드 제공자의 로드 밸런서 서비스와 연동되며, 외부 클라이언트는 할당된 외부 IP 주소 또는 DNS 이름을 통해 애플리케이션에 접근할 수 있습니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
```

LoadBalancer Service의 일반적인 트래픽 흐름은 다음과 같습니다.

1. Kubernetes가 클라우드 제공자의 로드 밸런서 프로비저닝을 요청합니다.
2. 클라우드 제공자가 외부 로드 밸런서를 생성합니다.
3. 로드 밸런서에 외부 IP 주소 또는 DNS 이름이 할당됩니다.
4. 외부 클라이언트의 요청이 로드 밸런서를 거쳐 Kubernetes Service의 Endpoint로 전달됩니다.

참고: LoadBalancer 유형은 Kubernetes 자체가 외부 로드 밸런서를 구현하는 것이 아닙니다. 실제 프로비저닝과 트래픽 전달은 클라우드 제공자의 컨트롤러 또는 별도의 LoadBalancer 구현체에 의해 처리됩니다.

### 5.4. ExternalName

ExternalName은 Kubernetes Service 이름을 클러스터 외부의 DNS 이름에 매핑하는 유형입니다.

다른 Service 유형과 달리 ClusterIP를 생성하거나 Pod로 트래픽을 직접 전달하지 않습니다. 대신 CoreDNS가 해당 Service 이름에 대한 DNS 조회 요청에 CNAME 레코드를 반환하도록 합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  type: ExternalName
  externalName: database.example.com
```

클러스터 내부의 Pod는 다음과 같이 접근할 수 있습니다.

```bash
nslookup external-database.default.svc.cluster.local
```

DNS 조회 결과는 다음과 같은 형태로 반환됩니다.

```text
external-database.default.svc.cluster.local
    canonical name = database.example.com
```

ExternalName은 외부 데이터베이스나 외부 API 서버처럼 클러스터 외부에 존재하는 서비스를 Kubernetes 내부의 일관된 DNS 이름으로 참조할 때 사용할 수 있습니다.

단, DNS 기반 매핑만 제공하므로 실제 네트워크 연결과 접근 제어는 별도로 구성해야 합니다.

## 6. Service의 주요 필드

Service를 정의할 때 자주 사용하는 주요 필드는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `spec.type` | Service의 유형을 지정합니다. |
| `spec.selector` | Service가 연결할 Pod의 Label 조건을 지정합니다. |
| `spec.ports` | Service가 노출할 포트와 대상 Pod의 포트를 정의합니다. |
| `spec.clusterIP` | Service에 할당되는 ClusterIP를 지정합니다. |
| `spec.externalIPs` | Service에 연결할 외부 IP 주소를 지정합니다. |
| `spec.externalName` | ExternalName 유형에서 매핑할 외부 DNS 이름을 지정합니다. |
| `spec.sessionAffinity` | 클라이언트의 요청을 동일한 Pod로 전달하도록 세션 어피니티를 설정합니다. |
| `spec.publishNotReadyAddresses` | 준비되지 않은 Endpoint도 게시할지 지정합니다. |
| `spec.externalTrafficPolicy` | 외부 트래픽의 전달 정책을 지정합니다. |
| `spec.internalTrafficPolicy` | 클러스터 내부 트래픽의 전달 정책을 지정합니다. |

### 6.1. Port 관련 필드

Service의 `spec.ports`에는 네트워크 연결에 필요한 여러 포트 정보가 정의됩니다.

| 필드 | 설명 |
|---|---|
| `port` | Service가 클라이언트 요청을 수신하는 포트입니다. |
| `targetPort` | 실제 대상 Pod에서 트래픽을 전달받는 포트입니다. |
| `nodePort` | NodePort 유형에서 Node에 개방하는 포트입니다. |
| `protocol` | 사용할 네트워크 프로토콜을 지정합니다. 기본값은 TCP입니다. |
| `name` | 포트에 부여하는 이름으로, 여러 포트를 정의할 때 구분에 사용됩니다. |
| `appProtocol` | 애플리케이션 프로토콜을 나타내는 선택적 힌트입니다. |

`port`와 `targetPort`는 서로 다른 개념입니다.

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
ports:
  - name: http
    port: 80
    targetPort: 8080
```

이 경우 클라이언트는 Service의 `80` 포트로 요청을 보내며, 실제 Pod의 `8080` 포트에서 요청을 처리합니다.

`targetPort`는 숫자뿐 아니라 문자열로도 지정할 수 있습니다. 문자열을 사용하는 경우 대상 Pod의 컨테이너 포트에 정의된 이름을 참조합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - name: http
      port: 80
      targetPort: http
```

위 예제에서 `targetPort: http`는 대상 Pod에 정의된 이름이 `http`인 컨테이너 포트를 의미합니다.

### 6.2. Session Affinity

`sessionAffinity`는 동일한 클라이언트의 요청을 가능한 한 동일한 Endpoint로 전달하도록 설정하는 기능입니다.

기본값은 `None`이며, `ClientIP`를 지정하면 클라이언트 IP 주소를 기준으로 세션 어피니티를 적용합니다.

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

위 설정에서는 동일한 클라이언트 IP 주소에서 발생하는 요청을 3시간 동안 동일한 Endpoint로 전달하도록 합니다.

세션 어피니티는 클라이언트별 트래픽을 특정 Pod에 유지해야 하는 경우 사용할 수 있지만, Pod 간 부하 불균형이 발생할 수 있으므로 주의해야 합니다.

### 6.3. External Traffic Policy

`externalTrafficPolicy`는 외부에서 유입된 트래픽을 처리하는 방식을 지정합니다.

| 값 | 설명 |
|---|---|
| `Cluster` | 클러스터 전체의 Endpoint를 대상으로 트래픽을 전달합니다. |
| `Local` | 요청을 수신한 Node에 존재하는 로컬 Endpoint로만 트래픽을 전달합니다. |

기본값은 `Cluster`입니다.

`Cluster`에서는 요청이 수신된 Node와 실제 Pod가 실행되는 Node가 다를 수 있으며, 이 과정에서 클라이언트의 원본 IP가 보존되지 않을 수 있습니다.

`Local`에서는 Node 내부의 Endpoint로만 트래픽을 전달하므로 불필요한 Node 간 트래픽을 줄이고 클라이언트의 원본 IP를 보존하는 데 유리합니다.

다만 해당 Node에 로컬 Endpoint가 존재하지 않으면 트래픽이 전달되지 않을 수 있으므로 Node별 Endpoint 배치를 고려해야 합니다.

### 6.4. Internal Traffic Policy

`internalTrafficPolicy`는 클러스터 내부에서 발생한 트래픽의 Endpoint 선택 범위를 지정합니다.

| 값 | 설명 |
|---|---|
| `Cluster` | 클러스터 전체의 Endpoint를 대상으로 트래픽을 전달합니다. |
| `Local` | 요청을 발생시킨 Node에 존재하는 로컬 Endpoint로만 트래픽을 전달합니다. |

기본값은 `Cluster`입니다.

`Local`을 사용하면 Node 간 네트워크 트래픽을 줄일 수 있지만, 해당 Node에 Endpoint가 없으면 요청이 정상적으로 전달되지 않을 수 있습니다.

## 7. Headless Service

Headless Service는 일반적인 ClusterIP를 할당하지 않고 개별 Pod의 네트워크 주소를 DNS를 통해 직접 조회할 수 있도록 하는 Service입니다.

`spec.clusterIP`를 `None`으로 지정하면 Headless Service가 생성됩니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database-service
spec:
  clusterIP: None
  selector:
    app: database
  ports:
    - port: 5432
      targetPort: 5432
```

일반적인 ClusterIP Service는 DNS 조회 시 Service의 ClusterIP를 반환합니다.

반면 Headless Service는 DNS 조회 시 해당 Service에 연결된 개별 Endpoint의 IP 주소를 반환합니다.

![Kubernetes Headless Service 동작 구조](https://images.openai.com/static-rsc-4/ox0PWefXeGRpjxdl9rlnAzGq5MuIPCsziPQjzZ09wOxySW4LkmVtGoBoqzntveExkfiT21Bi1HPZ6quBVdqOVSFjQ92W74_KjB54T2FwGt2FTHliFDEL_WOE3eRNjRhzkMtqG67_AhZyE1-0C4td_7pmqeh4B9iXnagms258558?purpose=inline)

참고: [Kubernetes Headless Service - Introduction and examples | Devops Junction](https://www.middlewareinventory.com/blog/kubernetes-headless-service/)

Headless Service는 다음과 같은 상황에서 주로 사용됩니다.

- StatefulSet의 개별 Pod에 직접 접근해야 하는 경우
- 데이터베이스 클러스터처럼 각 인스턴스의 주소를 구분해야 하는 경우
- 클라이언트가 자체적으로 Endpoint를 선택하고 연결을 관리해야 하는 경우

특히 StatefulSet과 함께 사용하면 각 Pod에 대해 안정적인 DNS 이름을 제공할 수 있습니다.

예를 들어 다음과 같은 DNS 이름을 사용할 수 있습니다.

```text
database-0.database-service.default.svc.cluster.local
database-1.database-service.default.svc.cluster.local
database-2.database-service.default.svc.cluster.local
```

이러한 구조는 데이터베이스 복제 구성이나 분산 시스템에서 개별 인스턴스에 접근해야 할 때 유용합니다.

## 8. Selector가 없는 Service

Service는 반드시 Label Selector를 사용해야 하는 것은 아닙니다.

`spec.selector`를 생략하면 Kubernetes는 Label을 기준으로 Pod를 자동으로 선택하지 않습니다. 이 경우 관리자가 EndpointSlice를 직접 생성하여 Service와 외부 시스템 또는 별도의 네트워크 대상 간의 연결을 구성할 수 있습니다.

예를 들어 클러스터 외부에 존재하는 데이터베이스 서버를 Kubernetes 내부 Service로 추상화할 수 있습니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  ports:
    - name: database
      port: 5432
      targetPort: 5432
```

다음은 해당 Service에 연결할 EndpointSlice 예제입니다.

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: external-database-1
  labels:
    kubernetes.io/service-name: external-database
    endpointslice.kubernetes.io/managed-by: example-controller
addressType: IPv4
ports:
  - name: database
    protocol: TCP
    port: 5432
endpoints:
  - addresses:
      - 192.168.1.100
    conditions:
      ready: true
```

이렇게 구성하면 클러스터 내부의 Pod는 외부 데이터베이스의 실제 IP 주소를 직접 알 필요 없이 Service 이름을 통해 접근할 수 있습니다.

단, EndpointSlice의 주소는 유효한 네트워크 대상이어야 하며, 일반적인 Pod IP 주소와 달리 외부 네트워크의 주소를 사용할 때는 라우팅과 방화벽 설정 등을 별도로 고려해야 합니다.

또한 EndpointSlice의 `kubernetes.io/service-name` Label이 Service 이름과 일치해야 하며, `endpointslice.kubernetes.io/managed-by` Label을 통해 해당 EndpointSlice를 관리하는 주체를 표시하는 것이 좋습니다.

## 9. Service와 Ingress, Gateway API의 관계

Service, Ingress, Gateway API는 모두 Kubernetes의 네트워킹 영역에 속하지만 담당하는 역할이 다릅니다.

| 리소스 | 주요 역할 |
|---|---|
| Service | Pod 집합에 안정적인 네트워크 접근 지점을 제공합니다. |
| Ingress | HTTP 및 HTTPS 기반 외부 트래픽의 라우팅 규칙을 정의합니다. |
| Gateway API | 외부 트래픽의 진입점과 라우팅 정책을 더 세분화하여 정의합니다. |

일반적인 HTTP 애플리케이션의 외부 접근 구조는 다음과 같습니다.

![Ingress를 통해 외부 트래픽이 Service와 Pod로 전달되는 구조](https://images.openai.com/static-rsc-4/N3WoFxY-KKSDv4MP14jeEhC0jAwuoR8uIT3TU2c4maNw6d0dCJlkUwNEgEbInSSWJ_k1_Q_3kU4K64OgDCmvjSHQJeIBoGUns-1eG3WjA8YMIppVdb8bdmD-tdpThY0PQqXuExpvk6ReZxi3KlWsB1AsIuE9-dcsn97CPv70A6Q?purpose=inline)

참고: [Inside the Cluster: Ingress in Focus — How External Traffic Reaches Your Services | Medium](https://medium.com/@kelvi030117/inside-the-cluster-ingress-in-focus-how-external-traffic-reaches-your-services-95006bd8cdb4)

- 외부 클라이언트는 Ingress Controller 또는 Gateway를 통해 클러스터에 접근합니다.
- Ingress 또는 HTTPRoute와 같은 라우팅 리소스는 요청을 전달할 Service를 지정합니다.
- Service는 연결된 Endpoint를 통해 실제 Pod로 트래픽을 전달합니다.

즉, Ingress와 Gateway API는 외부 트래픽의 라우팅을 담당하고, Service는 해당 트래픽을 실제 애플리케이션 Pod로 연결하는 역할을 담당합니다.

Ingress는 Kubernetes에서 오랫동안 사용된 HTTP 및 HTTPS 라우팅 API이며, Gateway API는 보다 확장성 있고 세분화된 트래픽 관리 기능을 제공하는 API입니다.

## 10. Service 확인 및 관리 명령어

Service를 확인하고 관리할 때 사용하는 대표적인 kubectl 명령어는 다음과 같습니다.

### Service 목록 조회

```bash
kubectl get services
```

또는 다음과 같이 축약할 수 있습니다.

```bash
kubectl get svc
```

### 특정 Service 상세 조회

```bash
kubectl describe service backend-service
```

### Service YAML 조회

```bash
kubectl get service backend-service -o yaml
```

### Service의 ClusterIP 확인

```bash
kubectl get svc backend-service -o wide
```

### EndpointSlice 조회

```bash
kubectl get endpointslices
```

특정 Service와 연결된 EndpointSlice만 조회하려면 다음과 같이 실행합니다.

```bash
kubectl get endpointslices \
  -l kubernetes.io/service-name=backend-service
```

### Service 삭제

```bash
kubectl delete service backend-service
```

Service를 삭제하더라도 해당 Service가 연결하고 있던 Pod 자체는 삭제되지 않습니다. Pod의 생성과 삭제는 Deployment, StatefulSet 등 해당 Pod를 관리하는 리소스의 역할입니다.

## 11. 정리

Service는 Kubernetes에서 Pod 집합에 안정적인 네트워크 접근 지점을 제공하는 핵심 네트워킹 리소스입니다.

Pod의 IP 주소가 변경되거나 Pod의 수가 증가 및 감소하더라도 Service를 통해 동일한 네트워크 주소로 애플리케이션에 접근할 수 있습니다.

Service의 주요 특징은 다음과 같습니다.

- Label Selector를 통해 대상 Pod를 식별할 수 있습니다.
- EndpointSlice를 통해 실제 네트워크 대상 정보를 관리합니다.
- ClusterIP, NodePort, LoadBalancer, ExternalName 등 다양한 접근 방식을 제공합니다.
- DNS 기반 서비스 디스커버리를 지원합니다.
- Headless Service를 통해 개별 Pod의 DNS 주소를 제공할 수 있습니다.
- Ingress 및 Gateway API와 연계하여 외부 트래픽을 애플리케이션 Pod로 전달할 수 있습니다.

Service는 Kubernetes 내부의 마이크로서비스 간 통신과 외부 트래픽 연결을 구성하는 기본 단위이며, Pod의 생명주기와 네트워크 주소 변경으로부터 클라이언트를 분리하는 역할을 수행합니다.

## 12. 공식 문서 및 참고 자료

- [Kubernetes 공식 문서 - Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes 공식 문서 - Service의 DNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Kubernetes 공식 문서 - EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Kubernetes 공식 문서 - Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kubernetes 공식 문서 - Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
