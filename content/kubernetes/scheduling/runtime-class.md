

# RuntimeClass

## 1. RuntimeClass란?

`RuntimeClass`는 Kubernetes에서 Pod를 실행할 때 사용할 컨테이너 런타임 구성(Runtime Configuration)을 선택하는 리소스입니다.

Kubernetes는 일반적으로 Node에 설정된 기본 컨테이너 런타임을 사용하여 Pod를 실행합니다. 하지만 RuntimeClass를 사용하면 Pod마다 서로 다른 런타임 구성이나 격리 방식을 선택할 수 있습니다.

예를 들어 다음과 같은 환경에서 활용할 수 있습니다.

- 일반적인 컨테이너 실행에는 기본 `runc` 런타임 사용
- 보안 격리가 필요한 Pod에는 `Kata Containers`와 같은 가상화 기반 런타임 사용
- 동일한 컨테이너 런타임에서 서로 다른 실행 옵션을 사용하는 구성 적용
- 특정 런타임을 지원하는 Node에만 Pod가 스케줄링되도록 제약 조건 설정

RuntimeClass는 컨테이너 런타임 자체를 설치하거나 구성하는 리소스가 아닙니다. 실제 런타임 구성은 각 Node의 Container Runtime Interface(CRI) 구현에 맞게 사전에 설정해야 합니다.

RuntimeClass는 해당 런타임 구성을 Kubernetes 리소스로 정의하고 Pod에서 참조할 수 있도록 연결하는 역할을 수행합니다.

RuntimeClass는 Kubernetes v1.20부터 Stable 상태로 제공됩니다.

## 2. RuntimeClass의 동작 원리

RuntimeClass는 Kubernetes API Server, Admission Controller, Scheduler, Kubelet 및 Container Runtime 간의 연동을 통해 동작합니다.

기본적인 동작 과정은 다음과 같습니다.

1. 관리자가 각 Node의 컨테이너 런타임에 사용할 Runtime Handler를 설정합니다.
2. Kubernetes에 해당 Handler를 참조하는 RuntimeClass 리소스를 생성합니다.
3. 사용자가 Pod의 `spec.runtimeClassName`에 RuntimeClass 이름을 지정합니다.
4. API Server의 RuntimeClass Admission Controller가 RuntimeClass에 정의된 Scheduling 설정과 Pod 설정을 병합하고, 필요한 경우 Pod Overhead를 반영합니다.
5. Scheduler가 Pod의 리소스 요청량과 스케줄링 조건을 고려하여 적절한 Node를 선택합니다.
6. 선택된 Node의 Kubelet이 RuntimeClass에 연결된 Runtime Handler를 사용하여 Pod를 실행합니다.

전체적인 구조는 다음과 같습니다.

```text
                 Kubernetes Cluster
                          |
                          |
                  +---------------+
                  |  API Server   |
                  +---------------+
                          |
                          |
                 RuntimeClass 조회
                          |
                          v
              +----------------------+
              | RuntimeClass         |
              |----------------------|
              | name: kata           |
              | handler: kata        |
              | scheduling           |
              | overhead             |
              +----------------------+
                          |
                          |
                 Pod Admission
                          |
                          v
              +----------------------+
              | Pod                  |
              |----------------------|
              | runtimeClassName     |
              | nodeSelector         |
              | tolerations          |
              | resource requests    |
              +----------------------+
                          |
                          |
                    kube-scheduler
                          |
                          v
              +----------------------+
              | Selected Node        |
              +----------------------+
                          |
                          |
                       Kubelet
                          |
                          v
              +----------------------+
              | Container Runtime    |
              | Runtime Handler      |
              +----------------------+
                          |
                          v
                    Running Pod
```

RuntimeClass는 Pod가 사용할 런타임 구성을 선택하는 역할을 수행하며, 실제 컨테이너 실행은 해당 Node의 Kubelet과 컨테이너 런타임이 담당합니다.

