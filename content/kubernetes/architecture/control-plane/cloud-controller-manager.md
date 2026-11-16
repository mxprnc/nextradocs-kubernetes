# cloud-controller-manager

## 1. cloud-controller-manager란?

cloud-controller-manager는 Kubernetes Control Plane의 구성 요소로, Kubernetes 클러스터와 클라우드 제공자(Cloud Provider)의 인프라를 연동하고 관리하는 역할을 담당합니다.

Kubernetes는 다양한 클라우드 환경과 온프레미스 환경에서 실행될 수 있도록 설계되어 있습니다. 그러나 클라우드 제공자마다 가상 머신, 네트워크 라우팅, Load Balancer, 스토리지 등의 인프라를 관리하는 방식과 API가 다릅니다.

예를 들어 AWS, Azure, Google Cloud와 같은 클라우드 환경에서는 Kubernetes Node가 실제 클라우드 가상 머신과 연결되어야 하며, `LoadBalancer` 타입 Service를 생성하면 클라우드 제공자가 제공하는 외부 Load Balancer를 구성해야 할 수 있습니다.

이러한 클라우드별 작업을 Kubernetes의 핵심 제어 로직과 분리하여 처리하는 구성 요소가 cloud-controller-manager입니다.

주요 역할은 다음과 같습니다.

- Kubernetes Node와 클라우드 인스턴스 정보를 연결합니다.
- 클라우드 환경의 네트워크 라우팅을 관리합니다.
- Kubernetes Service와 클라우드 Load Balancer를 연동합니다.
- 클라우드 제공자별 API를 호출하여 인프라 리소스를 생성하거나 변경합니다.
- Kubernetes의 클라우드 종속적인 제어 로직을 별도로 관리합니다.

cloud-controller-manager는 클라우드 인프라를 직접 실행하는 구성 요소가 아닙니다. 클라우드 제공자의 API와 연동하여 인프라 리소스를 관리하고, 그 결과를 Kubernetes API에 반영하는 Controller를 실행합니다.

## 2. Cloud Provider란?

### 2.1. Cloud Provider의 개념

Cloud Provider는 Kubernetes가 실행되는 클라우드 인프라와 상호작용하기 위한 추상화 계층입니다.

Kubernetes는 특정 클라우드 제공자에 종속되지 않도록 설계되어 있지만, 클라우드 인프라를 활용하려면 각 제공자가 제공하는 API와 기능에 접근할 수 있어야 합니다.

예를 들어 다음과 같은 작업은 클라우드 제공자별 구현이 필요합니다.

| 작업 | 클라우드 제공자별 처리 예시 |
| --- | --- |
| Node 식별 | 클라우드 인스턴스 ID 및 가용 영역 확인 |
| 네트워크 라우팅 | 클라우드 네트워크의 Route Table 관리 |
| Load Balancer | 외부 Load Balancer 생성 및 설정 |
| 네트워크 주소 | 클라우드 인스턴스의 내부 및 외부 IP 확인 |

Cloud Provider는 이러한 차이를 추상화하여 Kubernetes Controller가 클라우드 환경의 세부 구현에 직접 종속되지 않도록 지원합니다.

### 2.2. In-tree와 Out-of-tree Cloud Provider

Kubernetes의 Cloud Provider 구현 방식은 크게 In-tree와 Out-of-tree 방식으로 구분할 수 있습니다.

**In-tree Cloud Provider**

In-tree 방식은 Cloud Provider 구현 코드가 Kubernetes 소스 코드 내부에 포함되는 구조입니다.

초기 Kubernetes에서는 AWS, Azure, GCP 등 여러 클라우드 제공자의 구현이 Kubernetes 코드베이스에 포함되어 있었습니다.

그러나 클라우드 제공자별 기능을 Kubernetes 핵심 프로젝트에서 직접 유지보수하면 다음과 같은 문제가 발생할 수 있습니다.

