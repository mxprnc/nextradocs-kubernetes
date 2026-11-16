# Kubernetes Architecture

## 1. Kubernetes Architecture란?

Kubernetes Architecture는 Kubernetes 클러스터를 구성하는 주요 컴포넌트와 각 컴포넌트의 역할 및 상호작용 방식을 설명하는 아키텍처 구조입니다.

Kubernetes는 컨테이너화된 애플리케이션을 배포하고 관리하기 위한 컨테이너 오케스트레이션 플랫폼입니다. 클러스터의 전체 상태를 관리하는 Control Plane과 실제 애플리케이션 워크로드를 실행하는 Node를 중심으로 구성됩니다.

각 컴포넌트는 독립적인 역할을 수행하면서 Kubernetes API를 통해 서로 상호작용합니다. 이러한 구조를 통해 클러스터의 원하는 상태(Desired State)를 선언적으로 정의하고, 현재 상태(Actual State)가 원하는 상태에 도달하도록 지속적으로 조정할 수 있습니다.

Kubernetes 클러스터는 크게 다음 세 가지 영역으로 구분할 수 있습니다.

- **Control Plane:** 클러스터의 전체 상태를 관리하고 제어하는 영역입니다.
- **Node:** 컨테이너화된 애플리케이션의 워크로드를 실행하는 영역입니다.
- **Addons:** 클러스터의 기본 기능을 확장하고 보완하는 구성 요소입니다.

## 2. Kubernetes Cluster Architecture

### 2.1. 클러스터 구성

Kubernetes 클러스터는 하나 이상의 Control Plane 구성 요소와 하나 이상의 Node로 구성됩니다.

Control Plane은 클러스터 전체의 상태를 관리하고, Node는 실제 애플리케이션이 실행되는 환경을 제공합니다.

일반적인 Kubernetes 클러스터의 구성은 다음과 같습니다.

```text
+----------------------------------------------------------+
|                    Kubernetes Cluster                    |
|                                                          |
|  +----------------------------------------------------+  |
|  |                   Control Plane                    |  |
|  |                                                    |  |
|  |  +---------------+     +-----------------------+   |  |
|  |  | kube-apiserver|<--->|         etcd          |   |  |
|  |  +---------------+     +-----------------------+   |  |
|  |          ^                                         |  |
|  |          |                                         |  |
|  |  +-------+----------+  +----------------------+    |  |
|  |  | kube-scheduler   |  | kube-controller-     |    |  |
|  |  |                  |  | manager              |    |  |
|  |  +------------------+  +----------------------+    |  |
|  |                                                    |  |
|  +-------------------------+--------------------------+  |
|                            |                             |
|                  Kubernetes API                          |
|                            |                             |
|           +----------------+----------------+            |
|           |                                 |            |
|  +--------v----------------+  +-------------v----------+ |
|  |       Worker Node 1     |  |      Worker Node 2     | |
|  |                         |  |                        | |
|  |  +-------------------+  |  |  +-------------------+ | |
|  |  |     kubelet       |  |  |  |     kubelet       | | |
|  |  +-------------------+  |  |  +-------------------+ | |
|  |  |    kube-proxy     |  |  |  |    kube-proxy     | | |
|  |  +-------------------+  |  |  +-------------------+ | |
|  |  | Container Runtime |  |  |  | Container Runtime | | |
|  |  +-------------------+  |  |  +-------------------+ | |
|  |                         |  |                        | |
|  |  +-------------------+  |  |  +-------------------+ | |
|  |  |       Pod         |  |  |  |       Pod         | | |
|  |  |                   |  |  |  |                   | | |
|  |  |   Container       |  |  |  |   Container       | | |
|  |  +-------------------+  |  |  +-------------------+ | |
|  +-------------------------+  +------------------------+ |
|                                                          |
|  +----------------------------------------------------+  |
|  |                      Addons                        |  |
|  |                                                    |  |
|  |       CoreDNS / Metrics Server / Dashboard         |  |
|  +----------------------------------------------------+  |
|                                                          |
+----------------------------------------------------------+
```

