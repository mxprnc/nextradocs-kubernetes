# Node

## 1. Node란?

Node는 Kubernetes 클러스터에서 Pod를 실행하는 기본 컴퓨팅 자원입니다. 물리 서버(Physical Machine) 또는 가상 머신(Virtual Machine)으로 구성할 수 있으며, 클러스터의 Control Plane에 의해 관리됩니다.

Kubernetes는 컨테이너를 Pod에 배치하고, 해당 Pod를 Node에 스케줄링하여 애플리케이션을 실행합니다. 하나의 클러스터는 여러 Node로 구성될 수 있으며, 각 Node는 할당된 Pod를 실행하는 데 필요한 컴퓨팅 자원과 시스템 구성 요소를 제공합니다.

Node는 단순히 애플리케이션을 실행하는 서버에 그치지 않습니다. Kubernetes Scheduler가 Pod를 배치할 대상을 결정하는 기준이 되는 자원이며, 각 Node의 CPU, Memory, Pod 수용량, 상태 및 레이블 등의 정보를 통해 클러스터 전체의 워크로드 배치를 관리합니다.

주요 특징은 다음과 같습니다.

- **컴퓨팅 자원 제공:** Pod가 실행되는 CPU, Memory, Storage 및 네트워크 환경을 제공합니다.
- **워크로드 실행:** kubelet과 컨테이너 런타임을 통해 Pod에 정의된 컨테이너를 실행하고 관리합니다.
- **스케줄링 대상:** Kubernetes Scheduler가 Pod의 리소스 요구 사항과 Node의 상태 및 제약 조건을 고려하여 실행 위치를 결정합니다.
- **상태 관리:** Node의 가용성과 자원 상태를 Control Plane에 전달하여 장애 감지 및 워크로드 관리에 활용합니다.
- **확장성:** 클러스터에 Node를 추가하거나 제거하여 전체 컴퓨팅 자원을 확장하거나 축소할 수 있습니다.

## 2. Node의 아키텍처

Node는 Control Plane과 통신하면서 Pod를 실행하는 역할을 담당합니다.

일반적으로 Node에는 kubelet, Container Runtime, kube-proxy 등의 구성 요소가 존재합니다.

### 2.1. Node 구성 요소

| 구성 요소 | 설명 |
|---|---|
| kubelet | Node에서 실행되는 에이전트로, API Server와 통신하며 해당 Node에 할당된 Pod와 컨테이너의 상태를 관리합니다. |
| Container Runtime | 컨테이너 이미지를 가져오고 컨테이너를 생성 및 실행하는 역할을 담당합니다. containerd, CRI-O 등이 대표적입니다. |
| kube-proxy | Service의 네트워크 기능을 구현하고 Pod로 향하는 트래픽을 전달하기 위한 네트워크 규칙을 관리합니다. |

참고로 kube-proxy는 모든 클러스터에서 반드시 실행해야 하는 구성 요소는 아닙니다. 일부 네트워크 구현체는 kube-proxy의 기능을 자체적으로 대체할 수 있습니다.

### 2.2. 구성 요소별 동작 방식

**1) kubelet**

kubelet은 각 Node에서 실행되는 Kubernetes 에이전트입니다.

Control Plane의 API Server를 통해 자신에게 할당된 Pod의 정보를 확인하고, 해당 Pod가 정상적으로 실행되도록 Container Runtime에 필요한 작업을 요청합니다.

주요 역할은 다음과 같습니다.

- API Server와 통신하여 Node에 할당된 Pod의 정보를 확인합니다.
- Pod에 정의된 컨테이너가 정상적으로 실행되도록 관리합니다.
- 컨테이너의 상태와 리소스 사용량 등을 모니터링합니다.
- Node의 상태 정보를 API Server에 전달합니다.
- Liveness Probe, Readiness Probe 등의 상태 검사를 수행합니다.

**2) Container Runtime**

Container Runtime은 실제 컨테이너의 실행 환경을 담당합니다.

kubelet은 CRI(Container Runtime Interface)를 통해 Container Runtime과 통신하며, 컨테이너의 생성 및 실행, 종료 등의 작업을 요청합니다.

대표적인 Container Runtime은 다음과 같습니다.

- containerd
- CRI-O