- Kubernetes 릴리스 주기와 클라우드 제공자별 기능 업데이트 주기가 달라질 수 있습니다.
- 클라우드 제공자별 코드가 Kubernetes 핵심 코드베이스의 복잡도를 증가시킵니다.
- 특정 클라우드의 기능 변경이 Kubernetes 핵심 코드의 수정과 배포를 요구할 수 있습니다.

**Out-of-tree Cloud Provider**

Out-of-tree 방식은 Cloud Provider 구현을 Kubernetes 핵심 코드베이스 외부에서 별도로 개발하고 배포하는 구조입니다.

클라우드 제공자는 Kubernetes와 독립적으로 Cloud Controller Manager를 개발하고 유지보수할 수 있습니다.

현재 Kubernetes에서는 이러한 Out-of-tree Cloud Provider 구조를 중심으로 클라우드 연동 기능을 분리하고 있습니다.

| 구분 | In-tree | Out-of-tree |
| --- | --- | --- |
| 구현 위치 | Kubernetes 소스 코드 내부 | Kubernetes 외부 프로젝트 |
| 유지보수 | Kubernetes 프로젝트 중심 | 클라우드 제공자 또는 별도 프로젝트 |
| 업데이트 | Kubernetes 릴리스와 연계 | 독립적인 릴리스 가능 |
| 확장성 | Kubernetes 핵심 코드에 의존 | 독립적인 구현 및 확장 가능 |

새로운 클라우드 환경을 Kubernetes와 연동할 때는 일반적으로 Out-of-tree Cloud Provider 구현을 사용하는 방식을 고려합니다.

## 3. cloud-controller-manager의 구조

### 3.1. Controller Manager의 역할

cloud-controller-manager는 여러 클라우드 관련 Controller를 하나의 실행 프로세스에서 관리합니다.

대표적으로 다음 세 가지 Controller를 실행합니다.

- Node Controller
- Route Controller
- Service Controller

각 Controller는 특정 클라우드 인프라 영역을 담당하며, Kubernetes API Server와 Cloud Provider API를 통해 필요한 작업을 수행합니다.

```text
+------------------------------------------------+
|             cloud-controller-manager           |
|                                                |
|  +------------------------------------------+  |
|  |            Cloud Controllers             |  |
|  |                                          |  |
|  |  +------------------------------------+  |  |
|  |  |       Cloud Node Controller        |  |  |
|  |  +------------------------------------+  |  |
|  |  |       Route Controller             |  |  |
|  |  +------------------------------------+  |  |
|  |  |       Service Controller           |  |  |
|  |  +------------------------------------+  |  |
|  +------------------------------------------+  |
|                       |                        |
|                       v                        |
|              Cloud Provider Interface          |
+-----------------------+------------------------+
                        |
                        v
                Cloud Provider API
                        |
                        v
              Cloud Infrastructure
```

위 구조에서 각 Controller는 Kubernetes 리소스의 상태를 확인하고, 필요한 경우 클라우드 인프라를 변경합니다.

### 3.2. Kubernetes API와 Cloud Provider API

cloud-controller-manager는 Kubernetes API Server와 클라우드 제공자의 API를 모두 사용합니다.

두 API의 역할은 다음과 같이 구분됩니다.

| 구분 | 역할 |
| --- | --- |
| Kubernetes API | Node, Service 등의 Kubernetes 리소스를 조회하고 상태를 변경합니다. |
| Cloud Provider API | 클라우드 인스턴스, 네트워크 라우팅, Load Balancer 등의 인프라를 조회하고 관리합니다. |

예를 들어 사용자가 `type: LoadBalancer`인 Service를 생성하면 다음과 같은 작업이 발생할 수 있습니다.

1. cloud-controller-manager가 API Server를 통해 Service 생성 사실을 확인합니다.
2. Service Controller가 Cloud Provider API를 호출합니다.
3. 클라우드 제공자가 외부 Load Balancer를 생성합니다.
4. Service Controller가 생성된 Load Balancer의 정보를 확인합니다.
5. Kubernetes API Server를 통해 Service의 Status를 업데이트합니다.

이처럼 cloud-controller-manager는 Kubernetes 리소스와 클라우드 인프라 사이의 상태를 조정하는 역할을 담당합니다.

