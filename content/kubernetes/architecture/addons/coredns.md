# CoreDNS

## 1. CoreDNS란?

CoreDNS는 Kubernetes 클러스터 내부에서 DNS(Domain Name System) 서비스를 제공하는 DNS 서버입니다.

Kubernetes는 Service와 Pod에 DNS 이름을 부여하여 클러스터 내부의 워크로드가 IP 주소를 직접 사용하지 않고도 서로를 검색하고 통신할 수 있도록 지원합니다. CoreDNS는 Kubernetes API를 통해 클러스터의 Service 및 Pod 정보를 확인하고, 이에 대응하는 DNS 레코드를 제공하는 역할을 담당합니다.

CoreDNS는 Kubernetes의 핵심 Control Plane 구성 요소가 아닌 Addon에 해당합니다. 일반적으로 클러스터 내부에 Pod 형태로 배포되며, DNS 서비스를 제공하는 Kubernetes Service와 함께 동작합니다.

Kubernetes는 기존 kube-dns를 대체하는 DNS 구현체로 CoreDNS를 채택했습니다.

## 2. 주요 역할

### 2.1. Service Discovery

CoreDNS의 가장 중요한 역할은 클러스터 내부의 Service Discovery를 지원하는 것입니다.

Kubernetes에서는 Service가 생성되면 해당 Service에 대한 DNS 레코드가 제공됩니다. 이를 통해 클라이언트 Pod는 Service의 ClusterIP를 직접 알 필요 없이 Service 이름을 사용하여 대상 워크로드에 접근할 수 있습니다.

예를 들어 다음과 같은 Service가 존재한다고 가정하겠습니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: production
spec:
  selector:
    app: backend
  ports:
    - port: 8080
      targetPort: 8080
```

CoreDNS는 해당 Service에 대해 다음과 같은 DNS 이름을 제공합니다.

```text
backend.production.svc.cluster.local
```

클러스터 내부의 다른 Pod에서는 다음과 같이 Service에 접근할 수 있습니다.

```bash
curl http://backend.production.svc.cluster.local:8080
```

동일한 Namespace에 존재하는 Pod라면 검색 도메인을 활용하여 다음과 같이 축약된 이름으로도 접근할 수 있습니다.

```bash
curl http://backend:8080
```

이러한 방식은 애플리케이션이 Service의 실제 IP 주소나 개별 Pod의 IP 주소에 종속되지 않도록 지원합니다.

### 2.2. DNS 레코드 제공

CoreDNS는 Kubernetes의 Service 및 Pod 정보를 기반으로 DNS 레코드를 제공합니다.

주요 DNS 레코드 유형은 다음과 같습니다.

| 레코드 | 설명 |
|---|---|
| A | 도메인 이름을 IPv4 주소로 변환합니다. |
| AAAA | 도메인 이름을 IPv6 주소로 변환합니다. |
| SRV | 서비스의 이름이 지정된 포트와 대상 도메인 정보를 제공합니다. |
| PTR | IP 주소에 대응하는 도메인 이름을 조회하는 역방향 DNS 레코드입니다. |

일반적인 Service와 Headless Service는 서로 다른 방식으로 DNS 레코드를 제공합니다.

**일반 Service**

일반적인 ClusterIP Service는 Service의 ClusterIP를 반환합니다.

```text
backend.production.svc.cluster.local
    |
    v
10.96.100.10
```

클라이언트는 반환된 ClusterIP를 통해 Service에 접근합니다. 이후 실제 Pod로의 트래픽 전달은 Kubernetes의 Service 네트워킹 구현이 담당합니다.

**Headless Service**

Headless Service는 `clusterIP: None`으로 설정된 Service입니다.

일반 Service와 달리 ClusterIP를 생성하지 않으며, 해당 Service가 선택한 Pod들의 IP 주소를 DNS 응답으로 제공합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: production
spec:
  clusterIP: None
  selector:
    app: database
  ports:
    - port: 5432
      targetPort: 5432
```

