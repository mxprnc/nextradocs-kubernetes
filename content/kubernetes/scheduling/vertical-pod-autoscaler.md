# VerticalPodAutoscaler (VPA)

## 1. VerticalPodAutoscaler란?

VerticalPodAutoscaler(VPA)는 Kubernetes에서 Pod의 CPU 및 Memory 리소스 할당량을 사용량에 따라 자동으로 조정하는 기능입니다.

일반적으로 Deployment 또는 StatefulSet과 같은 워크로드 리소스를 대상으로 설정하며, 애플리케이션의 과거 및 현재 리소스 사용량을 분석하여 적절한 CPU와 Memory Requests를 추천합니다.

HPA가 Pod의 개수를 증가시키거나 감소시키는 Horizontal Scaling 방식이라면, VPA는 개별 Pod에 할당되는 리소스 크기를 조정하는 Vertical Scaling 방식입니다.

예를 들어 애플리케이션의 실제 CPU 사용량이 지속적으로 낮다면 VPA는 CPU Request를 낮추도록 추천할 수 있습니다. 반대로 Memory 사용량이 지속적으로 증가한다면 더 많은 Memory를 할당하도록 추천할 수 있습니다.

주요 특징은 다음과 같습니다.

- **리소스 자동 최적화:** CPU 및 Memory 사용량을 분석하여 적절한 리소스 Requests를 추천합니다.
- **리소스 낭비 방지:** 실제 사용량보다 과도하게 설정된 Requests를 줄여 Node의 자원 활용률을 개선할 수 있습니다.
- **리소스 부족 방지:** 현재 할당된 리소스가 부족한 경우 더 많은 리소스를 할당하도록 추천합니다.
- **다양한 업데이트 모드:** 기존 Pod를 유지하면서 리소스를 변경하거나 Pod를 재생성하여 변경 사항을 적용할 수 있습니다.
- **컨테이너별 정책 설정:** Pod 내부의 개별 컨테이너마다 리소스 관리 범위와 최소 및 최대값을 지정할 수 있습니다.

VPA는 Kubernetes의 기본 내장 리소스가 아니라 별도로 설치해야 하는 추가 구성 요소입니다. `VerticalPodAutoscaler`라는 Custom Resource Definition(CRD)을 등록하여 Kubernetes API에서 관리할 수 있습니다.

## 2. Vertical Scaling과 Horizontal Scaling

Kubernetes의 자동 스케일링은 크게 Horizontal Scaling과 Vertical Scaling으로 구분됩니다.

### 2.1. Horizontal Scaling

Horizontal Scaling은 애플리케이션을 실행하는 Pod의 개수를 증가시키거나 감소시키는 방식입니다.

예를 들어 현재 3개의 Pod가 실행 중인 상태에서 트래픽이 증가하면 6개로 확장하고, 트래픽이 감소하면 다시 3개로 축소할 수 있습니다.

HPA(HorizontalPodAutoscaler)가 이러한 방식을 사용합니다.

### 2.2. Vertical Scaling

Vertical Scaling은 기존 Pod의 개수를 유지하면서 각 Pod에 할당되는 CPU 및 Memory 등의 리소스를 조정하는 방식입니다.

예를 들어 Pod 3개를 유지하면서 각 Pod에 할당된 CPU Request를 500m에서 1000m로 증가시키는 방식입니다.

VPA(VerticalPodAutoscaler)가 이러한 방식을 사용합니다.

| 구분 | HPA | VPA |
|---|---|---|
| 스케일링 방식 | Horizontal Scaling | Vertical Scaling |
| 조정 대상 | Pod 개수 | Pod의 CPU 및 Memory Requests |
| 확장 방식 | Pod 추가 또는 제거 | 개별 Pod의 리소스 할당량 변경 |
| 주요 목적 | 처리량 및 동시 요청 처리 능력 확장 | 개별 Pod의 리소스 최적화 |
| 주요 기준 | CPU, Memory, Custom Metrics 등 | CPU 및 Memory 사용량 |

HPA와 VPA는 서로 다른 목적을 가지고 있으며 함께 사용할 수도 있습니다.

다만 동일한 CPU 또는 Memory 사용률을 기준으로 HPA와 VPA가 동시에 동작하면 서로 간섭할 수 있습니다.

예를 들어 CPU 사용률이 높아져 HPA가 Pod 개수를 증가시키는 동시에 VPA가 CPU Requests를 증가시키면 CPU 사용률 계산의 기준 자체가 변경됩니다.

따라서 두 리소스를 함께 사용할 때는 스케일링 기준을 분리하거나 각 리소스의 역할을 명확하게 설정해야 합니다.

## 3. VPA의 아키텍처

VPA는 일반적으로 세 가지 주요 구성 요소로 이루어집니다.

| 구성 요소 | 역할 |
|---|---|
| Recommender | Pod의 과거 및 현재 리소스 사용량을 분석하여 적절한 CPU 및 Memory Requests를 추천합니다. |
| Updater | 현재 Pod의 리소스 설정과 추천값을 비교하고, 필요한 경우 Pod를 재생성하거나 In-Place 방식으로 리소스를 변경합니다. |
| Admission Controller | 새로운 Pod가 생성될 때 VPA의 추천 리소스 값을 적용합니다. |

### 3.1. Recommender

Recommender는 VPA의 리소스 추천을 담당하는 핵심 구성 요소입니다.

각 Pod의 과거 및 현재 CPU와 Memory 사용량을 분석하여 적절한 리소스 Requests를 계산합니다.

