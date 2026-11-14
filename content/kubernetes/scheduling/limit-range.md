

# LimitRange

## 1. LimitRange란?

`LimitRange`는 Kubernetes에서 Namespace 내 개별 리소스의 CPU, Memory, Storage 등에 대한 최소·최대 할당량과 기본값을 정의하는 리소스입니다.

Namespace 내에서 특정 Pod 또는 Container가 과도한 리소스를 요청하거나 할당받지 않도록 제한하고, 리소스 설정이 누락된 Container에 기본 Request와 Limit을 자동으로 적용할 수 있습니다.

`LimitRange`를 사용하면 다음과 같은 정책을 적용할 수 있습니다.

- Container별 CPU 및 Memory의 최소·최대 Request와 Limit 설정
- Pod 전체의 CPU 및 Memory 최소·최대 할당량 설정
- PersistentVolumeClaim별 최소·최대 Storage 요청량 설정
- Resource Request와 Limit 사이의 최대 비율 설정
- Container에 CPU 및 Memory의 기본 Request와 Limit 자동 적용

`LimitRange`는 Namespace 단위로 적용됩니다. 따라서 특정 Namespace에 `LimitRange`를 생성하면 해당 Namespace에서 생성되는 적용 대상 리소스에 정책이 적용됩니다.

예를 들어 개발 환경에서는 Container의 Memory Limit을 최대 1Gi로 제한하고, 운영 환경에서는 최대 4Gi까지 허용하는 방식으로 Namespace별 리소스 정책을 구성할 수 있습니다.

## 2. LimitRange의 동작 원리

`LimitRange`는 Kubernetes API Server의 Admission Control 과정에서 동작합니다.

주요 동작 과정은 다음과 같습니다.

1. 관리자가 특정 Namespace에 `LimitRange`를 생성합니다.
2. 사용자가 해당 Namespace에 Pod 또는 PersistentVolumeClaim 등의 리소스를 생성하거나 수정합니다.
3. `LimitRanger` Admission Controller가 리소스에 기본 Request와 Limit이 필요한 경우 설정된 기본값을 적용합니다.
4. 요청된 리소스가 `LimitRange`에 정의된 최소·최대 조건 및 Request/Limit 비율을 만족하는지 검증합니다.
5. 조건을 만족하면 리소스 생성 또는 수정이 진행되고, 조건을 위반하면 API Server가 요청을 거부합니다.

리소스 생성 요청이 정책을 위반하면 일반적으로 HTTP `403 Forbidden` 응답과 함께 위반된 조건을 설명하는 오류 메시지가 반환됩니다.

### 2.1. 기본값 적용

Container에 CPU 또는 Memory Request와 Limit을 명시하지 않은 경우 `LimitRange`에 설정된 기본값이 적용될 수 있습니다.

예를 들어 다음과 같은 기본값을 설정할 수 있습니다.

| 항목 | CPU | Memory |
|---|---:|---:|
| Default Request | 200m | 256Mi |
| Default Limit | 500m | 512Mi |

이 정책이 적용된 Namespace에서 다음과 같이 리소스를 선언하지 않은 Container를 생성하면 기본값이 자동으로 적용됩니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  containers:
    - name: app
      image: nginx
```

Admission Controller가 기본값을 적용한 후의 Container 리소스 설정은 다음과 같은 형태가 됩니다.

```yaml
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

기본값 적용은 Pod 생성 과정에서 이루어지므로, 사용자가 제출한 YAML에 기본값이 직접 작성되어 있지 않더라도 실제 생성된 Pod에는 해당 값이 포함됩니다.

### 2.2. 최소·최대 조건 검증

`LimitRange`는 Container 또는 Pod에 설정된 리소스 Request와 Limit이 지정된 범위를 만족하는지 검증합니다.

예를 들어 다음과 같은 조건을 설정할 수 있습니다.

- CPU Request: 최소 100m
- CPU Limit: 최대 1
- Memory Request: 최소 128Mi
- Memory Limit: 최대 1Gi

이러한 조건이 적용된 Namespace에서는 CPU Request를 50m로 설정하거나 Memory Limit을 2Gi로 설정한 Container를 생성할 수 없습니다.

### 2.3. Request와 Limit 비율 검증