## 3. RuntimeClass 리소스 구조

RuntimeClass는 `node.k8s.io/v1` API 그룹을 사용하며, Namespace에 종속되지 않는 Cluster-scoped 리소스입니다.

기본적인 리소스 구조는 다음과 같습니다.

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata
```

주요 필드는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `metadata.name` | RuntimeClass의 이름 |
| `handler` | CRI에서 사용할 런타임 Handler 이름 |
| `scheduling` | RuntimeClass를 지원하는 Node를 선택하기 위한 스케줄링 조건 |
| `overhead` | 해당 RuntimeClass를 사용하는 Pod의 추가 리소스 오버헤드 |

`metadata.name`은 Pod의 `runtimeClassName`에서 참조하는 이름이며, `handler`는 Node에 설정된 실제 런타임 구성의 이름입니다.

두 값은 반드시 동일할 필요는 없습니다.

예를 들어 RuntimeClass 이름은 `kata-isolated`로 정의하고 실제 Runtime Handler는 `kata`로 지정할 수 있습니다.

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-isolated
handler: kata
```

이 경우 Pod에서는 `kata-isolated`를 지정하지만, 실제 런타임 실행에는 `kata` Handler가 사용됩니다.

## 4. Runtime Handler와 CRI 설정

RuntimeClass를 사용하려면 먼저 각 Node의 컨테이너 런타임에 Runtime Handler를 설정해야 합니다.

Runtime Handler는 CRI 구현에 따라 설정 방식이 다릅니다.

대표적으로 다음과 같은 구성이 있습니다.

| Container Runtime | 설정 위치 |
|---|---|
| containerd | `/etc/containerd/config.toml` |
| CRI-O | `/etc/crio/crio.conf` |

### 4.1. containerd

containerd에서는 CRI Plugin의 Runtime 설정에 Handler를 정의합니다.

예를 들어 `kata`라는 Runtime Handler를 구성하는 형태는 다음과 같습니다.

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
  runtime_type = "io.containerd.kata.v2"
```

위 설정은 개념적인 예시이며, 실제 `runtime_type` 및 추가 설정은 설치한 Kata Containers 버전과 containerd 구성에 따라 달라질 수 있습니다.

### 4.2. CRI-O

CRI-O에서는 Runtime 설정에 Handler와 실행 파일 경로 등을 정의합니다.

```toml
[crio.runtime.runtimes.kata]
  runtime_path = "/usr/bin/containerd-shim-kata-v2"
```

실제 실행 파일 경로와 설정 항목은 설치한 CRI-O 및 런타임 버전에 맞게 구성해야 합니다.

### 4.3. Runtime Handler 구성 시 주의사항

RuntimeClass는 기본적으로 클러스터 내 모든 Node에서 동일한 Runtime Handler가 사용 가능하다고 가정합니다.

따라서 특정 RuntimeClass를 모든 Node에서 사용할 계획이라면 해당 Handler가 모든 Node에 동일한 이름과 호환되는 설정으로 등록되어 있어야 합니다.

일부 Node에서만 특정 런타임을 지원하는 이종(Heterogeneous) 클러스터라면 RuntimeClass의 `scheduling` 필드를 사용하여 지원 Node를 제한해야 합니다.

또한 RuntimeClass는 실제 런타임 구성을 검증하거나 자동으로 설치하지 않습니다. Handler가 Node에 설정되어 있지 않으면 해당 RuntimeClass를 사용하는 Pod가 정상적으로 실행되지 않을 수 있습니다.

## 5. Pod에서 RuntimeClass 사용하기

RuntimeClass를 생성한 이후에는 Pod의 `spec.runtimeClassName`에 RuntimeClass 이름을 지정하여 사용할 수 있습니다.

다음은 `kata` RuntimeClass를 사용하는 Pod 예제입니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: isolated-pod
spec:
  runtimeClassName: kata
  containers:
    - name: app
      image: nginx
```

