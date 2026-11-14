# EndpointSlice

## 1. EndpointSlice란?

EndpointSlice는 Kubernetes에서 Service가 연결할 수 있는 네트워크 Endpoint 정보를 관리하는 리소스입니다.

Endpoint는 네트워크 통신의 대상이 되는 IP 주소와 포트의 조합을 의미합니다. Kubernetes에서는 일반적으로 애플리케이션을 실행하는 Pod가 Endpoint가 됩니다.

Service는 Label Selector를 사용하여 대상 Pod를 식별하지만, 실제 트래픽을 전달할 네트워크 대상 정보는 EndpointSlice를 통해 관리됩니다.

EndpointSlice는 기존 `Endpoints` 리소스가 가지고 있던 확장성 문제를 해결하기 위해 도입되었습니다. 기존 Endpoints는 `core/v1` API 그룹에 속하는 리소스로, 하나의 Service에 연결된 모든 Endpoint의 IP 주소와 포트 정보를 하나의 리소스에 저장합니다. 이 정보는 `subsets` 필드의 `addresses`, `notReadyAddresses`, `ports` 등을 통해 관리됩니다.

이러한 구조에서는 Service에 연결된 Pod가 많아질수록 하나의 Endpoints 리소스가 커지며, Endpoint가 일부 변경되더라도 해당 리소스 전체가 갱신되어야 합니다. 그 결과 API Server의 리소스 처리 부담이 증가하고, 변경 사항을 전달받는 네트워크 구성 요소에도 불필요한 데이터 전송과 갱신 작업이 발생할 수 있습니다.

EndpointSlice는 이러한 문제를 해결하기 위해 Endpoint 정보를 여러 개의 작은 리소스로 분할하여 관리합니다. 이를 통해 Endpoint 일부가 변경되었을 때 관련된 EndpointSlice만 갱신할 수 있으며, 대규모 클러스터에서도 Endpoint 정보를 보다 효율적으로 관리할 수 있습니다.

EndpointSlice는 `discovery.k8s.io/v1` API 그룹에 속하며, 일반적으로 Kubernetes 컨트롤 플레인의 EndpointSlice Controller가 Service와 Pod의 상태를 관찰하여 자동으로 생성하고 갱신합니다.