`maxLimitRequestRatio`를 사용하면 특정 리소스의 Limit이 Request보다 얼마나 큰 값까지 허용되는지 제한할 수 있습니다.

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
maxLimitRequestRatio:
  cpu: "4"
  memory: "2"
```

이 경우 다음 조건을 만족해야 합니다.

- CPU: Limit / Request ≤ 4
- Memory: Limit / Request ≤ 2

CPU Request가 200m인 Container의 CPU Limit은 최대 800m까지 허용됩니다.

Memory Request가 256Mi인 Container의 Memory Limit은 최대 512Mi까지 허용됩니다.

Request 또는 Limit이 0인 경우에는 해당 비율 검증을 적용할 수 없으므로, 비율 제한을 사용하려면 양쪽 값이 모두 0보다 커야 합니다.

## 3. LimitRange 리소스 구조

`LimitRange`는 Kubernetes Core API 그룹의 `v1` 버전을 사용합니다.

기본적인 리소스 구조는 다음과 같습니다.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: example-limit-range
  namespace: development
spec:
  limits:
    - type: Container
      min:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "1"
        memory: 1Gi
      defaultRequest:
        cpu: 200m
        memory: 256Mi
      default:
        cpu: 500m
        memory: 512Mi
      maxLimitRequestRatio:
        cpu: "4"
        memory: "2"
```

주요 필드는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `metadata.name` | LimitRange의 이름 |
| `metadata.namespace` | LimitRange가 적용될 Namespace |
| `spec.limits` | 리소스별 제한 정책 목록 |
| `spec.limits[].type` | 제한 정책을 적용할 리소스 종류 |
| `min` | 리소스의 최소 허용량 |
| `max` | 리소스의 최대 허용량 |
| `defaultRequest` | Request가 지정되지 않았을 때 적용할 기본값 |
| `default` | Limit이 지정되지 않았을 때 적용할 기본값 |
| `maxLimitRequestRatio` | Limit과 Request 사이에 허용되는 최대 비율 |

`spec.limits`에는 여러 개의 제한 정책을 정의할 수 있으며, 각 정책은 `type`을 통해 적용 대상을 구분합니다.

## 4. LimitRange의 Type

`LimitRange`는 다음 세 가지 Type을 지원합니다.

| Type | 적용 대상 | 설명 |
|---|---|---|
| `Container` | Container | 개별 Container의 리소스 할당량 제한 |
| `Pod` | Pod | Pod에 포함된 Container들의 리소스 합계 제한 |
| `PersistentVolumeClaim` | PersistentVolumeClaim | PVC의 Storage 요청량 제한 |

### 4.1. Container

`Container` Type은 Pod에 포함된 개별 Container의 리소스 할당량을 제한합니다.

예를 들어 다음과 같이 설정하면 Namespace 내에서 생성되는 Container마다 CPU와 Memory의 최소·최대 할당량이 적용됩니다.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limit-range
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
        cpu: 200m
        memory: 256Mi
      default:
        cpu: "1"
        memory: 1Gi
```

이 정책에서는 다음과 같은 조건이 적용됩니다.

- Container의 CPU Request는 최소 100m 이상이어야 합니다.
- Container의 Memory Request는 최소 128Mi 이상이어야 합니다.
- Container의 CPU Limit은 최대 2까지 허용됩니다.
- Container의 Memory Limit은 최대 2Gi까지 허용됩니다.
- Request와 Limit이 누락된 경우 설정된 기본값이 적용됩니다.

`Container` Type은 각 Container를 개별적으로 검증합니다. 따라서 여러 Container로 구성된 Pod에서는 각 Container에 정책이 별도로 적용됩니다.

### 4.2. Pod

`Pod` Type은 Pod에 포함된 Container들의 리소스 할당량 합계를 기준으로 제한합니다.

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: pod-limit-range
spec:
  limits:
    - type: Pod
      max:
        cpu: "2"
        memory: 2Gi
      min:
        cpu: 200m
        memory: 256Mi
```

이 정책에서는 Pod에 포함된 Container들의 CPU와 Memory Request 및 Limit 합계가 설정된 범위를 만족해야 합니다.

예를 들어 하나의 Pod에 다음과 같이 두 개의 Container가 존재한다고 가정하겠습니다.

