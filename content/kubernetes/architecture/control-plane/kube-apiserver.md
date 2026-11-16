# kube-apiserver

## 1. kube-apiserver란?

`kube-apiserver`는 Kubernetes Control Plane의 핵심 구성 요소로, Kubernetes API를 외부에 제공하는 API 서버입니다.

Kubernetes 클러스터의 모든 리소스에 대한 생성(Create), 조회(Read), 수정(Update), 삭제(Delete) 작업은 Kubernetes API를 통해 수행됩니다. `kube-apiserver`는 이러한 API 요청을 수신하고 처리하는 Control Plane의 진입점(Frontend) 역할을 담당합니다.

사용자는 `kubectl`을 통해 Kubernetes 리소스를 관리할 수 있으며, 내부적으로 `kubectl`은 Kubernetes API Server에 HTTP 요청을 전달합니다. 또한 Scheduler, Controller Manager, Kubelet 등 다른 Kubernetes 컴포넌트도 API Server를 통해 클러스터의 상태를 조회하거나 변경합니다.

`kube-apiserver`의 주요 특징은 다음과 같습니다.

- **API 제공:** Kubernetes 리소스를 관리하기 위한 RESTful HTTP API를 제공합니다.
- **요청 처리:** 클라이언트의 API 요청을 수신하고 유효성을 검증합니다.
- **접근 제어:** 인증(Authentication), 인가(Authorization), Admission Control을 통해 API 접근을 제어합니다.
- **상태 저장 연계:** Kubernetes 클러스터의 영속적인 상태를 etcd에 저장하고 조회합니다.
- **컴포넌트 간 통신:** Kubernetes의 다른 컴포넌트가 클러스터 상태를 조회하고 변경할 수 있도록 API 인터페이스를 제공합니다.

`kube-apiserver`는 Kubernetes 클러스터의 중앙 API 인터페이스이자 Control Plane의 핵심 통신 지점입니다.

## 2. 주요 역할

### 2.1 Kubernetes API 제공

`kube-apiserver`는 Kubernetes 리소스를 관리하기 위한 HTTP 기반 API를 제공합니다.

Kubernetes에서 사용하는 주요 리소스는 API를 통해 관리됩니다.

| API 리소스 | 주요 역할 |
|---|---|
| Pod | 컨테이너 실행 단위 관리 |
| Deployment | 애플리케이션 배포 및 Replica 관리 |
| Service | 애플리케이션 네트워크 접근 관리 |
| Node | 클러스터 노드 정보 관리 |
| ConfigMap | 애플리케이션 설정 데이터 관리 |
| Secret | 민감한 설정 데이터 관리 |
| PersistentVolume | 클러스터 스토리지 리소스 관리 |

예를 들어 다음 명령어를 실행하면 `kubectl`은 API Server에 Pod 목록을 조회하는 API 요청을 전달합니다.

```bash
kubectl get pods -n default
```

API Server는 요청을 처리한 후 조회 결과를 응답합니다.

Kubernetes API는 RESTful 인터페이스를 기반으로 하며, 일반적으로 HTTPS를 통해 제공됩니다.

### 2.2 클러스터 상태 관리

`kube-apiserver`는 Kubernetes 클러스터의 상태를 조회하고 변경하는 요청을 처리합니다.

클러스터의 영속적인 상태는 etcd에 저장되며, API Server는 etcd와 통신하는 주요 인터페이스 역할을 담당합니다.

예를 들어 Deployment를 생성하면 다음과 같은 과정이 수행됩니다.

1. 클라이언트가 Deployment 생성 요청을 API Server에 전달합니다.
2. API Server가 요청의 유효성과 접근 권한을 검증합니다.
3. 요청이 정상적으로 처리되면 Deployment 객체가 etcd에 저장됩니다.
4. Controller Manager가 API Server를 통해 Deployment의 상태를 확인합니다.
5. Deployment Controller가 원하는 Replica 수를 충족하기 위해 ReplicaSet을 생성합니다.
6. Scheduler가 새롭게 생성된 Pod를 적절한 Node에 할당합니다.

이처럼 API Server는 Kubernetes 리소스의 상태를 관리하는 중앙 인터페이스로 동작합니다.

