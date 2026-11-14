

# **ReplicaSet**

## **1. ReplicaSet이란?**

**ReplicaSet은 Kubernetes에서 지정된 수의 Pod 복제본(Replica)이 유지되도록 관리하는 Workload 리소스입니다.**

ReplicaSet은 사용자가 선언한 원하는 Pod 개수와 실제 관리 대상 Pod 개수를 지속적으로 비교하고, 두 값이 일치하도록 Pod를 생성하거나 삭제합니다.

예를 들어 ReplicaSet에 `replicas: 3`으로 설정하면 Kubernetes는 해당 ReplicaSet이 관리하는 Pod가 3개가 되도록 유지합니다.

Pod가 삭제되거나 비정상적으로 종료되어 관리 대상 Pod 수가 2개로 줄어들면 ReplicaSet Controller가 새로운 Pod를 생성하여 복제본 개수를 복구합니다.

ReplicaSet의 주요 특징은 다음과 같습니다.

- 지정된 수의 Pod 복제본을 유지합니다.
- Pod의 생성 및 삭제를 자동으로 관리합니다.
- Label Selector를 통해 관리 대상 Pod를 식별합니다.
- Pod Template을 기반으로 새로운 Pod를 생성합니다.
- Deployment와 함께 사용되어 애플리케이션의 배포 및 업데이트를 지원합니다.
- 일반적으로 동일한 구성을 가진 애플리케이션 Pod를 여러 개 실행할 때 사용합니다.

ReplicaSet은 Pod 복제본 개수를 유지하는 데 집중하며, 애플리케이션의 버전 관리나 Rolling Update와 같은 배포 전략은 직접 제공하지 않습니다.

이러한 기능은 ReplicaSet을 상위에서 관리하는 Deployment가 담당합니다.

ReplicaSet이 Pod 개수를 유지한다고 해서 애플리케이션이 반드시 정상적으로 동작하거나 요청을 처리할 수 있다는 의미는 아닙니다. ReplicaSet은 Pod의 개수를 기준으로 동작하며, 애플리케이션의 비즈니스 상태나 데이터 정합성을 직접 확인하지 않습니다.

## **2. ReplicaSet의 구조**

ReplicaSet은 자체적으로 컨테이너를 실행하지 않습니다. ReplicaSet Controller가 Pod를 생성하고 관리하며, 생성된 Pod는 Scheduler와 Kubelet 등의 구성 요소를 통해 실제 노드에서 실행됩니다.

전체 구조는 다음과 같습니다.

```text
Kubernetes Cluster
│
└── ReplicaSet
    │
    ├── Pod
    │   └── Container
    │
    ├── Pod
    │   └── Container
    │
    └── Pod
        └── Container
```

예를 들어 다음과 같이 ReplicaSet을 구성할 수 있습니다.

```text
ReplicaSet: nginx-replicaset
replicas: 3
│
├── Pod: nginx-replicaset-abcde
├── Pod: nginx-replicaset-fghij
└── Pod: nginx-replicaset-klmno
```

ReplicaSet은 지정된 복제본 개수와 실제 관리 대상 Pod 개수를 비교하여 차이가 발생하면 이를 조정합니다.

### **2.1. ReplicaSet Controller**

ReplicaSet Controller는 ReplicaSet의 원하는 상태와 실제 상태를 지속적으로 비교하고 조정하는 Kubernetes Controller입니다.

예를 들어 다음과 같은 ReplicaSet이 존재한다고 가정하겠습니다.

```yaml
spec:
  replicas: 3
```

현재 관리 대상 Pod가 2개라면 ReplicaSet Controller는 새로운 Pod를 생성합니다.

반대로 관리 대상 Pod가 4개라면 초과된 Pod를 삭제하여 개수를 조정합니다.

```text
ReplicaSet
replicas: 3
     │
     ▼
ReplicaSet Controller
     │
     ▼
현재 관리 대상 Pod 개수 확인
     │
     ├── Pod 2개 → Pod 1개 생성
     │
     ├── Pod 3개 → 변경 없음
     │
     └── Pod 4개 → Pod 1개 삭제
```

이러한 동작을 Reconciliation Loop(조정 루프)라고 합니다.