DNS 조회 결과는 다음과 같은 형태로 반환될 수 있습니다.

```text
database.production.svc.cluster.local
    |
    +-- 10.244.1.10
    |
    +-- 10.244.2.15
    |
    +-- 10.244.3.20
```

클라이언트는 반환된 Pod IP 주소들을 사용하여 개별 Pod에 직접 접근할 수 있습니다.

Headless Service는 StatefulSet과 같이 개별 Pod의 네트워크 식별자가 필요한 워크로드에서 유용하게 활용됩니다.

### 2.3. 외부 DNS 조회

CoreDNS는 클러스터 내부의 DNS 이름뿐 아니라 외부 도메인에 대한 DNS 조회도 지원합니다.

예를 들어 클러스터 내부의 Pod에서 다음과 같은 도메인을 조회할 수 있습니다.

```bash
nslookup www.google.com
```

CoreDNS는 Kubernetes 내부 DNS 영역에 해당하지 않는 요청을 설정된 상위 DNS 서버(Upstream DNS Server)로 전달할 수 있습니다.

이때 `forward` 플러그인을 사용하여 DNS 요청을 외부 DNS 서버로 전달합니다.

```text
Pod
 |
 | DNS Query: www.google.com
 v
CoreDNS
 |
 | Forward
 v
Upstream DNS Server
 |
 | DNS Response
 v
CoreDNS
 |
 v
Pod
```

이를 통해 클러스터 내부의 Service Discovery와 외부 도메인 조회를 하나의 DNS 서비스에서 처리할 수 있습니다.

## 3. 아키텍처 및 동작 방식

### 3.1. 배포 구조

CoreDNS는 일반적으로 Kubernetes 클러스터 내부에 Deployment 형태로 배포됩니다.

대표적인 구성 요소는 다음과 같습니다.

| 리소스 | 역할 |
|---|---|
| Deployment | CoreDNS Pod의 배포 및 복제본을 관리합니다. |
| Pod | 실제 DNS 요청을 처리하는 CoreDNS 서버를 실행합니다. |
| Service | 클러스터 내부에 DNS 서비스의 접근 지점을 제공합니다. |
| ConfigMap | CoreDNS의 DNS 처리 규칙과 플러그인 설정을 관리합니다. |
| ServiceAccount | CoreDNS가 Kubernetes API에 접근할 수 있도록 인증 정보를 제공합니다. |
| ClusterRole / ClusterRoleBinding | CoreDNS가 DNS 레코드 생성에 필요한 Kubernetes 리소스를 조회할 수 있도록 권한을 부여합니다. |

일반적으로 CoreDNS는 `kube-system` Namespace에 배포됩니다.

또한 DNS Service는 기존 kube-dns와의 호환성을 위해 일반적으로 `kube-dns`라는 이름을 사용합니다. 실제 DNS 요청을 처리하는 서버가 CoreDNS인 경우에도 Service 이름은 `kube-dns`일 수 있습니다.

### 3.2. DNS 요청 처리 과정

Pod에서 Service의 DNS 이름을 조회하면 다음과 같은 과정으로 요청이 처리됩니다.

1. 클라이언트 Pod에서 DNS 조회 요청이 발생합니다.
2. Pod 내부의 DNS Resolver는 `/etc/resolv.conf`에 설정된 DNS 서버로 요청을 전달합니다.
3. DNS 서버 주소는 일반적으로 클러스터의 DNS Service인 `kube-dns`의 ClusterIP입니다.
4. Kubernetes의 Service 네트워킹을 통해 요청이 CoreDNS Pod로 전달됩니다.
5. CoreDNS의 `kubernetes` 플러그인은 Kubernetes API를 통해 동기화한 Service 및 EndpointSlice 등의 정보를 기반으로 DNS 레코드를 조회합니다.
6. CoreDNS는 조회 결과를 DNS 응답으로 반환합니다.
7. 클라이언트 Pod는 반환된 IP 주소를 사용하여 대상 Service에 접근합니다.

