# Kubernetes Node Overview

## 1. Kubernetes Node란?

Node는 Kubernetes 클러스터에서 Pod와 컨테이너가 실제로 실행되는 작업 단위입니다.

Kubernetes 클러스터는 Control Plane과 하나 이상의 Node로 구성됩니다. Control Plane은 클러스터의 전체 상태와 워크로드를 관리하며, Node는 Control Plane의 지시에 따라 실제 애플리케이션 워크로드를 실행합니다.

Node는 물리 서버 또는 가상 머신으로 구성할 수 있으며, 각 Node에는 Pod를 실행하고 관리하기 위한 구성 요소가 설치되어야 합니다.

Kubernetes에서 Node는 단순한 서버가 아니라 API Server에 등록되어 클러스터의 일부로 관리되는 리소스입니다. Node의 상태, 리소스 용량 및 할당 가능한 리소스 등의 정보는 Kubernetes API를 통해 관리됩니다.

Node의 주요 역할은 다음과 같습니다.

- Pod와 컨테이너를 실행합니다.
- 컨테이너 이미지 및 실행 환경을 관리합니다.
- Pod의 네트워크 연결을 구성합니다.
- Service 트래픽을 전달하기 위한 네트워크 규칙을 관리합니다.
- Node의 리소스 사용량과 상태를 관리합니다.
- Control Plane과 통신하여 Pod 실행 상태 및 Node 상태를 보고합니다.

Kubernetes에서는 일반적으로 워크로드를 실행하는 Node를 Worker Node라고 부르지만, Node 자체는 Control Plane 구성 요소를 실행하는 데에도 사용할 수 있습니다.

## 2. Node의 주요 구성 요소

Node에는 Pod를 실행하고 관리하기 위한 여러 구성 요소가 존재합니다.

대표적인 구성 요소는 다음과 같습니다.

| 구성 요소 | 주요 역할 |
|---|---|
| kubelet | Pod의 생명주기를 관리하고 Container Runtime과 통신합니다. |
| Container Runtime | 컨테이너 이미지, Pod Sandbox 및 컨테이너의 실행을 관리합니다. |
| kube-proxy | Service의 네트워크 규칙을 관리하고 트래픽 전달을 처리합니다. |
| CNI Plugin | Pod의 네트워크 인터페이스와 연결 환경을 구성합니다. |

### 2.1. kubelet

kubelet은 각 Node에서 실행되는 에이전트로, 해당 Node에 할당된 Pod가 정상적으로 실행되도록 관리합니다.

kubelet은 API Server를 통해 자신에게 할당된 Pod 정보를 확인하고, PodSpec에 정의된 상태와 실제 실행 상태를 지속적으로 비교합니다.

주요 역할은 다음과 같습니다.

- Pod의 생성 및 종료를 관리합니다.
- Container Runtime을 통해 컨테이너의 실행을 요청합니다.
- Container Runtime으로부터 컨테이너 상태를 확인합니다.
- Liveness, Readiness, Startup Probe를 수행합니다.
- Node의 상태와 리소스 정보를 API Server에 보고합니다.
- Node의 리소스 부족 상황에서 Pod Eviction을 수행합니다.

kubelet은 Kubernetes API Server와 통신하며, Container Runtime과는 CRI(Container Runtime Interface)를 통해 통신합니다.

자세한 내용은 [kubelet.md](./kubelet.md)를 참고하시기 바랍니다.

### 2.2. Container Runtime

Container Runtime은 Node에서 컨테이너의 생성, 실행, 종료 및 삭제 등 컨테이너의 생명주기를 관리하는 소프트웨어입니다.

kubelet은 CRI를 통해 Container Runtime에 컨테이너 실행과 관련된 작업을 요청합니다.

대표적인 Container Runtime은 다음과 같습니다.

- containerd
- CRI-O

Container Runtime은 주요하게 다음과 같은 작업을 수행합니다.