ReplicaSet Controller는 Pod가 삭제되거나 노드 장애 등의 이유로 관리 대상 Pod의 개수가 변경되더라도 원하는 복제본 개수를 유지하도록 동작합니다.

다만 노드 자원 부족, 이미지 다운로드 실패, 스케줄링 제약 등의 문제가 있으면 새로운 Pod가 생성되더라도 실제로 실행 가능한 상태에 도달하지 못할 수 있습니다.

## **3. ReplicaSet YAML 구성**

ReplicaSet은 YAML Manifest를 통해 정의할 수 있습니다.

다음은 Nginx 컨테이너를 3개의 Pod로 실행하는 ReplicaSet 예제입니다.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  labels:
    app: nginx
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
          image: nginx:1.27
          ports:
            - containerPort: 80
```

각 필드의 역할은 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `apiVersion` | Kubernetes API 버전을 지정합니다. ReplicaSet은 `apps/v1`을 사용합니다. |
| `kind` | 생성할 리소스의 종류를 지정합니다. |
| `metadata` | ReplicaSet의 이름과 레이블 등의 메타데이터를 정의합니다. |
| `spec` | ReplicaSet의 원하는 상태를 정의합니다. |
| `replicas` | 유지하려는 Pod 복제본의 개수를 지정합니다. |
| `selector` | ReplicaSet이 관리할 Pod를 식별하는 조건을 지정합니다. |
| `template` | 새로운 Pod를 생성할 때 사용할 Pod Template을 정의합니다. |
| `template.metadata` | 생성될 Pod의 레이블 등의 메타데이터를 정의합니다. |
| `template.spec` | 생성될 Pod의 컨테이너 및 실행 환경을 정의합니다. |

### **3.1. replicas**

`replicas`는 ReplicaSet이 유지하려는 Pod 복제본의 개수를 지정합니다.

```yaml
spec:
  replicas: 3
```

위 설정은 동일한 Pod Template을 기반으로 한 Pod 3개를 유지하도록 지정합니다.

현재 관리 대상 Pod가 2개라면 ReplicaSet Controller가 Pod를 추가로 생성하고, 4개라면 초과된 Pod를 삭제합니다.

`replicas`를 지정하지 않으면 기본값은 1입니다.

`replicas`는 원하는 복제본 개수이므로 실제 실행 중이거나 Ready 상태인 Pod 개수와 항상 일치하는 것은 아닙니다. Pod가 Pending, ContainerCreating 또는 CrashLoopBackOff 상태인 경우에도 ReplicaSet은 원하는 개수를 충족하기 위한 조정을 계속 수행합니다.

### **3.2. selector**

`selector`는 ReplicaSet이 관리할 Pod를 식별하기 위한 조건입니다.

```yaml
spec:
  selector:
    matchLabels:
      app: nginx
```

위 설정은 `app: nginx` 레이블을 가진 Pod를 관리 대상으로 선택합니다.

ReplicaSet의 Selector는 Pod Template에 정의된 레이블과 일치해야 합니다.

```yaml
spec:
  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx
```

`apps/v1` API에서 ReplicaSet의 Selector는 필수이며, 생성 이후 변경할 수 없는 필드입니다.

또한 동일한 Pod를 여러 ReplicaSet이 중복으로 관리하지 않도록 Selector를 명확하게 정의해야 합니다.

ReplicaSet은 Selector에 일치하는 Pod 중 자신이 소유한 Pod와 그렇지 않은 Pod를 구분하여 관리합니다. 소유자가 없는 기존 Pod가 Selector에 일치하면 해당 Pod를 자신의 관리 대상으로 채택할 수도 있습니다. 반면 다른 Controller가 소유한 Pod를 무조건 가져와 관리하는 것은 아닙니다.

### **3.3. template**

`template`은 ReplicaSet이 새로운 Pod를 생성할 때 사용하는 템플릿입니다.

```yaml
spec:
  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

Pod Template에는 컨테이너 이미지, 포트, 환경 변수, 리소스 요청량, 볼륨 등의 Pod 실행 환경을 정의할 수 있습니다.

ReplicaSet은 이 Template을 기반으로 필요한 Pod를 생성합니다.

다만 ReplicaSet은 Deployment와 달리 Pod Template이 변경되었다고 해서 기존 Pod를 새로운 구성으로 자동 교체하지 않습니다.