전체 흐름은 다음과 같습니다.

```text
+---------------------------+
|       Client Pod          |
|                           |
|  Application              |
|       |                   |
|       v                   |
|  DNS Resolver             |
|  (/etc/resolv.conf)       |
+-------|-------------------+
        |
        | DNS Query
        v
+---------------------------+
|    kube-dns Service       |
|                           |
|    ClusterIP: 10.96.0.10  |
+-------|-------------------+
        |
        | Service Routing
        v
+---------------------------+
|       CoreDNS Pod         |
|                           |
|  +---------------------+  |
|  | Kubernetes Plugin   |  |
|  +---------------------+  |
|           |               |
|           v               |
|  DNS Record Lookup        |
+-------|-------------------+
        |
        | DNS Response
        v
+---------------------------+
|       Client Pod          |
|                           |
|  Resolved IP Address      |
+---------------------------+
```

참고로 CoreDNS가 DNS 요청을 처리하는 과정과 실제 Service 트래픽을 전달하는 과정은 구분해야 합니다.

CoreDNS는 Service 이름을 IP 주소로 변환하는 역할을 담당하며, 실제 애플리케이션 트래픽을 대상 Pod로 전달하는 작업은 Kubernetes의 Service 네트워킹이 담당합니다.

### 3.3. Kubernetes API와의 연동

CoreDNS는 `kubernetes` 플러그인을 통해 Kubernetes API와 연동합니다.

이 플러그인은 Kubernetes API를 감시하여 Service 및 EndpointSlice 등의 변경 사항을 확인하고, 해당 정보를 DNS 응답에 반영합니다.

예를 들어 다음과 같은 변경 사항이 발생하면 CoreDNS가 관련 정보를 동기화하여 DNS 응답에 반영할 수 있습니다.

- 새로운 Service가 생성되는 경우
- Service가 삭제되는 경우
- Service의 EndpointSlice에 등록된 Pod IP 주소가 변경되는 경우
- Headless Service에 연결된 Pod가 변경되는 경우

CoreDNS는 이러한 정보를 활용하여 별도의 DNS 레코드를 수동으로 관리하지 않고도 Kubernetes 리소스의 변경 사항에 대응할 수 있습니다.

또한 CoreDNS는 Kubernetes API에 접근하기 위해 ServiceAccount 및 RBAC 권한을 사용합니다.

## 4. 주요 구성 요소

CoreDNS는 플러그인 기반 아키텍처를 사용합니다.

각 플러그인은 DNS 요청을 처리하는 특정 기능을 담당하며, Corefile을 통해 활성화 및 설정할 수 있습니다.

Kubernetes 환경에서 주로 사용되는 플러그인은 다음과 같습니다.

| 플러그인 | 역할 |
|---|---|
| `kubernetes` | Kubernetes API와 연동하여 Service 및 Pod에 대한 DNS 레코드를 제공합니다. |
| `forward` | DNS 요청을 상위 DNS 서버로 전달합니다. |
| `cache` | DNS 응답을 캐싱하여 반복적인 DNS 조회에 대한 처리 부담을 줄입니다. |
| `errors` | DNS 처리 과정에서 발생하는 오류를 로그로 출력합니다. |
| `health` | CoreDNS의 헬스 체크 엔드포인트를 제공합니다. |
| `ready` | CoreDNS가 요청을 처리할 준비가 되었는지 확인할 수 있는 엔드포인트를 제공합니다. |
| `prometheus` | Prometheus가 수집할 수 있는 메트릭을 제공합니다. |
| `log` | DNS 요청 및 응답에 대한 로그를 출력합니다. |

### 4.1. Corefile

Corefile은 CoreDNS의 동작 방식을 정의하는 설정 파일입니다.