단, 실제 리소스의 상태를 지속적으로 조정하는 역할은 각 Controller와 다른 컴포넌트가 담당합니다.

### 2.3 인증 및 인가 처리

API Server는 Kubernetes API에 접근하는 클라이언트의 신원을 확인하고 요청 권한을 검증합니다.

이를 위해 다음과 같은 접근 제어 메커니즘을 제공합니다.

- Authentication: 요청을 보낸 클라이언트의 신원을 확인합니다.
- Authorization: 인증된 클라이언트가 요청한 작업을 수행할 권한이 있는지 확인합니다.
- Admission Control: 리소스 생성 및 변경 요청이 클러스터의 정책과 제약 조건을 만족하는지 검증합니다.

세부적인 처리 과정은 3장에서 설명합니다.

### 2.4 컴포넌트 간 통신의 중심

Kubernetes의 주요 컴포넌트는 API Server를 통해 클러스터 상태를 공유합니다.

대표적인 상호작용은 다음과 같습니다.

| 컴포넌트 | API Server와의 상호작용 |
|---|---|
| kubectl | 리소스 생성, 조회, 수정, 삭제 요청 |
| kube-scheduler | 스케줄링 대상 Pod 조회 및 Node 할당 결과 반영 |
| kube-controller-manager | 리소스 상태 감시 및 원하는 상태와 실제 상태의 차이 조정 |
| kubelet | 담당 Node의 Pod 상태 보고 및 실행에 필요한 정보 조회 |
| kube-proxy | Service 및 EndpointSlice 등의 네트워크 관련 정보 감시 |

API Server를 중심으로 컴포넌트 간 통신이 이루어지므로 각 컴포넌트는 서로의 내부 구현에 직접 의존하지 않고 Kubernetes API를 통해 협력할 수 있습니다.

## 3. API 요청 처리 과정

API Server는 클라이언트로부터 전달받은 요청에 대해 여러 단계의 검증과 처리를 수행합니다.

일반적인 리소스 생성 및 변경 요청은 요청의 인증과 인가, Admission Control, 객체 검증 및 저장 등의 과정을 거칩니다.

### 3.1 전체 처리 흐름

```text
Client
  |
  | HTTP Request
  v
kube-apiserver
  |
  v
Authentication
  |
  v
Authorization
  |
  v
Mutating Admission Control
  |
  v
Schema Validation
  |
  v
Validating Admission Control
  |
  v
etcd Persistence
  |
  v
HTTP Response
```

위 흐름은 API Server의 요청 처리 과정을 이해하기 위한 개념적인 표현입니다. 실제 API Server 내부에서는 요청의 종류와 처리 경로에 따라 디코딩, 기본값 설정, 필드 검증, 변환 등의 추가 단계가 수행되며, 모든 요청이 동일한 처리 경로를 거치는 것은 아닙니다.

특히 Authentication, Authorization, Admission Control은 서로 다른 목적을 가진 처리 단계입니다.

- **Authentication:** 요청을 보낸 클라이언트가 누구인지 확인합니다.
- **Authorization:** 해당 클라이언트가 요청한 작업을 수행할 권한이 있는지 확인합니다.
- **Admission Control:** 요청된 리소스의 생성이나 변경이 클러스터의 정책과 제약 조건을 만족하는지 검증하거나 객체를 변경합니다.

다음은 일반적인 리소스 생성 및 변경 요청을 기준으로 각 단계의 역할을 설명합니다.

### 3.2 Authentication

Authentication은 API 요청을 보낸 클라이언트의 신원을 확인하는 단계입니다.

Kubernetes API Server는 다양한 인증 방식을 지원합니다.

대표적인 인증 방식은 다음과 같습니다.

| 인증 방식 | 설명 |
|---|---|
| X.509 Client Certificate | 클라이언트 인증서를 이용한 신원 확인 |
| Bearer Token | HTTP Authorization 헤더에 포함된 토큰을 이용한 인증 |
| Service Account Token | Kubernetes ServiceAccount에 연결된 토큰을 이용한 인증 |
| Authentication Webhook | 외부 인증 서비스를 통해 토큰의 유효성을 확인 |
| OIDC | OpenID Connect 기반의 인증 정보 활용 |

인증이 성공하면 API Server는 요청을 보낸 사용자의 이름과 그룹 등의 신원 정보를 확인합니다.

