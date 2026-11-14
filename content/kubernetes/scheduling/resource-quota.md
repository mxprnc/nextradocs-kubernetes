# ResourceQuota

## 1. ResourceQuota란?

ResourceQuota는 Kubernetes에서 Namespace별로 사용할 수 있는 리소스의 총량을 제한하는 리소스입니다.

하나의 Kubernetes 클러스터를 여러 팀이나 애플리케이션이 공유하는 환경에서는 특정 Namespace가 CPU, Memory 또는 Pod 등의 리소스를 과도하게 사용하는 상황이 발생할 수 있습니다.

ResourceQuota는 이러한 문제를 방지하기 위해 Namespace별 리소스 사용량과 리소스 객체의 개수를 제한합니다.

예를 들어 하나의 Namespace에서 사용할 수 있는 CPU Request 총량을 10 Core, Memory Request 총량을 20Gi로 제한할 수 있습니다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: application-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
```

위 설정에서는 `production` Namespace에 속한 Pod들이 요청하는 CPU와 Memory의 총량 및 설정된 Limits의 총량을 제한합니다.

ResourceQuota의 주요 특징은 다음과 같습니다.

- **Namespace별 리소스 제한:** Namespace에서 사용할 수 있는 리소스의 총량을 제한합니다.
- **Compute Resource 관리:** CPU와 Memory의 Requests 및 Limits 총량을 제한할 수 있습니다.
- **Object Count 관리:** Pod, Service, ConfigMap, Secret 등의 객체 개수를 제한할 수 있습니다.
- **스토리지 사용량 관리:** PVC 및 StorageClass별 스토리지 사용량을 제한할 수 있습니다.
- **멀티테넌시 지원:** 여러 팀이나 애플리케이션이 하나의 클러스터를 공유하는 환경에서 리소스를 분리하여 관리할 수 있습니다.

ResourceQuota는 `core/v1` API 그룹에 속하는 리소스이며, 특정 Namespace에 종속됩니다.

## 2. ResourceQuota의 동작 원리

ResourceQuota는 Namespace별로 설정되며, 해당 Namespace에서 생성되는 리소스의 사용량을 추적합니다.

Kubernetes의 ResourceQuota Admission Controller는 리소스 생성 또는 변경 요청이 들어오면 Namespace에 설정된 ResourceQuota를 확인합니다.

요청된 리소스를 반영했을 때 제한된 사용량을 초과하는 경우 해당 요청을 거부합니다.

전체적인 동작 과정은 다음과 같습니다.

```text
┌──────────────────────────────┐
│       ResourceQuota          │
│                              │
│  Namespace: production      │
│  CPU Requests: 10 Core      │
│  Memory Requests: 20Gi      │
│  Pod Count: 20              │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       Resource Creation      │
│                              │
│  Pod / Service / PVC         │
│  Deployment / ConfigMap      │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│   ResourceQuota Admission    │
│        Controller            │
│                              │
│  Check Current Usage         │
│  Calculate Requested Usage  │
└──────────────┬───────────────┘
               │
               ▼
       ┌───────────────┐
       │ Quota Limit   │
       │ Exceeded?     │
       └───────┬───────┘
               │
       ┌───────┴────────┐
       │                │
       ▼                ▼
       No              Yes
       │                │
       ▼                ▼
   Allow Request    Reject Request
       │                │
       ▼                ▼
 Resource Created   Forbidden Error