위 구조는 Kubernetes의 일반적인 아키텍처를 이해하기 위한 개념도입니다. 실제 클러스터에서는 구성 요소의 배치 방식이나 실행 위치가 달라질 수 있습니다.

예를 들어 Control Plane 구성 요소를 별도의 서버에 배치하거나, 여러 Control Plane 인스턴스를 구성하여 고가용성(High Availability)을 확보할 수 있습니다.

### 2.2. Control Plane

Control Plane은 Kubernetes 클러스터의 전체 상태를 관리하고 제어하는 핵심 영역입니다.

클러스터에 생성된 리소스를 관리하고, 새로운 Pod의 배치 위치를 결정하며, 선언된 상태와 실제 상태 사이의 차이를 지속적으로 조정합니다.

주요 구성 요소는 다음과 같습니다.

| 컴포넌트 | 역할 |
|---|---|
| kube-apiserver | Kubernetes API를 제공하고 클러스터의 모든 API 요청을 처리합니다. |
| etcd | Kubernetes 클러스터의 상태 및 리소스 데이터를 저장하는 분산 Key-Value 저장소입니다. |
| kube-scheduler | 새롭게 생성된 Pod가 실행될 적절한 Node를 선택합니다. |
| kube-controller-manager | 다양한 Controller를 실행하여 클러스터의 현재 상태를 원하는 상태로 조정합니다. |
| cloud-controller-manager | 클라우드 제공자의 API와 연동하여 클라우드 인프라 관련 리소스를 관리합니다. |

Control Plane의 구성 요소는 서로 협력하여 클러스터의 상태를 관리합니다.

특히 kube-apiserver는 Kubernetes API를 제공하는 중심 인터페이스로, 다른 컴포넌트들이 클러스터의 상태를 조회하거나 변경할 때 사용하는 주요 통신 경로입니다.

각 구성 요소의 상세한 역할은 다음 문서에서 다룹니다.

- [Control Plane Components](./control-plane/)

### 2.3. Node

Node는 Kubernetes에서 Pod를 실행하는 작업 환경입니다.

Node는 물리 머신 또는 가상 머신으로 구성할 수 있으며, Control Plane의 관리하에 애플리케이션 컨테이너를 실행하고 유지합니다.

각 Node에는 Pod 실행과 관리를 위한 구성 요소가 배치됩니다.

주요 구성 요소는 다음과 같습니다.

| 컴포넌트 | 역할 |
|---|---|
| kubelet | Node에서 실행되는 에이전트로, 할당된 Pod의 컨테이너가 정상적으로 실행되도록 관리합니다. |
| kube-proxy | Kubernetes Service의 네트워크 동작을 구현하기 위해 Node의 네트워크 규칙을 관리합니다. |
| Container Runtime | 컨테이너의 실행 및 생명주기를 관리하는 런타임입니다. |

kubelet은 Control Plane에서 전달받은 PodSpec을 기반으로 컨테이너의 실행 상태를 관리합니다.

Container Runtime은 실제 컨테이너를 실행하는 역할을 담당하며, kubelet은 CRI(Container Runtime Interface)를 통해 Container Runtime과 상호작용합니다.

kube-proxy는 Service와 관련된 네트워크 규칙을 관리합니다. 다만 일부 네트워크 플러그인은 자체적인 Service 프록시 기능을 제공하므로 kube-proxy가 반드시 필요한 것은 아닙니다.

각 구성 요소의 상세한 역할은 다음 문서에서 다룹니다.

- [Node Components](./node/)

### 2.4. Addons

Addons는 Kubernetes 클러스터에 추가적인 기능을 제공하는 구성 요소입니다.

Control Plane과 Node의 기본 기능만으로도 Kubernetes 클러스터를 운영할 수 있지만, 실제 운영 환경에서는 DNS, 모니터링, 로깅 및 관리 인터페이스 등의 추가 기능이 필요합니다.

Addons는 일반적으로 Kubernetes의 Deployment, DaemonSet 등의 리소스를 사용하여 배포 및 관리됩니다.

대표적인 Addons는 다음과 같습니다.