새로운 Pod를 생성해야 하는 상황이 발생하면 변경된 Template을 사용하지만, 기존 Pod는 기존 설정으로 계속 실행됩니다.

이 부분은 Deployment와 ReplicaSet의 중요한 차이점입니다.

## **4. ReplicaSet의 동작 방식**

ReplicaSet은 선언적으로 원하는 복제본 개수를 정의하고, Kubernetes Controller가 실제 상태를 원하는 상태에 맞추는 방식으로 동작합니다.

### **4.1. ReplicaSet 생성 과정**

ReplicaSet을 생성하면 Kubernetes 내부에서 다음과 같은 작업이 진행됩니다.

```text
kubectl apply -f replicaset.yaml
              │
              ▼
         API Server
              │
              ▼
       ReplicaSet 생성
              │
              ▼
     ReplicaSet Controller
              │
              ▼
       Selector 확인
              │
              ▼
       현재 Pod 개수 확인
              │
              ▼
        부족한 Pod 생성
              │
              ▼
          Scheduler
              │
              ▼
        Node 선택 및 배치
              │
              ▼
           Kubelet
              │
              ▼
         Container 실행
```

각 구성 요소의 역할은 다음과 같습니다.

1. **API Server:** ReplicaSet 생성 요청을 받아 유효성을 검증하고 Kubernetes API 객체로 저장합니다.
2. **ReplicaSet Controller:** ReplicaSet의 원하는 복제본 개수와 실제 관리 대상 Pod 개수를 비교하고 필요한 Pod를 생성하거나 삭제합니다.
3. **Scheduler:** 새롭게 생성된 Pod 중 아직 Node가 할당되지 않은 Pod가 실행될 Node를 결정합니다.
4. **Kubelet:** 선택된 Node에서 Pod의 실행 상태를 관리합니다.
5. **Container Runtime:** 컨테이너 이미지를 가져오고 실제 컨테이너를 실행합니다.

ReplicaSet Controller가 Pod 객체를 생성하더라도 컨테이너를 직접 실행하는 것은 아닙니다. Pod가 실행될 Node를 선택하는 작업은 Scheduler가 담당하고, 해당 Node에서 컨테이너를 실행하고 관리하는 작업은 Kubelet과 Container Runtime이 담당합니다.

### **4.2. Pod 장애 발생 시 복구**

ReplicaSet이 관리하는 Pod가 삭제되거나 비정상적으로 종료되는 경우 ReplicaSet Controller는 현재 관리 대상 Pod 개수를 확인합니다.

예를 들어 3개의 Pod를 실행 중인 ReplicaSet에서 Pod 하나가 삭제되었다고 가정하겠습니다.

```text
Before

ReplicaSet
replicas: 3
│
├── Pod A
├── Pod B
└── Pod C


Pod C 삭제

ReplicaSet
replicas: 3
│
├── Pod A
└── Pod B


ReplicaSet Controller
현재 관리 대상 Pod 개수: 2
목표 Pod 개수: 3
        │
        ▼
새로운 Pod 생성


After

ReplicaSet
replicas: 3
│
├── Pod A
├── Pod B
└── Pod D
```

ReplicaSet Controller가 새로운 Pod를 생성하여 원하는 복제본 개수를 유지합니다.

단, 기존 Pod의 상태를 복구하는 것이 아니라 새로운 Pod를 생성하는 방식입니다. 따라서 기존 Pod의 UID, IP 주소, 컨테이너 내부의 임시 데이터 및 메모리 상태는 유지되지 않습니다.

Pod에 영구 저장소를 연결한 경우에도 실제 데이터의 보존 여부는 사용한 볼륨과 스토리지 구성에 따라 달라집니다.

### **4.3. Pod를 수동으로 삭제한 경우**

ReplicaSet이 관리하는 Pod를 `kubectl delete pod` 명령어로 직접 삭제하더라도 ReplicaSet Controller는 삭제 사실을 감지하고 새로운 Pod를 생성합니다.

예를 들어 다음 명령어를 실행했다고 가정하겠습니다.

```bash
kubectl delete pod nginx-replicaset-abcde
```

해당 Pod가 삭제되면 ReplicaSet은 관리 중인 Pod 개수가 감소한 것을 확인하고 새로운 Pod를 생성합니다.