```

ResourceQuota는 Namespace 내의 현재 사용량과 새로 요청된 리소스 사용량을 비교합니다.

예를 들어 CPU Requests의 총량이 10 Core로 제한되어 있고 현재 8 Core를 사용 중인 상태에서 새로운 Pod가 CPU 3 Core를 요청하면 생성 요청이 거부됩니다.

ResourceQuota는 리소스 사용량을 제한하는 기능이며, 실제 CPU나 Memory 사용량을 직접 측정하여 제한하는 기능은 아닙니다.

예를 들어 Pod가 CPU Request를 2 Core로 설정했더라도 실제 CPU 사용량이 0.5 Core라면 ResourceQuota는 실제 사용량인 0.5 Core가 아니라 선언된 Request 값인 2 Core를 기준으로 계산합니다.

## 3. ResourceQuota 리소스 구조

ResourceQuota는 다음과 같은 주요 필드로 구성됩니다.

| 필드 | 설명 |
|---|---|
| apiVersion | API 버전으로 `v1`을 사용합니다. |
| kind | 리소스의 종류를 나타내며 `ResourceQuota`를 사용합니다. |
| metadata.name | ResourceQuota의 이름입니다. |
| metadata.namespace | ResourceQuota가 적용될 Namespace입니다. |
| spec.hard | 제한할 리소스의 최대 사용량을 정의합니다. |
| spec.scopes | 특정 조건에 해당하는 리소스만 Quota 적용 대상으로 지정합니다. |
| spec.scopeSelector | 리소스의 조건을 기준으로 Quota 적용 대상을 선택합니다. |
| status.hard | 현재 적용 중인 리소스별 제한값입니다. |
| status.used | 현재 사용 중인 리소스별 사용량입니다. |

`spec.hard`는 ResourceQuota의 핵심 설정으로, Namespace에서 사용할 수 있는 리소스의 최대량을 지정합니다.

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "20"
```

위 설정은 다음과 같은 제한을 의미합니다.

| 항목 | 제한값 | 설명 |
|---|---:|---|
| requests.cpu | 10 | CPU Requests 총량 |
| requests.memory | 20Gi | Memory Requests 총량 |
| limits.cpu | 20 | CPU Limits 총량 |
| limits.memory | 40Gi | Memory Limits 총량 |
| pods | 20 | 생성할 수 있는 Pod 개수 |

## 4. Compute Resource Quota

Compute Resource Quota는 Namespace에서 사용할 수 있는 CPU와 Memory의 총량을 제한합니다.

Kubernetes에서는 CPU와 Memory를 Requests 및 Limits로 구분하여 관리합니다.

### 4.1. CPU Requests

`requests.cpu`는 Namespace 내 모든 Pod 컨테이너가 요청한 CPU Request의 총량을 제한합니다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: cpu-request-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
```

위 설정에서는 Namespace에 속한 모든 대상 컨테이너의 CPU Requests 합계가 10 Core를 초과할 수 없습니다.

예를 들어 다음과 같은 Pod가 있다고 가정하겠습니다.

| Pod | CPU Request |
|---|---:|
| Pod A | 2 Core |
| Pod B | 3 Core |
| Pod C | 4 Core |
| 합계 | 9 Core |

현재 CPU Requests 총량은 9 Core이므로 1 Core의 Quota가 남아 있습니다.

새로운 Pod가 CPU 2 Core를 요청하면 총량이 11 Core가 되므로 생성 요청이 거부됩니다.

### 4.2. Memory Requests

`requests.memory`는 Namespace 내 모든 Pod 컨테이너가 요청한 Memory Request의 총량을 제한합니다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: memory-request-quota
  namespace: production
spec:
  hard:
    requests.memory: 20Gi
```

위 설정에서는 Namespace 내 Memory Requests의 합계가 20Gi를 초과할 수 없습니다.

### 4.3. CPU Limits

`limits.cpu`는 Namespace 내 모든 Pod 컨테이너에 설정된 CPU Limits의 총량을 제한합니다.

```yaml
spec:
  hard:
    limits.cpu: "20"
```

위 설정에서는 CPU Limits의 총합이 20 Core를 초과할 수 없습니다.

CPU Limit은 컨테이너가 사용할 수 있는 CPU의 최대량을 제한하는 설정이며, ResourceQuota는 Namespace 전체에 선언된 CPU Limits의 합계를 관리합니다.

### 4.4. Memory Limits

`limits.memory`는 Namespace 내 모든 Pod 컨테이너에 설정된 Memory Limits의 총량을 제한합니다.

```yaml
spec:
  hard:
    limits.memory: 40Gi
```

위 설정에서는 Memory Limits의 합계가 40Gi를 초과할 수 없습니다.

Memory Limit은 컨테이너가 사용할 수 있는 메모리의 상한을 지정하며, 실제 메모리 사용량이 Limit을 초과하면 컨테이너가 OOMKilled될 수 있습니다.

### 4.5. Requests와 Limits의 차이

ResourceQuota에서 Requests와 Limits는 서로 독립적인 총량 제한으로 관리됩니다.