주요 역할은 다음과 같습니다.

- Pod의 리소스 사용량을 지속적으로 수집합니다.
- 과거 리소스 사용량을 저장하고 분석합니다.
- CPU 및 Memory의 사용 패턴을 분석합니다.
- 컨테이너별 적절한 리소스 Requests를 계산합니다.
- 계산된 추천값을 VerticalPodAutoscaler 리소스의 `.status`에 기록합니다.

Recommender는 단순히 현재 사용량만을 기준으로 추천값을 결정하지 않습니다. 과거 사용량과 시간에 따른 사용 패턴 등을 고려하여 리소스 요구량을 추정합니다.

따라서 일시적인 CPU 사용량 변화보다는 일정 기간 동안의 사용 패턴을 반영한 리소스 추천에 활용됩니다.

### 3.2. Updater

Updater는 Recommender가 계산한 추천값을 실제 Pod에 적용하는 역할을 담당합니다.

현재 Pod에 설정된 리소스와 추천된 리소스를 비교하고, 변경이 필요한 경우 설정된 Update Mode에 따라 업데이트를 수행합니다.

주요 역할은 다음과 같습니다.

- 관리 대상 Pod의 현재 리소스 설정을 확인합니다.
- Recommender가 제공한 추천값과 현재 설정을 비교합니다.
- 리소스 변경이 필요한 Pod를 판단합니다.
- Update Mode에 따라 In-Place 업데이트 또는 Pod Eviction을 수행합니다.
- Eviction API를 사용하는 경우 PodDisruptionBudget(PDB)을 고려합니다.

기존 Pod를 재생성하는 방식에서는 Updater가 Pod를 축출하고, 상위 워크로드 Controller가 새로운 Pod를 생성하도록 합니다.

새로운 Pod의 리소스 설정은 Admission Controller를 통해 추천값이 적용됩니다.

### 3.3. Admission Controller

Admission Controller는 Pod 생성 요청이 API Server에 전달될 때 VPA의 추천 리소스를 적용하는 역할을 담당합니다.

일반적으로 Mutating Admission Webhook 방식으로 동작합니다.

새로운 Pod가 생성되거나 기존 Pod가 재생성되면 Admission Controller가 해당 Pod가 VPA의 관리 대상인지 확인하고, 필요한 경우 컨테이너의 CPU 및 Memory Requests를 추천값으로 변경합니다.

주요 역할은 다음과 같습니다.

- 새로운 Pod 생성 요청을 감지합니다.
- 해당 Pod와 연결된 VPA 리소스를 확인합니다.
- Recommender가 계산한 추천값을 조회합니다.
- Pod의 컨테이너에 적절한 리소스 Requests를 적용합니다.
- 변경된 Pod 정의가 API Server에 저장되도록 처리합니다.

Admission Controller는 기존에 실행 중인 Pod의 리소스를 직접 변경하는 역할이 아닙니다. 새로운 Pod가 생성되는 시점에 추천값을 적용하는 역할을 담당합니다.

### 3.4. 전체 동작 구조

VPA의 전체 동작 구조는 다음과 같습니다.

```text
┌──────────────────────────────┐
│        Running Pods          │
│                              │
│     CPU / Memory Usage       │
└──────────────┬───────────────┘
               │
               │ Resource Metrics
               ▼
┌──────────────────────────────┐
│         Recommender          │
│                              │
│   Historical Usage Analysis  │
│                              │
│   Resource Recommendations   │
└──────────────┬───────────────┘
               │
               │ Update VPA Status
               ▼
┌──────────────────────────────┐
│     VerticalPodAutoscaler    │
│                              │
│       .status.recommendation │
└──────────────┬───────────────┘
               │
               │ Recommendation
               ▼
┌──────────────────────────────┐
│           Updater            │
│                              │
│  Compare Current Resources   │
│        with Target           │
└──────────────┬───────────────┘
               │
               │
       ┌───────┴────────┐
       │                │
       ▼                ▼
  In-Place           Eviction
   Update             Request
       │                │
       │                ▼
       │          Workload Controller
       │                │
       │                ▼
       │          New Pod Creation
       │                │
       └───────┬────────┘
               │
               ▼
┌──────────────────────────────┐
│      Admission Controller    │
│                              │
│   Apply Recommended Resources│
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│        Updated Pod           │
│                              │
│    CPU / Memory Requests     │
└──────────────────────────────┘
```

위 구조에서 Recommender는 리소스 추천을 담당하고, Updater는 기존 Pod에 추천값을 적용할지 결정하며, Admission Controller는 새롭게 생성되는 Pod에 추천값을 적용합니다.

## 4. VPA의 리소스 추천 방식

VPA는 Pod의 CPU 및 Memory 사용량을 분석하여 적절한 리소스 Requests를 추천합니다.

추천 결과는 VerticalPodAutoscaler 리소스의 `.status.recommendation` 필드에 저장됩니다.

### 4.1. Resource Recommendation

VPA는 컨테이너별로 다음과 같은 추천값을 제공합니다.

| 항목 | 설명 |
|---|---|
| target | 권장하는 리소스 Requests 값입니다. |
| lowerBound | 추천할 수 있는 리소스 Requests의 하한값입니다. |
| upperBound | 추천할 수 있는 리소스 Requests의 상한값입니다. |
| uncappedTarget | Resource Policy의 최소 및 최대 제한을 적용하기 전 계산된 추천값입니다. |

예를 들어 다음과 같은 추천 결과가 있다고 가정하겠습니다.