Kubernetes는 CRI를 통해 Container Runtime과 통신하므로 특정 Runtime에 종속되지 않고 다양한 컨테이너 실행 환경을 사용할 수 있습니다.

**3) kube-proxy**

kube-proxy는 Kubernetes Service의 네트워크 기능을 구현하는 구성 요소입니다.

Service에 연결된 Pod의 네트워크 엔드포인트를 기반으로 트래픽 전달 규칙을 관리합니다.

일반적으로 iptables 또는 IPVS 등의 방식을 사용하여 Service의 ClusterIP와 Pod 간의 트래픽 전달을 처리합니다.

단, kube-proxy는 선택적인 구성 요소이며, Cilium과 같은 일부 CNI 구현체는 eBPF 등을 활용하여 kube-proxy의 기능을 대체할 수 있습니다.

## 3. Node의 등록 및 관리

Node는 Kubernetes API Server에 Node 리소스로 등록되어 관리됩니다.

Node를 클러스터에 추가하는 방법은 크게 두 가지입니다.

### 3.1. 자동 등록 (Self-registration)

일반적으로 Kubernetes 클러스터에 Worker Node를 추가하면 해당 Node에서 실행되는 kubelet이 API Server에 자신을 등록합니다.

이 과정에서 kubelet은 자신의 Node 이름과 주소, 하드웨어 정보, 리소스 용량 등의 정보를 전달합니다.

등록이 완료되면 Kubernetes는 해당 Node를 관리 대상으로 인식하고, Scheduler가 Pod를 배치할 수 있는 대상으로 활용합니다.

### 3.2. 수동 등록 (Manual Registration)

관리자가 직접 Kubernetes API를 통해 Node 리소스를 생성할 수도 있습니다.

이 경우 Node의 이름과 주소, 레이블, 리소스 용량 등의 정보를 직접 설정해야 합니다.

수동으로 Node 리소스를 생성하는 경우 해당 Node에서 실행되는 kubelet이 자동으로 등록하지 않도록 `--register-node=false` 옵션을 설정할 수 있습니다.

다만 Node 리소스를 수동으로 생성했다고 해서 실제 컴퓨팅 환경이 자동으로 구성되는 것은 아닙니다. 해당 Node에서 kubelet과 Container Runtime 등의 구성 요소가 정상적으로 동작할 수 있어야 합니다.

### 3.3. Node 리소스 확인

클러스터에 등록된 Node는 다음 명령어로 확인할 수 있습니다.

```bash
kubectl get nodes
```

출력 예시는 다음과 같습니다.

```text
NAME           STATUS   ROLES           AGE   VERSION
master-node    Ready    control-plane   30d   v1.35.0
worker-node1   Ready    <none>          30d   v1.35.0
worker-node2   Ready    <none>          30d   v1.35.0
```

각 항목의 의미는 다음과 같습니다.

| 항목 | 설명 |
|---|---|
| NAME | Node의 이름입니다. |
| STATUS | Node의 상태를 나타냅니다. 일반적으로 Ready 또는 NotReady 등의 상태를 확인할 수 있습니다. |
| ROLES | Node에 설정된 역할 레이블을 표시합니다. |
| AGE | Node 리소스가 생성된 이후 경과한 시간입니다. |
| VERSION | 해당 Node에서 실행 중인 kubelet의 Kubernetes 버전입니다. |

특정 Node의 상세 정보를 확인하려면 다음 명령어를 사용합니다.

```bash
kubectl describe node worker-node1
```

Node의 레이블, 상태, 할당 가능한 리소스, 실행 중인 Pod 및 이벤트 등의 정보를 확인할 수 있습니다.

## 4. Node의 상태 (Node Status)

Kubernetes는 Node의 현재 상태를 지속적으로 확인하고 관리합니다.

Node의 상태 정보는 API 리소스의 `.status` 필드에 기록되며, 크게 다음과 같은 항목으로 구성됩니다.

- Addresses
- Conditions
- Capacity
- Allocatable
- Info

### 4.1. Addresses

Node의 네트워크 주소 정보를 나타냅니다.

| 항목 | 설명 |
|---|---|
| HostName | Node의 호스트 이름입니다. |
| InternalIP | 클러스터 내부에서 통신할 때 사용하는 Node의 IP 주소입니다. |
| ExternalIP | 클러스터 외부에서 접근할 수 있는 Node의 IP 주소입니다. |