Kubernetes에서는 일반적으로 ConfigMap을 통해 Corefile을 관리하며, CoreDNS Pod에 해당 설정을 마운트하여 사용합니다.

다음은 Kubernetes 클러스터에서 사용할 수 있는 대표적인 Corefile 구성 예시입니다.

```text
.:53 {
    errors
    health {
        lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

주요 설정의 의미는 다음과 같습니다.

- `.:53`: 모든 DNS 영역에 대해 53번 포트에서 DNS 요청을 처리합니다.
- `errors`: DNS 처리 오류를 출력합니다.
- `health`: CoreDNS의 헬스 체크 기능을 활성화합니다.
- `ready`: CoreDNS의 준비 상태를 확인할 수 있도록 합니다.
- `kubernetes cluster.local`: Kubernetes DNS 영역을 처리하도록 설정합니다.
- `pods insecure`: Pod DNS 조회에 대한 레코드 처리 방식을 설정합니다.
- `fallthrough`: 지정된 DNS 영역에서 레코드를 찾지 못한 경우 다음 플러그인으로 요청을 전달합니다.
- `ttl 30`: DNS 레코드의 TTL을 30초로 설정합니다.
- `prometheus :9153`: Prometheus 메트릭 엔드포인트를 활성화합니다.
- `forward . /etc/resolv.conf`: 처리 대상이 아닌 DNS 요청을 `/etc/resolv.conf`에 지정된 DNS 서버로 전달합니다.
- `cache 30`: DNS 응답을 최대 30초의 캐시 설정으로 관리합니다.
- `loop`: DNS 요청이 무한 반복되는 상황을 감지합니다.
- `reload`: Corefile 변경 사항을 감지하여 설정을 다시 로드합니다.
- `loadbalance`: DNS 응답에 포함되는 레코드의 순서를 조정합니다.

위 설정은 대표적인 구성 예시이며, 실제 배포 환경의 Corefile은 Kubernetes 버전, CoreDNS 버전 및 클러스터 구성에 따라 달라질 수 있습니다.

## 5. Pod의 DNS 설정

Kubernetes에서는 kubelet이 Pod 내부 컨테이너의 DNS 설정을 구성합니다.

일반적으로 `ClusterFirst` DNS 정책을 사용하는 Pod는 클러스터 DNS를 기본 DNS 서버로 사용합니다.

Pod 내부의 `/etc/resolv.conf`는 다음과 같은 형태로 구성될 수 있습니다.

```text
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

각 설정의 의미는 다음과 같습니다.

- `nameserver`: DNS 조회 요청을 전달할 DNS 서버의 IP 주소입니다.
- `search`: DNS 조회 시 자동으로 추가할 검색 도메인 목록입니다.
- `ndots`: DNS 이름에 포함된 점(`.`)의 개수가 지정된 값보다 적으면 검색 도메인을 활용한 조회를 먼저 수행하도록 하는 옵션입니다.

Kubernetes는 Pod의 `dnsPolicy`를 통해 DNS 설정 방식을 제어합니다.

| DNS Policy | 설명 |
|---|---|
| `Default` | Pod가 실행되는 Node의 DNS 설정을 상속합니다. |
| `ClusterFirst` | 클러스터 내부 DNS를 우선 사용합니다. 클러스터 도메인에 해당하지 않는 요청은 상위 DNS 서버로 전달됩니다. |
| `ClusterFirstWithHostNet` | Host Network를 사용하는 Pod에서 클러스터 DNS를 우선 사용하도록 설정합니다. |
| `None` | Kubernetes가 기본 DNS 설정을 제공하지 않으며, 사용자가 `dnsConfig`를 통해 직접 설정합니다. |

`dnsPolicy`는 Pod Spec에서 설정할 수 있으며, 필요에 따라 `dnsConfig`를 사용하여 DNS 서버, 검색 도메인 및 옵션을 추가로 지정할 수 있습니다.