| Container | CPU Request | Memory Request |
|---|---:|---:|
| app | 500m | 512Mi |
| sidecar | 300m | 256Mi |
| 합계 | 800m | 768Mi |

이 경우 Pod의 총 CPU Request는 800m, Memory Request는 768Mi입니다.

Pod Type의 제한은 개별 Container가 아닌 Pod 전체의 리소스 할당량을 대상으로 합니다.

### 4.3. PersistentVolumeClaim

`PersistentVolumeClaim` Type은 PVC가 요청하는 Storage 크기를 제한합니다.

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: pvc-limit-range
spec:
  limits:
    - type: PersistentVolumeClaim
      min:
        storage: 1Gi
      max:
        storage: 10Gi
```

이 정책에서는 Namespace 내에서 생성되는 PVC가 다음 조건을 만족해야 합니다.

- 최소 Storage 요청량은 1Gi입니다.
- 최대 Storage 요청량은 10Gi입니다.

따라서 500Mi를 요청하는 PVC나 20Gi를 요청하는 PVC는 생성할 수 없습니다.

다음은 해당 정책을 적용받는 PVC 예제입니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

위 PVC는 Storage 요청량이 1Gi 이상 10Gi 이하이므로 정책을 만족합니다.

`PersistentVolumeClaim` Type은 PVC의 Storage 요청량을 제한하며, 실제로 사용할 수 있는 Storage의 크기나 StorageClass의 Provisioning 정책을 직접 결정하지는 않습니다.

## 5. Default와 DefaultRequest

`LimitRange`에서 `default`와 `defaultRequest`는 서로 다른 목적으로 사용됩니다.

| 필드 | 역할 | 적용 대상 |
|---|---|---|
| `default` | 기본 Limit 설정 | Container의 Limit |
| `defaultRequest` | 기본 Request 설정 | Container의 Request |

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-resource-range
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 200m
        memory: 256Mi
      default:
        cpu: 500m
        memory: 512Mi
```

이 경우 Container가 Request와 Limit을 모두 지정하지 않으면 다음 값이 적용됩니다.

```yaml
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

각 항목이 적용되는 방식은 다음과 같습니다.

| Container 설정 | 적용 결과 |
|---|---|
| Request와 Limit 모두 미설정 | 각각 `defaultRequest`, `default` 적용 |
| Request만 설정 | Request는 사용자가 지정한 값, Limit은 `default` 적용 |
| Limit만 설정 | Limit은 사용자가 지정한 값, Request는 `defaultRequest` 적용 여부에 따라 결정 |
| Request와 Limit 모두 설정 | 사용자가 지정한 값 사용 |

단, Limit만 지정하고 Request를 생략한 경우 Kubernetes에는 Request를 Limit과 동일한 값으로 설정하는 기본 동작이 있습니다. 따라서 해당 상황에서는 `defaultRequest`와 Admission 과정의 기본값 적용을 함께 고려해야 합니다.

또한 `LimitRange`가 설정한 기본 Limit이 사용자가 명시한 Request보다 작은 경우, 최종 Pod의 Request와 Limit 관계가 유효하지 않아 Pod 생성이 거부될 수 있습니다.

예를 들어 다음과 같은 정책이 있다고 가정하겠습니다.

```yaml
default:
  cpu: 500m
defaultRequest:
  cpu: 500m
max:
  cpu: "1"
min:
  cpu: 100m
```

이 정책이 적용된 Namespace에서 다음 Pod를 생성하려고 하면 문제가 발생합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: invalid-resource-pod
spec:
  containers:
    - name: app
      image: nginx
      resources:
        requests:
          cpu: 700m
```

사용자가 CPU Request를 700m로 설정했지만 CPU Limit은 지정하지 않았습니다.

이 경우 `LimitRange`의 기본 CPU Limit인 500m가 적용되므로 Request가 Limit보다 커지는 문제가 발생합니다.

따라서 다음과 같이 Request와 Limit을 함께 지정하면 정상적으로 생성할 수 있습니다.

```yaml
resources:
  requests:
    cpu: 700m
  limits:
    cpu: 700m
```