실제 주소 구성은 클라우드 환경이나 Bare Metal 환경에 따라 달라질 수 있습니다.

### 4.2. Conditions

Conditions는 Node의 정상 여부와 자원 상태를 나타내는 조건입니다.

대표적인 Node Condition은 다음과 같습니다.

| Condition | 설명 |
|---|---|
| Ready | Node가 정상적으로 동작하며 Pod를 수용할 준비가 되었는지를 나타냅니다. |
| MemoryPressure | Node의 Memory 자원이 부족한 상태인지를 나타냅니다. |
| DiskPressure | Node의 디스크 자원이 부족한 상태인지를 나타냅니다. |
| PIDPressure | Node에서 사용할 수 있는 프로세스 ID 자원이 부족한 상태인지를 나타냅니다. |
| NetworkUnavailable | Node의 네트워크가 정상적으로 구성되어 있는지를 나타냅니다. |

Condition의 상태는 일반적으로 다음 세 가지 값으로 표현됩니다.

- `True`: 해당 조건이 발생한 상태입니다.
- `False`: 해당 조건이 발생하지 않은 상태입니다.
- `Unknown`: 해당 조건의 상태를 확인할 수 없는 상태입니다.

예를 들어 Ready Condition은 다음과 같이 해석할 수 있습니다.

| Ready 상태 | 의미 |
|---|---|
| True | Node가 정상적으로 동작하며 Pod를 수용할 수 있습니다. |
| False | Node가 정상적이지 않아 Pod를 수용할 수 없습니다. |
| Unknown | Control Plane이 Node의 상태를 확인할 수 없습니다. |

Node가 정상적으로 동작하지 않거나 Control Plane과의 통신이 끊기면 Ready Condition이 False 또는 Unknown으로 변경될 수 있습니다.

이러한 상태 변화는 Node Controller에 의해 감지되며, 이후 해당 Node에 배치된 Pod의 처리와 새로운 Pod의 스케줄링에 영향을 줍니다.

### 4.3. Capacity와 Allocatable

Capacity와 Allocatable은 Node가 제공하는 컴퓨팅 자원의 크기를 나타냅니다.

두 항목은 Pod 스케줄링 과정에서 중요한 기준으로 활용됩니다.

**Capacity**

Node가 보유한 전체 리소스 용량을 의미합니다.

예를 들어 8개의 CPU 코어와 16GiB의 Memory를 보유한 Node라면 다음과 같이 표현할 수 있습니다.

```yaml
capacity:
  cpu: "8"
  memory: 16Gi
  pods: "110"
```

**Allocatable**

Allocatable은 일반 Pod가 사용할 수 있도록 할당 가능한 리소스 용량을 의미합니다.

Node의 전체 리소스에서 운영체제와 Kubernetes 시스템 구성 요소 등을 위해 예약한 리소스를 제외한 값입니다.

```yaml
allocatable:
  cpu: "7500m"
  memory: 14Gi
  pods: "110"
```

Capacity와 Allocatable의 차이는 다음과 같습니다.

| 구분 | Capacity | Allocatable |
|---|---|---|
| 의미 | Node가 보유한 전체 리소스 | 일반 Pod에 할당 가능한 리소스 |
| CPU | 전체 CPU 용량 | 시스템 예약분 등을 제외한 CPU 용량 |
| Memory | 전체 Memory 용량 | 시스템 예약분 등을 제외한 Memory 용량 |
| 용도 | Node의 전체 자원 규모 파악 | Pod 스케줄링 가능 여부 판단 |

예를 들어 CPU Capacity가 8이고 Allocatable이 7500m라면 전체 CPU 8코어 중 7.5코어를 일반 Pod에 할당할 수 있다는 의미입니다.

Kubernetes Scheduler는 Pod의 CPU 및 Memory Requests를 기준으로 Node의 Allocatable에서 사용 가능한 리소스가 충분한지 확인합니다.

여기서 중요한 점은 실제 사용량이 아니라 **Pod에 예약된 Requests의 합계**를 기준으로 스케줄링 가능 여부를 판단한다는 것입니다.