```yaml
status:
  recommendation:
    containerRecommendations:
      - containerName: application
        target:
          cpu: 500m
          memory: 512Mi
        lowerBound:
          cpu: 250m
          memory: 256Mi
        upperBound:
          cpu: 1000m
          memory: 1Gi
        uncappedTarget:
          cpu: 500m
          memory: 512Mi
```

각 항목은 다음과 같이 해석할 수 있습니다.

- `target`: VPA가 권장하는 기본 리소스 Requests입니다.
- `lowerBound`: 이 값보다 낮은 리소스 할당은 적절하지 않을 수 있음을 나타냅니다.
- `upperBound`: 이 값보다 높은 리소스 할당은 필요하지 않을 수 있음을 나타냅니다.
- `uncappedTarget`: 사용자가 설정한 Resource Policy의 제한을 적용하기 전 계산된 추천값입니다.

이러한 값은 리소스 사용량을 분석한 결과이며, lowerBound와 upperBound가 실제 실행 가능한 절대적인 최소 및 최대값을 의미하는 것은 아닙니다.

### 4.2. CPU Recommendation

CPU Recommendation은 컨테이너의 CPU 사용량을 기반으로 적절한 CPU Requests를 계산합니다.

CPU 사용량은 시간에 따라 지속적으로 변동하므로 VPA는 사용 패턴을 분석하여 적절한 CPU 할당량을 추천합니다.

예를 들어 다음과 같은 CPU 사용량이 관측되었다고 가정하겠습니다.

| 구분 | CPU 사용량 |
|---|---:|
| 평균 사용량 | 300m |
| 일반적인 피크 사용량 | 450m |
| 추천 CPU Request | 500m |

이 경우 VPA는 애플리케이션의 일반적인 사용량과 피크 사용량 등을 고려하여 500m의 CPU Request를 추천할 수 있습니다.

CPU Requests가 변경되면 Kubernetes Scheduler가 해당 Pod를 배치할 때 고려하는 리소스 요구량도 변경됩니다.

### 4.3. Memory Recommendation

Memory Recommendation은 컨테이너의 Memory 사용량을 기반으로 적절한 Memory Requests를 계산합니다.

CPU와 달리 Memory는 순간적인 피크 사용량이 중요할 수 있습니다.

애플리케이션이 특정 작업을 수행하는 동안 일시적으로 Memory 사용량이 크게 증가할 수 있으므로 평균 사용량만을 기준으로 리소스를 할당하면 OOM(Out Of Memory)이 발생할 수 있습니다.

따라서 VPA는 Memory 사용량의 피크와 시간에 따른 사용 패턴 등을 고려하여 추천값을 계산합니다.

예를 들어 다음과 같은 사용 패턴을 가정할 수 있습니다.

| 구분 | Memory 사용량 |
|---|---:|
| 평균 사용량 | 400Mi |
| 일반적인 피크 사용량 | 700Mi |
| 추천 Memory Request | 768Mi |

실제 추천값은 수집된 사용량과 VPA의 추천 알고리즘 및 정책에 따라 달라집니다.

Memory Recommendation을 적용할 때는 Memory Limit과 OOM 발생 가능성도 함께 고려해야 합니다.

## 5. VPA 설치 및 구성

VPA는 Kubernetes의 기본 구성 요소가 아니므로 별도의 설치 과정이 필요합니다.

일반적으로 Kubernetes Autoscaler 프로젝트에서 제공하는 VPA 구현체를 사용합니다.

VPA를 설치하면 다음과 같은 구성 요소가 클러스터에 배포됩니다.

- VPA Recommender
- VPA Updater
- VPA Admission Controller
- VerticalPodAutoscaler Custom Resource Definition

설치 방법과 지원 기능은 VPA 버전 및 Kubernetes 버전에 따라 달라질 수 있습니다.

### 5.1. VPA 설치 확인

VPA 구성 요소가 정상적으로 실행되고 있는지 확인하려면 다음 명령어를 사용할 수 있습니다.

```bash
kubectl get pods -n kube-system
```

VPA를 별도의 Namespace에 설치했다면 해당 Namespace를 지정해야 합니다.

VPA의 CRD 등록 여부는 다음 명령어로 확인할 수 있습니다.

```bash
kubectl get crd verticalpodautoscalers.autoscaling.k8s.io
```

설치가 완료되면 다음과 같이 VPA 리소스를 조회할 수 있습니다.

```bash
kubectl get vpa -A
```

## 6. VPA 설정 (autoscaling.k8s.io/v1)

VPA는 `autoscaling.k8s.io` API 그룹에 정의된 `VerticalPodAutoscaler` 리소스를 사용합니다.

다음은 Deployment의 CPU 및 Memory 사용량을 분석하고 리소스를 자동으로 조정하는 VPA 예시입니다.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: application-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: application

  updatePolicy:
    updateMode: Recreate

  resourcePolicy:
    containerPolicies:
      - containerName: application
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 2
          memory: 2Gi
        controlledResources:
          - cpu
          - memory
        controlledValues: RequestsAndLimits
