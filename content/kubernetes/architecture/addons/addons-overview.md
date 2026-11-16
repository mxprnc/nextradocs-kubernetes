# Kubernetes Addons

## 1. Addons란?

Kubernetes Addons는 Kubernetes 클러스터에 기본적으로 제공되지 않는 기능을 추가하거나 기존 기능을 확장하기 위해 배포하는 구성 요소입니다.

Kubernetes는 Control Plane과 Node의 핵심 구성 요소를 통해 컨테이너 워크로드의 스케줄링, 실행, 상태 관리 등의 기능을 제공합니다. 그러나 실제 클러스터를 운영하려면 이러한 핵심 기능 외에도 DNS 기반 서비스 디스커버리, 리소스 모니터링, 로깅, 시각화 및 네트워크 관리와 같은 부가 기능이 필요합니다.

Addons는 이러한 기능을 제공하여 Kubernetes 클러스터를 실제 운영 환경에 적합한 형태로 확장합니다.

대표적인 Addons는 다음과 같습니다.

- **CoreDNS**: 클러스터 내부의 DNS 조회와 Service Discovery를 제공합니다.
- **Metrics Server**: Node와 Pod의 CPU 및 메모리 사용량을 수집하고 Metrics API를 통해 제공합니다.
- **Kubernetes Dashboard**: 웹 인터페이스를 통해 Kubernetes 리소스를 조회하고 관리할 수 있도록 지원합니다.

Addons는 Kubernetes의 핵심 구성 요소와 구분되는 개념입니다. 일반적으로 별도의 소프트웨어로 배포되며, Kubernetes API와 기존 리소스를 활용하여 클러스터 수준의 기능을 구현합니다.

## 2. Addons의 주요 특징

### 2.1. 클러스터 기능 확장

Addons는 Kubernetes의 기본 기능을 보완하거나 확장합니다.

예를 들어 Kubernetes는 Service 리소스를 통해 애플리케이션에 안정적인 네트워크 접근 지점을 제공하지만, 클러스터 내부에서 Service 이름을 DNS로 조회할 수 있도록 하려면 DNS 서비스가 필요합니다.

CoreDNS는 이러한 역할을 수행하는 대표적인 Addon입니다.

Metrics Server는 Kubernetes API Server가 기본적으로 제공하지 않는 Node 및 Pod의 CPU와 메모리 사용량 정보를 수집하여 Metrics API로 제공합니다. 이를 통해 HorizontalPodAutoscaler와 같은 자동 확장 기능이 리소스 사용량을 기반으로 동작할 수 있습니다.

이처럼 Addons는 Kubernetes의 기본 리소스와 API를 활용하여 클러스터 운영에 필요한 기능을 추가합니다.

### 2.2. Kubernetes 리소스를 활용한 배포

Addons는 일반적으로 Kubernetes 리소스를 사용하여 배포하고 관리합니다.

대표적으로 다음과 같은 리소스가 사용됩니다.

| 리소스 | 주요 역할 |
| --- | --- |
| Deployment | Addon 애플리케이션의 Replica 수와 배포 상태를 관리합니다. |
| DaemonSet | 각 Node 또는 지정된 Node에 Addon Pod를 배포합니다. |
| Service | Addon Pod에 안정적인 네트워크 접근 지점을 제공합니다. |
| ConfigMap | Addon 애플리케이션의 설정 정보를 관리합니다. |
| ServiceAccount | Addon이 Kubernetes API에 접근할 때 사용하는 서비스 계정을 제공합니다. |
| ClusterRole / ClusterRoleBinding | Addon에 클러스터 범위의 API 접근 권한을 부여합니다. |

예를 들어 CoreDNS는 일반적으로 Deployment와 Service를 사용하여 배포됩니다. Metrics Server 역시 Deployment로 실행되며, Kubernetes API에 접근하기 위한 ServiceAccount와 RBAC 권한을 함께 사용합니다.

다만 모든 Addon이 동일한 리소스 구성으로 배포되는 것은 아닙니다. Addon의 목적과 동작 방식에 따라 Deployment, DaemonSet, Custom Resource Definition(CRD) 등 다양한 리소스가 사용될 수 있습니다.