따라서 실제 CPU 사용량이 낮더라도 Pod의 CPU Requests가 크다면 새로운 Pod가 해당 Node에 배치되지 못할 수 있습니다.

### 4.4. Info

Info는 Node의 시스템 및 소프트웨어 환경에 대한 정보를 나타냅니다.

대표적으로 다음과 같은 정보가 포함됩니다.

- Architecture: CPU 아키텍처
- Operating System: 운영체제
- Kernel Version: 커널 버전
- OS Image: 운영체제 이미지 정보
- Container Runtime Version: 컨테이너 런타임 버전
- Kubelet Version: kubelet 버전
- Kube-Proxy Version: kube-proxy 버전

이러한 정보는 Node의 실행 환경을 확인하거나 버전 호환성 및 장애 원인을 분석할 때 활용됩니다.

## 5. Node Heartbeat

Node Heartbeat는 각 Node가 정상적으로 동작하고 있는지를 Control Plane이 확인할 수 있도록 전달하는 상태 신호입니다.

Kubernetes는 Node의 상태를 지속적으로 확인해야 하므로 각 Node는 주기적으로 자신의 상태 정보를 전달합니다.

Node Heartbeat는 다음 두 가지 방식으로 전달됩니다.

**1) Node Status Update**

kubelet이 Node 리소스의 `.status` 필드를 업데이트하는 방식입니다.

Node의 Condition이나 주소, 리소스 정보 등의 변경 사항을 Control Plane에 전달합니다.

**2) Lease Object**

각 Node는 `kube-node-lease` 네임스페이스에 자신의 이름과 동일한 Lease 리소스를 가집니다.

kubelet은 Lease의 갱신 시간을 주기적으로 업데이트하여 자신이 정상적으로 동작하고 있음을 알립니다.

Lease는 Node Status 전체를 업데이트하는 것보다 가벼운 방식이므로 대규모 클러스터에서 Heartbeat를 효율적으로 처리하는 데 활용됩니다.

### 5.1. Heartbeat와 Node 장애 감지

Node가 정상적으로 동작하면 kubelet은 주기적으로 Heartbeat를 전달합니다.

반면 Node 장애, 네트워크 단절 또는 kubelet 장애 등이 발생하면 Heartbeat가 전달되지 않을 수 있습니다.

이 경우 Control Plane의 Node Controller는 일정 시간 동안 Node의 응답이 없는 것을 확인한 후 해당 Node의 Ready Condition을 Unknown으로 변경할 수 있습니다.

이후 Node Controller는 `node.kubernetes.io/unreachable` 등의 Taint를 추가하여 해당 Node에 새로운 Pod가 배치되지 않도록 처리하고, 기존 Pod에 대한 Eviction 절차를 진행할 수 있습니다.

Node의 장애 감지부터 Pod 처리까지의 과정은 다음과 같습니다.

1. Node에서 kubelet 또는 네트워크 장애가 발생합니다.
2. Node에서 전달하던 Heartbeat가 중단됩니다.
3. Node Controller가 일정 시간 동안 응답이 없는 것을 감지합니다.
4. Node의 Ready Condition을 Unknown으로 변경합니다.
5. `node.kubernetes.io/unreachable` Taint가 추가됩니다.
6. 해당 Taint를 허용하지 않는 Pod는 설정된 Toleration 시간 등에 따라 Eviction 대상이 될 수 있습니다.

Node Controller는 이러한 과정을 통해 장애가 발생한 Node가 클러스터 전체의 워크로드에 영향을 미치는 것을 최소화합니다.

## 6. Node Controller

Node Controller는 Control Plane에서 Node의 상태와 생명주기를 관리하는 컨트롤러입니다.

Node의 등록 상태와 가용성을 확인하고, 장애가 발생한 Node에 대한 후속 처리를 담당합니다.

주요 역할은 다음과 같습니다.

- Node의 상태를 주기적으로 모니터링합니다.
- Node의 Heartbeat를 확인하여 가용성을 판단합니다.
- Node의 Ready Condition을 업데이트합니다.
- Node 장애가 발생하면 관련 Taint를 추가합니다.
- 장애가 발생한 Node에 배치된 Pod의 Eviction을 처리합니다.
- 클라우드 환경에서는 Cloud Provider와 연동하여 실제 인스턴스의 존재 여부를 확인할 수 있습니다.