이 Pod는 기본 Runtime Handler가 아닌 `kata` Runtime Handler를 사용하여 실행됩니다.

Deployment에서도 동일한 방식으로 설정할 수 있습니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: isolated-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: isolated-app
  template:
    metadata:
      labels:
        app: isolated-app
    spec:
      runtimeClassName: kata
      containers:
        - name: app
          image: nginx
```

Deployment의 Pod Template에 `runtimeClassName`을 지정하면 해당 Deployment가 생성하는 Pod에도 동일한 RuntimeClass가 적용됩니다.

### 5.1. RuntimeClass를 지정하지 않은 경우

Pod에 `runtimeClassName`을 지정하지 않으면 Node에 설정된 기본 Runtime Handler가 사용됩니다.

이는 RuntimeClass 기능을 사용하지 않았을 때의 기본적인 컨테이너 실행 방식과 동일합니다.

### 5.2. 존재하지 않는 RuntimeClass를 지정한 경우

Pod에서 존재하지 않는 RuntimeClass 이름을 지정하면 Pod Admission 과정에서 해당 참조를 검증할 수 없어 Pod 생성 요청이 거부됩니다.

반면 RuntimeClass 리소스는 존재하지만 실제 Node의 CRI에 해당 Handler가 설정되어 있지 않거나 실행할 수 없는 경우에는 Pod가 정상적으로 실행되지 못하고 `Failed` 상태에 도달할 수 있습니다.

따라서 RuntimeClass 리소스의 존재 여부뿐 아니라 각 Node의 Runtime Handler 설정도 확인해야 합니다.

## 6. RuntimeClass Scheduling

RuntimeClass의 `scheduling` 필드는 특정 RuntimeClass를 지원하는 Node에만 Pod가 배치되도록 스케줄링 조건을 정의합니다.

이 기능은 클러스터 내 Node마다 지원하는 Runtime Handler가 다른 이종 클러스터 환경에서 유용합니다.

예를 들어 다음과 같은 Node 구성이 있다고 가정하겠습니다.

| Node | 지원 Runtime | Label |
|---|---|---|
| node-1 | runc | `runtime=standard` |
| node-2 | runc, kata | `runtime=kata` |
| node-3 | runc, kata | `runtime=kata` |

이 경우 `kata` RuntimeClass를 사용하는 Pod가 `node-1`에 배치되면 해당 Node에서 Runtime Handler를 찾지 못해 실행에 실패할 수 있습니다.

이를 방지하기 위해 RuntimeClass의 `scheduling.nodeSelector`를 사용하여 `node-2`, `node-3`만 선택하도록 구성할 수 있습니다.

### 6.1. nodeSelector

`nodeSelector`는 RuntimeClass를 지원하는 Node에 공통으로 설정된 Label을 기준으로 스케줄링 대상을 제한합니다.

다음은 `kata` RuntimeClass의 예제입니다.

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata
scheduling:
  nodeSelector:
    runtime: kata
```

이 설정은 `runtime: kata` Label이 존재하는 Node에서만 해당 RuntimeClass를 사용하는 Pod가 실행되도록 제한합니다.

Pod 자체에도 `nodeSelector`를 설정할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kata-pod
spec:
  runtimeClassName: kata
  nodeSelector:
    disktype: ssd
  containers:
    - name: app
      image: nginx