`LimitRange`는 기본값을 적용할 때 Request와 Limit의 관계가 항상 유효한지 자동으로 보정하지 않으므로 기본값을 설계할 때 주의해야 합니다.

## 6. LimitRange와 ResourceQuota 비교

`LimitRange`와 `ResourceQuota`는 모두 Namespace 단위로 리소스 사용을 관리하지만 적용 대상과 목적이 다릅니다.

| 구분 | LimitRange | ResourceQuota |
|---|---|---|
| 목적 | 개별 리소스의 할당 범위 제한 | Namespace 전체 리소스 사용량 제한 |
| 적용 범위 | Container, Pod, PVC | Namespace 내 리소스 전체 |
| CPU / Memory | 최소·최대 Request 및 Limit 설정 | Request 및 Limit 총량 제한 |
| Storage | PVC별 최소·최대 요청량 | Storage 총 요청량 및 PVC 개수 제한 |
| 기본값 설정 | 가능 | 불가능 |
| Request / Limit 비율 | 제한 가능 | 제한 불가능 |
| 주요 목적 | 개별 리소스의 과도한 할당 방지 | Namespace의 리소스 총량 관리 |

예를 들어 다음과 같은 정책을 구성할 수 있습니다.

**LimitRange**

- Container별 CPU Request는 최소 100m 이상이어야 합니다.
- Container별 CPU Limit은 최대 1까지 허용됩니다.
- Container별 Memory Limit은 최대 1Gi까지 허용됩니다.

**ResourceQuota**

- Namespace 전체 CPU Request 합계는 최대 10 Core입니다.
- Namespace 전체 Memory Request 합계는 최대 20Gi입니다.
- Namespace 내 Pod 개수는 최대 50개입니다.

두 리소스를 함께 사용하면 개별 Container의 과도한 리소스 할당을 방지하면서 Namespace 전체 사용량도 제한할 수 있습니다.

## 7. LimitRange 적용 시 주의사항

### 7.1. 기존 Pod에는 소급 적용되지 않습니다.

`LimitRange`의 기본값 적용 및 유효성 검사는 Pod Admission 단계에서 수행됩니다.

따라서 이미 생성되어 실행 중인 Pod에는 새로운 `LimitRange` 정책이 소급 적용되지 않습니다.

기존 Pod가 정책을 위반하더라도 자동으로 Request나 Limit이 변경되거나 종료되지 않습니다.

새로운 정책을 기존 Pod에 적용하려면 해당 Pod를 재생성하거나 별도의 리소스 업데이트 절차를 수행해야 합니다.

### 7.2. LimitRange가 여러 개 존재하면 기본값 적용이 비결정적입니다.

하나의 Namespace에 두 개 이상의 `LimitRange`가 존재할 수 있습니다.

다만 여러 `LimitRange`가 동일한 리소스에 서로 다른 기본값을 정의하면 어떤 기본값이 적용될지 결정적이지 않습니다.

따라서 동일한 Namespace에서 여러 `LimitRange`를 사용하더라도 기본값 정책이 충돌하지 않도록 구성하는 것이 좋습니다.

가능하면 Namespace별로 하나의 일관된 기본 리소스 정책을 관리하는 것이 좋습니다.

### 7.3. LimitRange는 실제 리소스 사용량을 직접 제한하지 않습니다.

`LimitRange`는 리소스 생성 및 수정 시 Request와 Limit이 정책을 만족하는지 검증하고 기본값을 적용하는 정책입니다.

실제 실행 중인 Container의 리소스 사용량을 제어하는 것은 Request와 Limit의 역할입니다.

- CPU Request: Scheduler가 Pod 배치 시 고려하는 리소스 요청량입니다.
- CPU Limit: Container가 사용할 수 있는 CPU 상한을 설정합니다.
- Memory Request: Scheduler가 Pod 배치 시 고려하는 메모리 요청량입니다.
- Memory Limit: Container의 메모리 사용 상한을 설정합니다.

따라서 `LimitRange` 자체가 CPU 사용량을 제한하거나 메모리 사용량을 강제로 줄이는 것은 아닙니다.

### 7.4. Node의 가용 리소스를 보장하지 않습니다.

`LimitRange`는 Namespace 내 리소스 할당 범위를 검증하지만 Node의 실제 가용 리소스를 확인하거나 예약하지 않습니다.