인증에 실패한 요청은 일반적으로 거부됩니다.

### 3.3 Authorization

Authorization은 인증된 클라이언트가 요청한 Kubernetes 리소스와 작업에 대한 권한을 가지고 있는지 확인하는 단계입니다.

Kubernetes는 여러 가지 인가 방식을 지원합니다.

| 인가 방식 | 설명 |
|---|---|
| Node | Kubelet의 API 요청 권한을 제어하는 인가 모듈 |
| RBAC | Role과 RoleBinding 등을 이용한 역할 기반 접근 제어 |
| ABAC | 속성 기반 정책을 이용한 접근 제어 |
| Webhook | 외부 서비스에 권한 확인을 위임하는 방식 |

일반적으로 Kubernetes 클러스터에서는 RBAC를 주요 인가 방식으로 사용합니다.

예를 들어 특정 사용자가 `default` Namespace의 Pod를 조회할 수 있는지 확인하려면 다음 요소를 검토합니다.

- 요청 사용자
- API Group
- Resource
- Namespace
- Verb

다음은 RBAC를 통해 Pod 조회 권한을 부여하는 예시입니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

API Server는 해당 요청의 속성을 인가 정책과 비교하여 요청의 허용 여부를 결정합니다.

인가 단계에서 요청이 거부되면 일반적으로 HTTP 403 Forbidden 응답이 반환됩니다.

### 3.4 Admission Control

Admission Control은 인증과 인가가 완료된 요청에 대해 리소스 생성 또는 변경을 허용할지 추가로 검증하는 단계입니다.

Admission Controller는 API Server 내부에서 동작하며, 리소스 생성(Create), 수정(Update), 삭제(Delete) 등의 요청에 적용됩니다.

Admission Control은 크게 두 가지 유형으로 구분됩니다.

**Mutating Admission Controller**

요청으로 전달된 리소스 객체를 변경할 수 있는 Admission Controller입니다.

주요 역할은 다음과 같습니다.

- 누락된 필드에 기본값 적용
- 리소스 객체의 특정 필드 자동 변경
- 정책에 따른 리소스 설정 보정

예를 들어 Pod 생성 요청에 별도의 ServiceAccount가 지정되지 않았다면 기본 ServiceAccount를 적용하는 등의 처리가 이루어질 수 있습니다.

**Validating Admission Controller**

요청으로 전달된 리소스 객체가 특정 정책이나 제약 조건을 만족하는지 검증하는 Admission Controller입니다.

주요 역할은 다음과 같습니다.

- 리소스 생성 및 변경 정책 검증
- 허용되지 않는 설정 차단
- 보안 정책 및 리소스 제약 조건 검증

예를 들어 특정 Namespace에서 Privileged Container 실행을 금지하는 정책을 적용할 수 있습니다.

Admission Control은 일반적으로 다음 순서로 진행됩니다.

1. Mutating Admission Controller 실행
2. Validating Admission Controller 실행
3. 모든 검증이 통과하면 다음 처리 단계 진행

Admission Controller 중 하나라도 요청을 거부하면 해당 요청은 처리되지 않습니다.

Admission Webhook을 이용하면 클러스터 외부의 별도 서비스에 리소스 검증이나 변경 작업을 위임할 수도 있습니다.

### 3.5 Schema Validation

API Server는 요청으로 전달된 Kubernetes 리소스 객체가 해당 API 리소스의 스키마와 유효성 조건을 만족하는지 검증합니다.

대표적인 검증 대상은 다음과 같습니다.

- 필드의 데이터 타입
- 필수 필드의 존재 여부
- 필드 값의 허용 범위
- API 버전 및 리소스 형식
- 리소스별 필드 제약 조건

예를 들어 Pod의 `spec.containers`에 필수 컨테이너 정보가 누락되거나 허용되지 않는 필드 값이 전달되면 요청이 거부될 수 있습니다.

Custom Resource Definition(CRD)을 통해 추가된 사용자 정의 리소스 역시 정의된 스키마에 따라 검증할 수 있습니다.

### 3.6 Persistence

모든 검증 및 처리 과정을 통과한 리소스 변경 요청은 etcd에 저장됩니다.