## 4. 주요 Controller

### 4.1. Cloud Node Controller

Cloud Node Controller는 Kubernetes Node와 클라우드 인스턴스 정보를 연결하고 관리하는 Controller입니다.

클라우드 환경에서 Kubernetes Node는 일반적으로 클라우드 제공자가 생성한 가상 머신 또는 인스턴스에 해당합니다.

Cloud Node Controller는 클라우드 제공자 API를 통해 해당 인스턴스의 정보를 확인하고 Kubernetes Node 리소스에 반영합니다.

주요 역할은 다음과 같습니다.

- Node와 클라우드 인스턴스의 연결 정보를 관리합니다.
- 클라우드 인스턴스 ID를 확인합니다.
- Node의 내부 및 외부 IP 정보를 설정합니다.
- Node의 가용 영역(Availability Zone) 및 지역(Region) 정보를 반영합니다.
- 클라우드 인스턴스와 관련된 Node 정보를 관리합니다.

예를 들어 AWS 환경에서는 EC2 인스턴스 정보가 Kubernetes Node의 메타데이터와 연결될 수 있습니다.

Node에는 다음과 같은 정보가 반영될 수 있습니다.

```yaml
apiVersion: v1
kind: Node
metadata:
  name: worker-node-1
  labels:
    topology.kubernetes.io/region: ap-northeast-2
    topology.kubernetes.io/zone: ap-northeast-2a
spec:
  providerID: aws:///ap-northeast-2a/i-0123456789abcdef0
```

위 예시에서 `providerID`는 Kubernetes Node와 클라우드 인스턴스를 연결하는 식별자입니다.

실제 값의 형식은 Cloud Provider 구현에 따라 달라집니다.

Node Controller는 이러한 정보를 활용하여 Kubernetes가 클라우드 인프라의 Node 정보를 이해하고 관리할 수 있도록 지원합니다.

### 4.2. Route Controller

Route Controller는 Kubernetes Pod 네트워크와 클라우드 네트워크 사이의 라우팅을 관리하는 Controller입니다.

일부 클라우드 환경에서는 서로 다른 Node에 배치된 Pod 간 통신이 가능하도록 클라우드 네트워크의 Route Table을 구성해야 합니다.

Route Controller는 각 Node에 할당된 Pod CIDR 정보를 확인하고, 해당 네트워크로 트래픽이 전달될 수 있도록 클라우드 네트워크의 라우팅 정보를 관리합니다.

예를 들어 다음과 같은 구조를 가정할 수 있습니다.

```text
             Cloud Network
                    |
             +------+------+
             |             |
             v             v
          Node A         Node B
          Pod CIDR       Pod CIDR
        10.244.1.0/24   10.244.2.0/24
             |             |
             v             v
          Pod A          Pod B
```

Node A의 Pod가 Node B의 Pod와 통신하려면 목적지 Pod CIDR에 도달할 수 있는 네트워크 경로가 필요합니다.

Route Controller는 Cloud Provider API를 통해 이러한 라우팅 정보를 구성할 수 있습니다.

다만 모든 Kubernetes 클러스터가 Route Controller를 사용하는 것은 아닙니다.

CNI(Container Network Interface) 플러그인이나 클라우드 네트워크 자체가 Pod 간 라우팅을 처리하는 환경에서는 별도의 Route Controller가 필요하지 않을 수 있습니다.

따라서 Route Controller의 동작 여부와 책임은 Cloud Provider 및 네트워크 구현 방식에 따라 달라집니다.

### 4.3. Service Controller

Service Controller는 Kubernetes Service와 클라우드 Load Balancer를 연동하는 Controller입니다.

특히 `type: LoadBalancer`로 선언된 Service를 클라우드 환경에서 외부에 노출할 때 중요한 역할을 담당합니다.