| Addon | 역할 |
|---|---|
| CoreDNS | 클러스터 내부의 Service 및 Pod에 대한 DNS 조회 기능을 제공합니다. |
| Metrics Server | 클러스터 내 Node와 Pod의 리소스 사용량 지표를 수집하고 제공합니다. |
| Kubernetes Dashboard | Kubernetes 클러스터를 관리할 수 있는 웹 기반 사용자 인터페이스를 제공합니다. |
| Cluster-level Logging | 컨테이너 및 클러스터의 로그를 중앙에서 수집하고 관리합니다. |
| Container Network Plugin | Pod 간 네트워크 연결과 IP 주소 할당 등의 네트워크 기능을 제공합니다. |

Addons는 Kubernetes의 핵심 컴포넌트와 달리 클러스터의 요구사항에 따라 선택적으로 설치하고 운영할 수 있습니다.

단, 클러스터 DNS와 같은 기능은 일반적인 애플리케이션 운영에 중요하게 사용되므로 대부분의 클러스터에서 함께 구성합니다.

각 Addon의 상세한 역할은 다음 문서에서 다룹니다.

- [Addons](./addons/)

## 3. Kubernetes Architecture의 동작 원리

Kubernetes Architecture의 핵심은 선언적 상태 관리(Declarative State Management)와 지속적인 상태 조정(Reconciliation)입니다.

사용자가 원하는 클러스터의 상태를 Kubernetes API를 통해 선언하면, Control Plane의 각 컴포넌트가 이를 기반으로 실제 클러스터의 상태를 조정합니다.

### 3.1. Desired State와 Actual State

Kubernetes는 사용자가 정의한 Desired State와 현재 클러스터에서 관찰되는 Actual State를 비교하여 차이를 조정합니다.

예를 들어 Deployment에 다음과 같이 3개의 Pod를 유지하도록 선언할 수 있습니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
```

위 설정은 nginx Pod를 3개 유지하겠다는 Desired State를 정의합니다.

Kubernetes는 다음과 같은 방식으로 상태를 관리합니다.

1. 사용자가 Deployment 리소스를 생성합니다.
2. kube-apiserver가 요청을 처리하고 API 리소스의 상태를 저장합니다.
3. kube-controller-manager의 Deployment Controller가 Desired State를 확인합니다.
4. 필요한 경우 ReplicaSet을 생성하거나 변경합니다.
5. ReplicaSet Controller가 원하는 개수만큼 Pod를 생성합니다.
6. kube-scheduler가 아직 Node에 할당되지 않은 Pod의 실행 위치를 결정합니다.
7. 해당 Node의 kubelet이 PodSpec을 확인하고 Container Runtime을 통해 컨테이너를 실행합니다.
8. Kubernetes는 이후에도 Desired State와 Actual State를 지속적으로 비교하고 필요한 조정을 수행합니다.

이러한 과정을 통해 Pod가 비정상적으로 종료되거나 삭제되더라도 Kubernetes는 선언된 상태를 유지하기 위해 새로운 Pod를 생성합니다.

### 3.2. Reconciliation Loop

Reconciliation Loop는 Kubernetes Controller가 클러스터의 현재 상태를 지속적으로 관찰하고 원하는 상태와의 차이를 조정하는 동작 방식입니다.

일반적인 Controller의 동작 과정은 다음과 같습니다.

```text
+-------------------------+
|       Watch Events      |
|                         |
|  Kubernetes API 변경 감지 |
+------------+------------+
             |
             v
+-------------------------+
|      Observe State      |
|                         |
|   현재 상태 확인         |
+------------+------------+
             |
             v
+-------------------------+
|      Compare State      |
|                         |
| Desired State와         |
| Actual State 비교       |
+------------+------------+
             |
             v
+-------------------------+
|       Reconcile         |
|                         |
| 필요한 리소스 생성/수정  |
+------------+------------+
             |
             |
             +--------------------+
                                  |
                                  v
                         Watch Events