### 2.3. kube-system Namespace

클러스터 수준의 기능을 제공하는 Addon은 일반적으로 `kube-system` Namespace에 배포됩니다.

`kube-system`은 Kubernetes 시스템 구성 요소와 클러스터 운영에 필요한 리소스를 관리하기 위한 Namespace입니다.

예를 들어 일반적인 클러스터에서는 다음과 같은 리소스를 확인할 수 있습니다.

```bash
kubectl get pods -n kube-system
```

출력 예시:

```text
NAME                               READY   STATUS    RESTARTS
coredns-7b9c7d6f8f-abcde           1/1     Running   0
coredns-7b9c7d6f8f-fghij           1/1     Running   0
metrics-server-6f7b8c9d5e-klmno    1/1     Running   0
```

위 출력은 이해를 돕기 위한 예시이며, 실제 배포 환경에 따라 Pod 이름과 실행 중인 Addon은 달라질 수 있습니다.

모든 Addon이 반드시 `kube-system`에 배포되어야 하는 것은 아닙니다. Addon의 종류와 설치 방식에 따라 별도의 Namespace를 사용할 수 있으며, 일부 구성 요소는 Node에서 직접 실행되거나 Kubernetes 리소스 이외의 방식으로 설치될 수도 있습니다.

## 3. Addons의 배포 및 관리

Addons는 Kubernetes 클러스터에 추가적인 기능을 제공하는 독립적인 소프트웨어인 경우가 많으므로, 클러스터를 구성하는 방식과 운영 환경에 따라 다양한 방법으로 설치할 수 있습니다.

### 3.1. Manifest 기반 배포

Kubernetes Manifest를 사용하여 Addon에 필요한 리소스를 직접 생성하는 방식입니다.

Deployment, DaemonSet, Service, ConfigMap, RBAC 리소스 등을 YAML 파일로 정의하고 Kubernetes API Server에 적용합니다.

```bash
kubectl apply -f addon.yaml
```

이 방식은 Addon의 구성 요소와 배포 설정을 명시적으로 관리할 수 있다는 장점이 있습니다.

다만 Addon이 여러 Kubernetes 리소스로 구성되어 있거나 버전별 설정이 복잡한 경우에는 Manifest를 직접 관리하는 데 추가적인 작업이 필요할 수 있습니다.

### 3.2. Helm 기반 배포

Helm은 Kubernetes 애플리케이션을 패키징하고 배포하기 위한 패키지 관리자입니다.

Addon에서 제공하는 Helm Chart를 사용하면 여러 Kubernetes 리소스와 설정을 하나의 배포 단위로 관리할 수 있습니다.

```bash
helm install metrics-server metrics-server/metrics-server \
  --namespace kube-system
```

Helm은 다음과 같은 기능을 제공합니다.

- 여러 Kubernetes 리소스의 통합 배포
- `values.yaml`을 통한 설정 관리
- 릴리스 버전 및 배포 이력 관리
- Upgrade 및 Rollback 지원

실제 설치 시에는 Addon에서 제공하는 Chart 저장소와 설치 문서에 따라 Repository 등록 및 Chart 버전 지정 등의 추가 설정이 필요할 수 있습니다.

### 3.3. 클라우드 및 Kubernetes 배포 도구를 통한 관리

Managed Kubernetes 환경에서는 클라우드 제공자가 Addon의 설치와 업그레이드를 지원하기도 합니다.

예를 들어 Amazon EKS, Google Kubernetes Engine(GKE), Azure Kubernetes Service(AKS) 등은 클러스터 운영에 필요한 일부 Addon을 관리형 기능으로 제공합니다.

이러한 환경에서는 클라우드 제공자의 관리 기능을 사용하거나 별도의 Helm Chart 및 Manifest를 통해 Addon을 배포할 수 있습니다.

배포 방식에 따라 버전 관리, 설정 변경, 업그레이드 및 장애 대응의 책임 범위가 달라질 수 있으므로 클러스터 운영 정책에 맞는 관리 방식을 선택해야 합니다.

## 4. 주요 Addons