- 컨테이너 이미지 다운로드 및 관리
- Pod Sandbox 생성 및 관리
- 컨테이너 생성 및 실행
- 컨테이너 상태 조회 및 종료
- OCI Runtime을 통한 실제 컨테이너 실행

Container Runtime은 CRI를 구현하며, 실제 컨테이너 실행에는 runc와 같은 OCI Runtime을 활용할 수 있습니다.

자세한 내용은 [container-runtime.md](./container-runtime.md)를 참고하시기 바랍니다.

### 2.3. kube-proxy

kube-proxy는 Node에서 Kubernetes Service의 네트워크 규칙을 관리하는 구성 요소입니다.

Kubernetes Service는 일반적으로 고정된 가상 IP(ClusterIP)를 통해 Pod에 접근할 수 있도록 제공합니다. kube-proxy는 Service와 EndpointSlice 정보를 기반으로 네트워크 규칙을 구성하여 Service 트래픽이 적절한 Pod로 전달되도록 합니다.

주요 역할은 다음과 같습니다.

- Service의 가상 IP에 대한 트래픽 전달 규칙을 관리합니다.
- EndpointSlice 정보를 기반으로 트래픽 대상 Pod를 반영합니다.
- ClusterIP, NodePort, LoadBalancer 등의 Service 트래픽 전달을 지원합니다.
- iptables, IPVS, nftables 등의 Proxy Mode를 활용합니다.

kube-proxy는 일반적으로 각 Node에서 DaemonSet 형태로 실행됩니다.

다만 Cilium과 같이 eBPF 기반으로 Service 네트워크 기능을 구현하는 일부 네트워크 플러그인은 kube-proxy의 기능을 대체할 수 있습니다.

자세한 내용은 [kube-proxy.md](./kube-proxy.md)를 참고하시기 바랍니다.

### 2.4. CNI Plugin

CNI(Container Network Interface)는 컨테이너 및 Pod의 네트워크 연결을 구성하기 위한 인터페이스 규격입니다.

CNI Plugin은 Container Runtime과 연동하여 Pod Sandbox의 네트워크 인터페이스, IP 주소 및 라우팅 등의 네트워크 환경을 구성합니다.

대표적인 CNI 기반 네트워크 플러그인은 다음과 같습니다.

- Calico
- Cilium
- Flannel
- Weave Net

CNI Plugin의 주요 역할은 다음과 같습니다.

- Pod Network Namespace에 네트워크 인터페이스를 구성합니다.
- Pod에 IP 주소를 할당합니다.
- Pod 간 통신을 위한 네트워크 연결을 구성합니다.
- Node 간 Pod 통신을 위한 라우팅 또는 터널링을 구성합니다.
- 네트워크 정책(NetworkPolicy) 등의 네트워크 기능을 구현할 수 있습니다.

CNI는 Pod의 네트워크 연결을 구성하는 역할을 담당하며, kube-proxy는 Service의 가상 IP와 트래픽 전달 규칙을 관리하는 역할을 담당합니다.

두 구성 요소는 서로 다른 목적을 가지고 있지만, Cilium과 같이 eBPF 기반으로 Service 네트워크 기능까지 제공하는 구현에서는 kube-proxy의 기능을 대체할 수 있습니다.

## 3. Node 아키텍처

Node의 구성 요소들은 서로 협력하여 Pod의 실행 환경을 구성하고 유지합니다.

전체적인 아키텍처는 다음과 같습니다.