| 구분 | 의미 | ResourceQuota 기준 |
|---|---|---|
| requests.cpu | 컨테이너가 필요로 하는 CPU 양 | CPU Requests 총합 |
| limits.cpu | 컨테이너가 사용할 수 있는 CPU 상한 | CPU Limits 총합 |
| requests.memory | 컨테이너가 필요로 하는 Memory 양 | Memory Requests 총합 |
| limits.memory | 컨테이너가 사용할 수 있는 Memory 상한 | Memory Limits 총합 |

예를 들어 다음과 같은 Pod가 있다고 가정하겠습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: application
spec:
  containers:
    - name: application
      image: nginx:1.27
      resources:
        requests:
          cpu: "500m"
          memory: 512Mi
        limits:
          cpu: "1"
          memory: 1Gi
```

이 Pod는 ResourceQuota 계산에서 다음과 같은 양을 사용합니다.

| 리소스 | 사용량 |
|---|---:|
| CPU Requests | 500m |
| CPU Limits | 1 Core |
| Memory Requests | 512Mi |
| Memory Limits | 1Gi |

여러 Pod가 생성되면 각 Pod의 리소스 선언값이 Namespace 단위로 합산됩니다.

## 5. Object Count Quota

ResourceQuota는 CPU와 Memory뿐만 아니라 Namespace 내에서 생성할 수 있는 Kubernetes 리소스 객체의 개수도 제한할 수 있습니다.

대표적으로 다음과 같은 리소스의 개수를 제한할 수 있습니다.

| Quota 항목 | 설명 |
|---|---|
| pods | Pod 개수 |
| services | Service 개수 |
| services.loadbalancers | LoadBalancer 타입 Service 개수 |
| services.nodeports | NodePort 타입 Service 개수 |
| configmaps | ConfigMap 개수 |
| secrets | Secret 개수 |
| persistentvolumeclaims | PVC 개수 |
| replicationcontrollers | ReplicationController 개수 |
| count/deployments.apps | Deployment 개수 |
| count/statefulsets.apps | StatefulSet 개수 |
| count/jobs.batch | Job 개수 |
| count/cronjobs.batch | CronJob 개수 |

### 5.1. Pod 개수 제한

다음은 Namespace에서 생성할 수 있는 Pod 개수를 최대 20개로 제한하는 예시입니다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-count-quota
  namespace: production
spec:
  hard:
    pods: "20"
```

위 설정에서는 `production` Namespace에서 생성되는 Pod의 개수가 20개를 초과할 수 없습니다.

Deployment가 3개의 Replica를 생성하려고 하더라도 현재 Pod 개수가 이미 20개라면 추가 Pod 생성이 거부됩니다.

### 5.2. Service 개수 제한

다음은 Namespace에서 생성할 수 있는 Service 개수를 최대 10개로 제한하는 예시입니다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: service-count-quota
  namespace: production
spec:
  hard:
    services: "10"
    services.loadbalancers: "2"
    services.nodeports: "3"
```

위 설정에서는 다음과 같이 제한됩니다.

- 전체 Service: 최대 10개
- LoadBalancer 타입 Service: 최대 2개
- NodePort 타입 Service: 최대 3개

특정 유형의 Service 개수를 별도로 제한하면 클라우드 환경에서 불필요한 LoadBalancer 생성이나 NodePort 사용을 관리하는 데 도움이 됩니다.

### 5.3. API Group을 포함한 객체 개수 제한

일부 리소스는 API Group을 포함한 `count/<resource>.<group>` 형식으로 제한할 수 있습니다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: workload-count-quota
  namespace: production
spec:
  hard:
    count/deployments.apps: "10"
    count/statefulsets.apps: "5"
    count/jobs.batch: "20"
    count/cronjobs.batch: "5"
```

위 설정에서는 다음과 같은 객체 개수를 제한합니다.

| Quota 항목 | 최대 개수 |
|---|---:|
| Deployment | 10 |
| StatefulSet | 5 |
| Job | 20 |
| CronJob | 5 |

Object Count Quota는 리소스 객체의 개수를 제한하는 기능이며, 해당 리소스가 생성하는 Pod의 CPU나 Memory 사용량을 직접 제한하지는 않습니다.

## 6. Storage Resource Quota

ResourceQuota는 Namespace에서 사용하는 스토리지 리소스의 총량도 제한할 수 있습니다.

대표적으로 다음과 같은 항목을 사용할 수 있습니다.