```

각 필드의 의미는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| apiVersion | VPA 리소스의 API 버전입니다. |
| kind | 리소스의 종류를 나타내며 `VerticalPodAutoscaler`를 사용합니다. |
| metadata.name | VPA 리소스의 이름입니다. |
| targetRef | VPA가 관리할 대상 워크로드를 지정합니다. |
| updatePolicy | 추천된 리소스 값을 실제 Pod에 적용하는 방식을 설정합니다. |
| updateMode | 리소스 업데이트 방식을 지정합니다. |
| resourcePolicy | 컨테이너별 리소스 추천 및 적용 정책을 정의합니다. |
| containerPolicies | 개별 컨테이너에 적용할 정책 목록입니다. |
| containerName | 정책을 적용할 컨테이너의 이름입니다. |
| minAllowed | 추천할 수 있는 최소 리소스 값입니다. |
| maxAllowed | 추천할 수 있는 최대 리소스 값입니다. |
| controlledResources | VPA가 관리할 리소스 종류입니다. |
| controlledValues | Requests와 Limits 중 어떤 값을 관리할지 지정합니다. |

위 예시에서는 `application`이라는 Deployment의 `application` 컨테이너를 대상으로 CPU와 Memory를 관리합니다.

CPU는 최소 100m에서 최대 2코어까지, Memory는 최소 128Mi에서 최대 2Gi까지 추천할 수 있도록 제한합니다.

또한 `controlledValues: RequestsAndLimits`를 설정했으므로 Requests와 Limits를 함께 관리하며, Limit은 Request에 비례하도록 조정됩니다.

## 7. VPA Update Mode

VPA는 추천한 리소스 값을 실제 Pod에 적용하는 방식을 `updatePolicy.updateMode`로 제어합니다.

현재 VPA에서 제공하는 주요 Update Mode는 다음과 같습니다.

| Update Mode | 설명 |
|---|---|
| Off | 리소스 추천값만 계산하고 실제 Pod에는 적용하지 않습니다. |
| Initial | Pod가 처음 생성될 때만 추천 리소스 값을 적용합니다. |
| Recreate | 기존 Pod를 Eviction하고 새로운 Pod를 생성하여 리소스 변경 사항을 적용합니다. |
| InPlaceOrRecreate | 가능한 경우 기존 Pod의 리소스를 In-Place 방식으로 변경하고, 불가능하면 Pod를 재생성합니다. |
| InPlace | In-Place 방식으로만 리소스를 변경하며, 실패하더라도 Pod를 Eviction하지 않습니다. |

### 7.1. Off

Off는 VPA가 리소스 추천값을 계산하지만 실제 Pod에는 자동으로 적용하지 않는 모드입니다.

Recommender는 계속해서 리소스 사용량을 분석하고 추천값을 생성하지만, Updater가 기존 Pod를 변경하지 않습니다.

추천값은 VPA 리소스의 `.status.recommendation`에서 확인할 수 있습니다.

```yaml
updatePolicy:
  updateMode: Off
```

주요 활용 사례는 다음과 같습니다.

- 기존 애플리케이션의 적절한 CPU 및 Memory Requests를 분석할 때
- 현재 설정된 Requests와 VPA 추천값을 비교할 때
- 자동 업데이트를 적용하기 전에 추천값의 적절성을 검증할 때
- 리소스 최적화 작업을 위한 사전 분석을 수행할 때

운영 환경에 VPA를 처음 도입하는 경우 Off 모드로 추천값을 관찰한 후 실제 적용 여부를 결정하는 방법을 사용할 수 있습니다.

### 7.2. Initial

Initial은 새로운 Pod가 생성되는 시점에만 추천 리소스 값을 적용하는 모드입니다.

이미 실행 중인 Pod의 리소스는 변경하지 않으며, 이후 추천값이 변경되더라도 기존 Pod에는 영향을 주지 않습니다.

```yaml
updatePolicy:
  updateMode: Initial
```

동작 방식은 다음과 같습니다.

1. Deployment가 새로운 Pod 생성을 요청합니다.
2. Admission Controller가 해당 Pod가 VPA 관리 대상인지 확인합니다.
3. VPA의 추천 리소스 값을 조회합니다.
4. Pod의 CPU 및 Memory Requests에 추천값을 적용합니다.
5. 변경된 Pod가 생성되어 실행됩니다.

Initial 모드는 Pod가 생성될 때만 리소스 추천값을 적용하므로, 실행 중인 Pod를 강제로 재생성하지 않는다는 특징이 있습니다.

다만 이미 실행 중인 Pod는 추천값이 변경되더라도 자동으로 업데이트되지 않습니다.

### 7.3. Recreate

Recreate는 기존 Pod를 Eviction하고 새로운 Pod를 생성하여 추천 리소스 값을 적용하는 모드입니다.

VPA는 현재 Pod의 리소스 설정과 추천값을 비교하고, 변경이 필요하다고 판단하면 Updater가 해당 Pod를 축출합니다.

이후 Deployment 또는 StatefulSet과 같은 상위 Controller가 새로운 Pod를 생성하며, Admission Controller가 추천 리소스 값을 적용합니다.

```yaml
updatePolicy:
  updateMode: Recreate
```

동작 방식은 다음과 같습니다.

1. Recommender가 새로운 리소스 추천값을 계산합니다.
2. Updater가 기존 Pod의 리소스 설정과 추천값을 비교합니다.
3. 리소스 변경이 필요한 Pod를 선택합니다.
4. Updater가 Eviction API를 통해 기존 Pod를 축출합니다.
5. 상위 Controller가 새로운 Pod를 생성합니다.
6. Admission Controller가 새로운 추천 리소스 값을 적용합니다.
7. 새로운 Pod가 실행됩니다.

Recreate 모드는 실행 중인 Pod를 재생성하기 때문에 애플리케이션에 일시적인 중단이 발생할 수 있습니다.

따라서 운영 환경에서는 PodDisruptionBudget(PDB)을 함께 고려하는 것이 중요합니다.

### 7.4. InPlaceOrRecreate

InPlaceOrRecreate는 가능한 경우 기존 Pod를 재생성하지 않고 리소스를 변경하는 방식입니다.

Kubernetes의 In-Place Pod Resize 기능을 활용하여 실행 중인 Pod의 CPU 및 Memory Requests와 Limits를 변경합니다.

```yaml
updatePolicy:
  updateMode: InPlaceOrRecreate