특히 Node Controller는 Node의 상태가 비정상적으로 변경되었을 때 해당 Node에 배치된 Pod를 어떻게 처리할 것인지 결정하는 데 중요한 역할을 합니다.

## 7. Node의 스케줄링 제어

Kubernetes Scheduler는 Pod를 실행할 적절한 Node를 선택합니다.

이 과정에서 Node의 리소스 상태뿐만 아니라 레이블, Taint, 스케줄링 가능 여부 등의 조건을 함께 고려합니다.

Node 자체에서 스케줄링을 제어하는 대표적인 방법은 다음과 같습니다.

### 7.1. Node Label

Node Label은 Node에 특정 속성이나 역할을 부여하기 위한 Key-Value 형태의 메타데이터입니다.

예를 들어 GPU가 장착된 Node에 다음과 같은 레이블을 설정할 수 있습니다.

```bash
kubectl label node worker-node1 accelerator=nvidia-gpu
```

이후 Pod의 `nodeSelector`를 사용하면 해당 레이블을 가진 Node에만 Pod가 배치되도록 제한할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  nodeSelector:
    accelerator: nvidia-gpu
  containers:
    - name: app
      image: nginx:latest
```

위 예시에서는 `accelerator=nvidia-gpu` 레이블을 가진 Node만 해당 Pod의 스케줄링 대상으로 선택됩니다.

Node Label은 특정 하드웨어, 인프라 환경 또는 Node의 역할에 따라 워크로드를 분리할 때 활용됩니다.

### 7.2. Unschedulable

Node의 Unschedulable 설정은 해당 Node에 새로운 일반 Pod가 스케줄링되지 않도록 제한하는 기능입니다.

주로 Node 유지보수나 시스템 업데이트를 준비할 때 활용됩니다.

다음 명령어로 Node를 스케줄 불가능한 상태로 변경할 수 있습니다.

```bash
kubectl cordon worker-node1
```

이 명령어를 실행하면 Node의 `.spec.unschedulable` 값이 `true`로 설정됩니다.

이후 Scheduler는 해당 Node를 새로운 일반 Pod의 스케줄링 대상으로 선택하지 않습니다.

다만 이미 해당 Node에서 실행 중인 Pod에는 영향을 주지 않습니다.

스케줄링을 다시 허용하려면 다음 명령어를 사용합니다.

```bash
kubectl uncordon worker-node1
```

### 7.3. Node Taint

Node Taint는 특정 조건을 가진 Node에 일반 Pod가 무분별하게 배치되지 않도록 제한하는 기능입니다.

Node에 Taint가 설정되면 해당 Taint를 허용하는 Toleration을 가진 Pod만 스케줄링 대상이 될 수 있습니다.

예를 들어 특정 Node를 GPU 워크로드 전용으로 사용하려면 다음과 같이 Taint를 설정할 수 있습니다.

```bash
kubectl taint nodes worker-node1 accelerator=nvidia-gpu:NoSchedule
```

이후 해당 Node에 Pod를 배치하려면 Pod에 대응하는 Toleration을 설정해야 합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  tolerations:
    - key: "accelerator"
      operator: "Equal"
      value: "nvidia-gpu"
      effect: "NoSchedule"
  containers:
    - name: app
      image: nginx:latest
```

Taint와 Toleration은 Node와 Pod 사이의 스케줄링 조건을 제어하는 기능입니다.

Node Taint의 주요 Effect는 다음과 같습니다.

| Effect | 설명 |
|---|---|
| NoSchedule | 해당 Taint를 허용하지 않는 새로운 Pod의 스케줄링을 차단합니다. |
| PreferNoSchedule | 가능한 경우 해당 Taint를 허용하지 않는 Pod를 다른 Node에 배치하도록 선호합니다. |
| NoExecute | 해당 Taint를 허용하지 않는 새로운 Pod의 스케줄링을 차단하고, 이미 실행 중인 Pod도 축출할 수 있습니다. |

Taint와 Toleration의 구체적인 동작 방식은 별도의 Taint/Toleration 문서에서 설명합니다.

## 8. Node 유지보수 (Cordon & Drain)

Node는 운영체제 업데이트, 커널 패치, 하드웨어 교체 및 시스템 장애 복구 등의 이유로 유지보수가 필요할 수 있습니다.