따라서 ReplicaSet이 관리하는 Pod를 직접 삭제하는 것은 일반적으로 애플리케이션의 복제본 개수를 줄이는 방법이 아닙니다.

복제본 개수를 변경하려면 ReplicaSet의 `replicas` 값을 수정해야 합니다.

## **5. ReplicaSet의 Label Selector**

ReplicaSet은 Label Selector를 통해 관리할 Pod를 식별합니다.

Label은 Kubernetes 리소스에 부여하는 Key-Value 형태의 메타데이터이며, Selector는 특정 레이블을 가진 리소스를 선택하는 조건입니다.

예를 들어 다음과 같은 Pod가 존재한다고 가정하겠습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    environment: production
spec:
  containers:
    - name: nginx
      image: nginx:1.27
```

다음과 같이 ReplicaSet의 Selector를 설정하면 `app: nginx` 레이블을 가진 Pod를 관리 대상으로 선택합니다.

```yaml
spec:
  selector:
    matchLabels:
      app: nginx
```

ReplicaSet은 Selector에 일치하는 Pod를 확인하여 원하는 복제본 개수와 비교합니다.

### **5.1. matchLabels**

`matchLabels`는 지정한 Key-Value 레이블이 모두 일치하는 리소스를 선택합니다.

```yaml
selector:
  matchLabels:
    app: nginx
    environment: production
```

위 설정은 다음 두 조건을 모두 만족하는 Pod를 선택합니다.

- `app: nginx`
- `environment: production`

### **5.2. matchExpressions**

`matchExpressions`는 레이블에 대해 보다 다양한 선택 조건을 지정할 수 있습니다.

```yaml
selector:
  matchExpressions:
    - key: environment
      operator: In
      values:
        - production
        - staging
```

위 설정은 `environment` 레이블의 값이 `production` 또는 `staging`인 Pod를 선택합니다.

주요 Operator는 다음과 같습니다.

| Operator | 설명 |
|---|---|
| `In` | 지정한 값 중 하나와 일치하는 경우 선택합니다. |
| `NotIn` | 지정한 값에 포함되지 않는 경우 선택합니다. |
| `Exists` | 지정한 레이블 Key가 존재하는 경우 선택합니다. |
| `DoesNotExist` | 지정한 레이블 Key가 존재하지 않는 경우 선택합니다. |

`matchLabels`와 `matchExpressions`를 함께 사용하는 경우 모든 조건을 만족해야 합니다.

### **5.3. Selector와 Pod Template의 관계**

ReplicaSet의 Selector는 기존 Pod를 관리 대상으로 식별하는 조건이며, Pod Template은 새로운 Pod를 생성할 때 사용할 설정입니다.

두 설정은 서로 다른 역할을 담당하지만 반드시 일치해야 합니다.

```yaml
spec:
  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx
```

`apps/v1` API에서는 Pod Template의 레이블이 Selector와 일치하지 않으면 ReplicaSet 생성 요청이 거부됩니다.

또한 ReplicaSet의 Selector가 너무 광범위하게 설정되면 의도하지 않은 기존 Pod까지 관리 대상으로 선택할 수 있으므로 주의해야 합니다.

특히 서로 다른 ReplicaSet이 겹치는 Selector를 사용하면 Pod 소유권이 예상과 다르게 변경되거나 복제본 관리가 충돌할 수 있습니다. 각 ReplicaSet에는 다른 ReplicaSet과 구분할 수 있는 명확한 Selector를 사용하는 것이 좋습니다.

## **6. ReplicaSet의 스케일링 (Scaling)**

ReplicaSet은 `replicas` 값을 변경하여 관리하는 Pod의 개수를 조절할 수 있습니다.

### **6.1. 수동 스케일링**

`kubectl scale` 명령어를 사용하여 Pod 복제본 개수를 변경할 수 있습니다.

기존 ReplicaSet의 Pod 개수를 3개에서 5개로 변경하는 예제입니다.

```bash
kubectl scale replicaset nginx-replicaset --replicas=5
```

또는 YAML Manifest에서 `replicas` 값을 변경할 수 있습니다.

```yaml
spec:
  replicas: 5