```

동작 방식은 다음과 같습니다.

1. Recommender가 새로운 리소스 추천값을 계산합니다.
2. Updater가 기존 Pod의 리소스 설정과 추천값을 비교합니다.
3. 리소스 변경이 필요하면 In-Place 업데이트를 시도합니다.
4. In-Place 업데이트가 성공하면 기존 Pod를 유지하면서 리소스 설정을 변경합니다.
5. In-Place 업데이트가 불가능하면 Pod를 Eviction하고 재생성하는 방식으로 전환합니다.

주요 특징은 다음과 같습니다.

- 기존 Pod의 재생성을 줄여 애플리케이션 중단을 최소화할 수 있습니다.
- In-Place 업데이트가 가능한 경우 기존 Pod의 UID와 실행 상태를 유지할 수 있습니다.
- Node의 가용 리소스 부족 등으로 In-Place 업데이트가 불가능하면 Recreate 방식으로 전환할 수 있습니다.
- Kubernetes의 In-Place Pod Resize 기능 지원 여부와 VPA 버전에 따라 사용 조건이 달라집니다.

### 7.5. InPlace

InPlace는 Pod를 재생성하지 않고 In-Place 방식으로만 리소스를 변경하는 모드입니다.

```yaml
updatePolicy:
  updateMode: InPlace
```

InPlaceOrRecreate와 달리 In-Place 업데이트가 불가능하더라도 Pod를 Eviction하지 않습니다.

대신 리소스 변경이 가능한 상태가 될 때까지 업데이트를 보류하고 재시도합니다.

주요 특징은 다음과 같습니다.

- 기존 Pod를 Eviction하지 않습니다.
- Pod를 재생성하지 않고 리소스를 변경합니다.
- Node의 자원 부족 등으로 업데이트가 불가능하면 재시도합니다.
- 애플리케이션의 중단을 최소화해야 하는 워크로드에 활용할 수 있습니다.

다만 InPlace 모드는 InPlaceOrRecreate와 동일한 안정화 수준으로 일반화해서 사용해서는 안 됩니다. VPA 버전과 Feature Gate 설정에 따라 지원 상태가 달라질 수 있으며, 현재 VPA v1.7.0 기준 Alpha 기능입니다.

또한 Kubernetes 클러스터에서 `InPlacePodVerticalScaling` 기능이 활성화되어 있어야 하며, VPA의 Updater 및 Admission Controller에도 관련 Feature Gate 설정이 필요합니다.

### 7.6. Update Mode 비교

| 항목 | Off | Initial | Recreate | InPlaceOrRecreate | InPlace |
|---|---|---|---|---|---|
| 추천값 계산 | O | O | O | O | O |
| 신규 Pod에 적용 | X | O | O | O | O |
| 기존 Pod 업데이트 | X | X | O | O | O |
| Pod 재생성 | X | X | O | 필요시 | X |
| In-Place 업데이트 | X | X | X | O | O |
| 업데이트 실패 시 Eviction | X | X | - | O | X |
| 주요 목적 | 추천값 분석 | 신규 Pod 최적화 | 리소스 자동 적용 | 중단 최소화 및 자동 적용 | Pod 재생성 없는 변경 |

참고로 `Auto` Update Mode는 과거 VPA에서 사용되던 모드이며, VPA v1.4.0부터 Deprecated 상태입니다. 현재는 명시적으로 `Recreate` 또는 `InPlaceOrRecreate` 등을 설정하는 것이 권장됩니다.

### 7.7. VPA 및 Kubernetes 버전 호환성

VPA는 Kubernetes와 별도로 배포되는 프로젝트이므로 Update Mode의 지원 여부와 안정화 수준은 VPA 버전 및 Kubernetes 버전에 따라 달라질 수 있습니다.

특히 `InPlaceOrRecreate`와 `InPlace`는 Kubernetes의 In-Place Pod Resize 기능을 활용하므로, Kubernetes 버전과 관련 Feature Gate 설정을 함께 확인해야 합니다.

따라서 In-Place 업데이트를 운영 환경에 적용하기 전에는 다음 사항을 확인해야 합니다.

- 설치된 VPA 버전에서 해당 Update Mode를 지원하는지 확인합니다.
- Kubernetes 버전이 In-Place Pod Resize 기능을 지원하는지 확인합니다.
- 필요한 Feature Gate가 활성화되어 있는지 확인합니다.
- VPA의 Updater 및 Admission Controller에 필요한 설정이 적용되어 있는지 확인합니다.
- 해당 기능의 Alpha, Beta, GA 등 안정화 수준을 확인합니다.

기능의 지원 상태는 버전에 따라 변경될 수 있으므로 실제 클러스터에 적용할 때는 [Kubernetes 공식 문서의 In-Place Pod Resize](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/)와 [VPA 공식 GitHub 저장소](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)의 버전별 문서를 기준으로 확인해야 합니다.

## 8. Resource Policy

Resource Policy는 VPA가 추천하고 관리할 리소스의 범위를 제한하는 설정입니다.

`spec.resourcePolicy` 필드를 통해 컨테이너별로 서로 다른 리소스 정책을 지정할 수 있습니다.

### 8.1. Container Policies

`containerPolicies`는 Pod 내부의 개별 컨테이너에 적용할 리소스 정책을 정의합니다.

예를 들어 하나의 Pod에 애플리케이션 컨테이너와 Sidecar 컨테이너가 함께 실행되는 경우 각각 다른 리소스 제한을 설정할 수 있습니다.

```yaml
resourcePolicy:
  containerPolicies:
    - containerName: application
      minAllowed:
        cpu: 200m
        memory: 256Mi
      maxAllowed:
        cpu: 2
        memory: 2Gi

    - containerName: sidecar
      mode: Off