| Quota 항목 | 설명 |
|---|---|
| requests.storage | Namespace 내 PVC가 요청한 스토리지 총량 |
| persistentvolumeclaims | PVC 개수 |
| requests.ephemeral-storage | 컨테이너가 요청한 Ephemeral Storage 총량 |
| limits.ephemeral-storage | 컨테이너에 설정된 Ephemeral Storage Limit 총량 |

### 6.1. requests.storage

`requests.storage`는 Namespace 내 모든 PVC가 요청한 스토리지 용량의 총합을 제한합니다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: production
spec:
  hard:
    requests.storage: 100Gi
    persistentvolumeclaims: "10"
```

위 설정에서는 다음과 같이 제한됩니다.

- PVC가 요청한 스토리지 총량: 최대 100Gi
- PVC 개수: 최대 10개

예를 들어 현재 3개의 PVC가 각각 20Gi, 30Gi, 10Gi를 요청하고 있다면 총 사용량은 60Gi입니다.

이 상태에서 새로운 PVC가 50Gi를 요청하면 총량이 110Gi가 되므로 PVC 생성 요청이 거부됩니다.

### 6.2. StorageClass별 Quota

StorageClass별로 스토리지 사용량을 별도로 제한할 수도 있습니다.

다음은 `fast-storage` StorageClass를 사용하는 PVC의 스토리지 요청 총량을 50Gi로 제한하는 예시입니다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-class-quota
  namespace: production
spec:
  hard:
    fast-storage.storageclass.storage.k8s.io/requests.storage: 50Gi
    fast-storage.storageclass.storage.k8s.io/persistentvolumeclaims: "5"
```

위 설정에서는 다음과 같이 제한됩니다.

- `fast-storage` StorageClass의 PVC 스토리지 요청 총량: 최대 50Gi
- `fast-storage` StorageClass를 사용하는 PVC 개수: 최대 5개

이를 활용하면 Namespace 내에서 StorageClass별 스토리지 사용량을 구분하여 관리할 수 있습니다.

### 6.3. Ephemeral Storage Quota

Ephemeral Storage는 컨테이너가 사용하는 임시 로컬 스토리지입니다.

컨테이너의 쓰기 가능한 레이어, 컨테이너 로그 및 일부 EmptyDir 사용량 등이 포함될 수 있습니다.

다음과 같이 Namespace 전체의 Ephemeral Storage Requests와 Limits를 제한할 수 있습니다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ephemeral-storage-quota
  namespace: production
spec:
  hard:
    requests.ephemeral-storage: 20Gi
    limits.ephemeral-storage: 40Gi
```

위 설정에서는 다음과 같이 제한됩니다.

- Ephemeral Storage Requests 총량: 최대 20Gi
- Ephemeral Storage Limits 총량: 최대 40Gi

Ephemeral Storage는 Node의 로컬 디스크 자원을 사용하므로 컨테이너의 로그나 임시 파일로 인해 디스크가 과도하게 사용되는 상황을 관리하는 데 활용할 수 있습니다.

## 7. Scope를 이용한 ResourceQuota 적용 대상 지정

ResourceQuota는 기본적으로 Namespace 내의 해당 리소스를 대상으로 적용됩니다.

하지만 `scopes` 또는 `scopeSelector`를 사용하면 특정 조건에 해당하는 리소스에만 Quota를 적용할 수 있습니다.

예를 들어 특정 PriorityClass를 사용하는 Pod만 대상으로 CPU와 Memory 사용량을 제한할 수 있습니다.

### 7.1. scopes

`scopes`는 사전에 정의된 Scope를 사용하여 ResourceQuota 적용 대상을 제한합니다.

대표적인 Scope는 다음과 같습니다.

| Scope | 설명 |
|---|---|
| Terminating | 종료 시간이 설정된 Pod를 대상으로 합니다. |
| NotTerminating | 종료 시간이 설정되지 않은 Pod를 대상으로 합니다. |
| BestEffort | BestEffort QoS Class에 해당하는 Pod를 대상으로 합니다. |
| NotBestEffort | BestEffort QoS Class가 아닌 Pod를 대상으로 합니다. |
| PriorityClass | 특정 PriorityClass를 사용하는 Pod를 대상으로 합니다. |

예를 들어 BestEffort QoS Class에 해당하는 Pod의 개수를 제한할 수 있습니다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: besteffort-quota
  namespace: production
spec:
  hard:
    pods: "10"
  scopes:
    - BestEffort
```