```

복제본 개수가 변경되면 ReplicaSet Controller가 이에 맞게 Pod를 생성하거나 삭제합니다.

스케일링은 원하는 복제본 개수를 변경하는 작업이며, 실제 Pod의 생성과 삭제는 Controller의 조정 과정을 통해 수행됩니다.

### **6.2. HorizontalPodAutoscaler 연동**

ReplicaSet은 HorizontalPodAutoscaler(HPA)와 연동하여 Pod 개수를 자동으로 조절할 수 있습니다.

HPA는 CPU 사용률, 메모리 사용률 또는 사용자 정의 메트릭 등을 기준으로 적절한 복제본 개수를 계산하고 대상 리소스의 Scale subresource를 통해 복제본 개수를 조정할 수 있습니다.

ReplicaSet을 직접 대상으로 지정하면 ReplicaSet의 복제본 개수를 조절할 수 있습니다.

```text
ReplicaSet
replicas: 3
     │
     ▼
HorizontalPodAutoscaler
     │
     ▼
메트릭 수집 및 목표 사용률 비교
     │
     ├── 사용률 증가
     │       │
     │       ▼
     │    Pod 증가
     │
     └── 사용률 감소
             │
             ▼
          Pod 감소
```

다만 실제 운영 환경에서는 HPA를 ReplicaSet에 직접 연결하기보다는 Deployment에 연결하는 것이 일반적입니다.

Deployment를 대상으로 HPA를 구성하면 HPA가 Deployment의 복제본 개수를 조정하고, Deployment가 이를 관리하는 ReplicaSet에 반영합니다.

또한 HPA와 함께 사용할 경우 YAML Manifest에서 `spec.replicas`를 반복적으로 적용하면 HPA가 조정한 값이 다시 덮어써질 수 있으므로, 운영 방식에 맞게 복제본 개수의 관리 주체를 정해야 합니다.

## **7. ReplicaSet과 Deployment의 관계**

ReplicaSet과 Deployment는 밀접하게 연관되어 있지만 담당하는 역할에는 차이가 있습니다.

ReplicaSet은 Pod의 복제본 개수를 유지하는 데 집중하며, Deployment는 ReplicaSet을 관리하여 애플리케이션의 배포와 업데이트를 수행합니다.

일반적인 Kubernetes 애플리케이션 배포 구조는 다음과 같습니다.

```text
Deployment
│
├── ReplicaSet v1
│   ├── Pod A
│   ├── Pod B
│   └── Pod C
│
└── ReplicaSet v2
    ├── Pod D
    ├── Pod E
    └── Pod F
```

애플리케이션 버전을 변경하면 Deployment는 새로운 Pod Template을 가진 ReplicaSet을 생성하고, 배포 전략에 따라 기존 ReplicaSet의 복제본을 줄이는 동시에 새로운 ReplicaSet의 복제본을 늘립니다.

이러한 방식으로 Rolling Update를 수행합니다.

### **7.1. Deployment가 ReplicaSet을 사용하는 이유**

ReplicaSet은 Pod 복제본을 유지하는 기능만 제공합니다.

애플리케이션 업데이트를 위해 Pod Template이 변경되더라도 기존 Pod를 자동으로 새로운 구성으로 교체하지 않습니다.

예를 들어 ReplicaSet에서 다음과 같이 이미지를 변경했다고 가정하겠습니다.

```yaml
spec:
  template:
    spec:
      containers:
        - name: nginx
          image: nginx:1.28