![Kubernetes EndpointSlice의 개념과 Service 연결 구조](https://images.openai.com/static-rsc-4/Poqv_OQXQzXUuHjuLFmAkbKv0nvZTo5gckLbFnB512Zz9asKLP6EgyzLpXbOM2bmshrJFbqLzjWJ_sQHM0LZCYyuOt2Cg5JSZYSTV6eyw5mNdYiFqkWYcKtBUPsmrRh7I_SubryS3H62JOLDqChCCowOEugQkK-N2ha1cz7N5ng?purpose=inline)

## 2. EndpointSlice의 주요 역할

EndpointSlice는 다음과 같은 역할을 수행합니다.

- **Endpoint 정보 관리:** Service에 연결된 Pod의 IP 주소와 포트 정보를 관리합니다.
- **확장성 개선:** 많은 Endpoint를 여러 EndpointSlice로 분할하여 관리합니다.
- **상태 정보 제공:** Endpoint의 준비 상태, 종료 상태, Serving 여부 등을 제공합니다.
- **다양한 네트워크 주소 지원:** IPv4와 IPv6 주소를 지원합니다.
- **서비스 디스커버리 지원:** kube-proxy와 같은 네트워크 구성 요소가 Service의 실제 트래픽 전달 대상을 확인할 수 있도록 합니다.

## 3. EndpointSlice의 동작 구조

일반적인 Service와 EndpointSlice의 관계는 다음과 같습니다.

![Service와 EndpointSlice 및 Pod 간의 연결 구조](https://images.openai.com/static-rsc-4/U_yk9UayxRJW084kU8F7pAdg8EwB84G3JcD3RAR9uCu2ZmjC713Eb70vLbkai8FzLCv0Eye8NDQjPLO7lkWvND0og1vM4IuKTy7ngZe6zS-LWi8_8S9kBNtnGrCCgzK1-6DaLdGW6UIFc3UQ6FaT1rh8tvon0s0MgtDG9LO8CqE?purpose=inline)

1. Service가 생성되고 Label Selector를 통해 대상 Pod를 지정합니다.
2. EndpointSlice Controller가 Service와 Pod의 상태를 관찰합니다.
3. Selector 조건에 부합하는 Pod를 기반으로 EndpointSlice를 생성하거나 갱신합니다.
4. EndpointSlice에 Pod의 IP 주소, 포트 및 상태 정보가 기록됩니다.
5. kube-proxy 또는 이에 상응하는 네트워크 구현체가 EndpointSlice 정보를 참조하여 트래픽 전달 규칙을 구성합니다.

예를 들어 `app: backend` Label을 가진 Pod가 3개 존재하고, 이를 대상으로 하는 Service가 생성되었다고 가정하겠습니다.

이 경우 EndpointSlice에는 다음과 같은 정보가 등록됩니다.

| Endpoint | IP 주소 | Port |
|---|---|---:|
| backend-1 | 10.0.1.10 | 8080 |
| backend-2 | 10.0.1.11 | 8080 |
| backend-3 | 10.0.1.12 | 8080 |

Pod가 추가되거나 삭제되면 EndpointSlice Controller가 변경 사항을 감지하여 EndpointSlice 정보를 갱신합니다.

이를 통해 네트워크 구성 요소는 개별 Pod의 생성 및 삭제를 직접 추적하지 않고도 최신 Endpoint 정보를 확인할 수 있습니다.

**참고:** EndpointSlice Controller는 EndpointSlice를 관리하는 컨트롤 플레인 구성 요소이며, kube-proxy는 EndpointSlice 정보를 이용하여 Service 트래픽 전달 규칙을 구성하는 네트워크 구성 요소입니다. 두 구성 요소의 역할은 구분됩니다.

## 4. EndpointSlice의 주요 필드

EndpointSlice는 다음과 같은 주요 필드로 구성됩니다.

| 필드 | 설명 |
|---|---|
| `addressType` | Endpoint 주소의 유형을 지정합니다. |
| `endpoints` | 실제 네트워크 Endpoint 정보를 정의합니다. |
| `ports` | Endpoint에서 사용하는 포트 정보를 정의합니다. |
| `metadata.labels` | EndpointSlice의 소속 Service 및 관리 정보를 나타냅니다. |

### 4.1 addressType

`addressType`은 EndpointSlice에 포함된 Endpoint 주소의 유형을 지정합니다.

현재 `discovery.k8s.io/v1` API에서 지원하는 값은 다음과 같습니다.

| 값 | 설명 |
|---|---|
| `IPv4` | IPv4 주소를 사용하는 Endpoint입니다. |
| `IPv6` | IPv6 주소를 사용하는 Endpoint입니다. |

하나의 EndpointSlice에는 동일한 주소 유형의 Endpoint만 포함할 수 있습니다.

예를 들어 IPv4와 IPv6 Endpoint를 함께 관리해야 한다면 서로 다른 EndpointSlice를 생성해야 합니다.

### 4.2 endpoints

`endpoints`는 실제 네트워크 통신 대상의 정보를 정의하는 필드입니다.

각 Endpoint는 다음과 같은 주요 정보를 포함합니다.

| 필드 | 설명 |
|---|---|
| `addresses` | Endpoint의 IP 주소를 지정합니다. |
| `conditions` | Endpoint의 준비 상태 및 종료 상태를 나타냅니다. |
| `hostname` | Endpoint에 부여된 호스트 이름입니다. |
| `nodeName` | Endpoint가 위치한 Node의 이름입니다. |
| `targetRef` | Endpoint가 참조하는 Kubernetes 객체를 나타냅니다. |
| `zone` | Endpoint가 위치한 가용 영역(Zone)을 나타냅니다. |
| `hints` | Endpoint 선택 시 고려할 수 있는 토폴로지 정보를 제공합니다. |

일반적인 Pod Endpoint에서는 다음과 같은 정보가 사용됩니다.

```yaml
endpoints:
  - addresses:
      - 10.0.1.10
    conditions:
      ready: true
      serving: true
      terminating: false
    hostname: backend-1
    nodeName: worker-node-1
    targetRef:
      kind: Pod
      name: backend-1
      namespace: default
      uid: 12345678-1234-1234-1234-123456789012
    zone: ap-northeast-2a
```

위 예제는 `10.0.1.10` 주소를 가진 Endpoint가 `backend-1` Pod를 참조하며, 현재 준비된 상태임을 나타냅니다.

### 4.3 conditions

`conditions`는 Endpoint의 현재 상태를 나타내는 필드입니다.

주요 상태 필드는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `ready` | Endpoint가 일반적인 트래픽을 처리할 준비가 되었는지를 나타냅니다. |
| `serving` | Endpoint가 현재 트래픽을 처리할 수 있는 상태인지를 나타냅니다. |
| `terminating` | Endpoint가 종료 과정에 있는지를 나타냅니다. |

각 필드의 의미는 다음과 같습니다.

- `ready`: 일반적으로 트래픽을 전달할 수 있는 상태인지 나타냅니다.
- `serving`: 종료 과정과 관계없이 실제로 트래픽을 처리할 수 있는 상태인지 나타냅니다.
- `terminating`: 해당 Endpoint가 종료 과정에 진입했는지를 나타냅니다.

이러한 상태 정보는 Pod의 생성, 준비, 종료 과정에서 트래픽을 적절하게 제어하는 데 사용됩니다.

특히 `ready`와 `serving`은 서로 다른 의미를 가집니다.

예를 들어 Pod가 종료 과정에 진입하면 일반적인 Service 트래픽 대상에서는 제외될 수 있지만, 기존 연결을 처리하거나 종료 유예 시간 동안 요청을 처리할 수 있습니다. 이때 `serving`은 `true`이고 `terminating`은 `true`인 상태가 될 수 있습니다.

또한 상태 필드의 기본 해석은 다음과 같습니다.

- `ready`가 생략되면 `true`로 간주됩니다.
- `serving`이 생략되면 `true`로 간주됩니다.
- `terminating`이 생략되면 `false`로 간주됩니다.

### 4.4 ports

`ports`는 Endpoint에서 사용하는 네트워크 포트 정보를 정의합니다.

| 필드 | 설명 |
|---|---|
| `name` | 포트의 이름입니다. |
| `protocol` | 네트워크 프로토콜입니다. |
| `port` | Endpoint에서 사용하는 실제 포트 번호입니다. |
| `appProtocol` | 애플리케이션 프로토콜을 나타내는 선택적 힌트입니다. |

예를 들어 다음과 같이 정의할 수 있습니다.

```yaml
ports:
  - name: http
    protocol: TCP
    port: 8080
    appProtocol: http
```

여기서 `port: 8080`은 실제 Endpoint에서 요청을 수신하는 포트입니다.

Service의 `port`와 혼동하지 않도록 주의해야 합니다. Service의 `port`는 클라이언트가 접근하는 Service 포트이며, EndpointSlice의 `ports`는 실제 Endpoint에서 사용하는 포트입니다.

## 5. EndpointSlice 예제

다음은 일반적인 Service에 연결되는 EndpointSlice의 예제입니다.

먼저 EndpointSlice가 참조하는 `backend-1`과 `backend-2` Pod를 다음과 같이 정의합니다. 두 Pod는 모두 `app: backend` Label을 가지고 있으며, 각각 `default` Namespace에 생성됩니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend-1
  namespace: default
  labels:
    app: backend
spec:
  containers:
    - name: backend
      image: nginx
      ports:
        - name: http
          containerPort: 8080
---
apiVersion: v1
kind: Pod
metadata:
  name: backend-2
  namespace: default
  labels:
    app: backend
spec:
  containers:
    - name: backend
      image: nginx
      ports:
        - name: http
          containerPort: 8080
```

다음은 위 Pod들을 참조하는 EndpointSlice 예제입니다.

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: backend-service-abcde
  namespace: default
  labels:
    kubernetes.io/service-name: backend-service
    endpointslice.kubernetes.io/managed-by: endpointslice-controller.k8s.io
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 8080
endpoints:
  - addresses:
      - 10.0.1.10
    conditions:
      ready: true
    targetRef:
      kind: Pod
      name: backend-1
      namespace: default
  - addresses:
      - 10.0.1.11
    conditions:
      ready: true
    targetRef:
      kind: Pod
      name: backend-2
      namespace: default
```

위 예제에서 `backend-1`과 `backend-2`는 앞서 정의한 Pod의 이름입니다.

- `targetRef.kind`: 참조 대상이 Pod임을 나타냅니다.
- `targetRef.name`: 참조할 Pod의 이름을 지정합니다.
- `targetRef.namespace`: 참조할 Pod가 속한 Namespace를 지정합니다.
- `addresses`: 해당 Pod에 할당된 IP 주소를 나타냅니다.

여기서 `10.0.1.10`과 `10.0.1.11`은 설명을 위한 예시 IP 주소입니다. 실제 환경에서는 각 Pod에 Kubernetes가 할당한 IP 주소를 사용해야 합니다.

각 필드의 의미는 다음과 같습니다.

- `kubernetes.io/service-name`: EndpointSlice가 연결된 Service의 이름입니다.
- `endpointslice.kubernetes.io/managed-by`: EndpointSlice를 관리하는 주체를 나타냅니다.
- `addressType`: Endpoint 주소가 IPv4임을 나타냅니다.
- `ports`: Endpoint가 사용하는 포트를 지정합니다.
- `endpoints`: 실제 네트워크 대상의 주소와 상태 정보를 정의합니다.

`kubernetes.io/service-name` Label은 EndpointSlice와 Service를 연결하는 데 사용됩니다.

하나의 Service에 여러 EndpointSlice가 연결될 수 있으며, 해당 Service의 네트워크 구성 요소는 연결된 EndpointSlice들을 종합하여 Endpoint 정보를 확인합니다.

**참고:** 일반적인 Selector 기반 Service에서는 EndpointSlice Controller가 EndpointSlice를 자동으로 생성하고 관리합니다. 따라서 운영 환경에서 Controller가 관리하는 EndpointSlice를 직접 수정하는 방식은 권장되지 않습니다.

## 6. EndpointSlice의 분할 관리 방식

EndpointSlice의 가장 중요한 특징 중 하나는 Endpoint 정보를 여러 리소스로 분할하여 관리한다는 점입니다.

기존 Endpoints 리소스는 하나의 리소스에 모든 Endpoint 정보를 저장했습니다. 따라서 Service에 연결된 Pod가 많아질수록 하나의 큰 리소스를 갱신하고 전달해야 하는 부담이 증가했습니다.

EndpointSlice는 이러한 문제를 해결하기 위해 Endpoint 정보를 여러 개의 EndpointSlice로 나누어 관리합니다.

### 6.1 EndpointSlice의 최대 Endpoint 개수

기본적으로 Kubernetes는 하나의 EndpointSlice에 최대 100개의 Endpoint가 포함되도록 관리합니다.

이 값은 `kube-controller-manager`의 다음 설정을 통해 변경할 수 있습니다.

```bash
--max-endpoints-per-slice
```

최대 1,000개까지 설정할 수 있습니다.

예를 들어 Service에 연결된 Pod가 250개라면 다음과 같이 분할하여 관리할 수 있습니다.

![EndpointSlice를 여러 리소스로 분할하여 관리하는 구조](https://images.openai.com/static-rsc-4/_gjDdMbnJQuZaf3rL2UQmzdWpjYM5X35gNdqlsBg_67XO1HFni7NBbrwFhOEF6mf3n38SGrjC2C7btn9JfqhwTPI2OqGeZphklYbYIYUDsDo1jGjIRNE77KEvY8ZAHFH_DiNHAn2Bv0xBY_80Qv3XR4JFrEGF6iol-RUOOc30xo?purpose=inline)

| EndpointSlice | Endpoint 개수 |
|---|---:|
| backend-abc | 100 |
| backend-def | 100 |
| backend-ghi | 50 |
| **합계** | **250** |

실제 분할 결과가 항상 위와 같이 정확하게 구성되는 것은 아닙니다.

EndpointSlice Controller는 기존 리소스의 변경을 최소화하는 방향으로 Endpoint를 배치합니다. 따라서 모든 EndpointSlice를 최대한 가득 채우는 것보다 불필요한 리소스 갱신을 줄이는 것을 우선할 수 있습니다.

### 6.2 EndpointSlice 분할 기준

EndpointSlice는 다음과 같은 조건의 조합을 기준으로 Endpoint를 그룹화합니다.

- Service 이름
- IP 주소 패밀리
- 네트워크 프로토콜
- 포트 번호 및 포트 구성

특히 하나의 EndpointSlice에 포함된 `ports` 정보는 해당 EndpointSlice의 모든 Endpoint에 공통으로 적용됩니다.

예를 들어 동일한 Service를 대상으로 하더라도 Pod마다 동일한 이름의 포트가 서로 다른 번호로 해석된다면 별도의 EndpointSlice가 필요할 수 있습니다.

### 6.3 EndpointSlice의 변경 최소화

EndpointSlice Controller는 Endpoint가 변경될 때 모든 EndpointSlice를 재구성하지 않습니다.

일반적으로 다음과 같은 순서로 처리합니다.

1. 기존 EndpointSlice에서 더 이상 필요하지 않은 Endpoint를 제거하거나 변경된 정보를 갱신합니다.
2. 변경된 EndpointSlice에 새로운 Endpoint를 추가합니다.
3. 추가할 Endpoint가 남아 있으면 기존 EndpointSlice 중 변경되지 않은 리소스를 활용하거나 새로운 EndpointSlice를 생성합니다.

이러한 방식은 EndpointSlice의 완전한 균등 분배보다 변경 횟수를 줄이는 데 초점을 맞춥니다.

EndpointSlice의 변경 사항은 kube-proxy를 실행하는 각 Node에도 전달되므로, 불필요한 갱신을 줄이는 것은 대규모 클러스터의 네트워크 관리 효율성에 중요합니다.

## 7. Topology Aware Routing

EndpointSlice는 Endpoint가 위치한 Node와 Zone 정보를 제공하며, 이를 활용하여 네트워크 트래픽의 이동 범위를 최적화할 수 있습니다.

Topology Aware Routing은 가능한 경우 요청을 발생시킨 Zone과 동일한 Zone에 위치한 Endpoint로 전달하도록 유도하는 기능입니다.

예를 들어 다음과 같이 서로 다른 Zone에 Pod가 배포되어 있다고 가정하겠습니다.

| Zone | Pod |
|---|---|
| ap-northeast-2a | backend-1, backend-2 |
| ap-northeast-2b | backend-3, backend-4 |
| ap-northeast-2c | backend-5, backend-6 |

일반적인 Service 트래픽 전달에서는 클러스터 전체의 Endpoint가 선택 대상이 될 수 있습니다.

Topology Aware Routing을 적용하면 다음과 같은 동작을 유도할 수 있습니다.

- `2a` Zone에서 발생한 요청은 `2a` Zone의 Endpoint를 우선 사용합니다.
- `2b` Zone에서 발생한 요청은 `2b` Zone의 Endpoint를 우선 사용합니다.
- `2c` Zone에서 발생한 요청은 `2c` Zone의 Endpoint를 우선 사용합니다.

이를 통해 Zone 간 네트워크 통신을 줄이고 지연 시간과 네트워크 비용을 절감할 수 있습니다.

### 7.1 Hints

EndpointSlice의 `hints` 필드는 Endpoint를 어떤 Zone의 트래픽에 사용할지에 대한 정보를 제공합니다.

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: backend-service-abc
  labels:
    kubernetes.io/service-name: backend-service
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 8080
endpoints:
  - addresses:
      - 10.0.1.10
    conditions:
      ready: true
    zone: ap-northeast-2a
    hints:
      forZones:
        - name: ap-northeast-2a
```

위 예제에서 해당 Endpoint는 `ap-northeast-2a` Zone의 트래픽을 처리하도록 할당된 힌트를 가지고 있습니다.

EndpointSlice Controller는 설정된 라우팅 정책과 클러스터의 Zone별 리소스 정보를 고려하여 이러한 힌트를 생성할 수 있습니다.

kube-proxy와 같은 네트워크 구성 요소는 EndpointSlice의 힌트를 활용하여 Endpoint 선택 범위를 조정합니다.

**주의사항:** Topology Aware Routing은 동일 Zone의 Endpoint 사용을 선호하도록 하는 기능이지, 모든 요청이 반드시 동일 Zone에서 처리된다는 보장은 아닙니다. Endpoint 수가 부족하거나 Zone별 분포가 적절하지 않은 경우에는 다른 Zone의 Endpoint가 사용될 수 있습니다.

## 8. Endpoint 중복과 집계

하나의 Service에 여러 EndpointSlice가 연결되어 있을 때 동일한 Endpoint가 서로 다른 EndpointSlice에 중복으로 나타날 수 있습니다.

이는 EndpointSlice가 개별 리소스로 갱신되고 Kubernetes API의 Watch 및 Cache를 통해 변경 사항이 전달되는 과정에서 발생할 수 있는 현상입니다.

예를 들어 다음과 같은 상황을 가정할 수 있습니다.

| EndpointSlice | Endpoint |
|---|---|
| backend-abc | 10.0.1.10, 10.0.1.11 |
| backend-def | 10.0.1.11, 10.0.1.12 |

이 경우 `10.0.1.11`이 두 EndpointSlice에 중복으로 포함되어 있습니다.

따라서 EndpointSlice를 사용하는 클라이언트나 네트워크 구현체는 특정 Service에 연결된 모든 EndpointSlice를 조회하고, Endpoint 정보를 종합하여 중복을 처리해야 합니다.

EndpointSlice 하나만 조회하여 Service의 전체 Endpoint 목록이라고 판단해서는 안 됩니다.

## 9. EndpointSlice와 Endpoints의 차이

Endpoints는 EndpointSlice가 도입되기 전에 사용되던 Kubernetes 리소스입니다.

두 리소스 모두 Service에 연결된 네트워크 대상 정보를 관리하지만, Endpoint 정보를 구성하고 관리하는 방식에 차이가 있습니다.

| 구분 | Endpoints | EndpointSlice |
|---|---|---|
| API 그룹 | `v1` | `discovery.k8s.io/v1` |
| 관리 방식 | 하나의 리소스에 전체 Endpoint 저장 | 여러 리소스로 분할 관리 |
| 확장성 | Endpoint가 많아질수록 비효율적 | 대규모 Endpoint 관리에 유리 |
| 주소 패밀리 | IPv4, IPv6 주소 정보 | IPv4, IPv6별로 분리 관리 |
| 상태 정보 | 제한적인 상태 정보 | Ready, Serving, Terminating 제공 |
| 토폴로지 정보 | 제한적 | Node, Zone, Hints 제공 |
| 권장 사용 | 기존 호환성 유지 | 신규 구현 및 일반적인 사용 |

### 9.1 Endpoints의 한계

기존 Endpoints 리소스는 하나의 Service에 연결된 모든 Endpoint를 하나의 리소스에 저장합니다.

따라서 Endpoint가 수천 개 이상으로 증가하면 다음과 같은 문제가 발생할 수 있습니다.

- 하나의 큰 리소스를 갱신해야 하므로 API Server의 처리 부담이 증가합니다.
- Endpoint 일부만 변경되어도 전체 리소스가 갱신될 수 있습니다.
- 변경된 리소스가 클러스터의 네트워크 구성 요소에 전달되면서 불필요한 데이터 전송이 발생할 수 있습니다.

EndpointSlice는 이러한 문제를 해결하기 위해 Endpoint 정보를 여러 리소스로 분할하여 관리합니다.

### 9.2 Endpoints API의 사용 중단

Endpoints는 기존 워크로드와의 호환성을 위해 여전히 제공되지만, Kubernetes에서는 EndpointSlice 사용을 권장합니다.

Kubernetes v1.33부터 Endpoints API는 Deprecated 상태입니다.

기존 Endpoints 리소스를 사용하는 시스템은 EndpointSlice 기반으로 전환하는 것이 바람직합니다.

또한 기존 Endpoints 리소스를 생성하면 호환성을 위해 EndpointSlice로 미러링되는 기능이 제공되지만, 이 미러링 기능 역시 Kubernetes v1.33부터 Deprecated 상태입니다.

따라서 Selector가 없는 Service에 외부 Endpoint를 연결하는 경우에도 기존 Endpoints를 생성하여 미러링하는 방식보다는 EndpointSlice를 직접 생성하는 방식을 권장합니다.

## 10. EndpointSlice 확인 및 관리 명령어

EndpointSlice를 조회하고 관리할 때 사용하는 대표적인 kubectl 명령어는 다음과 같습니다.

### EndpointSlice 목록 조회

```bash
kubectl get endpointslices
```

또는 다음과 같이 축약할 수 있습니다.

```bash
kubectl get endpointslice
```

### 특정 Service에 연결된 EndpointSlice 조회

```bash
kubectl get endpointslices \
  -l kubernetes.io/service-name=backend-service
```

### EndpointSlice 상세 정보 조회

```bash
kubectl describe endpointslice backend-service-abc
```

### EndpointSlice YAML 조회

```bash
kubectl get endpointslice backend-service-abc -o yaml
```

### 모든 Namespace의 EndpointSlice 조회

```bash
kubectl get endpointslices -A
```

### EndpointSlice의 관리 주체 확인

```bash
kubectl get endpointslices \
  -l kubernetes.io/service-name=backend-service \
  -o custom-columns=NAME:.metadata.name,MANAGED-BY:.metadata.labels.endpointslice\\.kubernetes\\.io/managed-by
```

### EndpointSlice 삭제

```bash
kubectl delete endpointslice backend-service-abc
```

Selector가 지정된 일반적인 Service의 EndpointSlice는 EndpointSlice Controller가 관리하므로, 수동으로 삭제하더라도 조건에 따라 다시 생성될 수 있습니다.

운영 환경에서는 Controller가 관리하는 EndpointSlice를 직접 수정하거나 삭제하기보다는 Service와 대상 Pod의 설정을 변경하는 것이 일반적입니다.

## 11. EndpointSlice 사용 시 주의사항

EndpointSlice를 활용할 때는 다음 사항을 고려해야 합니다.

1. **전체 EndpointSlice 조회:** 하나의 Service에 여러 EndpointSlice가 연결될 수 있으므로 특정 Service의 Endpoint를 조회할 때는 모든 EndpointSlice를 확인해야 합니다.
2. **중복 Endpoint 처리:** 서로 다른 EndpointSlice에 동일한 Endpoint가 중복으로 나타날 수 있으므로 클라이언트 구현 시 중복 처리가 필요합니다.
3. **Controller 관리 리소스 직접 수정 금지:** EndpointSlice Controller가 관리하는 리소스는 Service와 Pod의 상태에 따라 자동으로 갱신되므로 직접 수정하는 방식은 피해야 합니다.
4. **Endpoint 상태 확인:** `ready`, `serving`, `terminating`을 구분하여 Pod의 준비 및 종료 상태를 정확하게 해석해야 합니다.
5. **Zone 정보 활용:** 다중 Zone 환경에서는 `zone`과 `hints`를 활용하여 네트워크 트래픽을 최적화할 수 있습니다.
6. **Endpoints 대신 EndpointSlice 사용:** Deprecated된 Endpoints API보다 EndpointSlice API를 사용하는 것이 바람직합니다.

**참고:** `ready`, `serving`, `terminating` 필드가 생략된 경우의 기본 해석은 서로 다릅니다. API 정의상 `ready`와 `serving`은 `true`, `terminating`은 `false`로 해석합니다.

또한 주소 유형과 관련하여 현재 `discovery.k8s.io/v1` EndpointSlice에서 지원하는 주소 유형은 IPv4와 IPv6입니다. FQDN은 현재 해당 API의 지원 주소 유형에 포함되지 않습니다.

## 12. 정리

EndpointSlice는 Kubernetes에서 Service에 연결된 네트워크 Endpoint 정보를 효율적으로 관리하기 위한 리소스입니다.

기존 Endpoints 리소스의 확장성 문제를 해결하기 위해 도입되었으며, 대규모 클러스터에서 다수의 Pod를 안정적으로 관리할 수 있도록 지원합니다.

EndpointSlice의 주요 특징은 다음과 같습니다.

- Service에 연결된 Pod의 IP 주소와 포트 정보를 관리합니다.
- Endpoint 정보를 여러 EndpointSlice로 분할하여 관리합니다.
- `ready`, `serving`, `terminating`을 통해 Endpoint의 상태를 구분합니다.
- IPv4와 IPv6 주소 패밀리를 지원합니다.
- Node 및 Zone 정보를 활용하여 네트워크 토폴로지를 파악할 수 있습니다.
- Topology Aware Routing을 통해 Zone 간 네트워크 트래픽을 최적화할 수 있습니다.
- Selector가 없는 Service에서도 직접 생성한 EndpointSlice를 연결하여 외부 네트워크 대상을 관리할 수 있습니다.

Service가 클라이언트에 안정적인 네트워크 접근 지점을 제공한다면, EndpointSlice는 해당 Service가 실제로 트래픽을 전달할 네트워크 대상을 관리합니다.

두 리소스는 Kubernetes의 서비스 디스커버리와 네트워크 트래픽 전달 구조를 구성하는 핵심 요소입니다.

## 공식 문서 및 참고 자료

- [Kubernetes 공식 문서 - EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Kubernetes API Reference - EndpointSlice v1](https://kubernetes.io/docs/reference/kubernetes-api/discovery/endpoint-slice-v1/)
- [Kubernetes 공식 문서 - Topology Aware Routing](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/)
- [Kubernetes 공식 블로그 - Scaling Kubernetes Networking with EndpointSlices](https://kubernetes.io/blog/2020/09/02/scaling-kubernetes-networking-with-endpointslices/)
- [Kubernetes v1.33 Release Announcement](https://kubernetes.io/blog/2025/04/23/kubernetes-v1-33-release/)