```text
                 Kubernetes Control Plane
              ┌────────────────────────────┐
              │                            │
              │        API Server          │
              │             ▲              │
              │             │              │
              │      Pod / Node 상태       │
              │                            │
              └─────────────┬──────────────┘
                            │
                    Kubernetes API
                            │
                            ▼
┌────────────────────────────────────────────────┐
│                    Worker Node                 │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │                  kubelet                 │  │
│  │                                          │  │
│  │  - Pod 생명주기 관리                     │  │
│  │  - Container Runtime 관리                │  │
│  │  - Node 상태 보고                        │  │
│  └───────────────────┬──────────────────────┘  │
│                      │                         │
│                      │ CRI                     │
│                      ▼                         │
│  ┌──────────────────────────────────────────┐  │
│  │             Container Runtime            │  │
│  │                                          │  │
│  │  - Image Management                      │  │
│  │  - Pod Sandbox                           │  │
│  │  - Container Lifecycle                   │  │
│  └───────────────────┬──────────────────────┘  │
│                      │                         │
│                      ▼                         │
│  ┌──────────────────────────────────────────┐  │
│  │                OCI Runtime               │  │
│  │                                          │  │
│  │  - Container Process Execution           │  │
│  └───────────────────┬──────────────────────┘  │
│                      │                         │
│                      ▼                         │
│  ┌──────────────────────────────────────────┐  │
│  │               Pod / Containers           │  │
│  └──────────────────────────────────────────┘  │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │                kube-proxy                │  │
│  │                                          │  │
│  │  - Service 네트워크 규칙 관리            │  │
│  │  - Service 트래픽 전달                   │  │
│  └──────────────────────────────────────────┘  │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │                 CNI Plugin               │  │
│  │                                          │  │
│  │  - Pod Network 구성                      │  │
│  │  - Network Interface 설정                │  │
│  └──────────────────────────────────────────┘  │
│                                                │
└────────────────────────────────────────────────┘
```

각 구성 요소의 관계는 다음과 같습니다.

- **kubelet과 API Server**: kubelet은 API Server를 통해 Pod 및 Node 정보를 확인하고 실행 상태를 보고합니다.
- **kubelet과 Container Runtime**: kubelet은 CRI를 통해 Pod Sandbox와 컨테이너의 생성 및 실행을 요청합니다.
- **Container Runtime과 OCI Runtime**: Container Runtime은 runc 등의 OCI Runtime을 활용하여 실제 컨테이너 프로세스를 실행합니다.
- **Container Runtime과 CNI Plugin**: Container Runtime은 CNI Plugin을 호출하여 Pod의 네트워크 환경을 구성할 수 있습니다.
- **kube-proxy와 API Server**: kube-proxy는 Service 및 EndpointSlice 정보를 감시하고 Node의 네트워크 규칙에 반영합니다.

위 구조는 일반적인 구성 관계를 나타냅니다. 실제 네트워크 플러그인 및 Container Runtime의 구현 방식에 따라 세부적인 호출 관계와 구성 요소는 달라질 수 있습니다.

## 4. Pod 생성 및 실행 과정

Pod는 Control Plane에서 스케줄링된 이후 해당 Node의 kubelet에 의해 실제 실행 환경이 구성됩니다.

일반적인 Pod 생성 및 실행 과정은 다음과 같습니다.

1. 사용자가 API Server에 Pod 생성을 요청합니다.
2. API Server가 Pod 객체를 생성합니다.
3. kube-scheduler가 Pod를 실행할 Node를 결정하고 할당 정보를 기록합니다.
4. 해당 Node의 kubelet이 자신에게 할당된 Pod 정보를 확인합니다.
5. kubelet이 PodSpec을 기반으로 필요한 컨테이너 이미지와 실행 설정을 확인합니다.
6. kubelet이 CRI를 통해 Container Runtime에 Pod Sandbox 생성을 요청합니다.
7. Container Runtime이 Pod Sandbox를 생성하고 CNI Plugin 등을 통해 필요한 네트워크 설정을 수행합니다.
8. Container Runtime이 필요한 컨테이너 이미지를 다운로드하거나 로컬 이미지를 확인합니다.
9. kubelet이 CRI를 통해 컨테이너 생성을 요청합니다.
10. Container Runtime이 OCI Runtime 등을 활용하여 컨테이너를 생성하고 실행합니다.
11. kubelet이 컨테이너 및 Pod의 상태를 확인하고 API Server에 보고합니다.

전체적인 흐름은 다음과 같습니다.