사용자가 다음과 같이 Service를 생성한다고 가정하겠습니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
```

Service Controller는 해당 Service를 감지하고 Cloud Provider API를 통해 외부 Load Balancer를 생성하거나 기존 Load Balancer를 조정합니다.

일반적인 처리 흐름은 다음과 같습니다.

```text
          User
            |
            | kubectl apply
            v
       kube-apiserver
            |
            v
           etcd
            |
            | Service 저장
            v
  cloud-controller-manager
            |
            v
      Service Controller
            |
            | Cloud Provider API
            v
     Cloud Load Balancer 생성
            |
            v
     Load Balancer 정보 확인
            |
            v
       kube-apiserver
            |
            | Service Status 업데이트
            v
          etcd
```

Load Balancer 생성이 완료되면 일반적으로 Service의 `status.loadBalancer`에 외부 접근을 위한 주소가 반영됩니다.

예를 들면 다음과 같습니다.

```yaml
status:
  loadBalancer:
    ingress:
      - ip: 203.0.113.10
```

실제 주소는 클라우드 제공자와 Load Balancer 구성에 따라 IP 또는 Hostname 형태로 제공될 수 있습니다.

Service Controller는 Load Balancer 생성뿐 아니라 Service 설정 변경이나 삭제에 따른 클라우드 리소스 변경 및 정리 작업도 담당할 수 있습니다.

### 4.4. Controller별 역할 비교

세 가지 주요 Controller의 역할을 정리하면 다음과 같습니다.

| Controller | 관리 대상 | 주요 작업 |
| --- | --- | --- |
| Cloud Node Controller | Kubernetes Node 및 클라우드 인스턴스 | 인스턴스 식별, IP 및 토폴로지 정보 관리 |
| Route Controller | 클라우드 네트워크 Route | Node별 Pod CIDR에 대한 라우팅 구성 |
| Service Controller | Kubernetes Service 및 Cloud Load Balancer | 외부 Load Balancer 생성, 변경 및 삭제 |

## 5. kube-controller-manager와의 차이

### 5.1. 역할 분리

kube-controller-manager와 cloud-controller-manager는 모두 Kubernetes 리소스의 상태를 조정하는 Controller를 실행합니다.

그러나 관리하는 영역이 다릅니다.

kube-controller-manager는 Kubernetes 자체의 핵심 제어 로직을 담당합니다.

반면 cloud-controller-manager는 클라우드 제공자별 인프라와 연동하는 제어 로직을 담당합니다.

이러한 분리는 Kubernetes 핵심 기능이 특정 클라우드 제공자의 구현에 종속되지 않도록 하기 위한 아키텍처입니다.

### 5.2. 주요 책임 비교

| 구분 | kube-controller-manager | cloud-controller-manager |
| --- | --- | --- |
| 주요 목적 | Kubernetes 내부 리소스 상태 관리 | 클라우드 인프라와 Kubernetes 연동 |
| 관리 영역 | Deployment, ReplicaSet, Node, Job, Namespace 등 | Cloud Node, Route, Load Balancer 등 |
| 주요 API | Kubernetes API | Kubernetes API 및 Cloud Provider API |
| 클라우드 의존성 | 핵심 Controller는 일반적으로 클라우드 비종속적입니다. | 클라우드 제공자별 구현에 의존합니다. |
| 대표 Controller | Deployment, ReplicaSet, Job, Node 등 | Cloud Node, Route, Service |

### 5.3. Node Controller와 Cloud Node Controller

두 컴포넌트의 Controller 중에는 Node와 관련된 역할이 있어 혼동하기 쉽습니다.

kube-controller-manager의 Node Controller와 cloud-controller-manager의 Cloud Node Controller는 서로 다른 책임을 담당합니다.

| 구분 | Node Controller | Cloud Node Controller |
| --- | --- | --- |
| 소속 | kube-controller-manager | cloud-controller-manager |
| 주요 책임 | Node 상태 감시 및 장애 대응 | 클라우드 인스턴스와 Node 정보 연결 |
| 주요 작업 | Node 상태 확인, 장애 관련 Taint 관리 등 | Provider ID, IP, Region, Zone 정보 관리 |
| 클라우드 API | 일반적으로 직접 사용하지 않습니다. | Cloud Provider API를 사용합니다. |

예를 들어 Node가 정상적으로 응답하지 않는 상황에서 장애 상태를 감지하고 후속 조치를 관리하는 것은 kube-controller-manager의 Node Controller 역할입니다.

반면 해당 Node가 어떤 클라우드 인스턴스에 해당하는지 확인하고 클라우드 인프라 정보를 반영하는 것은 cloud-controller-manager의 Cloud Node Controller 역할입니다.

## 6. 외부 Load Balancer 연동 과정

### 6.1. LoadBalancer 타입 Service

Kubernetes Service는 여러 Pod를 하나의 네트워크 접근 지점으로 제공하는 리소스입니다.

Service의 `type`에 따라 외부 노출 방식이 달라집니다.

| Service Type | 설명 |
| --- | --- |
| ClusterIP | 클러스터 내부에서 접근할 수 있는 IP를 제공합니다. |
| NodePort | 각 Node의 특정 포트를 통해 Service에 접근할 수 있도록 합니다. |
| LoadBalancer | 외부 Load Balancer를 통해 Service에 접근할 수 있도록 합니다. |
| ExternalName | DNS 이름을 다른 이름으로 매핑합니다. |

`LoadBalancer` 타입 Service는 일반적으로 클라우드 제공자의 Load Balancer를 생성하는 방식으로 외부 접근을 지원합니다.

다만 실제 Load Balancer의 생성과 구성은 Cloud Provider 구현 및 클러스터 환경에 따라 달라집니다.

### 6.2. Load Balancer 생성 과정

`LoadBalancer` 타입 Service를 생성하면 일반적으로 다음과 같은 과정이 진행됩니다.

1. 사용자가 `type: LoadBalancer`인 Service를 생성합니다.
2. API Server가 Service 리소스를 저장합니다.
3. cloud-controller-manager의 Service Controller가 변경 사항을 감지합니다.
4. Service Controller가 Cloud Provider API를 호출합니다.
5. 클라우드 제공자가 Load Balancer를 생성합니다.
6. Load Balancer의 Listener, Backend 등의 설정이 구성됩니다.
7. Service Controller가 Load Balancer의 외부 주소를 확인합니다.
8. Service의 `status.loadBalancer`가 업데이트됩니다.

이 과정에서 클라우드 인프라의 생성 및 설정 작업은 Cloud Provider가 담당하고, Kubernetes 리소스 상태와 클라우드 인프라 사이의 조정은 Service Controller가 담당합니다.

### 6.3. Load Balancer와 Kubernetes Service의 관계

Load Balancer는 외부 클라이언트의 트래픽을 Kubernetes Service로 전달하기 위한 진입점입니다.

일반적인 구조는 다음과 같습니다.

```text
              External Client
                     |
                     v
             Cloud Load Balancer
                     |
                     v
              Kubernetes Node
                     |
                     v
                  Service
                     |
                     v
                    Pod