예를 들어 Container의 CPU Request가 `LimitRange`의 최대값 이하이더라도 해당 Request를 수용할 수 있는 Node가 없다면 Pod는 스케줄링되지 않을 수 있습니다.

### 7.5. ResourceQuota와 함께 사용할 때 주의해야 합니다.

Namespace에 `ResourceQuota`가 적용되어 있다면 Pod 생성 시 `LimitRange`와 `ResourceQuota`의 조건을 모두 만족해야 합니다.

특히 CPU 또는 Memory에 대한 ResourceQuota가 설정된 Namespace에서 Request와 Limit이 누락되면 기본값 적용 여부에 따라 ResourceQuota 검증에 필요한 리소스 값이 달라질 수 있습니다.

따라서 `LimitRange`의 기본값과 `ResourceQuota`의 총량을 함께 설계해야 합니다.

### 7.6. In-Place Pod Resize에도 제약이 적용됩니다.

Kubernetes의 In-Place Pod Resize 기능을 사용하면 실행 중인 Pod의 CPU 및 Memory 리소스를 재시작 없이 변경할 수 있습니다.

이때 리소스 변경으로 인해 `LimitRange`의 최소·최대 조건을 위반하게 되면 Resize 요청이 거부될 수 있습니다.

따라서 In-Place Pod Resize를 사용하는 환경에서는 `LimitRange` 정책도 함께 고려해야 합니다.

## 8. LimitRange 확인 및 관리 명령어

### 8.1. LimitRange 목록 조회

```bash
kubectl get limitrange -n development
```

특정 Namespace의 LimitRange 목록을 조회합니다.

### 8.2. LimitRange 상세 정보 조회

```bash
kubectl describe limitrange example-limit-range -n development
```

LimitRange에 설정된 리소스 제한과 기본값을 확인합니다.

### 8.3. LimitRange YAML 조회

```bash
kubectl get limitrange example-limit-range -n development -o yaml
```

LimitRange의 전체 설정을 YAML 형식으로 확인합니다.

### 8.4. LimitRange 생성 및 수정

```bash
kubectl apply -f limit-range.yaml
```

YAML 파일에 정의된 LimitRange를 생성하거나 기존 리소스를 수정합니다.

### 8.5. LimitRange 삭제

```bash
kubectl delete limitrange example-limit-range -n development
```

지정한 Namespace의 LimitRange를 삭제합니다.

### 8.6. Pod에 적용된 Request와 Limit 확인

```bash
kubectl get pod example-pod -n development -o yaml
```

Admission Controller가 적용한 기본 Request와 Limit을 포함하여 실제 Pod의 리소스 설정을 확인할 수 있습니다.

특정 Container의 리소스 설정만 확인하려면 다음 명령어를 사용할 수 있습니다.

```bash
kubectl get pod example-pod -n development \
  -o jsonpath='{range .spec.containers[*]}{.name}{"\n"}{.resources}{"\n"}{end}'
```

## 9. 정리

`LimitRange`는 Namespace 내 개별 리소스의 CPU, Memory, Storage 할당 범위를 관리하는 Kubernetes 정책 리소스입니다.

주요 기능은 다음과 같습니다.

- Container별 CPU 및 Memory의 최소·최대 할당량 제한
- Pod 전체의 CPU 및 Memory 할당량 제한
- PVC별 Storage 요청량 제한
- Container의 기본 Request와 Limit 자동 적용
- Request와 Limit 사이의 최대 비율 제한

`ResourceQuota`가 Namespace 전체의 리소스 사용 총량을 관리한다면, `LimitRange`는 개별 리소스가 할당받을 수 있는 범위를 관리합니다.

따라서 두 리소스를 함께 사용하면 Namespace 전체의 리소스 사용량과 개별 Container 및 Pod의 리소스 할당량을 동시에 관리할 수 있습니다.

## 참고자료

- [Kubernetes 공식 문서 - Limit Ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)
- [Kubernetes 공식 문서 - Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes 공식 문서 - Configure Minimum and Maximum Memory Constraints for a Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-constraint-namespace/)
- [Kubernetes API Reference - LimitRange v1](https://kubernetes.io/docs/reference/kubernetes-api/policy/limit-range-v1/)