```text
       User
        │
        │ Pod 생성 요청
        ▼
   ┌───────────┐
   │ API Server│
   └─────┬─────┘
         │
         ▼
   ┌───────────┐
   │ Scheduler │
   └─────┬─────┘
         │
         │ Node 할당
         ▼
   ┌───────────┐
   │  kubelet  │
   └─────┬─────┘
         │
         │ CRI
         ▼
   ┌──────────────────┐
   │ Container Runtime│
   └────────┬─────────┘
            │
       ┌────┴─────┐
       │          │
       ▼          ▼
   Pod Sandbox  Image Pull
       │          │
       └────┬─────┘
            │
            ▼
      Container 생성
            │
            ▼
      OCI Runtime
            │
            ▼
      Container 실행
            │
            ▼
       Pod Running
```

Pod 생성 과정에서 kubelet은 Pod의 Desired State와 실제 실행 상태를 지속적으로 비교합니다. 컨테이너가 종료되거나 실행 조건을 만족하지 못하는 경우 Pod의 Restart Policy와 상태에 따라 필요한 조치를 수행합니다.

Pod가 정상적으로 실행되면 kubelet은 실행 상태를 API Server에 보고합니다.

## 5. Node의 네트워크 구조

Kubernetes Node의 네트워크는 Pod 간 통신과 Service를 통한 트래픽 전달을 지원하도록 구성됩니다.

이를 위해 CNI Plugin과 kube-proxy가 서로 다른 역할을 수행합니다.

### 5.1. Pod Network

Pod Network는 클러스터 내 Pod들이 서로 통신할 수 있도록 구성된 네트워크입니다.

각 Pod는 일반적으로 고유한 IP 주소를 할당받으며, 동일한 Pod에 속한 컨테이너들은 네트워크 Namespace를 공유합니다.

CNI Plugin은 Pod Sandbox 생성 과정에서 네트워크 인터페이스와 IP 주소를 구성합니다.

Pod Network의 주요 특징은 다음과 같습니다.

- Pod마다 고유한 IP 주소를 할당할 수 있습니다.
- 동일한 Node에 있는 Pod 간 통신을 지원합니다.
- 서로 다른 Node에 있는 Pod 간 통신을 지원합니다.
- 네트워크 플러그인에 따라 라우팅, 오버레이 네트워크 또는 eBPF 등의 구현 방식을 사용할 수 있습니다.

### 5.2. Service Network

Service는 Pod에 안정적인 네트워크 접근 경로를 제공하는 Kubernetes 리소스입니다.

Pod는 생성 및 종료에 따라 IP 주소가 변경될 수 있으므로, Service는 일반적으로 고정된 가상 IP인 ClusterIP를 통해 여러 Pod에 접근할 수 있도록 합니다.

kube-proxy는 Service와 EndpointSlice 정보를 기반으로 Node의 네트워크 규칙을 구성합니다.

대표적인 Service 유형은 다음과 같습니다.

| Service Type | 설명 |
|---|---|
| ClusterIP | 클러스터 내부에서 접근할 수 있는 가상 IP를 제공합니다. |
| NodePort | 각 Node의 특정 포트를 통해 Service에 접근할 수 있도록 합니다. |
| LoadBalancer | 외부 Load Balancer를 통해 Service에 접근할 수 있도록 합니다. |
| ExternalName | DNS CNAME 레코드를 통해 외부 도메인 이름을 연결합니다. |

ExternalName은 DNS 기반으로 동작하므로 kube-proxy의 일반적인 트래픽 전달 규칙을 사용하지 않습니다.

### 5.3. CNI와 kube-proxy의 차이

CNI Plugin과 kube-proxy는 모두 Kubernetes 네트워크에 관여하지만 담당하는 기능은 다릅니다.