```

실제 트래픽 전달 경로는 클라우드 제공자, Load Balancer 유형, CNI 및 Service 데이터 플레인 구현에 따라 달라질 수 있습니다.

예를 들어 일부 환경에서는 Load Balancer가 NodePort를 통해 Node로 트래픽을 전달하고, Node의 kube-proxy 또는 대체 데이터 플레인 구현이 이를 Pod로 전달합니다.

또한 일부 환경에서는 Pod IP를 직접 Backend로 등록하는 방식도 사용할 수 있습니다.

따라서 Load Balancer가 항상 동일한 네트워크 경로로 트래픽을 전달한다고 가정해서는 안 됩니다.

## 7. 배포 및 운영

### 7.1. Cloud Provider 환경에서의 배포

cloud-controller-manager는 일반적으로 Kubernetes Control Plane에서 실행됩니다.

배포 방식은 클라우드 제공자와 Kubernetes 설치 방식에 따라 달라질 수 있습니다.

예를 들어 kubeadm 기반 클러스터에서는 Control Plane 구성 요소를 Static Pod로 실행할 수 있으며, cloud-controller-manager 역시 Static Pod로 배포할 수 있습니다.

Managed Kubernetes 환경에서는 클라우드 제공자가 Control Plane과 Cloud Controller Manager의 운영을 관리하므로 사용자가 직접 배포하거나 관리할 필요가 없는 경우도 있습니다.

### 7.2. External Cloud Controller Manager

Out-of-tree Cloud Provider는 별도의 cloud-controller-manager 바이너리로 제공될 수 있습니다.

이러한 구현은 Kubernetes 핵심 프로젝트와 독립적으로 개발 및 배포됩니다.

일반적인 배포 구조는 다음과 같습니다.

```text
+--------------------------------------+
|          Kubernetes Control Plane   |
|                                      |
|  +--------------------------------+  |
|  |         kube-apiserver         |  |
|  +--------------------------------+  |
|  |     kube-controller-manager    |  |
|  +--------------------------------+  |
|  |     kube-scheduler             |  |
|  +--------------------------------+  |
|  |     cloud-controller-manager   |  |
|  +--------------------------------+  |
|                                      |
+------------------+-------------------+
                   |
                   v
          Cloud Provider API
                   |
                   v
          Cloud Infrastructure