```

위 예시에서는 다음과 같이 동작합니다.

- `application` 컨테이너는 CPU와 Memory를 자동으로 추천하고 적용합니다.
- `application` 컨테이너의 CPU는 최소 200m, 최대 2코어로 제한됩니다.
- `application` 컨테이너의 Memory는 최소 256Mi, 최대 2Gi로 제한됩니다.
- `sidecar` 컨테이너는 VPA의 자동 리소스 관리를 수행하지 않습니다.

컨테이너 이름 대신 `*`를 지정하면 개별 정책이 없는 컨테이너에 기본 정책을 적용할 수 있습니다.

### 8.2. minAllowed

`minAllowed`는 VPA가 추천할 수 있는 리소스의 최소값을 설정합니다.

```yaml
minAllowed:
  cpu: 100m
  memory: 128Mi
```

위 설정에서는 CPU를 100m 미만으로, Memory를 128Mi 미만으로 추천하지 않도록 제한합니다.

애플리케이션의 최소 실행 요구량을 유지하거나 지나치게 작은 리소스 할당을 방지할 때 활용할 수 있습니다.

### 8.3. maxAllowed

`maxAllowed`는 VPA가 추천할 수 있는 리소스의 최대값을 설정합니다.

```yaml
maxAllowed:
  cpu: 2
  memory: 2Gi
```

위 설정에서는 CPU를 2코어 초과로, Memory를 2Gi 초과로 추천하지 않도록 제한합니다.

특정 애플리케이션이 과도한 리소스를 할당받지 않도록 제한하거나 Node의 자원 용량을 고려한 리소스 관리에 활용할 수 있습니다.

### 8.4. controlledResources

`controlledResources`는 VPA가 관리할 리소스 종류를 지정합니다.

```yaml
controlledResources:
  - cpu
  - memory
```

일반적으로 CPU와 Memory를 모두 관리하지만, 특정 리소스만 관리하도록 설정할 수도 있습니다.

예를 들어 CPU만 관리하려면 다음과 같이 설정합니다.

```yaml
controlledResources:
  - cpu
```

이 경우 VPA는 CPU를 추천 및 관리 대상으로 사용하고 Memory는 자동 관리 대상에서 제외합니다.

### 8.5. controlledValues

`controlledValues`는 VPA가 Requests와 Limits 중 어떤 값을 관리할 것인지 지정합니다.

| 값 | 설명 |
|---|---|
| RequestsAndLimits | Requests와 Limits를 함께 관리합니다. Limit은 Request에 비례하여 조정됩니다. |
| RequestsOnly | Requests만 관리하고 Limits는 변경하지 않습니다. |

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
controlledValues: RequestsOnly
```

이 경우 VPA는 CPU 및 Memory Requests를 추천값에 따라 변경하지만 기존 Limits는 그대로 유지합니다.

반대로 다음과 같이 설정하면 Requests와 Limits를 함께 관리합니다.

```yaml
controlledValues: RequestsAndLimits
```

리소스 Limit까지 자동으로 관리할지 여부는 애플리케이션의 특성과 리소스 제한 정책을 고려하여 결정해야 합니다.

## 9. VPA와 PodDisruptionBudget(PDB)

VPA의 Recreate 모드는 기존 Pod를 Eviction하고 새로운 Pod를 생성하는 방식으로 리소스 변경을 적용합니다.

이 과정에서 여러 Pod가 동시에 종료되면 애플리케이션의 가용성이 저하될 수 있습니다.

이를 방지하기 위해 PodDisruptionBudget(PDB)을 함께 활용할 수 있습니다.