```

이 경우 RuntimeClass와 Pod의 `nodeSelector`가 병합되어 두 조건을 모두 만족하는 Node가 선택됩니다.

즉, 다음 조건을 모두 만족해야 합니다.

- `runtime: kata`
- `disktype: ssd`

RuntimeClass의 `nodeSelector`와 Pod의 `nodeSelector`는 교집합 방식으로 적용됩니다.

동일한 Label Key에 서로 다른 Value를 지정하는 등 조건이 충돌하면 Pod Admission 단계에서 요청이 거부됩니다.

### 6.2. tolerations

RuntimeClass의 `scheduling.tolerations`는 해당 RuntimeClass를 지원하는 Node에 설정된 Taint를 허용하기 위한 설정입니다.

예를 들어 Kata Runtime을 지원하는 Node에 다음과 같은 Taint가 설정되어 있다고 가정하겠습니다.

```bash
kubectl taint nodes node-2 runtime=kata:NoSchedule
kubectl taint nodes node-3 runtime=kata:NoSchedule
```

일반 Pod는 해당 Node의 Taint를 허용하지 않으므로 스케줄링되지 않습니다.

RuntimeClass에 다음과 같이 Toleration을 설정하면 해당 RuntimeClass를 사용하는 Pod가 Taint를 허용할 수 있습니다.

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata
scheduling:
  nodeSelector:
    runtime: kata
  tolerations:
    - key: runtime
      operator: Equal
      value: kata
      effect: NoSchedule
```

RuntimeClass의 Toleration은 Pod의 Toleration에 추가되는 방식으로 적용됩니다.

따라서 Pod 자체에 정의된 Toleration과 RuntimeClass에 정의된 Toleration은 합집합 형태로 동작합니다.

이때 Toleration은 Taint를 허용하는 조건일 뿐, 해당 Node에 반드시 Pod를 배치하도록 강제하지는 않습니다.

Node 선택은 `nodeSelector`, Node Affinity 및 기타 스케줄링 조건에 의해 결정됩니다.

### 6.3. Scheduling 설정의 적용 방식

RuntimeClass의 Scheduling 설정은 다음과 같이 동작합니다.

| 설정 | 적용 방식 | 목적 |
|---|---|---|
| `nodeSelector` | Pod의 Node Selector와 교집합 | 런타임을 지원하는 Node로 배치 제한 |
| `tolerations` | Pod의 Toleration에 추가 | 런타임 전용 Node의 Taint 허용 |

RuntimeClass의 Scheduling 설정은 Node의 Label이나 Taint를 자동으로 생성하지 않습니다.

따라서 관리자가 먼저 Node에 적절한 Label과 Taint를 설정해야 합니다.

## 7. Pod Overhead

Pod Overhead는 특정 RuntimeClass를 사용하는 Pod를 실행할 때 추가로 소비되는 CPU 및 Memory 리소스를 정의하는 기능입니다.

일반적인 컨테이너 실행 환경에서는 컨테이너의 Request를 기준으로 Pod의 리소스 요구량을 계산할 수 있습니다.

그러나 가상화 기반 런타임처럼 컨테이너 실행을 위해 별도의 VM이나 추가 프로세스가 필요한 경우에는 컨테이너에 선언된 Request 외에도 추가 리소스가 소비될 수 있습니다.

예를 들어 Kata Containers와 같은 런타임은 Pod별로 VM과 Guest OS를 실행하는 데 추가 Memory와 CPU를 사용할 수 있습니다.

이러한 추가 리소스를 Kubernetes가 인식하지 못하면 Scheduler가 실제보다 적은 리소스를 기준으로 Pod를 배치할 수 있습니다.

Pod Overhead는 이러한 문제를 해결하기 위해 RuntimeClass에 런타임별 추가 리소스 사용량을 선언하는 기능입니다.

Pod Overhead는 Kubernetes v1.24부터 Stable 상태로 제공됩니다.

### 7.1. overhead 설정

RuntimeClass의 `overhead.podFixed`를 사용하여 Pod 실행에 필요한 추가 CPU와 Memory를 정의할 수 있습니다.

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata
overhead:
  podFixed:
    cpu: 250m
    memory: 120Mi