```

실제 배포 구조에서는 Cloud Controller Manager가 별도의 Pod로 실행되거나 클라우드 제공자가 관리하는 형태로 구성될 수 있습니다.

### 7.3. --cloud-provider 옵션

Kubernetes의 일부 컴포넌트는 Cloud Provider 사용 여부를 설정하는 `--cloud-provider` 옵션을 제공합니다.

Out-of-tree Cloud Provider를 사용하는 환경에서는 일반적으로 다음과 같은 방식으로 설정합니다.

```bash
--cloud-provider=external
```

이 설정은 해당 컴포넌트가 클라우드 제공자별 기능을 자체적으로 구현하지 않고 외부 Cloud Controller Manager와 연동하는 구조임을 나타냅니다.

일반적으로 외부 Cloud Controller Manager를 사용하는 환경에서는 kubelet 및 관련 컴포넌트의 설정도 일관되게 구성해야 합니다.

특히 kubelet이 `--cloud-provider=external`로 설정된 경우, Cloud Node Controller가 클라우드 정보를 초기화하기 전까지 Node에 `node.cloudprovider.kubernetes.io/uninitialized` Taint가 적용될 수 있습니다.

Cloud Node Controller가 초기화 작업을 완료하면 해당 Taint를 제거하여 Node가 정상적으로 스케줄링될 수 있도록 합니다.

### 7.4. 주요 실행 옵션

cloud-controller-manager의 실행 옵션은 Kubernetes 버전과 Cloud Provider 구현에 따라 달라질 수 있습니다.

대표적인 옵션은 다음과 같습니다.

| 옵션 | 설명 |
| --- | --- |
| `--cloud-provider` | 사용할 Cloud Provider 구현을 지정합니다. |
| `--cloud-config` | Cloud Provider 설정 파일을 지정합니다. |
| `--kubeconfig` | API Server에 접근할 때 사용할 kubeconfig 파일을 지정합니다. |
| `--master` | API Server 주소를 지정합니다. |
| `--leader-elect` | Leader Election 사용 여부를 설정합니다. |
| `--controllers` | 실행할 Controller 목록을 지정합니다. |
| `--use-service-account-credentials` | 개별 Controller가 별도의 ServiceAccount 자격 증명을 사용하도록 설정합니다. |

모든 Cloud Provider가 동일한 옵션을 지원하는 것은 아닙니다. 특히 Cloud Provider별 설정은 별도의 설정 파일, 환경 변수 또는 클라우드 인증 체계를 통해 관리할 수 있습니다.

운영 환경에서는 사용 중인 Kubernetes 버전과 Cloud Provider 구현의 공식 문서를 확인해야 합니다.

### 7.5. Leader Election

cloud-controller-manager는 여러 인스턴스를 실행하여 고가용성을 구성할 수 있습니다.

일반적으로 여러 cloud-controller-manager 인스턴스를 실행하고 Leader Election을 통해 하나의 인스턴스만 활성 Leader로 동작하도록 구성합니다.

Leader Election은 Kubernetes API의 Lease 리소스를 사용하여 수행할 수 있습니다.

```text
       cloud-controller-manager
                   |
          +--------+--------+
          |        |        |
          v        v        v
        CCM 1    CCM 2    CCM 3
          |        |        |
          +--------+--------+
                   |
             Leader Election
                   |
                   v
             Active Leader
                   |
                   v
         Cloud Controller 실행