이러한 작업을 진행할 때는 기존 Pod를 안전하게 다른 Node로 이동시키고, 유지보수 대상 Node에 새로운 Pod가 배치되지 않도록 처리해야 합니다.

Kubernetes는 이를 위해 `cordon`, `drain`, `uncordon` 명령어를 제공합니다.

### 8.1. Cordon

Cordon은 특정 Node를 스케줄링 불가능한 상태로 변경하는 작업입니다.

```bash
kubectl cordon worker-node1
```

실행 결과는 다음과 같습니다.

- Node의 `.spec.unschedulable` 값이 `true`로 설정됩니다.
- 새로운 일반 Pod가 해당 Node에 스케줄링되지 않습니다.
- 기존에 실행 중인 Pod는 그대로 유지됩니다.

Cordon은 Node 유지보수를 시작하기 전에 새로운 워크로드가 유입되는 것을 차단하는 용도로 사용됩니다.

### 8.2. Drain

Drain은 특정 Node에서 실행 중인 Pod를 안전하게 종료하거나 축출하여 해당 Node를 비우는 작업입니다.

```bash
kubectl drain worker-node1 --ignore-daemonsets
```

Drain을 실행하면 다음과 같은 작업이 수행됩니다.

1. 대상 Node를 Unschedulable 상태로 변경합니다.
2. 해당 Node에서 실행 중인 Pod를 확인합니다.
3. Eviction API를 사용할 수 있는 경우 Eviction을 요청합니다.
4. Pod가 종료되면 Deployment, ReplicaSet 등의 상위 컨트롤러가 필요한 경우 다른 Node에 새로운 Pod를 생성합니다.
5. Node에서 처리 대상 Pod가 제거되면 Drain 작업이 완료됩니다.

Drain은 단순히 Pod를 삭제하는 것이 아니라 Kubernetes의 Eviction 절차를 활용하여 기존 워크로드를 안전하게 종료하는 기능입니다.

다음 사항에 유의해야 합니다.

- DaemonSet에 의해 관리되는 Pod는 일반적인 Drain 과정에서 삭제되지 않습니다. 따라서 `--ignore-daemonsets` 옵션이 필요할 수 있습니다.
- Static Pod는 Mirror Pod로 API Server에 표시되며 Drain으로 제거할 수 없습니다.
- 상위 컨트롤러가 없는 Pod는 기본적으로 Drain 과정에서 삭제되지 않으며, `--force` 옵션이 필요한 경우가 있습니다.
- PodDisruptionBudget(PDB)이 설정된 경우 허용 가능한 중단 범위를 초과하는 Eviction은 차단될 수 있습니다.

특히 PDB는 애플리케이션의 가용성을 유지하면서 Pod를 축출할 수 있도록 제어하는 리소스입니다.

따라서 운영 환경에서는 Drain을 실행하기 전에 대상 Node의 Pod 구성과 PDB 설정을 확인하는 것이 중요합니다.

### 8.3. Uncordon

유지보수가 완료된 Node를 다시 스케줄링 가능한 상태로 변경하는 작업입니다.

```bash
kubectl uncordon worker-node1
```

실행하면 Node의 Unschedulable 설정이 해제되고 Scheduler가 해당 Node를 다시 스케줄링 대상으로 고려할 수 있습니다.

단, Uncordon은 기존 Pod를 자동으로 복구하거나 다시 배치하는 작업이 아닙니다. 이후 새로운 Pod가 해당 Node에 스케줄링될 수 있도록 허용하는 작업입니다.

### 8.4. 유지보수 작업 흐름

일반적인 Node 유지보수 절차는 다음과 같습니다.

```bash
# 1. Node 상태 확인
kubectl get nodes

# 2. 새로운 Pod 스케줄링 차단
kubectl cordon worker-node1

# 3. 기존 Pod를 안전하게 축출
kubectl drain worker-node1 --ignore-daemonsets

# 4. Node 유지보수 작업 수행
# 운영체제 업데이트, 커널 패치, 재부팅 등

# 5. Node 상태 확인
kubectl get nodes

# 6. 스케줄링 허용
kubectl uncordon worker-node1
```