```

위 설정은 `kata` RuntimeClass를 사용하는 Pod마다 다음과 같은 추가 리소스가 필요하다는 것을 의미합니다.

- CPU: 250m
- Memory: 120Mi

예를 들어 Pod 내부 Container들의 리소스 Request 합계가 다음과 같다고 가정하겠습니다.

| 항목 | CPU Request | Memory Request |
|---|---:|---:|
| app Container | 500m | 512Mi |
| sidecar Container | 200m | 256Mi |
| Pod Overhead | 250m | 120Mi |
| 최종 Pod 리소스 요구량 | 950m | 888Mi |

이 경우 Kubernetes는 Pod의 리소스 요구량을 계산할 때 Container들의 Request 합계에 Pod Overhead를 추가로 반영합니다.

### 7.2. Pod Overhead가 반영되는 영역

Pod Overhead는 다음과 같은 Kubernetes 동작에 반영됩니다.

- **Scheduler:** Node에 Pod를 배치할 때 추가 리소스 요구량을 고려합니다.
- **ResourceQuota:** Namespace의 리소스 사용량을 계산할 때 Pod Overhead를 반영합니다.
- **Pod cgroup:** Pod의 리소스 제한을 구성할 때 Overhead를 고려합니다.

따라서 Pod Overhead를 정의하면 가상화 기반 런타임이나 추가 실행 환경이 소비하는 리소스를 Kubernetes의 리소스 관리 체계에 포함할 수 있습니다.

### 7.3. Pod Overhead 적용 시 주의사항

Pod Overhead는 RuntimeClass를 사용하는 Pod에 적용되는 추가 리소스 요구량입니다.

실제 런타임이 소비하는 리소스보다 지나치게 작게 설정하면 Node의 리소스 사용량을 과소평가할 수 있으며, 반대로 지나치게 크게 설정하면 Node의 가용 리소스를 불필요하게 제한할 수 있습니다.

따라서 실제 런타임의 추가 CPU 및 Memory 소비량을 측정한 뒤 적절한 값을 설정해야 합니다.

또한 Pod Overhead는 Container의 Request나 Limit을 직접 수정하는 기능이 아닙니다. Pod 전체의 리소스 계산에 추가되는 별도의 값입니다.

## 8. RuntimeClass와 다른 Kubernetes 리소스의 관계

### 8.1. RuntimeClass와 Container Runtime

Container Runtime은 실제 컨테이너를 실행하는 소프트웨어이며, RuntimeClass는 Kubernetes에서 사용할 런타임 구성을 선택하는 리소스입니다.

| 구분 | Container Runtime | RuntimeClass |
|---|---|---|
| 역할 | 컨테이너 실행 | 런타임 구성 선택 |
| 설정 위치 | Node | Kubernetes API |
| 주요 구성 | Runtime Handler, 실행 옵션 | `handler`, `scheduling`, `overhead` |
| 담당 구성요소 | containerd, CRI-O 등 | Kubernetes API Server 및 Kubelet |

RuntimeClass는 Container Runtime을 대체하지 않으며, 실제 실행은 Node에 설치된 Container Runtime이 담당합니다.

### 8.2. RuntimeClass와 Scheduler

RuntimeClass는 Scheduler가 고려해야 하는 추가적인 스케줄링 조건을 제공할 수 있습니다.

특히 `scheduling.nodeSelector`를 통해 특정 런타임을 지원하는 Node만 선택하도록 제한할 수 있습니다.

또한 `overhead`를 사용하면 런타임 실행에 필요한 추가 CPU와 Memory가 Scheduler의 리소스 배치 계산에 반영됩니다.

다만 Scheduler가 Runtime Handler를 직접 실행하거나 런타임 설치 여부를 확인하는 것은 아닙니다.

### 8.3. RuntimeClass와 NodeSelector / Taint / Toleration

RuntimeClass의 Scheduling 설정은 Pod의 기존 스케줄링 조건과 결합됩니다.

- `nodeSelector`: 런타임을 지원하는 Node로 배치 대상을 제한합니다.
- `tolerations`: 런타임 전용 Node에 설정된 Taint를 허용합니다.

이를 통해 Pod 작성자가 개별적으로 런타임 지원 Node를 지정하지 않아도 RuntimeClass를 참조하는 것만으로 기본적인 런타임별 스케줄링 조건을 적용할 수 있습니다.

### 8.4. RuntimeClass와 ResourceQuota

Pod Overhead가 설정된 RuntimeClass를 사용하는 Pod는 ResourceQuota의 리소스 사용량 계산에도 Overhead가 반영됩니다.

따라서 Namespace의 CPU 및 Memory Request 총량을 제한하는 ResourceQuota가 존재한다면 Pod Overhead를 포함한 최종 리소스 요구량이 Quota를 초과하지 않는지 확인해야 합니다.

## 9. RuntimeClass 확인 및 관리 명령어

### 9.1. RuntimeClass 목록 조회

```bash
kubectl get runtimeclass
```

클러스터에 등록된 RuntimeClass 목록을 조회합니다.

### 9.2. RuntimeClass 상세 정보 조회

```bash
kubectl describe runtimeclass kata
```

RuntimeClass의 Handler, Scheduling 및 Overhead 설정을 확인합니다.

### 9.3. RuntimeClass YAML 조회

```bash
kubectl get runtimeclass kata -o yaml
```

RuntimeClass의 전체 설정을 YAML 형식으로 확인합니다.

### 9.4. RuntimeClass 생성 및 수정

```bash
kubectl apply -f runtime-class.yaml
```

YAML 파일에 정의된 RuntimeClass를 생성하거나 수정합니다.

### 9.5. RuntimeClass 삭제

```bash
kubectl delete runtimeclass kata
```

지정한 RuntimeClass를 삭제합니다.

RuntimeClass는 Cluster-scoped 리소스이므로 Namespace를 지정하지 않습니다.

### 9.6. Pod의 RuntimeClass 확인

```bash
kubectl get pod isolated-pod -o jsonpath='{.spec.runtimeClassName}'
```

Pod에 지정된 RuntimeClass 이름을 확인합니다.

### 9.7. Pod 이벤트 확인

```bash
kubectl describe pod isolated-pod
```

RuntimeClass를 찾을 수 없거나 Runtime Handler 실행에 문제가 발생한 경우 Pod 이벤트를 확인하여 원인을 파악할 수 있습니다.

## 10. 정리

`RuntimeClass`는 Kubernetes에서 Pod가 사용할 컨테이너 런타임 구성을 선택하는 리소스입니다.

주요 기능은 다음과 같습니다.

- Pod별 컨테이너 런타임 구성 선택
- 보안 격리 및 가상화 기반 런타임 활용
- Runtime Handler를 지원하는 Node로 스케줄링 제한
- RuntimeClass별 Toleration 적용
- Pod Overhead를 통한 추가 CPU 및 Memory 사용량 반영

RuntimeClass를 사용하려면 먼저 Node의 CRI에 Runtime Handler를 구성해야 합니다. 이후 Kubernetes에 RuntimeClass를 생성하고 Pod의 `runtimeClassName`으로 참조하면 해당 런타임 구성을 사용할 수 있습니다.

특히 `scheduling`은 런타임을 지원하는 Node에 Pod를 배치하기 위한 기능이며, `overhead`는 런타임 실행에 필요한 추가 리소스를 Kubernetes의 리소스 관리 체계에 반영하기 위한 기능입니다.

따라서 RuntimeClass는 단순히 컨테이너 실행 방식을 선택하는 것뿐 아니라, 서로 다른 런타임 환경이 공존하는 클러스터에서 스케줄링과 리소스 관리를 일관되게 수행할 수 있도록 지원합니다.

## 참고자료

- [Kubernetes 공식 문서 - RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/)
- [Kubernetes 공식 문서 - Pod Overhead](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-overhead/)
- [Kubernetes API Reference - RuntimeClass v1](https://kubernetes.io/docs/reference/kubernetes-api/node/runtime-class-v1/)
- [Kubernetes 공식 문서 - Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes Enhancement Proposal - RuntimeClass](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/585-runtime-class)