## 6. 주요 특징

### 6.1. Kubernetes API 기반의 동적 DNS 관리

CoreDNS는 Kubernetes API를 통해 Service 및 EndpointSlice 등의 정보를 동기화합니다.

따라서 Service나 연결된 Pod의 변경 사항이 발생하더라도 DNS 레코드를 수동으로 관리할 필요가 없습니다.

### 6.2. 플러그인 기반 아키텍처

CoreDNS는 플러그인 기반으로 동작하므로 필요한 기능을 선택적으로 활성화할 수 있습니다.

이를 통해 Kubernetes DNS 서비스뿐 아니라 외부 DNS Forwarding, DNS 캐싱, 메트릭 수집 및 헬스 체크 등의 기능을 함께 구성할 수 있습니다.

### 6.3. 확장성과 고가용성

CoreDNS는 Deployment의 복제본 수를 조정하여 여러 Pod로 운영할 수 있습니다.

여러 CoreDNS Pod를 운영하면 특정 Pod에 장애가 발생하더라도 다른 Pod가 DNS 요청을 처리할 수 있도록 구성할 수 있습니다.

일반적으로 CoreDNS Pod는 Kubernetes Service를 통해 접근하므로 클라이언트 Pod는 개별 CoreDNS Pod의 IP 주소를 직접 관리할 필요가 없습니다.

### 6.4. DNS 캐싱

CoreDNS는 `cache` 플러그인을 통해 DNS 응답을 캐싱할 수 있습니다.

캐싱을 활용하면 동일한 DNS 요청에 대한 반복적인 처리 부담을 줄이고 DNS 응답 성능을 개선할 수 있습니다.

다만 DNS 캐시의 TTL 및 캐시 정책에 따라 Kubernetes 리소스 변경 사항이 DNS 응답에 반영되기까지 일정한 지연이 발생할 수 있습니다.

## 7. 다른 Kubernetes 컴포넌트와의 관계

CoreDNS는 Kubernetes의 여러 컴포넌트 및 리소스와 연동하여 클러스터 내부의 서비스 검색 기능을 제공합니다.

| 구성 요소 | 관계 |
|---|---|
| kube-apiserver | CoreDNS가 Kubernetes 리소스 정보를 조회하고 변경 사항을 감시하기 위해 접근하는 API 엔드포인트입니다. |
| kubelet | Pod의 DNS 설정을 구성하고 컨테이너 내부의 `/etc/resolv.conf`를 관리합니다. |
| kube-proxy | DNS 요청이 전달되는 `kube-dns` Service의 ClusterIP 트래픽을 CoreDNS Pod로 전달하는 역할을 담당할 수 있습니다. |
| Service | DNS 이름을 통해 클러스터 내부에서 접근할 수 있는 네트워크 엔드포인트를 제공합니다. |
| EndpointSlice | Service를 구성하는 실제 백엔드 엔드포인트 정보를 제공하며, CoreDNS가 DNS 응답을 구성하는 데 활용합니다. |
| Pod | DNS 요청을 발생시키고 CoreDNS의 응답을 사용하여 다른 Service 및 Pod를 검색합니다. |

참고로 kube-proxy는 Service 트래픽을 처리하는 일반적인 구현 방식 중 하나입니다. Cilium과 같이 kube-proxy를 대체하는 네트워킹 구현을 사용하는 환경에서는 실제 트래픽 전달 방식이 달라질 수 있습니다.

## 8. 참고자료

- [Kubernetes Documentation - DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Kubernetes Documentation - Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Kubernetes Documentation - Installing Addons](https://kubernetes.io/docs/concepts/cluster-administration/addons/)
- [CoreDNS Documentation - Kubernetes Plugin](https://coredns.io/plugins/kubernetes/)
- [CoreDNS Documentation](https://coredns.io/manual/toc/)