위 설정에서는 `production` Namespace의 BestEffort Pod 개수를 최대 10개로 제한합니다.

### 7.2. scopeSelector

`scopeSelector`는 Scope에 대한 추가적인 조건을 지정하여 적용 대상을 선택합니다.

특히 PriorityClass와 같은 조건을 기준으로 Quota를 적용할 때 활용할 수 있습니다.

다음은 `high-priority` PriorityClass를 사용하는 Pod의 CPU Requests 총량을 제한하는 예시입니다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: high-priority-quota
  namespace: production
spec:
  hard:
    requests.cpu: "5"
    pods: "10"
  scopeSelector:
    matchExpressions:
      - scopeName: PriorityClass
        operator: In
        values:
          - high-priority
```

위 설정에서는 `high-priority` PriorityClass를 사용하는 Pod를 대상으로 다음과 같이 제한합니다.

- CPU Requests 총량: 최대 5 Core
- Pod 개수: 최대 10개

`scopeSelector`에서 사용할 수 있는 주요 연산자는 다음과 같습니다.

| Operator | 설명 |
|---|---|
| In | 지정한 값 중 하나와 일치하는 경우 선택합니다. |
| NotIn | 지정한 값에 포함되지 않는 경우 선택합니다. |
| Exists | 해당 Scope가 존재하는 경우 선택합니다. |
| DoesNotExist | 해당 Scope가 존재하지 않는 경우 선택합니다. |

Scope와 ScopeSelector를 함께 사용하면 Namespace 내부에서도 워크로드의 특성에 따라 서로 다른 Quota 정책을 적용할 수 있습니다.

## 8. ResourceQuota와 LimitRange의 관계

ResourceQuota와 LimitRange는 모두 Namespace의 리소스 사용을 관리하지만 서로 다른 목적을 가지고 있습니다.

ResourceQuota는 Namespace 전체에서 사용할 수 있는 리소스의 총량을 제한합니다.

반면 LimitRange는 개별 Pod 또는 컨테이너가 설정할 수 있는 리소스의 최소값, 최대값 및 기본값을 관리합니다.

### 8.1. ResourceQuota

ResourceQuota는 Namespace 전체의 리소스 사용량을 관리합니다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: application-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
```

위 설정은 Namespace 전체에서 사용할 수 있는 CPU와 Memory의 총량을 제한합니다.

### 8.2. LimitRange

LimitRange는 개별 컨테이너의 리소스 설정에 대한 최소값, 최대값 및 기본값을 지정합니다.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: application-limit-range
  namespace: production
spec:
  limits:
    - type: Container
      min:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "2"
        memory: 2Gi
      defaultRequest:
        cpu: 250m
        memory: 256Mi
      default:
        cpu: "1"
        memory: 1Gi
```

위 설정에서는 개별 컨테이너가 설정할 수 있는 리소스의 범위와 기본값을 지정합니다.

### 8.3. ResourceQuota와 LimitRange 비교

| 구분 | ResourceQuota | LimitRange |
|---|---|---|
| 관리 범위 | Namespace 전체 | 개별 Pod 또는 Container |
| 주요 목적 | 전체 리소스 사용량 제한 | 개별 리소스 설정 범위 제한 |
| 주요 설정 | hard | min, max, default, defaultRequest |
| 관리 대상 | CPU, Memory, Storage, 객체 개수 등 | CPU, Memory, Storage 등 |
| 적용 시점 | 리소스 생성 및 변경 요청 시 Quota 검증 | 리소스 생성 시 기본값 적용 및 범위 검증 |

두 리소스를 함께 사용하면 Namespace 전체의 사용량과 개별 컨테이너의 리소스 설정을 동시에 관리할 수 있습니다.

예를 들어 ResourceQuota에서 Namespace의 CPU Requests 총량을 10 Core로 제한하고, LimitRange에서 개별 컨테이너의 CPU Request를 최소 100m, 최대 2 Core로 제한할 수 있습니다.

## 9. ResourceQuota 적용 시 주의사항

ResourceQuota를 사용하는 Namespace에서는 리소스 생성 시 추가적인 제약이 발생할 수 있습니다.

### 9.1. Requests와 Limits의 누락

ResourceQuota에서 `requests.cpu`, `requests.memory`, `limits.cpu`, `limits.memory` 등을 지정하면 Pod의 리소스 설정도 해당 Quota를 충족해야 합니다.

예를 들어 다음과 같은 ResourceQuota가 있다고 가정하겠습니다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
```