```

Leader가 장애를 일으키거나 Lease를 갱신하지 못하면 다른 인스턴스가 Leader로 선출되어 Controller 작업을 이어받을 수 있습니다.

이러한 구조는 여러 인스턴스가 동시에 동일한 클라우드 인프라 변경 작업을 수행하는 상황을 방지하고 가용성을 높이는 데 도움이 됩니다.

## 8. 다른 Control Plane 컴포넌트와의 관계

cloud-controller-manager는 Kubernetes API Server를 중심으로 다른 Control Plane 컴포넌트와 협력합니다.

### 8.1. kube-apiserver

kube-apiserver는 cloud-controller-manager가 Kubernetes 리소스를 조회하고 변경하기 위해 사용하는 API 인터페이스입니다.

cloud-controller-manager는 API Server를 통해 다음과 같은 작업을 수행합니다.

- Node 리소스 조회 및 수정
- Service 리소스 조회 및 수정
- 리소스 변경 사항 감시
- 클라우드 인프라 상태를 Kubernetes 리소스에 반영

cloud-controller-manager는 etcd에 직접 접근하지 않으며, Kubernetes API를 통해 리소스 상태를 관리합니다.

### 8.2. kube-controller-manager

kube-controller-manager는 Kubernetes 자체의 핵심 리소스 상태를 관리하고, cloud-controller-manager는 클라우드 인프라와 관련된 상태를 관리합니다.

두 컴포넌트는 API Server를 통해 서로 관련된 리소스 상태를 확인하고 조정합니다.

예를 들어 Node Controller가 Node 장애를 감지하는 작업은 kube-controller-manager가 담당하고, Cloud Node Controller가 해당 Node의 클라우드 인스턴스 정보를 관리하는 작업은 cloud-controller-manager가 담당합니다.

### 8.3. kube-scheduler

kube-scheduler는 Pod를 실행할 적절한 Node를 선택합니다.

cloud-controller-manager의 Cloud Node Controller가 Node의 Region, Zone, Instance Type 등 클라우드 인프라 정보를 Kubernetes Node에 반영하면 kube-scheduler는 이러한 정보를 활용하여 Pod를 배치할 수 있습니다.

예를 들어 Pod의 Node Affinity나 Topology Spread Constraints가 특정 Zone을 요구하는 경우 Node에 설정된 토폴로지 정보가 스케줄링 판단에 활용될 수 있습니다.

### 8.4. kubelet

kubelet은 각 Node에서 실행되며 해당 Node에 할당된 Pod의 실행을 관리합니다.

cloud-controller-manager는 kubelet과 직접 컨테이너 실행을 제어하는 관계가 아니라, 클라우드 인스턴스와 Node의 정보를 관리하는 역할을 담당합니다.

Out-of-tree Cloud Provider 환경에서는 kubelet이 외부 Cloud Controller Manager의 Node 초기화 작업을 기다리는 구조로 동작할 수 있습니다.

### 8.5. 전체 컴포넌트 상호작용

클라우드 환경에서 Kubernetes Node가 등록되고 Load Balancer가 생성되는 과정을 기준으로 각 컴포넌트의 역할을 정리하면 다음과 같습니다.

```text
                  Cloud Provider
                        |
                        | Instance 생성
                        v
                   Cloud Instance
                        |
                        v
                      kubelet
                        |
                        | Node 등록
                        v
                  kube-apiserver
                        |
                        v
                       etcd
                        |
                        | Node 리소스 저장
                        v
             cloud-controller-manager
                        |
             +----------+----------+
             |                     |
             v                     v
       Cloud Node Controller   Service Controller
             |                     |
             | Cloud API           | Cloud API
             v                     v
       Instance 정보 확인      Load Balancer 생성
             |                     |
             v                     v
       kube-apiserver         Cloud Load Balancer
             |                     |
             v                     |
           etcd                    |
                                   |
                         Service Status 업데이트
                                   |
                                   v
                             kube-apiserver
                                   |
                                   v
                                  etcd