| 구분 | CNI Plugin | kube-proxy |
|---|---|---|
| 주요 목적 | Pod 네트워크 연결 구성 | Service 트래픽 전달 |
| 주요 역할 | Network Interface 및 IP 주소 구성 | Service 가상 IP 및 트래픽 전달 규칙 관리 |
| 적용 대상 | Pod Network | Service Network |
| 주요 동작 시점 | Pod Sandbox 생성 및 네트워크 설정 | Service 및 EndpointSlice 변경 감지 |
| 대표 구현 방식 | Bridge, Routing, Overlay, eBPF 등 | iptables, IPVS, nftables 등 |

CNI Plugin은 Pod가 네트워크에 연결될 수 있도록 구성하고, kube-proxy는 Service를 통해 Pod에 접근할 수 있도록 네트워크 규칙을 관리합니다.

다만 Cilium과 같이 eBPF 기반으로 Pod 네트워크와 Service 트래픽 전달 기능을 함께 제공하는 구현에서는 kube-proxy 없이도 해당 기능을 제공할 수 있습니다.

## 6. Node 상태 및 장애 관리

Kubernetes는 Node의 상태를 지속적으로 확인하여 정상적으로 워크로드를 실행할 수 있는지 판단합니다.

Node의 상태 정보는 Kubernetes API의 Node 객체를 통해 관리됩니다.

### 6.1. Node Status

Node Status는 Node의 현재 상태와 리소스 정보를 나타냅니다.

주요 항목은 다음과 같습니다.

| 항목 | 설명 |
|---|---|
| Capacity | Node가 제공할 수 있는 전체 리소스 용량입니다. |
| Allocatable | Kubernetes가 Pod에 할당할 수 있는 리소스 용량입니다. |
| Conditions | Node의 현재 상태를 나타냅니다. |
| Addresses | Node의 IP 주소 및 Hostname 등의 정보를 포함합니다. |
| NodeInfo | OS, Kernel, Container Runtime 및 kubelet 버전 등의 정보를 포함합니다. |

예를 들어 Capacity와 Allocatable은 다음과 같이 구분됩니다.

- Capacity: Node가 제공할 수 있는 전체 CPU 및 Memory 용량입니다.
- Allocatable: 시스템 및 Kubernetes 구성 요소가 사용하는 리소스를 고려하여 Pod에 할당할 수 있는 용량입니다.

따라서 Pod의 리소스 스케줄링은 일반적으로 Node의 Capacity가 아니라 Allocatable을 기준으로 판단합니다.

### 6.2. Node Conditions

Node Conditions는 Node의 상태를 나타내는 조건 정보입니다.

대표적인 Condition은 다음과 같습니다.

| Condition | 설명 |
|---|---|
| Ready | Node가 정상적으로 동작하며 Pod를 실행할 수 있는 상태인지 나타냅니다. |
| MemoryPressure | Node의 Memory 리소스가 부족한 상태인지 나타냅니다. |
| DiskPressure | Node의 디스크 공간 또는 Inode가 부족한 상태인지 나타냅니다. |
| PIDPressure | Node에서 사용할 수 있는 프로세스 ID가 부족한 상태인지 나타냅니다. |
| NetworkUnavailable | Node의 네트워크 구성이 정상적으로 완료되었는지 나타냅니다. |

`Ready` Condition이 `False` 또는 `Unknown`인 경우 해당 Node는 정상적인 워크로드 실행이 어려운 상태로 판단될 수 있습니다.

### 6.3. Node Heartbeat와 Lease

Kubernetes는 Node의 상태를 확인하기 위해 Heartbeat를 사용합니다.

kubelet은 Node Status를 갱신하고, 별도의 Lease 객체를 주기적으로 갱신하여 Node가 정상적으로 동작하고 있음을 알립니다.

Lease는 `kube-node-lease` Namespace에 저장됩니다.

Node의 Lease 갱신이 일정 시간 동안 확인되지 않으면 Control Plane은 해당 Node와의 통신이 원활하지 않은 것으로 판단할 수 있습니다.

이러한 방식으로 Kubernetes는 Node의 장애 및 통신 상태를 감지합니다.