```

Controller는 일반적으로 Kubernetes API를 통해 리소스의 상태를 관찰하고, 필요한 경우 API 요청을 통해 상태를 변경합니다.

이러한 제어 루프(Control Loop)는 특정 작업이 한 번 실행되고 종료되는 방식이 아니라, 지속적으로 상태를 확인하고 조정하는 방식으로 동작합니다.

따라서 Kubernetes는 특정 작업의 실행 순서에만 의존하지 않고, 여러 Controller가 독립적으로 동작하면서 전체 클러스터의 상태를 원하는 방향으로 수렴시킵니다.

## 4. Kubernetes Components 간의 통신

Kubernetes의 각 컴포넌트는 Kubernetes API를 중심으로 상호작용합니다.

kube-apiserver는 클러스터의 API 접근을 위한 중심 인터페이스 역할을 수행하며, 다른 Control Plane 구성 요소와 Node의 컴포넌트는 API를 통해 필요한 정보를 조회하거나 변경합니다.

일반적인 통신 관계는 다음과 같습니다.

| 통신 주체 | 통신 대상 | 목적 |
|---|---|---|
| kubectl | kube-apiserver | 사용자의 API 요청 전달 |
| kube-scheduler | kube-apiserver | 미할당 Pod 조회 및 Node 할당 정보 반영 |
| kube-controller-manager | kube-apiserver | 리소스 상태 감시 및 원하는 상태로 조정 |
| kubelet | kube-apiserver | Node 및 Pod 상태 관리 |
| kubelet | Container Runtime | 컨테이너 실행 및 생명주기 관리 |
| kube-apiserver | etcd | 클러스터 상태 데이터 저장 및 조회 |
| kube-proxy | kube-apiserver | Service 및 EndpointSlice 정보 조회 |
| kube-apiserver | kubelet | Pod 로그 조회, 원격 명령 실행 및 포트 포워딩 등의 기능 제공 |

일반적으로 Node에서 발생하는 API 통신은 kube-apiserver를 중심으로 이루어집니다.

Kubernetes는 이를 Hub-and-Spoke 형태의 API 통신 구조로 설명합니다. Node의 컴포넌트가 다른 Control Plane 컴포넌트에 직접 연결되는 것이 아니라 kube-apiserver를 통해 클러스터 상태를 확인하고 관리하는 방식입니다.

단, 실제 데이터 플레인(Data Plane)의 Pod 간 네트워크 트래픽은 kube-apiserver를 경유하지 않습니다. 이러한 트래픽은 CNI 네트워크 플러그인과 Node의 네트워크 구성을 통해 전달됩니다.

## 5. Kubernetes Architecture의 주요 특징

### 5.1. Declarative Configuration

Kubernetes는 명령형(Imperative) 방식보다 선언형(Declarative) 방식의 리소스 관리를 중심으로 설계되었습니다.

사용자는 원하는 상태를 YAML 등의 매니페스트로 정의하고, Kubernetes는 해당 상태를 실제 클러스터에 반영합니다.

예를 들어 Deployment의 replicas 값을 3으로 설정하면 Kubernetes는 해당 개수의 Pod가 유지되도록 관리합니다.

### 5.2. Distributed Components

Kubernetes는 여러 컴포넌트가 분산된 환경에서 독립적으로 동작하는 구조를 갖습니다.

각 컴포넌트는 담당하는 역할이 분리되어 있으며, Kubernetes API를 중심으로 필요한 정보를 교환합니다.

이를 통해 특정 컴포넌트의 책임을 명확하게 구분하고 개별 구성 요소의 확장 및 운영을 유연하게 수행할 수 있습니다.

### 5.3. Reconciliation-based Control

Kubernetes는 지속적인 상태 관찰과 조정을 통해 클러스터의 원하는 상태를 유지합니다.

Controller는 이벤트를 감지하거나 주기적으로 상태를 확인하고, Desired State와 Actual State 사이의 차이를 줄이기 위한 작업을 수행합니다.

이러한 방식은 Pod 장애, Node 장애 및 리소스 변경과 같은 다양한 상황에 대응할 수 있도록 합니다.

### 5.4. High Availability and Scalability

Kubernetes는 Control Plane의 고가용성과 수평 확장을 지원하는 구조를 제공합니다.

예를 들어 kube-apiserver는 여러 인스턴스로 실행하여 요청을 분산할 수 있으며, etcd는 여러 멤버로 구성하여 데이터의 고가용성을 확보할 수 있습니다.

또한 여러 Node에 워크로드를 분산하여 클러스터의 실행 용량을 확장할 수 있습니다.

다만 고가용성을 확보하려면 컴포넌트별 복제 구성뿐 아니라 네트워크, 데이터 저장소 및 장애 복구 전략도 함께 고려해야 합니다.

### 5.5. Extensibility

Kubernetes는 다양한 확장 지점을 제공하여 클러스터의 기능을 확장할 수 있도록 설계되었습니다.

대표적인 확장 방식은 다음과 같습니다.

- **CustomResourceDefinition (CRD):** Kubernetes API에 사용자 정의 리소스를 추가합니다.
- **Custom Controller:** 사용자 정의 리소스 또는 기존 리소스의 상태를 관리하는 Controller를 구현합니다.
- **Scheduler Extension:** 기본 Scheduler의 동작을 확장하거나 별도의 Scheduler를 구성합니다.
- **CNI Plugin:** Pod 네트워크 연결 및 IP 주소 할당 기능을 구현합니다.
- **CSI Driver:** 다양한 스토리지 시스템과 Kubernetes를 연동합니다.
- **Admission Webhook:** API 리소스의 생성 및 변경 요청에 대한 검증이나 변경 작업을 수행합니다.

이러한 확장 구조를 통해 Kubernetes의 기본 소스 코드를 직접 수정하지 않고도 다양한 플랫폼 기능을 구현할 수 있습니다.

## 6. Kubernetes Architecture 구성 요소 요약

Kubernetes Architecture의 주요 구성 요소를 정리하면 다음과 같습니다.

| 구분 | 구성 요소 | 주요 책임 |
|---|---|---|
| Control Plane | kube-apiserver | Kubernetes API 제공 |
| Control Plane | etcd | 클러스터 상태 데이터 저장 |
| Control Plane | kube-scheduler | Pod 스케줄링 |
| Control Plane | kube-controller-manager | Controller 실행 및 상태 조정 |
| Control Plane | cloud-controller-manager | 클라우드 인프라 연동 |
| Node | kubelet | Pod 및 컨테이너 상태 관리 |
| Node | kube-proxy | Service 네트워크 규칙 관리 |
| Node | Container Runtime | 컨테이너 실행 및 관리 |
| Addons | CoreDNS | 클러스터 DNS 제공 |
| Addons | Metrics Server | 리소스 사용량 지표 제공 |
| Addons | Dashboard | 웹 기반 클러스터 관리 인터페이스 |
| Addons | Logging | 중앙 집중식 로그 수집 및 관리 |
| Addons | Network Plugin | Pod 네트워크 구현 |

위 구성 요소들은 Kubernetes 클러스터의 핵심 기능을 담당하거나 이를 확장하는 역할을 수행합니다. 실제 클러스터에서는 배포 환경과 운영 요구사항에 따라 일부 구성 요소가 생략되거나 다른 구현체로 대체될 수 있습니다.

## 7. 참고자료

### Kubernetes Official Documentation

- [Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
  - Kubernetes 클러스터의 전체 아키텍처와 Control Plane, Node, Addons 구성 요소를 설명합니다.
- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
  - Kubernetes 클러스터를 구성하는 주요 컴포넌트의 역할을 개괄적으로 설명합니다.
- [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)
  - Node의 구조와 관리 방식 및 Node 구성 요소를 설명합니다.
- [Communication between Nodes and the Control Plane](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/)
  - Control Plane과 Node 사이의 통신 구조 및 보안 고려사항을 설명합니다.
- [Kubernetes Overview](https://kubernetes.io/docs/concepts/overview/)
  - Kubernetes의 기본 개념과 주요 기능 및 설계 방향을 설명합니다.

### Related Documentation

- [Control Plane Components](./control-plane/)
- [Node Components](./node/)
- [Addons](./addons/)