```

위 구조는 주요 상호작용을 개념적으로 표현한 것입니다. 실제 Node 초기화와 Service 생성의 실행 순서는 클라우드 제공자 및 클러스터 구성에 따라 달라질 수 있습니다.

## 9. 정리

cloud-controller-manager는 Kubernetes Control Plane에서 클라우드 제공자별 인프라와 Kubernetes 리소스를 연결하고 관리하는 구성 요소입니다.

Kubernetes 핵심 제어 로직과 클라우드 제공자별 구현을 분리하여 Kubernetes가 다양한 클라우드 환경에서 동작할 수 있도록 지원합니다.

cloud-controller-manager를 이해할 때는 다음 개념을 특히 구분해서 이해해야 합니다.

- **In-tree와 Out-of-tree Cloud Provider:** In-tree는 Cloud Provider 구현이 Kubernetes 소스 코드 내부에 포함된 구조이며, Out-of-tree는 Kubernetes 외부에서 별도로 개발하고 배포하는 구조입니다. 현재는 클라우드 제공자별 구현을 분리하는 Out-of-tree 방식을 중심으로 사용합니다.
- **Cloud Node Controller:** Kubernetes Node와 클라우드 인스턴스를 연결하고, Provider ID, IP, Region, Zone 등의 정보를 관리합니다.
- **Route Controller:** 클라우드 네트워크의 라우팅 정보를 관리하여 Node 간 Pod 네트워크 통신을 지원합니다. 단, 실제 사용 여부는 클라우드 네트워크 및 CNI 구현에 따라 달라집니다.
- **Service Controller:** Kubernetes Service와 클라우드 Load Balancer를 연동합니다. 특히 `type: LoadBalancer` Service의 생성, 변경 및 삭제에 따른 클라우드 리소스 관리를 담당합니다.
- **kube-controller-manager와의 책임 분리:** kube-controller-manager는 Kubernetes 자체의 핵심 리소스 제어를 담당하고, cloud-controller-manager는 클라우드 인프라와 관련된 제어 로직을 담당합니다.

cloud-controller-manager는 Kubernetes의 클라우드 종속성을 분리하고, Kubernetes 리소스와 클라우드 인프라 사이의 상태를 일관되게 유지하기 위한 핵심 구성 요소입니다.

**Kubernetes는 Cloud Controller Manager를 통해 클라우드 제공자별 인프라 제어 로직을 핵심 Control Plane으로부터 분리하여 다양한 클라우드 환경을 지원합니다.**

## 10. 참고자료

### Kubernetes 공식 문서

- [Cloud Controller Manager](https://kubernetes.io/docs/concepts/architecture/cloud-controller/)
- [Cloud Controller Manager Administration](https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/)
- [Kubernetes Cloud Controller Manager](https://kubernetes.io/blog/2016/10/outsourcing-cloud-provider-integration/)
- [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)
- [Kubernetes Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)

### Kubernetes 개발 및 구현 관련 자료

- [Kubernetes Cloud Provider Repository](https://github.com/kubernetes/cloud-provider)
- [Kubernetes Cloud Provider Interface](https://pkg.go.dev/k8s.io/cloud-provider)
- [Kubernetes Cloud Controller Manager Command Line Reference](https://kubernetes.io/docs/reference/command-line-tools-reference/cloud-controller-manager/)
- [Kubernetes Service Controller Interface](https://pkg.go.dev/k8s.io/cloud-provider/controllers/service)
- [Kubernetes Node Controller Interface](https://pkg.go.dev/k8s.io/cloud-provider/controllers/node)