### 6.4. Node 장애와 Pod 처리

Node가 장애 상태가 되면 Kubernetes는 해당 Node의 상태를 확인하고 필요한 조치를 수행합니다.

일반적인 처리 과정은 다음과 같습니다.

1. kubelet이 Node의 Heartbeat 및 Lease를 갱신하지 못합니다.
2. Control Plane의 Node Lifecycle Controller가 Node의 상태를 확인합니다.
3. 일정 시간 동안 정상적인 상태가 확인되지 않으면 Node의 Ready Condition이 `Unknown` 등으로 변경될 수 있습니다.
4. 해당 Node에 대한 Pod 스케줄링이 제한될 수 있습니다.
5. 설정된 조건과 시간에 따라 Pod에 Taint가 적용되고, 기존 Pod의 Eviction이 진행될 수 있습니다.
6. 관리 대상 워크로드는 새로운 Node에 재생성될 수 있습니다.

Node 장애 이후 Pod가 다른 Node에서 실행되는 것은 기존 컨테이너가 이동하는 것이 아니라, 컨트롤러가 관리하는 워크로드의 새로운 Pod가 생성되는 방식입니다.

## 7. Node와 Control Plane의 상호작용

Node는 Control Plane과 지속적으로 통신하며 클러스터의 Desired State를 유지합니다.

각 구성 요소의 상호작용은 다음과 같습니다.

| Control Plane 구성 요소 | Node와의 관계 |
|---|---|
| kube-apiserver | Node 및 Pod 정보를 제공하고 Node 구성 요소의 상태 보고를 수신합니다. |
| kube-scheduler | Pod를 실행할 Node를 선택하고 Pod의 Node 할당 정보를 기록합니다. |
| kube-controller-manager | Node 상태를 감시하고 장애 상황에서 Node 및 Pod 관련 처리를 수행합니다. |

Node에서 실행되는 구성 요소는 다음과 같이 Control Plane과 상호작용합니다.

- kubelet은 API Server를 통해 Pod 정보를 확인하고 Node 및 Pod 상태를 보고합니다.
- kube-proxy는 API Server의 Service 및 EndpointSlice 정보를 확인하여 네트워크 규칙을 관리합니다.
- Container Runtime은 kubelet의 CRI 요청을 처리하여 실제 컨테이너를 실행합니다.
- CNI Plugin은 Pod의 네트워크 연결을 구성합니다.

이러한 구조를 통해 Kubernetes는 Control Plane에서 정의한 Desired State를 각 Node의 실제 실행 상태에 반영합니다.

## 8. Node 확인 명령어

### 8.1. Node 목록 확인

클러스터에 등록된 Node 목록을 확인합니다.

```bash
kubectl get nodes
```

Node의 상세 정보를 확인합니다.

```bash
kubectl describe node <node-name>
```

### 8.2. Node 리소스 확인

Node의 CPU 및 Memory 용량과 할당 가능 리소스를 확인할 수 있습니다.

```bash
kubectl describe node <node-name>
```

Node의 리소스 사용량을 확인하려면 Metrics API가 제공되어야 합니다.

```bash
kubectl top nodes
```

### 8.3. Node에 실행 중인 Pod 확인

특정 Node에 할당된 Pod 목록을 확인합니다.

```bash
kubectl get pods -A -o wide --field-selector spec.nodeName=<node-name>
```

전체 Node에서 실행 중인 Pod 목록을 확인합니다.

```bash
kubectl get pods -A -o wide
```

### 8.4. Node 구성 요소 확인

kubelet 상태를 확인합니다.

```bash
systemctl status kubelet
```

Container Runtime 상태를 확인합니다.

```bash
systemctl status containerd
```

kube-proxy가 실행 중인지 확인합니다.

```bash
kubectl get pods -n kube-system -o wide
```

CNI Plugin의 설치 및 실행 상태는 사용 중인 네트워크 플러그인에 따라 확인 방법이 다릅니다.