이 상태에서 CPU Request를 지정하지 않은 Pod를 생성하려고 하면 요청이 거부될 수 있습니다.

다만 LimitRange에서 기본 Request 값을 설정하여 누락된 리소스 값을 보완할 수 있습니다.

따라서 ResourceQuota와 LimitRange를 함께 사용하는 환경에서는 기본 리소스 설정을 명확하게 정의하는 것이 좋습니다.

### 9.2. ResourceQuota 초과

현재 사용량에 새로운 리소스 요청량을 더했을 때 Quota를 초과하면 Kubernetes API Server가 요청을 거부합니다.

예를 들어 다음과 같은 오류가 발생할 수 있습니다.

```text
Error from server (Forbidden):
pods "application" is forbidden:
exceeded quota: application-quota,
requested: requests.cpu=2,
used: requests.cpu=9,
limited: requests.cpu=10
```

이 경우 현재 CPU Requests 사용량은 9 Core이며, 새로운 Pod가 2 Core를 요청하여 총량이 11 Core가 되므로 생성이 거부된 것입니다.

문제를 해결하려면 다음과 같은 방법을 사용할 수 있습니다.

- 불필요한 Pod 또는 워크로드를 제거합니다.
- 기존 Pod의 Requests를 조정합니다.
- ResourceQuota의 제한값을 조정합니다.
- Namespace를 분리하여 리소스를 관리합니다.

### 9.3. Deployment 생성과 Pod 생성의 차이

ResourceQuota는 실제로 생성되는 리소스의 사용량을 검증합니다.

예를 들어 Deployment 객체 자체의 개수와 해당 Deployment가 생성하는 Pod의 개수는 별개의 Quota 항목입니다.

다음과 같이 설정했다고 가정하겠습니다.

```yaml
spec:
  hard:
    count/deployments.apps: "10"
    pods: "20"
```

이 경우 다음과 같이 동작합니다.

- Deployment 객체는 최대 10개까지 생성할 수 있습니다.
- Namespace 내 Pod는 최대 20개까지 생성할 수 있습니다.

Deployment 객체 생성 자체는 성공하더라도 해당 Deployment가 생성하려는 Pod의 개수가 Quota를 초과하면 Pod 생성이 거부될 수 있습니다.

이때 Deployment 객체는 존재하지만 Pod는 정상적으로 생성되지 않아 Replica 수를 충족하지 못하는 상황이 발생할 수 있습니다.

### 9.4. ResourceQuota는 Namespace 단위로 적용

ResourceQuota는 특정 Namespace에만 적용됩니다.

예를 들어 `production` Namespace에 다음과 같은 Quota를 설정하더라도 `development` Namespace의 리소스 사용량에는 영향을 주지 않습니다.

```yaml
metadata:
  name: production-quota
  namespace: production
```

따라서 여러 팀이 하나의 클러스터를 공유하는 환경에서는 Namespace별로 ResourceQuota를 설정하여 리소스 사용량을 분리하는 것이 일반적입니다.

### 9.5. ResourceQuota와 실제 Node 리소스의 차이

ResourceQuota는 Namespace에 할당된 리소스 사용량을 관리하지만, 특정 Node의 실제 가용 리소스를 예약하거나 보장하지는 않습니다.

예를 들어 Namespace에 CPU Requests Quota가 10 Core로 설정되어 있더라도 클러스터 전체의 Node에 실제로 10 Core의 여유 리소스가 있다는 의미는 아닙니다.

Pod가 스케줄링될 수 있는지는 Node의 Allocatable 리소스와 다른 스케줄링 조건에 따라 별도로 판단됩니다.

따라서 ResourceQuota는 Namespace 수준의 사용량 제한 기능이며, Node의 리소스 할당이나 Pod의 스케줄링 가능성을 보장하는 기능과 구분해야 합니다.

## 10. ResourceQuota 상태 확인

ResourceQuota를 생성하면 Kubernetes는 현재 제한값과 사용량을 Status에 기록합니다.

다음 명령어로 확인할 수 있습니다.

```bash
kubectl get resourcequota -n production
```

출력 예시는 다음과 같습니다.