etcd는 Kubernetes 클러스터의 영속적인 상태 데이터를 저장하는 분산 Key-Value 저장소입니다.

API Server는 etcd와 통신하여 다음과 같은 작업을 수행합니다.

- 리소스 생성 및 저장
- 리소스 조회
- 리소스 변경 및 삭제
- 리소스 상태 변경 사항 감시를 위한 Watch 처리

리소스 변경 요청이 정상적으로 저장되면 API Server는 클라이언트에 처리 결과를 응답합니다.

중요한 점은 API Server가 애플리케이션 컨테이너를 직접 실행하거나 Pod를 특정 Node에 배치하지 않는다는 것입니다. 이러한 작업은 Scheduler, Controller Manager, Kubelet 등의 다른 컴포넌트가 담당합니다.

## 4. API Server와 etcd의 관계

`kube-apiserver`와 `etcd`는 Kubernetes Control Plane에서 밀접하게 연관된 구성 요소입니다.

두 컴포넌트의 역할은 명확하게 구분됩니다.

| 구성 요소 | 주요 책임 |
|---|---|
| kube-apiserver | Kubernetes API 제공 및 클러스터 상태 변경 요청 처리 |
| etcd | Kubernetes 클러스터의 영속적인 상태 데이터 저장 |

API Server는 etcd에 직접 접근하는 대신 Kubernetes API를 제공하는 중앙 인터페이스 역할을 담당합니다.

일반적으로 Kubernetes의 다른 컴포넌트와 클라이언트는 etcd에 직접 접근하지 않고 API Server를 통해 데이터를 조회하거나 변경합니다.

이러한 구조는 데이터 접근 경로를 중앙화하여 인증, 인가, 검증 및 정책 적용을 일관되게 수행할 수 있도록 합니다.

```text
                 Client
                   |
                   v
             kube-apiserver
                   |
          +--------+--------+
          |                 |
          v                 v
    Authentication    Authorization
          |                 |
          +--------+--------+
                   |
                   v
             Admission Control
                   |
                   v
                 etcd
                   |
                   v
          Persistent Cluster State
```

etcd의 데이터가 손상되거나 접근할 수 없는 상태가 되면 API Server의 리소스 조회 및 변경 작업에도 영향을 줄 수 있습니다.

따라서 etcd 백업과 복구 전략은 Kubernetes 클러스터 운영에서 중요한 요소입니다.

## 5. API Server와 다른 컴포넌트의 관계

API Server는 Kubernetes Control Plane 및 Node Components가 협력할 수 있도록 중앙 API 인터페이스를 제공합니다.

### 5.1 kube-scheduler

`kube-scheduler`는 아직 Node에 할당되지 않은 Pod를 감시하고 적절한 Node를 선택합니다.

일반적인 상호작용 과정은 다음과 같습니다.

1. API Server에 새로운 Pod가 생성됩니다.
2. kube-scheduler가 Node에 할당되지 않은 Pod를 감시합니다.
3. Scheduler가 Node의 리소스와 제약 조건 등을 검토합니다.
4. 적절한 Node를 선택합니다.
5. API Server를 통해 Pod의 Node 할당 정보를 반영합니다.

API Server는 Pod 생성 및 변경 결과를 관리하고, Scheduler는 실제 스케줄링 결정을 담당합니다.

### 5.2 kube-controller-manager

`kube-controller-manager`는 Kubernetes 리소스의 원하는 상태(Desired State)와 실제 상태(Actual State)를 지속적으로 비교하고 차이를 조정하는 Controller들을 실행합니다.

예를 들어 Deployment의 Replica 수가 3으로 설정되어 있다면 Deployment Controller는 현재 ReplicaSet의 상태를 확인하고 필요한 리소스를 생성하거나 변경합니다.

일반적인 상호작용 과정은 다음과 같습니다.

1. Controller가 API Server를 통해 관심 있는 리소스를 감시합니다.
2. 리소스의 원하는 상태와 실제 상태를 비교합니다.
3. 차이가 발견되면 필요한 작업을 결정합니다.
4. API Server에 리소스 생성 또는 변경 요청을 전달합니다.

Controller는 API Server를 통해 리소스 상태를 확인하고 변경합니다.

### 5.3 kubelet