```

기존 Pod는 이전 이미지인 `nginx:1.27`을 사용하고 있더라도 계속 실행됩니다.

ReplicaSet은 새로운 Pod가 필요한 상황에서 변경된 Template을 사용하여 새로운 Pod를 생성하지만, 기존 Pod를 업데이트된 Template에 맞게 자동으로 교체하지는 않습니다.

반면 Deployment는 Pod Template의 변경을 감지하여 새로운 ReplicaSet을 생성하고, RollingUpdate 또는 Recreate 전략에 따라 Pod를 교체합니다.

### **7.2. Deployment와 ReplicaSet의 역할 비교**

| 구분 | Deployment | ReplicaSet |
|---|---|---|
| 주요 목적 | 애플리케이션 배포 및 업데이트 관리 | Pod 복제본 개수 유지 |
| Pod 생성 | ReplicaSet을 통해 간접적으로 관리 | 직접 관리 |
| 복제본 관리 | ReplicaSet을 통해 관리 | 직접 관리 |
| Rolling Update | 지원 | 직접 지원하지 않음 |
| Rollback | 이전 ReplicaSet Revision을 활용하여 지원 | 배포 Revision을 관리하는 기능은 제공하지 않음 |
| Pod Template 변경 | 새로운 ReplicaSet을 생성하여 업데이트 | 기존 Pod를 자동 교체하지 않음 |
| 일반적인 사용 방식 | 애플리케이션 배포에 직접 사용 | Deployment의 하위 리소스로 사용 |

**일반적인 애플리케이션 배포에서는 ReplicaSet을 직접 생성하기보다 Deployment를 사용하는 것이 권장됩니다.**

Deployment가 ReplicaSet의 생성과 관리를 담당하므로 ReplicaSet을 직접 관리할 필요가 줄어들고, Rolling Update와 Rollback 등의 기능을 활용할 수 있기 때문입니다.

## **8. ReplicaSet과 다른 Workload 리소스의 관계**

Kubernetes에는 Pod를 관리하는 다양한 Workload 리소스가 존재합니다.

각 리소스는 애플리케이션의 특성과 실행 목적에 따라 서로 다른 방식으로 Pod를 관리합니다.

| 리소스 | Pod 관리 방식 | 주요 사용 사례 |
|---|---|---|
| ReplicaSet | 지정된 수의 Pod 복제본 유지 | Pod 복제본 관리 |
| Deployment | ReplicaSet을 통해 Pod 관리 | 일반적인 애플리케이션 배포 |
| StatefulSet | 안정적인 식별자와 스토리지를 가진 Pod 관리 | 데이터베이스, 분산 시스템 |
| DaemonSet | 각 노드 또는 지정된 노드에 Pod 배치 | 로그 수집기, 모니터링 에이전트 |
| Job | 작업이 성공적으로 완료될 때까지 Pod 실행 | 배치 작업, 데이터 처리 |
| CronJob | 일정에 따라 Job 생성 | 정기적인 배치 작업 |

ReplicaSet은 Deployment의 하위 리소스로 가장 많이 사용되지만, StatefulSet이나 DaemonSet과 같은 다른 Workload 리소스가 ReplicaSet을 통해 Pod를 관리하는 것은 아닙니다.

각 Workload 리소스는 서로 다른 Controller를 통해 Pod를 관리합니다.

## **9. ReplicaSet 관리 명령어**

ReplicaSet을 생성하고 관리할 때 사용하는 주요 kubectl 명령어는 다음과 같습니다.

| 명령어 | 설명 |
|---|---|
| `kubectl get replicasets` | 현재 네임스페이스의 ReplicaSet 목록을 조회합니다. |
| `kubectl get rs` | ReplicaSet 목록을 간략하게 조회합니다. |
| `kubectl describe rs <name>` | ReplicaSet의 상세 정보와 이벤트를 조회합니다. |
| `kubectl apply -f replicaset.yaml` | YAML Manifest를 기반으로 ReplicaSet을 생성하거나 변경합니다. |
| `kubectl delete rs <name>` | ReplicaSet을 삭제합니다. |
| `kubectl scale rs <name> --replicas=5` | ReplicaSet의 Pod 복제본 개수를 변경합니다. |
| `kubectl get rs <name> -o yaml` | ReplicaSet의 전체 YAML 정보를 조회합니다. |
| `kubectl get pods --show-labels` | Pod 목록과 각 Pod의 레이블을 함께 조회합니다. |

ReplicaSet과 해당 ReplicaSet이 관리하는 Pod를 함께 확인하려면 다음 명령어를 사용할 수 있습니다.

```bash
kubectl get rs,pods
```

특정 ReplicaSet의 Selector에 해당하는 Pod를 조회하려면 다음과 같이 실행합니다.

```bash
kubectl get pods -l app=nginx
```

특정 ReplicaSet이 관리하는 Pod의 소유 관계를 확인하려면 다음 명령어를 사용할 수 있습니다.

```bash
kubectl get pods -l app=nginx -o wide
kubectl describe rs nginx-replicaset
```

## **10. ReplicaSet 사용 시 고려사항**

### **10.1. 일반적인 애플리케이션 배포에는 Deployment를 사용합니다.**

ReplicaSet은 Pod 복제본을 유지하는 기능에 집중합니다.

Rolling Update, Rollback, 배포 이력 관리 등의 기능이 필요하다면 Deployment를 사용하는 것이 적절합니다.

### **10.2. Selector를 신중하게 설정해야 합니다.**

ReplicaSet은 Selector를 기반으로 관리 대상 Pod를 식별합니다.

Selector가 너무 광범위하게 설정되면 의도하지 않은 Pod까지 관리 대상으로 포함될 수 있습니다.

특히 여러 ReplicaSet이 동일한 Pod를 대상으로 지정하지 않도록 주의해야 합니다.

### **10.3. Pod Template 변경이 기존 Pod에 자동으로 반영되지 않습니다.**

ReplicaSet의 Pod Template을 변경하더라도 기존 Pod의 설정이 자동으로 변경되지는 않습니다.

새로운 Pod가 생성될 때 변경된 Template이 적용됩니다.

기존 Pod까지 새로운 구성으로 교체해야 한다면 Deployment를 사용하는 것이 적절합니다.

### **10.4. ReplicaSet은 Pod의 데이터와 상태를 보존하지 않습니다.**

ReplicaSet은 Pod의 개수를 유지하는 역할을 수행할 뿐, Pod 내부에 저장된 데이터를 보존하는 기능은 제공하지 않습니다.

Pod가 삭제되고 새로운 Pod가 생성되면 기존 Pod의 임시 파일이나 메모리 상태는 유지되지 않습니다.

영구적인 데이터 저장이 필요한 경우 PersistentVolumeClaim(PVC) 등을 사용해야 합니다. 단, PVC를 사용하더라도 데이터의 보존 여부는 스토리지 구성과 PVC의 생명주기 정책에 따라 달라집니다.

### **10.5. ReplicaSet 삭제 시 하위 Pod도 삭제됩니다.**

ReplicaSet을 일반적인 방식으로 삭제하면 해당 ReplicaSet이 소유한 Pod도 함께 삭제됩니다.

이는 Kubernetes의 OwnerReference와 Garbage Collection 정책에 따라 하위 리소스가 정리되기 때문입니다.

특정 상황에서 ReplicaSet만 삭제하고 Pod를 유지하려면 `--cascade=orphan` 옵션을 사용할 수 있습니다.

```bash
kubectl delete rs nginx-replicaset --cascade=orphan
```

이 경우 ReplicaSet은 삭제되지만 기존 Pod는 남아 있게 됩니다. 남아 있는 Pod는 기존 ReplicaSet의 소유 관계에서 분리되며, 이후 다른 Controller가 해당 Pod를 관리 대상으로 채택할 수도 있습니다.

따라서 이 옵션을 사용할 때는 다른 Controller의 Selector와 Pod 소유 관계를 신중하게 확인해야 합니다.

## **11. 핵심 정리**

ReplicaSet은 Kubernetes에서 지정된 수의 Pod 복제본이 유지되도록 관리하는 Workload 리소스입니다.

- `replicas`를 통해 원하는 Pod 복제본 개수를 지정합니다.
- `selector`를 통해 관리 대상 Pod를 식별합니다.
- `template`을 통해 새로운 Pod의 구성을 정의합니다.
- ReplicaSet Controller가 실제 Pod 개수를 지속적으로 확인하고 조정합니다.
- Pod가 삭제되거나 장애가 발생하면 새로운 Pod를 생성하여 복제본 개수를 유지합니다.
- HPA와 연동하여 Pod 개수를 자동으로 조절할 수 있습니다.
- Pod Template이 변경되어도 기존 Pod를 자동으로 업데이트하지 않습니다.
- Deployment는 ReplicaSet을 관리하여 Rolling Update, Rollback 등의 배포 기능을 제공합니다.

**ReplicaSet은 Pod의 복제본 개수를 유지하는 데 집중하는 리소스이며, Deployment는 ReplicaSet을 활용하여 애플리케이션의 배포와 업데이트를 관리하는 상위 리소스입니다.**

따라서 Kubernetes의 일반적인 애플리케이션 배포에서는 Deployment를 사용하고, ReplicaSet은 Deployment 내부에서 Pod 복제본을 관리하는 구성 요소로 이해하는 것이 중요합니다.

## **참고자료**

- [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [Owners and Dependents](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/)