예를 들어 다음과 같은 PDB를 설정할 수 있습니다.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: application-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: application
```

위 설정에서는 자발적 중단(Voluntary Disruption) 이후에도 최소 2개의 Pod가 사용 가능한 상태로 유지되도록 제한합니다.

VPA Updater는 Eviction API를 사용하므로 PDB의 제약을 고려하여 Pod를 축출합니다.

따라서 Recreate 모드를 사용하는 운영 환경에서는 다음 사항을 함께 고려해야 합니다.

- 충분한 Replica 개수 확보
- PDB를 통한 최소 가용 Pod 개수 설정
- Pod의 시작 시간 및 Readiness Probe
- Node의 가용 CPU 및 Memory
- Eviction이 지연될 가능성

PDB가 허용하는 중단 범위를 초과하는 경우 VPA의 Pod 업데이트가 지연될 수 있습니다.

또한 PDB는 자발적 중단을 제어하는 리소스이므로 In-Place 업데이트에는 동일한 방식으로 적용되지 않습니다.

## 10. VPA와 HPA의 조합

VPA와 HPA는 서로 다른 스케일링 방식을 사용하므로 함께 활용할 수 있습니다.

다만 동일한 리소스 메트릭을 기준으로 두 기능이 동시에 동작하면 스케일링 결과가 서로 영향을 줄 수 있습니다.

### 10.1. CPU 및 Memory를 동시에 사용하는 경우

예를 들어 HPA가 CPU 사용률을 기준으로 Pod 개수를 조정하고, VPA가 CPU Requests를 동시에 조정한다고 가정하겠습니다.

HPA는 CPU 사용률을 CPU Requests 대비 실제 사용량으로 계산합니다.

\[
CPU\ Utilization =
\frac{CPU\ Usage}{CPU\ Request} \times 100
\]

이때 VPA가 CPU Requests를 증가시키면 실제 CPU 사용량이 동일하더라도 CPU 사용률은 감소합니다.

반대로 VPA가 CPU Requests를 감소시키면 CPU 사용률은 증가합니다.

결과적으로 다음과 같은 문제가 발생할 수 있습니다.

1. CPU 사용률이 높아져 HPA가 Pod 개수를 증가시킵니다.
2. VPA가 CPU Requests를 증가시킵니다.
3. CPU 사용률 계산의 기준이 변경됩니다.
4. HPA의 스케일링 판단에도 영향을 미칩니다.

따라서 CPU Requests를 관리하는 VPA와 CPU 사용률을 기준으로 동작하는 HPA를 함께 사용할 때는 주의해야 합니다.

### 10.2. 권장 조합 방식

일반적으로 다음과 같은 방식으로 두 기능의 역할을 분리할 수 있습니다.

| HPA 기준 | VPA 관리 대상 | 설명 |
|---|---|---|
| CPU | Memory | HPA는 CPU 사용률을 기준으로 Pod 개수를 조정하고 VPA는 Memory를 최적화합니다. |
| Custom Metrics | CPU / Memory | HPA는 요청량이나 메시지 대기량 등을 기준으로 Pod 개수를 조정하고 VPA는 리소스 Requests를 관리합니다. |
| External Metrics | CPU / Memory | 외부 시스템의 부하를 기준으로 Pod 개수를 조정하고 VPA는 개별 Pod의 리소스를 최적화합니다. |

HPA와 VPA를 함께 사용하는 경우에는 동일한 리소스를 기준으로 서로 다른 스케일링 제어가 발생하지 않도록 설계하는 것이 중요합니다.

## 11. VPA와 Kubernetes Scheduler의 관계

VPA는 단순히 Pod에 할당되는 CPU 및 Memory 값을 변경하는 기능에 그치지 않습니다.

VPA가 추천한 Requests는 Kubernetes Scheduler의 Pod 배치 판단에도 영향을 미칩니다.

Kubernetes Scheduler는 Pod의 Requests를 기준으로 Node에 충분한 리소스가 있는지 확인합니다.

예를 들어 기존 Pod의 CPU Request가 500m였지만 VPA가 1500m를 추천하여 변경했다고 가정하겠습니다.

기존 Node에 500m의 CPU만 추가로 할당할 수 있는 상황이라면 변경된 Pod를 기존 Node에 다시 배치하지 못할 수 있습니다.

이러한 경우 다음과 같은 문제가 발생할 수 있습니다.

- Pod가 Pending 상태에 머무를 수 있습니다.
- 기존 Node의 Allocatable이 부족할 수 있습니다.
- 다른 Node로 스케줄링이 필요할 수 있습니다.
- 클러스터 전체의 Node 자원이 부족하면 추가 Node가 필요할 수 있습니다.

따라서 VPA를 사용할 때는 추천 리소스의 크기뿐만 아니라 클러스터의 전체 자원 용량과 Node별 Allocatable도 함께 고려해야 합니다.

Cluster Autoscaler를 함께 사용하면 새로운 Pod를 배치할 Node 자원이 부족한 상황에서 Node를 자동으로 확장하는 구조를 구성할 수 있습니다.

## 12. VPA 관련 주요 명령어

| 명령어 | 설명 |
|---|---|
| `kubectl get vpa` | 현재 Namespace의 VPA 목록을 확인합니다. |
| `kubectl get vpa -A` | 모든 Namespace의 VPA 목록을 확인합니다. |
| `kubectl describe vpa <name>` | 특정 VPA의 상세 설정과 상태를 확인합니다. |
| `kubectl get vpa <name> -o yaml` | VPA 리소스의 전체 YAML 정보를 확인합니다. |
| `kubectl get crd verticalpodautoscalers.autoscaling.k8s.io` | VPA CRD 등록 여부를 확인합니다. |
| `kubectl top pods` | Pod의 CPU 및 Memory 사용량을 확인합니다. |
| `kubectl top pods --containers` | Pod 내부의 컨테이너별 CPU 및 Memory 사용량을 확인합니다. |
| `kubectl get pods -n kube-system` | kube-system Namespace의 VPA 구성 요소 상태를 확인합니다. |

VPA의 추천 리소스 값을 확인하려면 다음 명령어를 사용할 수 있습니다.

```bash
kubectl get vpa application-vpa -o yaml
```

출력 결과의 `.status.recommendation`에서 컨테이너별 추천값을 확인할 수 있습니다.

특정 컨테이너의 추천값만 확인하려면 다음 명령어를 사용할 수 있습니다.

```bash
kubectl get vpa application-vpa \
  -o jsonpath='{.status.recommendation.containerRecommendations}'