`kubelet`은 각 Node에서 실행되는 에이전트로, 해당 Node에 할당된 Pod의 컨테이너가 정상적으로 실행되도록 관리합니다.

일반적인 상호작용 과정은 다음과 같습니다.

1. API Server를 통해 Pod가 특정 Node에 할당됩니다.
2. 해당 Node의 kubelet이 자신에게 할당된 Pod 정보를 확인합니다.
3. kubelet이 Container Runtime과 협력하여 컨테이너를 실행합니다.
4. kubelet이 Pod 및 컨테이너 상태를 API Server에 보고합니다.

API Server와 kubelet의 통신은 양방향으로 이루어집니다.

kubelet은 API Server에 접근하여 Pod 정보를 조회하고 상태를 보고하며, API Server는 로그 조회, 원격 명령 실행, 포트 포워딩 등의 일부 기능을 위해 kubelet의 HTTPS 엔드포인트에 연결할 수 있습니다.

### 5.4 전체 컴포넌트 상호작용

```text
                  kubectl
                     |
                     v
              kube-apiserver
                     |
                     v
                    etcd
                     |
          +----------+----------+
          |          |          |
          v          v          v
      Scheduler  Controller  Other
                 Manager     Components
          |          |
          |          |
          +-----+----+
                |
                v
          kube-apiserver
                |
                v
              kubelet
                |
                v
         Container Runtime
                |
                v
          Running Containers
```

위 구조에서 API Server는 컴포넌트 간의 상태 공유와 변경 요청을 중개합니다.

각 컴포넌트는 자신의 책임에 해당하는 작업을 수행하며, API Server를 통해 클러스터 상태를 공유합니다.

## 6. 주요 아키텍처 특징

### 6.1 수평 확장(Horizontal Scaling)

`kube-apiserver`는 수평 확장이 가능하도록 설계되어 있습니다.

단일 인스턴스에 모든 API 요청을 집중시키는 대신 여러 개의 API Server 인스턴스를 실행하고 Load Balancer를 통해 요청을 분산할 수 있습니다.

```text
                API Clients
                     |
                     v
                Load Balancer
                     |
           +---------+---------+
           |         |         |
           v         v         v
        API Server API Server API Server
           |         |         |
           +---------+---------+
                     |
                     v
                    etcd
```

이러한 구성은 다음과 같은 장점을 제공합니다.

- API 요청 처리량 증가
- API Server 인스턴스 장애에 대한 가용성 향상
- 클러스터 규모 증가에 따른 확장성 확보

다만 API Server 인스턴스만 확장한다고 해서 전체 Control Plane의 가용성이 자동으로 보장되는 것은 아닙니다. etcd의 가용성과 네트워크 및 Load Balancer 구성도 함께 고려해야 합니다.

### 6.2 Stateless API Server

API Server 인스턴스는 리소스의 영속적인 상태를 자체 로컬 데이터에 저장하지 않고 etcd를 통해 관리합니다.

따라서 여러 API Server 인스턴스가 동일한 etcd를 사용하도록 구성할 수 있습니다.

이러한 구조는 API Server 인스턴스의 수평 확장과 장애 대응에 유리합니다.

다만 API Server는 인증 정보, 설정, 캐시 등 런타임 상태를 가질 수 있으므로 완전히 무상태(Stateless)인 서버라고 단정하기보다는 영속적인 클러스터 상태를 etcd에 저장하는 구조로 이해하는 것이 적절합니다.

### 6.3 Watch 메커니즘

Kubernetes API는 리소스 변경 사항을 지속적으로 감시할 수 있는 Watch 기능을 제공합니다.

Controller와 Scheduler 등의 컴포넌트는 API Server의 Watch API를 이용하여 관심 있는 리소스의 변경 사항을 전달받을 수 있습니다.

예를 들어 Deployment Controller는 Pod 또는 ReplicaSet의 상태 변경을 감시하고, 변경 사항에 따라 필요한 조정 작업을 수행합니다.

Watch 메커니즘은 모든 리소스를 반복적으로 조회하는 방식에 비해 효율적인 상태 감시를 가능하게 합니다.

다만 Watch 연결이 항상 유지되는 것은 아니므로 클라이언트는 연결 종료, 재연결 및 Resource Version을 고려한 재동기화 처리를 수행해야 합니다.