Kubernetes는 클러스터의 목적과 운영 환경에 따라 다양한 Addons를 사용할 수 있습니다.

대표적인 Addons는 다음과 같습니다.

### 4.1. CoreDNS

CoreDNS는 Kubernetes 클러스터 내부의 DNS 서비스를 제공하는 구성 요소입니다.

Pod는 CoreDNS를 통해 Service 이름을 DNS로 조회할 수 있으며, 이를 기반으로 클러스터 내부의 서비스 디스커버리를 수행합니다.

예를 들어 동일한 Namespace에 있는 애플리케이션은 Service 이름을 사용하여 다른 애플리케이션에 접근할 수 있습니다.

```text
my-service
```

다른 Namespace에 있는 Service는 다음과 같은 DNS 이름으로 조회할 수 있습니다.

```text
my-service.default
```

CoreDNS는 일반적으로 `kube-system` Namespace에 Deployment로 배포되며, `kube-dns`라는 이름의 Service를 통해 클러스터 내부 DNS 요청을 전달받습니다.

자세한 내용은 [CoreDNS](./coredns.md) 문서를 참고하시기 바랍니다.

### 4.2. Metrics Server

Metrics Server는 Kubernetes 클러스터의 Node와 Pod에서 CPU 및 메모리 사용량을 수집하고 이를 Metrics API로 제공하는 구성 요소입니다.

Metrics Server는 각 Node의 kubelet으로부터 리소스 사용량 정보를 수집하고, Kubernetes API Aggregation Layer를 통해 `metrics.k8s.io` API를 제공합니다.

이를 활용하는 대표적인 기능은 다음과 같습니다.

- `kubectl top`: Node와 Pod의 CPU 및 메모리 사용량을 조회합니다.
- HorizontalPodAutoscaler(HPA): 리소스 사용량을 기준으로 Pod의 Replica 수를 자동으로 조정합니다.

Metrics Server는 장기적인 시계열 모니터링을 위한 시스템이 아닙니다. 클러스터의 현재 리소스 사용량을 제공하는 데 목적이 있으며, 장기적인 메트릭 수집과 분석에는 Prometheus와 같은 별도의 모니터링 시스템을 사용할 수 있습니다.

자세한 내용은 [Metrics Server](./metrics-server.md) 문서를 참고하시기 바랍니다.

### 4.3. Kubernetes Dashboard

Kubernetes Dashboard는 웹 브라우저를 통해 Kubernetes 클러스터의 리소스를 조회하고 관리할 수 있도록 지원하는 웹 기반 사용자 인터페이스입니다.

사용자는 Dashboard를 통해 다음과 같은 작업을 수행할 수 있습니다.

- Node, Namespace, Pod, Deployment, Service 등의 리소스 조회
- 리소스의 상태 및 이벤트 확인
- Kubernetes 리소스 생성 및 수정
- 컨테이너 로그 조회
- RBAC 권한에 따른 클러스터 리소스 접근

Dashboard는 Kubernetes API Server를 통해 클러스터 리소스에 접근합니다. 따라서 Dashboard를 사용하는 계정에는 수행하려는 작업에 적합한 Kubernetes 인증 및 RBAC 권한이 필요합니다.

참고로 Kubernetes Dashboard 프로젝트는 현재 유지보수가 종료된 상태이므로, 신규 환경에서는 대체 UI인 Headlamp 등의 사용을 검토할 수 있습니다.

자세한 내용은 [Kubernetes Dashboard](./dashboard.md) 문서를 참고하시기 바랍니다.

## 5. Addons와 Kubernetes Architecture의 관계

Kubernetes Architecture는 Control Plane과 Node의 핵심 구성 요소를 중심으로 구성됩니다.

Addons는 이러한 핵심 구성 요소를 대체하는 것이 아니라 Kubernetes API와 기존 클러스터 리소스를 활용하여 추가적인 클러스터 기능을 제공합니다.

대표적인 구성 요소 간의 관계는 다음과 같습니다.