```text
NAME                REQUEST                                  LIMIT
application-quota   requests.cpu: 8/10, requests.memory: ... limits.cpu: ...
```

보다 자세한 정보를 확인하려면 다음 명령어를 사용합니다.

```bash
kubectl describe resourcequota application-quota -n production
```

출력 예시는 다음과 같습니다.

```text
Name:                   application-quota
Namespace:              production
Resource                Used    Hard
--------                ----    ----
limits.cpu              12      20
limits.memory           24Gi    40Gi
requests.cpu            8       10
requests.memory         16Gi    20Gi
pods                    12      20
```

각 항목의 의미는 다음과 같습니다.

| 항목 | 설명 |
|---|---|
| Resource | 제한 대상 리소스입니다. |
| Used | 현재 사용 중인 리소스 양입니다. |
| Hard | 해당 리소스에 설정된 최대 허용량입니다. |

위 예시에서는 다음과 같은 상태를 확인할 수 있습니다.

- CPU Requests: 8 Core / 10 Core
- Memory Requests: 16Gi / 20Gi
- CPU Limits: 12 Core / 20 Core
- Memory Limits: 24Gi / 40Gi
- Pod 개수: 12개 / 20개

특정 Quota의 YAML 정보를 확인하려면 다음 명령어를 사용할 수 있습니다.

```bash
kubectl get resourcequota application-quota \
  -n production \
  -o yaml
```

## 11. ResourceQuota 관련 주요 명령어

| 명령어 | 설명 |
|---|---|
| `kubectl get resourcequota` | 현재 Namespace의 ResourceQuota 목록을 확인합니다. |
| `kubectl get resourcequota -A` | 모든 Namespace의 ResourceQuota 목록을 확인합니다. |
| `kubectl describe resourcequota <name>` | ResourceQuota의 상세 설정과 사용량을 확인합니다. |
| `kubectl get resourcequota <name> -o yaml` | ResourceQuota의 전체 YAML 정보를 확인합니다. |
| `kubectl get pods -n <namespace>` | Namespace의 Pod 목록을 확인합니다. |
| `kubectl get limitrange -n <namespace>` | Namespace의 LimitRange 목록을 확인합니다. |
| `kubectl describe limitrange <name>` | LimitRange의 상세 설정을 확인합니다. |

ResourceQuota를 생성하려면 다음 명령어를 사용할 수 있습니다.

```bash
kubectl apply -f resource-quota.yaml
```

특정 Namespace의 ResourceQuota 사용량을 확인하려면 다음 명령어를 사용합니다.

```bash
kubectl describe resourcequota -n production
```

## 12. 정리

ResourceQuota는 Kubernetes에서 Namespace별 리소스 사용량과 객체 개수를 제한하는 리소스입니다.

CPU와 Memory의 Requests 및 Limits 총량을 제한할 수 있으며, PVC의 스토리지 요청량, Ephemeral Storage, Pod 및 Service 등의 객체 개수도 관리할 수 있습니다.

또한 `scopes`와 `scopeSelector`를 사용하면 특정 PriorityClass나 QoS Class 등에 해당하는 리소스만 대상으로 Quota를 적용할 수 있습니다.

Kubernetes의 ResourceQuota Admission Controller는 리소스 생성 및 변경 요청을 검증하여 Namespace의 사용량이 설정된 한도를 초과하지 않도록 관리합니다.

다만 ResourceQuota는 실제 Node의 가용 리소스를 예약하거나 Pod의 스케줄링을 보장하지 않습니다. Namespace에서 사용할 수 있는 리소스의 총량을 제한하는 기능이므로 개별 컨테이너의 리소스 설정 범위를 관리하는 LimitRange와 함께 사용하는 것이 효과적입니다.

따라서 ResourceQuota는 멀티테넌시 환경에서 Namespace별 리소스 사용량을 통제하고, 특정 팀이나 애플리케이션이 클러스터 자원을 과도하게 점유하는 상황을 방지하는 데 활용됩니다.

## 13. 참고자료

- [Kubernetes 공식 문서 - Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Kubernetes 공식 문서 - ResourceQuota API Reference](https://kubernetes.io/docs/reference/kubernetes-api/policy/resource-quota-v1/)
- [Kubernetes 공식 문서 - Limit Ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)
- [Kubernetes 공식 문서 - Managing Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes 공식 문서 - Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Kubernetes 공식 문서 - Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)