### 6.4 API 확장성

Kubernetes API Server는 기본 제공 리소스 외에도 API 확장 기능을 지원합니다.

대표적인 확장 방식은 다음과 같습니다.

**Custom Resource Definition(CRD)**

CRD를 이용하면 Kubernetes API에 사용자 정의 리소스 타입을 추가할 수 있습니다.

예를 들어 기본 Kubernetes API에서 제공하지 않는 `Database`라는 리소스를 정의하고 Kubernetes API를 통해 관리할 수 있습니다.

**API Aggregation**

API Aggregation은 별도의 API Server를 Kubernetes API에 연결하여 API 기능을 확장하는 방식입니다.

기본 API Server는 API Aggregation Layer를 통해 외부 API Server로 요청을 전달할 수 있습니다.

두 방식은 모두 Kubernetes API를 확장하지만 동작 방식은 다릅니다.

| 구분 | CRD | API Aggregation |
|---|---|---|
| 확장 방식 | 리소스 타입 추가 | 별도 API Server 연결 |
| API 구현 | Kubernetes API Server가 제공 | 외부 API Server가 제공 |
| 주요 목적 | 사용자 정의 리소스 관리 | 독립적인 API 구현 및 통합 |

## 7. 주요 실행 옵션

`kube-apiserver`는 다양한 실행 옵션을 통해 API 제공 주소, 인증, 인가, Admission Control 및 etcd 연결 등을 설정할 수 있습니다.

대표적인 옵션은 다음과 같습니다.

| 옵션 | 설명 |
|---|---|
| `--bind-address` | API Server가 바인딩할 네트워크 주소 |
| `--secure-port` | HTTPS API 요청을 수신할 포트 |
| `--advertise-address` | 다른 컴포넌트에 알릴 API Server 주소 |
| `--etcd-servers` | etcd 서버 연결 주소 |
| `--client-ca-file` | 클라이언트 인증서 검증에 사용할 CA 인증서 |
| `--authorization-mode` | API 요청의 인가 방식 설정 |
| `--enable-admission-plugins` | 활성화할 Admission Controller 지정 |
| `--disable-admission-plugins` | 비활성화할 Admission Controller 지정 |
| `--audit-log-path` | Audit Log 저장 경로 |

실행 옵션은 Kubernetes 버전과 배포 방식에 따라 차이가 있을 수 있습니다.

일반적인 kubeadm 기반 클러스터에서는 API Server가 Static Pod 형태로 실행되며, 설정은 다음 경로의 Manifest에서 확인할 수 있습니다.

```text
/etc/kubernetes/manifests/kube-apiserver.yaml
```

해당 파일에는 API Server의 실행 이미지, Command Arguments, Volume Mount 및 기타 실행 설정이 포함됩니다.

## 8. 정리

`kube-apiserver`는 Kubernetes Control Plane의 중심에서 Kubernetes API를 제공하고 클러스터 상태에 대한 접근 및 변경 요청을 처리하는 핵심 컴포넌트입니다.

주요 역할은 다음과 같습니다.

1. Kubernetes 리소스를 관리하기 위한 RESTful API 제공
2. API 요청에 대한 Authentication 및 Authorization 처리
3. Admission Control을 통한 리소스 정책 검증 및 변경
4. etcd와의 연계를 통한 클러스터 상태 조회 및 영속화
5. Scheduler, Controller, Kubelet 등 다른 컴포넌트와의 API 기반 통신
6. 수평 확장 및 API 확장 기능 제공

Kubernetes의 다른 컴포넌트는 API Server를 중심으로 클러스터 상태를 공유하고 각자의 책임에 해당하는 작업을 수행합니다.

따라서 `kube-apiserver`는 단순한 HTTP API 서버가 아니라 Kubernetes의 선언적 리소스 관리와 분산 제어 구조를 연결하는 핵심 인터페이스입니다.

## 9. 참고자료

- [Kubernetes Documentation - Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Kubernetes Documentation - Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Kubernetes Documentation - kube-apiserver](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)
- [Kubernetes Documentation - Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [Kubernetes Documentation - Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [Kubernetes Documentation - Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Kubernetes Documentation - Communication between Nodes and the Control Plane](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/)
- [Kubernetes Documentation - Extending Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/)