| Addon | 연관된 Kubernetes 구성 요소 | 관계 |
| --- | --- | --- |
| CoreDNS | kube-apiserver, Service, EndpointSlice, kubelet | Kubernetes API를 통해 Service 및 Endpoint 정보를 확인하고 DNS 응답을 제공합니다. |
| Metrics Server | kube-apiserver, kubelet, API Aggregation Layer | kubelet에서 리소스 메트릭을 수집하고 Metrics API를 제공합니다. |
| Kubernetes Dashboard | kube-apiserver, RBAC | Kubernetes API를 통해 리소스를 조회하고 권한이 허용된 작업을 수행합니다. |

### 5.1. Control Plane과의 관계

Addons는 일반적으로 Kubernetes API Server를 통해 클러스터의 상태를 조회하거나 리소스를 생성하고 변경합니다.

예를 들어 CoreDNS는 Kubernetes API를 통해 Service와 EndpointSlice 정보를 확인하여 DNS 레코드를 구성합니다.

Metrics Server는 Kubernetes API Aggregation Layer를 통해 Metrics API를 제공하며, HPA Controller와 같은 Kubernetes 구성 요소가 해당 API를 사용하여 리소스 사용량을 확인할 수 있도록 합니다.

Kubernetes Dashboard 역시 API Server를 통해 클러스터 리소스에 접근하며, RBAC에 정의된 권한 범위 내에서 작업을 수행합니다.

이처럼 Addons는 Control Plane과 연동하여 클러스터 수준의 기능을 제공합니다.

### 5.2. Node와의 관계

일부 Addons는 Node에서 실행되는 구성 요소와 직접 연동합니다.

예를 들어 Metrics Server는 각 Node의 kubelet에 접근하여 해당 Node에서 실행 중인 Pod와 Node 자체의 리소스 사용량을 수집합니다.

또한 네트워크 및 스토리지 관련 확장 구성 요소는 Node에 배포되어 네트워크 인터페이스 구성이나 볼륨 연결 및 마운트와 같은 작업을 수행하기도 합니다.

Addon의 실행 위치와 배포 방식은 제공하는 기능에 따라 달라집니다.

### 5.3. CNI, CSI 및 Device Plugin과의 관계

Kubernetes 클러스터를 운영할 때는 CoreDNS나 Metrics Server와 같은 Addon 외에도 CNI, CSI, Device Plugin과 같은 확장 메커니즘을 함께 이해해야 합니다.

이들은 Kubernetes가 외부 인프라 및 런타임 기능과 연동할 수 있도록 정의한 인터페이스 또는 플러그인 구조입니다.

| 구분 | 역할 | 주요 연동 대상 |
| --- | --- | --- |
| CNI (Container Network Interface) | Pod 네트워크 구성 및 연결을 담당합니다. | Container Runtime, 네트워크 플러그인 |
| CSI (Container Storage Interface) | 외부 스토리지 시스템과 연동하여 볼륨 생성, 연결 및 마운트를 지원합니다. | kubelet, 스토리지 플러그인 |
| Device Plugin | GPU 등의 특수 하드웨어 리소스를 Kubernetes에 노출하고 할당할 수 있도록 지원합니다. | kubelet, 컨테이너 런타임 |

CNI, CSI, Device Plugin은 각각 Kubernetes와 외부 기능을 연결하는 표준화된 인터페이스를 기반으로 동작합니다. 실제 구현체는 Kubernetes 리소스와 함께 배포되는 경우도 있지만, 모든 구현체를 일반적인 애플리케이션 형태의 Addon과 동일하게 취급해서는 안 됩니다.

특히 네트워크와 스토리지처럼 클러스터의 기본 동작에 직접 영향을 미치는 구성 요소는 설치 방식과 설정에 따라 클러스터 전체에 영향을 줄 수 있으므로 주의가 필요합니다.

## 6. 참고자료

- [Kubernetes Documentation - Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Kubernetes Documentation - Installing Addons](https://kubernetes.io/docs/concepts/cluster-administration/addons/)
- [Kubernetes Documentation - Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Kubernetes Documentation - Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
- [Kubernetes Documentation - Container Storage Interface (CSI)](https://kubernetes.io/docs/concepts/storage/volumes/#csi)
- [Kubernetes Documentation - Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)