위 절차는 일반적인 유지보수 흐름이며, 실제 운영 환경에서는 PodDisruptionBudget, DaemonSet, 로컬 스토리지 및 워크로드의 복구 가능 여부 등을 사전에 확인해야 합니다.

## 9. Node와 Pod의 관계

Node와 Pod는 Kubernetes의 워크로드 실행 구조에서 밀접하게 연결되어 있습니다.

Node는 Pod가 실행될 컴퓨팅 자원을 제공하며, Pod는 Scheduler에 의해 적절한 Node에 배치됩니다.

하나의 Node에는 여러 Pod가 실행될 수 있으며, 하나의 Pod는 특정 시점에 하나의 Node에 할당되어 실행됩니다.

Pod가 Node에 배치되는 과정은 다음과 같습니다.

1. 사용자가 Pod를 생성하거나 Deployment 등의 컨트롤러가 Pod를 생성합니다.
2. API Server에 Pod 리소스가 등록됩니다.
3. Scheduler가 아직 Node에 할당되지 않은 Pod를 확인합니다.
4. Node의 리소스 여유 공간, 레이블, Taint 및 기타 스케줄링 조건을 확인합니다.
5. 적절한 Node를 선택하고 Pod의 `.spec.nodeName`을 설정합니다.
6. 해당 Node의 kubelet이 Pod 정보를 확인합니다.
7. kubelet이 Container Runtime에 컨테이너 실행을 요청합니다.
8. 컨테이너가 실행되면서 Pod가 동작합니다.

이 과정에서 Node는 단순한 실행 환경을 넘어 Scheduler의 배치 판단과 kubelet의 워크로드 실행이 연결되는 핵심 자원입니다.

## 10. Node 관련 주요 명령어

| 명령어 | 설명 |
|---|---|
| `kubectl get nodes` | 클러스터에 등록된 Node 목록을 확인합니다. |
| `kubectl get nodes -o wide` | Node의 IP 주소와 추가 정보를 확인합니다. |
| `kubectl describe node <node-name>` | 특정 Node의 상세 정보와 상태를 확인합니다. |
| `kubectl get node <node-name> -o yaml` | Node 리소스의 전체 YAML 정보를 확인합니다. |
| `kubectl label node <node-name> key=value` | Node에 Label을 추가합니다. |
| `kubectl taint nodes <node-name> key=value:NoSchedule` | Node에 Taint를 추가합니다. |
| `kubectl cordon <node-name>` | Node를 스케줄링 불가능한 상태로 변경합니다. |
| `kubectl drain <node-name>` | Node의 Pod를 안전하게 축출하여 비웁니다. |
| `kubectl uncordon <node-name>` | Node를 다시 스케줄링 가능한 상태로 변경합니다. |

## 11. 정리

Node는 Kubernetes 클러스터에서 Pod를 실행하기 위한 기본 컴퓨팅 자원입니다.

각 Node는 kubelet과 Container Runtime을 통해 Pod의 실행을 관리하며, Control Plane과 통신하면서 자신의 상태와 리소스 정보를 전달합니다.

Kubernetes Scheduler는 Node의 Capacity와 Allocatable, Label, Taint 및 스케줄링 가능 여부 등을 고려하여 Pod를 배치합니다.

또한 Node Controller는 Node의 Heartbeat와 Condition을 모니터링하여 장애를 감지하고, 필요한 경우 Taint 적용 및 Pod Eviction 등의 작업을 수행합니다.

운영 환경에서는 Cordon과 Drain을 활용하여 Node 유지보수를 안전하게 진행할 수 있습니다.

따라서 Node는 Kubernetes의 워크로드 실행 기반이자 스케줄링, 리소스 관리 및 장애 대응을 구성하는 핵심 리소스입니다.

## 12. 참고자료

- [Kubernetes 공식 문서 - Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)
- [Kubernetes 공식 문서 - Node Status](https://kubernetes.io/docs/reference/node/node-status/)
- [Kubernetes 공식 문서 - Node API Reference](https://kubernetes.io/docs/reference/kubernetes-api/core/node-v1/)
- [Kubernetes 공식 문서 - Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
- [Kubernetes 공식 문서 - kubectl drain](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_drain/)
- [Kubernetes 공식 문서 - Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Kubernetes 공식 문서 - Reserve Compute Resources for System Daemons](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/)