## 9. 핵심 정리

Kubernetes Node는 Pod와 컨테이너가 실제로 실행되는 환경이며, Control Plane과 협력하여 클러스터의 워크로드를 유지합니다.

Node의 주요 구성 요소와 역할은 다음과 같습니다.

| 구성 요소 | 핵심 역할 |
|---|---|
| kubelet | Pod 생명주기와 Node 상태를 관리합니다. |
| Container Runtime | Pod Sandbox 및 컨테이너를 생성하고 실행합니다. |
| OCI Runtime | 실제 컨테이너 프로세스의 실행 환경을 구성합니다. |
| CNI Plugin | Pod의 네트워크 연결을 구성합니다. |
| kube-proxy | Service의 네트워크 규칙과 트래픽 전달을 관리합니다. |

Node의 전체적인 동작은 다음과 같이 이해할 수 있습니다.

1. kube-scheduler가 Pod를 실행할 Node를 결정합니다.
2. 해당 Node의 kubelet이 API Server를 통해 Pod 정보를 확인합니다.
3. kubelet이 CRI를 통해 Container Runtime에 Pod Sandbox 및 컨테이너 생성을 요청합니다.
4. Container Runtime은 필요한 이미지와 실행 환경을 준비하고 컨테이너를 실행합니다.
5. CNI Plugin은 Pod의 네트워크 연결을 구성합니다.
6. kube-proxy는 Service 트래픽 전달에 필요한 네트워크 규칙을 관리합니다.
7. kubelet은 Pod 및 Node의 상태를 지속적으로 확인하고 API Server에 보고합니다.

결과적으로 Node는 단순히 컨테이너를 실행하는 서버가 아니라, Pod의 실행과 네트워크 연결, 리소스 관리 및 장애 감지를 담당하는 Kubernetes의 핵심 실행 환경입니다.

## 10. 참고자료

- [Kubernetes Documentation - Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)
  - Node의 개념과 구성 요소, Node 상태 및 관리 방식을 설명합니다.
- [Kubernetes Documentation - Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
  - Control Plane과 Node를 포함한 Kubernetes 전체 아키텍처를 설명합니다.
- [Kubernetes Documentation - Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
  - Kubernetes에서 사용하는 Container Runtime의 종류와 설정 방법을 설명합니다.
- [Kubernetes Documentation - Container Runtime Interface (CRI)](https://kubernetes.io/docs/concepts/containers/cri/)
  - kubelet과 Container Runtime 사이의 통신 구조를 설명합니다.
- [Kubernetes Documentation - Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
  - Kubernetes에서 사용하는 CNI 기반 네트워크 플러그인의 역할과 동작 방식을 설명합니다.
- [Kubernetes Documentation - Service](https://kubernetes.io/docs/concepts/services-networking/service/)
  - Service의 개념과 유형, Pod에 대한 네트워크 접근 방식을 설명합니다.
- [Kubernetes Documentation - Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/)
  - kube-proxy의 동작 방식과 Service 가상 IP 구현 방식을 설명합니다.
- [Kubernetes Documentation - EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
  - Service의 트래픽 전달 대상인 EndpointSlice의 구조와 관리 방식을 설명합니다.
- [Kubernetes Documentation - Node Status](https://kubernetes.io/docs/reference/node/node-status/)
  - Node Status, Conditions, Capacity 및 Allocatable 등의 상태 정보를 설명합니다.
- [Kubernetes Documentation - Node Heartbeats](https://kubernetes.io/docs/concepts/architecture/nodes/#heartbeats)
  - Node Heartbeat와 Lease를 이용한 Node 상태 확인 방식을 설명합니다.
- [Kubernetes Documentation - Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
  - Node의 리소스 부족 상황에서 수행되는 Pod Eviction 동작을 설명합니다.
- [Kubernetes Documentation - RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/)
  - Pod별 Container Runtime Handler 선택 및 실행 환경 설정 방법을 설명합니다.