```

VPA가 관리하는 Pod의 실제 리소스 설정은 다음 명령어로 확인할 수 있습니다.

```bash
kubectl get pods -l app=application -o yaml
```

이를 통해 VPA의 추천값과 실제 Pod에 적용된 Requests 및 Limits를 비교할 수 있습니다.

## 13. VPA 운영 시 고려사항

VPA를 실제 운영 환경에 적용할 때는 다음 사항을 고려해야 합니다.

### 13.1. Pod 재생성에 따른 서비스 영향

Recreate 모드는 기존 Pod를 Eviction하고 새로운 Pod를 생성하므로 애플리케이션의 일시적인 중단이 발생할 수 있습니다.

특히 Replica가 1개인 애플리케이션은 Pod가 재생성되는 동안 서비스가 중단될 가능성이 있습니다.

따라서 운영 환경에서는 충분한 Replica 개수와 PDB를 함께 고려해야 합니다.

### 13.2. Resource Requests와 Limits의 관계

VPA는 설정에 따라 Requests와 Limits를 함께 관리할 수 있습니다.

특히 Requests와 Limits의 비율을 유지하도록 구성할 수 있으므로 기존 애플리케이션의 리소스 제한 정책을 확인해야 합니다.

Requests만 관리하려는 경우 `controlledValues: RequestsOnly`를 사용할 수 있습니다.

### 13.3. Memory 사용량과 OOM

Memory는 CPU와 달리 일시적인 사용량 증가가 애플리케이션의 OOM으로 이어질 수 있습니다.

따라서 Memory Recommendation을 적용할 때는 평균 사용량뿐만 아니라 피크 사용량과 OOM 발생 이력 등을 고려해야 합니다.

VPA는 OOM 이벤트를 기반으로 Memory 추천값을 상향 조정하는 기능도 제공합니다.

### 13.4. Node 자원 부족

VPA가 Pod의 Requests를 증가시키면 Scheduler가 새로운 리소스 요구량을 충족할 수 있는 Node를 찾아야 합니다.

클러스터의 전체 자원이 부족하다면 Pod가 Pending 상태에 머무를 수 있습니다.

따라서 VPA를 적용할 때는 Node의 Capacity와 Allocatable, Cluster Autoscaler의 동작 여부 등을 확인해야 합니다.

### 13.5. JVM 기반 애플리케이션

JVM 기반 애플리케이션은 Memory 사용량을 해석할 때 주의가 필요합니다.

Java 애플리케이션은 Heap뿐만 아니라 Metaspace, Thread Stack, Direct Buffer 및 JVM Native Memory 등 다양한 영역을 사용합니다.

컨테이너의 Memory 사용량만으로 JVM 내부의 실제 메모리 요구량을 정확하게 판단하기 어려울 수 있습니다.

따라서 JVM 기반 애플리케이션에 VPA를 적용할 때는 Heap 설정과 컨테이너 Memory Limit, Native Memory 사용량 등을 함께 고려해야 합니다.

### 13.6. HPA와의 충돌

HPA와 VPA가 동일한 CPU 또는 Memory 메트릭을 기준으로 동작하면 서로 간섭할 수 있습니다.

특히 CPU Utilization 기반 HPA와 CPU Requests를 변경하는 VPA를 함께 사용할 때는 스케일링 결과가 불안정해질 수 있습니다.

두 기능을 함께 사용하는 경우 메트릭 기준과 리소스 관리 범위를 명확하게 분리하는 것이 중요합니다.

## 14. 정리

VerticalPodAutoscaler(VPA)는 Kubernetes에서 Pod의 CPU 및 Memory 리소스 할당량을 자동으로 최적화하는 기능입니다.

VPA는 Recommender, Updater, Admission Controller의 세 가지 주요 구성 요소를 통해 동작합니다.

Recommender는 과거 및 현재 리소스 사용량을 분석하여 적절한 리소스 Requests를 추천하고, Updater는 추천값을 실제 Pod에 적용할지 결정합니다. Admission Controller는 새로운 Pod가 생성될 때 추천된 리소스 값을 적용합니다.

VPA는 Off, Initial, Recreate, InPlaceOrRecreate 및 InPlace 등의 Update Mode를 제공하여 리소스 변경 방식을 제어할 수 있습니다.

또한 Resource Policy를 통해 컨테이너별 최소 및 최대 리소스 할당량과 관리 대상 리소스를 지정할 수 있습니다.

HPA가 Pod 개수를 조정하는 방식이라면 VPA는 개별 Pod의 리소스 할당량을 조정하는 방식입니다. 두 기능을 함께 활용하면 애플리케이션의 처리 용량과 개별 Pod의 리소스 효율성을 함께 관리할 수 있습니다.

따라서 VPA는 Kubernetes 클러스터에서 리소스 Requests를 최적화하고 컴퓨팅 자원의 효율적인 활용을 지원하는 핵심적인 자동 리소스 관리 기능입니다.

## 15. 참고자료

- [Kubernetes 공식 문서 - Vertical Pod Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/)
- [Kubernetes Autoscaler 공식 GitHub - Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [Kubernetes Autoscaler 공식 문서 - VPA Components](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/docs/components.md)
- [Kubernetes Autoscaler 공식 문서 - VPA API Reference](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/docs/api.md)
- [Kubernetes Autoscaler 공식 문서 - VPA Quick Start](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/docs/quickstart.md)
- [Kubernetes Autoscaler 공식 문서 - VPA Features](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/docs/features.md)
- [Kubernetes 공식 문서 - In-Place Pod Resize](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/)
- [Kubernetes 공식 문서 - Horizontal Pod Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)
- [Kubernetes 공식 문서 - Pod Disruption Budget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)