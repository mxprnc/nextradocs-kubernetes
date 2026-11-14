

# **Deployment**

## **1. Deployment이란?**

**Deployment는 Kubernetes에서 애플리케이션을 선언적으로 배포하고 관리하기 위한 Workload 리소스입니다.**

Deployment는 Pod의 생성과 삭제, 복제본 유지, 업데이트 및 롤백 등의 작업을 관리합니다.

Pod를 직접 생성하여 관리하는 경우 Pod가 삭제되거나 노드 장애가 발생했을 때 이를 대신할 Pod를 자동으로 생성하기 어렵습니다. Deployment를 사용하면 원하는 Pod의 개수와 구성을 선언하고, Kubernetes가 실제 상태를 원하는 상태에 맞추도록 관리할 수 있습니다.

Deployment는 내부적으로 ReplicaSet을 생성하고 관리하며, ReplicaSet을 통해 실제 Pod의 개수를 유지합니다.

Deployment의 주요 특징은 다음과 같습니다.

- 선언적 방식으로 애플리케이션을 배포하고 관리합니다.
- 지정한 수의 Pod 복제본이 유지되도록 관리합니다.
- 새로운 버전의 애플리케이션을 배포할 수 있습니다.
- RollingUpdate를 통해 기존 Pod를 점진적으로 교체할 수 있습니다.
- 업데이트 과정에서 문제가 발생하면 이전 버전으로 롤백할 수 있습니다.
- ReplicaSet을 생성하고 관리하여 Pod의 복제본을 유지합니다.
- HorizontalPodAutoscaler(HPA)와 연동하여 Pod 개수를 자동으로 조절할 수 있습니다.

Deployment는 웹 애플리케이션, REST API 서버, 백엔드 애플리케이션 등 여러 개의 동일한 인스턴스를 실행해야 하는 서비스에 적합합니다.

일반적으로 Stateless 애플리케이션에 많이 사용하지만, Deployment가 Stateful 애플리케이션을 실행할 수 없는 것은 아닙니다. 다만 안정적인 Pod 식별자, Pod별 영구 스토리지, 순서가 보장되는 생성 및 삭제가 필요한 경우에는 StatefulSet이 더 적합합니다.

## **2. Deployment의 구조**

Deployment는 자체적으로 컨테이너를 실행하지 않습니다. Deployment Controller가 ReplicaSet을 생성하고 관리하며, ReplicaSet Controller가 실제 Pod를 생성하고 유지합니다.

전체 구조는 다음과 같습니다.

```text
Kubernetes Cluster
│
└── Deployment
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

예를 들어 Deployment에 복제본을 3개로 설정하면 다음과 같은 구조가 만들어집니다.

```text
Deployment: nginx-deployment
│
└── ReplicaSet: nginx-deployment-7d8f9c
    │
    ├── Pod: nginx-deployment-7d8f9c-abcde
    ├── Pod: nginx-deployment-7d8f9c-fghij
    └── Pod: nginx-deployment-7d8f9c-klmno
```

각 리소스의 역할은 다음과 같습니다.

| 리소스 | 역할 |
|---|---|
| Deployment | 애플리케이션의 배포 상태와 업데이트를 관리합니다. |
| ReplicaSet | 지정된 수의 Pod 복제본이 유지되도록 관리합니다. |
| Pod | 실제 컨테이너를 실행하는 단위입니다. |
| Container | 애플리케이션 프로세스를 실행합니다. |

Deployment는 ReplicaSet을 관리하고, ReplicaSet은 Pod를 관리하는 계층 구조를 가집니다.

따라서 Pod의 개수를 변경하거나 애플리케이션 버전을 업데이트할 때 Deployment를 수정하면 하위 리소스들이 이에 맞게 변경됩니다.

Deployment는 업데이트 이력을 관리하기 위해 이전 ReplicaSet을 유지할 수 있습니다. 따라서 업데이트가 완료된 시점에도 이전 ReplicaSet 객체가 남아 있을 수 있으며, 일반적으로 이전 ReplicaSet의 복제본 개수는 0으로 조정됩니다.

## **3. Deployment YAML 구성**

Deployment는 YAML Manifest를 통해 정의할 수 있습니다.

다음은 Nginx 컨테이너를 3개의 Pod로 실행하는 Deployment 예제입니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
| `apiVersion` | Kubernetes API 버전을 지정합니다. Deployment는 `apps/v1`을 사용합니다. |
| `kind` | 생성할 리소스의 종류를 지정합니다. |
| `metadata` | Deployment의 이름과 레이블 등의 메타데이터를 정의합니다. |
| `spec` | Deployment의 원하는 상태를 정의합니다. |
| `replicas` | 유지하려는 Pod 복제본의 개수를 지정합니다. |
| `selector` | Deployment가 관리할 Pod를 식별하는 조건을 지정합니다. |
| `template` | ReplicaSet이 생성할 Pod의 템플릿을 정의합니다. |
| `template.metadata` | 생성될 Pod의 레이블 등의 메타데이터를 정의합니다. |
| `template.spec` | 생성될 Pod의 컨테이너 및 실행 환경을 정의합니다. |

### **3.1. replicas**

`replicas`는 Deployment가 유지하려는 Pod 복제본의 개수를 지정합니다.

```yaml
spec:
  replicas: 3
```

위 설정은 동일한 구성을 가진 Pod 3개가 유지되도록 지정합니다.

현재 관리 중인 Pod가 2개라면 ReplicaSet Controller가 Pod를 추가로 생성합니다. 반대로 관리 대상 Pod가 4개라면 지정된 개수에 맞게 초과된 Pod를 삭제합니다.

`replicas`를 지정하지 않으면 기본값은 1입니다.

단, `replicas`는 원하는 복제본 개수이므로 실제 실행 중이거나 Ready 상태인 Pod 개수와 항상 일치하는 것은 아닙니다. 노드 자원 부족, 이미지 다운로드 실패, 스케줄링 제약 등의 문제가 있으면 Pod가 생성되더라도 정상적으로 실행되지 못할 수 있습니다.

### **3.2. selector**

`selector`는 Deployment가 관리할 Pod를 식별하기 위한 조건입니다.

```yaml
spec:
  selector:
    matchLabels:
      app: nginx
```

위 설정은 `app: nginx` 레이블을 가진 Pod를 대상으로 지정합니다.

Deployment의 `selector`는 `template.metadata.labels`에 정의된 레이블과 일치해야 합니다.

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

`apps/v1` API에서 Deployment의 Selector는 필수이며, 생성 이후 변경할 수 없는 필드입니다.

Deployment의 Selector는 하위 ReplicaSet의 관리 대상 Pod를 결정하는 데 사용됩니다. 따라서 서로 다른 Deployment가 동일한 Pod를 대상으로 지정하지 않도록 명확하고 고유한 레이블을 사용하는 것이 중요합니다.

### **3.3. template**

`template`은 Deployment가 관리하는 ReplicaSet에서 생성할 Pod의 템플릿입니다.

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

Pod의 컨테이너 이미지, 포트, 리소스 요청량, 환경 변수, 볼륨 등의 설정을 정의할 수 있습니다.

Deployment의 Pod Template이 변경되면 Deployment Controller는 변경된 Template을 사용하는 새로운 ReplicaSet을 생성하거나, 기존 ReplicaSet 중 해당 Template과 일치하는 ReplicaSet을 다시 활용합니다.

새로운 ReplicaSet을 통해 업데이트가 진행되므로 기존 Pod가 새로운 Template의 설정으로 직접 변경되는 것은 아닙니다.

## **4. Deployment의 동작 방식**

Deployment는 선언적으로 원하는 상태를 정의하고, Kubernetes Controller가 실제 상태를 원하는 상태에 맞추는 방식으로 동작합니다.

이러한 동작 방식을 **Reconciliation Loop(조정 루프)**라고 합니다.

예를 들어 Deployment에 `replicas: 3`으로 지정되어 있다면 Deployment Controller는 원하는 상태에 맞는 ReplicaSet을 관리하고, ReplicaSet Controller는 실제 관리 대상 Pod의 개수가 3개가 되도록 조정합니다.

```text
Deployment
replicas: 3
     │
     ▼
Deployment Controller
     │
     ▼
ReplicaSet 생성 및 관리
     │
     ▼
ReplicaSet Controller
     │
     ▼
현재 Pod 개수 확인
     │
     ├── Pod 2개 → Pod 1개 추가 생성
     │
     ├── Pod 3개 → 변경 없음
     │
     └── Pod 4개 → Pod 1개 삭제
```

Deployment Controller와 ReplicaSet Controller는 서로 다른 역할을 수행합니다. Deployment Controller는 ReplicaSet의 생성과 복제본 수를 관리하고, ReplicaSet Controller는 Pod의 생성과 삭제를 관리합니다.

### **4.1. Deployment 생성 과정**

Deployment를 생성하면 Kubernetes 내부에서 다음과 같은 작업이 진행됩니다.

```text
kubectl apply -f deployment.yaml
              │
              ▼
         API Server
              │
              ▼
       Deployment 생성
              │
              ▼
     Deployment Controller
              │
              ▼
         ReplicaSet 생성
              │
              ▼
      ReplicaSet Controller
              │
              ▼
          Pod 생성
              │
              ▼
          Scheduler
              │
              ▼
       Node에 Pod 배치
              │
              ▼
           Kubelet
              │
              ▼
       Container 실행
```

각 구성 요소의 역할은 다음과 같습니다.

1. **API Server:** Deployment 생성 요청을 받아 유효성을 검증하고 Kubernetes API 객체로 저장합니다.
2. **Deployment Controller:** Deployment의 원하는 상태를 확인하고 필요한 ReplicaSet을 생성하거나 관리합니다.
3. **ReplicaSet Controller:** 지정된 복제본 개수에 맞게 Pod를 생성하거나 삭제합니다.
4. **Scheduler:** 생성된 Pod가 실행될 Node를 결정합니다.
5. **Kubelet:** 선택된 Node에서 Pod와 컨테이너의 실행 상태를 관리합니다.
6. **Container Runtime:** 컨테이너 이미지를 가져오고 실제 컨테이너를 실행합니다.

Deployment는 이러한 Controller들의 협력을 통해 원하는 상태를 유지합니다.

### **4.2. Pod 장애 발생 시 복구**

Deployment가 관리하는 Pod가 삭제되거나 비정상적으로 종료되는 경우 ReplicaSet Controller는 현재 관리 대상 Pod 개수를 확인합니다.

예를 들어 3개의 Pod를 실행 중인 Deployment에서 Pod 하나가 삭제되면 다음과 같이 동작합니다.

```text
Deployment
replicas: 3
     │
     ▼
ReplicaSet
     │
     ├── Pod A
     ├── Pod B
     └── Pod C
          │
          ▼
       Pod C 삭제
          │
          ▼
현재 관리 대상 Pod 개수: 2
          │
          ▼
새로운 Pod 생성
          │
          ▼
Pod A
Pod B
Pod D
```

ReplicaSet Controller가 새로운 Pod를 생성하여 원하는 복제본 개수를 유지합니다.

단, Deployment가 Pod의 기존 상태를 복구하는 것은 아닙니다. 새로운 Pod를 생성하는 것이므로 기존 Pod와 동일한 IP 주소나 컨테이너 내부의 임시 데이터가 유지되지 않을 수 있습니다.

## **5. Deployment의 업데이트 전략**

Deployment는 애플리케이션의 새로운 버전을 배포할 때 업데이트 전략을 사용할 수 있습니다.

대표적인 업데이트 전략은 다음 두 가지입니다.

- `RollingUpdate`
- `Recreate`

### **5.1. RollingUpdate**

RollingUpdate는 기존 Pod를 한 번에 모두 종료하지 않고, 새로운 Pod를 점진적으로 생성하면서 기존 Pod를 교체하는 방식입니다.

Deployment의 기본 업데이트 전략입니다.

예를 들어 Nginx 1.27을 실행하는 Pod 3개를 Nginx 1.28로 업데이트하는 경우 다음과 같이 동작할 수 있습니다.

```text
Before Update

ReplicaSet v1
├── Pod A (nginx:1.27)
├── Pod B (nginx:1.27)
└── Pod C (nginx:1.27)


Rolling Update

ReplicaSet v1
├── Pod A (nginx:1.27)
├── Pod B (nginx:1.27)
└── Pod C (nginx:1.27)

ReplicaSet v2
└── Pod D (nginx:1.28)


After Update

ReplicaSet v2
├── Pod D (nginx:1.28)
├── Pod E (nginx:1.28)
└── Pod F (nginx:1.28)
```

실제 업데이트 과정에서는 새로운 ReplicaSet이 생성되고, 새로운 ReplicaSet의 Pod가 증가하는 동시에 기존 ReplicaSet의 Pod가 감소합니다.

RollingUpdate의 주요 설정은 다음과 같습니다.

| 설정 | 설명 | 기본값 |
|---|---|---|
| `maxSurge` | 업데이트 과정에서 원하는 복제본 개수를 초과하여 추가로 생성할 수 있는 Pod의 최대 개수 또는 비율입니다. | `25%` |
| `maxUnavailable` | 업데이트 과정에서 사용 불가능한 상태가 될 수 있는 Pod의 최대 개수 또는 비율입니다. | `25%` |

다음은 RollingUpdate 설정 예제입니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1

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
          image: nginx:1.28
          ports:
            - containerPort: 80
```

위 설정의 의미는 다음과 같습니다.

- 원하는 Pod 개수는 3개입니다.
- 업데이트 과정에서 최대 1개의 Pod를 추가로 생성할 수 있습니다.
- 업데이트 과정에서 최대 1개의 Pod를 사용할 수 없는 상태로 허용합니다.

따라서 업데이트 과정에서 관리 대상 Pod의 수는 최대 4개까지 증가할 수 있습니다.

`maxSurge`와 `maxUnavailable`은 정수 또는 백분율로 지정할 수 있습니다.

백분율로 지정한 경우 다음과 같이 계산합니다.

- `maxSurge`: 계산 결과를 올림합니다.
- `maxUnavailable`: 계산 결과를 내림합니다.

두 설정을 모두 0으로 지정할 수는 없습니다. 최소한 하나는 0보다 큰 값이어야 합니다.

`maxUnavailable`은 업데이트 중 Deployment가 유지해야 하는 Available Pod의 최소 개수에 영향을 줍니다. Pod가 Ready 상태가 되는 시점과 Available 상태로 인정되는 시점은 다를 수 있습니다. `minReadySeconds`가 설정되어 있다면 Pod가 Ready 상태를 일정 시간 유지해야 Available 상태로 인정됩니다.

또한 Readiness Probe를 적절하게 구성해야 실제 요청을 처리할 수 있는 Pod를 기준으로 업데이트가 진행되도록 할 수 있습니다.

### **5.2. Recreate**

Recreate는 새로운 버전의 Pod를 생성하기 전에 기존 Pod를 모두 종료하는 방식입니다.

```text
Before Update

ReplicaSet v1
├── Pod A (nginx:1.27)
├── Pod B (nginx:1.27)
└── Pod C (nginx:1.27)


Recreate

기존 ReplicaSet의 Pod 종료
├── Pod A (Terminated)
├── Pod B (Terminated)
└── Pod C (Terminated)


새로운 ReplicaSet의 Pod 생성
├── Pod D (nginx:1.28)
├── Pod E (nginx:1.28)
└── Pod F (nginx:1.28)
```

기존 Pod가 모두 종료된 후 새로운 버전의 Pod가 생성됩니다.

다음과 같이 설정할 수 있습니다.

```yaml
spec:
  strategy:
    type: Recreate
```

Recreate는 기존 애플리케이션과 새로운 애플리케이션이 동시에 실행되면 안 되는 경우에 사용할 수 있습니다.

다만 기존 Pod가 종료된 시점부터 새로운 Pod가 준비될 때까지 서비스 중단이 발생할 수 있습니다.

Recreate 전략은 Deployment의 Pod Template이 변경되어 업데이트가 발생할 때 적용됩니다. 단순히 `replicas` 값을 변경하는 스케일링 작업에 기존 Pod를 모두 종료하는 방식으로 적용되는 것은 아닙니다.

| 구분 | RollingUpdate | Recreate |
|---|---|---|
| 업데이트 방식 | 기존 Pod와 새로운 Pod를 점진적으로 교체합니다. | 기존 Pod를 모두 종료한 후 새로운 Pod를 생성합니다. |
| 서비스 중단 | 적절한 설정을 통해 중단을 최소화할 수 있습니다. | 업데이트 과정에서 서비스 중단이 발생할 수 있습니다. |
| 리소스 사용 | 업데이트 중 일시적으로 추가 리소스가 필요할 수 있습니다. | 기존 Pod와 신규 Pod가 동시에 실행되는 업데이트 과정이 없습니다. |
| 주요 사용 사례 | 일반적인 웹 애플리케이션 및 API 서버 | 버전 간 동시 실행이 불가능한 애플리케이션 |

## **6. Deployment의 롤백 (Rollback)**

Deployment는 업데이트 이력을 관리하며, 문제가 발생했을 때 이전 버전으로 되돌릴 수 있는 롤백 기능을 제공합니다.

예를 들어 새로운 애플리케이션 버전을 배포했지만 오류가 발생했다면 이전 ReplicaSet을 다시 활성화하여 기존 버전으로 복구할 수 있습니다.

```text
Deployment
│
├── ReplicaSet v1 (nginx:1.26)
│
├── ReplicaSet v2 (nginx:1.27)
│
└── ReplicaSet v3 (nginx:1.28)
        │
        ▼
   배포 오류 발생
        │
        ▼
   Rollback to v2
        │
        ▼
ReplicaSet v2 활성화
```

Deployment의 업데이트 이력을 확인하는 명령어는 다음과 같습니다.

```bash
kubectl rollout history deployment/nginx-deployment
```

특정 Revision의 상세 정보를 확인하려면 다음 명령어를 사용합니다.

```bash
kubectl rollout history deployment/nginx-deployment --revision=2
```

이전 Revision으로 롤백하려면 다음과 같이 실행합니다.

```bash
kubectl rollout undo deployment/nginx-deployment
```

특정 Revision으로 롤백하려면 다음과 같이 실행합니다.

```bash
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

업데이트가 정상적으로 진행되고 있는지 확인하려면 다음 명령어를 사용합니다.

```bash
kubectl rollout status deployment/nginx-deployment
```

Deployment의 롤백은 이전 ReplicaSet의 Pod Template을 다시 사용하는 방식으로 동작합니다.

롤백에 필요한 이전 ReplicaSet이 보존되어 있어야 하므로 `revisionHistoryLimit` 설정도 중요합니다. 기본값은 10이며, 0으로 설정하면 이전 ReplicaSet을 보존하지 않으므로 롤백할 수 있는 이력이 남지 않을 수 있습니다.

다만 롤백은 애플리케이션의 실행 구성에 대한 복구 기능이며, 데이터베이스 스키마 변경이나 외부 시스템의 데이터 변경까지 자동으로 되돌려 주지는 않습니다.

### **6.1. revisionHistoryLimit**

`revisionHistoryLimit`은 Deployment가 롤백을 위해 보존할 이전 ReplicaSet의 개수를 지정합니다.

```yaml
spec:
  revisionHistoryLimit: 5
```

위 설정은 이전 ReplicaSet을 최대 5개까지 보존하도록 지정합니다.

기본값은 10이며, 0으로 설정하면 이전 ReplicaSet을 보존하지 않습니다.

이 설정은 업데이트가 완료된 이후 오래된 ReplicaSet을 정리하는 데 사용됩니다. 따라서 롤백이 필요한 운영 환경에서는 적절한 이력 보존 개수를 지정해야 합니다.

## **7. Deployment의 스케일링 (Scaling)**

Deployment는 실행 중인 Pod의 복제본 개수를 변경하여 애플리케이션의 처리 용량을 조절할 수 있습니다.

이를 스케일링이라고 합니다.

### **7.1. 수동 스케일링**

`replicas` 값을 변경하거나 `kubectl scale` 명령어를 사용하여 Pod 개수를 조정할 수 있습니다.

기존 Deployment의 Pod 개수를 3개에서 5개로 변경하는 예제입니다.

```bash
kubectl scale deployment nginx-deployment --replicas=5
```

또는 YAML Manifest의 `replicas` 값을 변경할 수 있습니다.

```yaml
spec:
  replicas: 5
```

변경된 복제본 개수에 맞게 ReplicaSet Controller가 Pod를 생성하거나 삭제합니다.

스케일링은 원하는 복제본 개수를 변경하는 작업이며, 실제 Pod의 생성과 삭제는 Controller의 조정 과정을 통해 수행됩니다.

### **7.2. 자동 스케일링**

Deployment는 HorizontalPodAutoscaler(HPA)와 연동하여 Pod 개수를 자동으로 조절할 수 있습니다.

HPA는 CPU 사용률, 메모리 사용률 또는 사용자 정의 메트릭 등을 기준으로 원하는 Pod 복제본 개수를 계산합니다.

예를 들어 CPU 사용률이 지정된 기준을 초과하면 Pod를 추가로 생성하고, 사용률이 낮아지면 Pod 개수를 줄일 수 있습니다.

```text
Deployment
replicas: 3
     │
     ▼
HorizontalPodAutoscaler
     │
     ▼
메트릭 수집 및 목표 사용률 비교
     │
     ├── CPU 사용률 증가
     │       │
     │       ▼
     │    Pod 증가
     │
     └── CPU 사용률 감소
             │
             ▼
          Pod 감소
```

HPA는 Deployment의 Scale subresource를 통해 복제본 개수를 조정합니다. Deployment Controller는 변경된 복제본 개수를 반영하여 ReplicaSet을 관리합니다.

HPA를 사용하는 경우 YAML Manifest에서 `spec.replicas`를 반복적으로 적용하면 HPA가 조정한 값이 다시 덮어써질 수 있으므로, 복제본 개수의 관리 주체를 명확하게 정해야 합니다.

HPA에 대해서는 별도의 `horizontal-pod-autoscaler.md` 문서에서 설명합니다.

## **8. Deployment와 Service의 관계**

Deployment와 Service는 서로 다른 역할을 담당하지만 일반적으로 함께 사용됩니다.

Deployment는 Pod의 생성과 복제본 유지를 담당하며, Service는 Pod에 접근할 수 있는 안정적인 네트워크 엔드포인트를 제공합니다.

Deployment가 관리하는 Pod는 업데이트나 장애로 인해 삭제되고 새로운 Pod로 교체될 수 있습니다. 이 과정에서 Pod IP 주소가 변경될 수 있습니다.

Service는 Pod의 IP 주소가 변경되더라도 동일한 서비스 주소를 통해 애플리케이션에 접근할 수 있도록 지원합니다.

```text
                    Client
                       │
                       ▼
                  Service
                nginx-service
                       │
                 Label Selector
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
        Pod A        Pod B        Pod C
          │            │            │
          └────────────┼────────────┘
                       │
                  Deployment
                       │
                   ReplicaSet
```

예를 들어 Deployment에서 다음과 같이 Pod 레이블을 지정했다고 가정하겠습니다.

```yaml
template:
  metadata:
    labels:
      app: nginx
```

Service는 동일한 레이블을 Selector로 지정하여 해당 Pod들을 서비스 대상으로 선택할 수 있습니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

이러한 구조를 통해 Deployment가 Pod를 교체하거나 복제본 개수를 변경하더라도 Service는 현재 조건에 맞는 Pod를 대상으로 트래픽을 전달할 수 있습니다.

단, Deployment가 Service를 자동으로 생성하거나 관리하는 것은 아닙니다. Deployment와 Service는 별도의 리소스이며, 일반적으로 각각의 Manifest를 통해 생성하고 관리합니다.

## **9. Deployment와 ReplicaSet의 관계**

Deployment와 ReplicaSet은 서로 다른 역할을 가진 리소스입니다.

ReplicaSet은 지정된 개수의 Pod 복제본을 유지하는 데 집중하며, Deployment는 ReplicaSet의 생성과 변경을 관리하여 애플리케이션의 배포 및 업데이트를 수행합니다.

특히 Deployment의 주요 기능인 RollingUpdate와 Rollback은 여러 ReplicaSet을 관리하는 방식으로 구현됩니다.

예를 들어 애플리케이션 버전을 순차적으로 변경하면 다음과 같이 ReplicaSet이 생성됩니다.

```text
Deployment: nginx-deployment
│
├── ReplicaSet v1
│   ├── Pod (nginx:1.26)
│   ├── Pod (nginx:1.26)
│   └── Pod (nginx:1.26)
│
├── ReplicaSet v2
│   ├── Pod (nginx:1.27)
│   ├── Pod (nginx:1.27)
│   └── Pod (nginx:1.27)
│
└── ReplicaSet v3
    ├── Pod (nginx:1.28)
    ├── Pod (nginx:1.28)
    └── Pod (nginx:1.28)
```

위 구조는 업데이트 이력을 개념적으로 표현한 것입니다. 실제 업데이트가 완료된 이후에는 일반적으로 최신 ReplicaSet만 원하는 복제본을 유지하고, 이전 ReplicaSet은 Pod가 0개인 상태로 남아 롤백에 활용됩니다.

이전 ReplicaSet은 Deployment의 Revision History 관리 정책에 따라 보존되거나 삭제될 수 있습니다.

따라서 ReplicaSet은 Pod 복제본 관리에 집중하는 리소스이며, Deployment는 ReplicaSet을 활용하여 애플리케이션의 버전 관리와 배포 전략을 제공하는 상위 리소스라고 이해할 수 있습니다.

## **10. Deployment 관리 명령어**

Deployment를 생성하고 관리할 때 사용하는 주요 kubectl 명령어는 다음과 같습니다.

| 명령어 | 설명 |
|---|---|
| `kubectl get deployments` | Deployment 목록을 조회합니다. |
| `kubectl get deployment <name>` | 특정 Deployment를 조회합니다. |
| `kubectl describe deployment <name>` | Deployment의 상세 정보와 이벤트를 조회합니다. |
| `kubectl apply -f deployment.yaml` | YAML Manifest를 기반으로 Deployment를 생성하거나 변경합니다. |
| `kubectl delete deployment <name>` | Deployment를 삭제합니다. |
| `kubectl scale deployment <name> --replicas=5` | Pod 복제본 개수를 변경합니다. |
| `kubectl rollout status deployment/<name>` | Deployment 업데이트 진행 상태를 확인합니다. |
| `kubectl rollout history deployment/<name>` | Deployment 업데이트 이력을 확인합니다. |
| `kubectl rollout undo deployment/<name>` | 이전 Revision으로 롤백합니다. |
| `kubectl rollout restart deployment/<name>` | Pod Template에 변경을 발생시켜 Pod를 순차적으로 재시작합니다. |
| `kubectl rollout pause deployment/<name>` | Deployment의 업데이트 진행을 일시 중지합니다. |
| `kubectl rollout resume deployment/<name>` | 일시 중지된 Deployment 업데이트를 재개합니다. |
| `kubectl edit deployment <name>` | 실행 중인 Deployment 설정을 편집합니다. |

Deployment가 관리하는 ReplicaSet과 Pod를 함께 확인하려면 다음 명령어를 사용할 수 있습니다.

```bash
kubectl get deployment,replicaset,pod
```

Deployment의 전체 YAML 설정을 확인하려면 다음 명령어를 사용합니다.

```bash
kubectl get deployment nginx-deployment -o yaml
```

## **11. Deployment 사용 시 고려사항**

### **11.1. Stateless 애플리케이션에 적합합니다.**

Deployment는 일반적으로 동일한 구성을 가진 여러 애플리케이션 인스턴스를 실행하는 데 적합합니다.

각 Pod가 독립적으로 요청을 처리할 수 있는 웹 애플리케이션이나 API 서버 등에 주로 사용합니다.

반면 안정적인 네트워크 식별자나 개별 영구 스토리지, 순서가 보장되는 Pod 생성 및 삭제가 필요한 애플리케이션에는 StatefulSet을 고려해야 합니다.

### **11.2. Pod의 상태와 데이터는 자동으로 보존되지 않습니다.**

Deployment가 관리하는 Pod는 업데이트나 장애 복구 과정에서 새로운 Pod로 교체될 수 있습니다.

따라서 Pod 내부의 임시 파일 시스템에 저장된 데이터는 Pod 교체 과정에서 손실될 수 있습니다.

영구적인 데이터 저장이 필요한 경우 PersistentVolumeClaim(PVC) 등을 사용해야 합니다.

단, PVC를 사용하더라도 데이터의 보존 여부는 스토리지 구성과 PVC의 생명주기 정책에 따라 달라집니다.

### **11.3. RollingUpdate가 항상 무중단 배포를 보장하지는 않습니다.**

RollingUpdate는 기존 Pod를 점진적으로 교체하여 서비스 중단을 최소화하는 방식입니다.

그러나 다음과 같은 상황에서는 서비스 장애가 발생할 수 있습니다.

- 애플리케이션이 정상적으로 시작되지 않는 경우
- Readiness Probe가 적절하게 설정되지 않은 경우
- 기존 버전과 새로운 버전 간 API 호환성이 없는 경우
- 업데이트 과정에서 사용 가능한 Pod가 부족한 경우
- 데이터베이스 스키마 변경이 애플리케이션 버전과 호환되지 않는 경우
- Pod 종료 과정에서 진행 중인 요청을 적절하게 처리하지 못하는 경우

따라서 안정적인 RollingUpdate를 위해서는 Readiness Probe, 적절한 리소스 설정, 애플리케이션의 버전 호환성 및 Graceful Shutdown을 함께 고려해야 합니다.

### **11.4. Deployment의 Selector는 신중하게 설정해야 합니다.**

Deployment의 Selector는 관리 대상 Pod를 식별하는 중요한 조건입니다.

`apps/v1`에서는 Selector를 생성 이후 변경할 수 없으며, Pod Template의 레이블과 일치해야 합니다.

또한 동일한 Pod를 여러 Controller가 중복으로 관리하지 않도록 Selector를 명확하게 정의해야 합니다.

### **11.5. Deployment 삭제 시 하위 리소스도 함께 삭제됩니다.**

Deployment를 일반적인 방식으로 삭제하면 해당 Deployment가 관리하는 ReplicaSet과 Pod도 함께 삭제됩니다.

따라서 Deployment 삭제 시 실행 중인 애플리케이션도 종료될 수 있습니다.

단, Kubernetes의 OwnerReference와 Garbage Collection 정책에 따라 하위 리소스의 삭제 동작이 결정되므로, 특정 리소스를 보존해야 하는 경우에는 삭제 정책을 별도로 고려해야 합니다.

### **11.6. 업데이트 진행 상태와 실패 조건을 확인해야 합니다.**

Deployment는 업데이트가 일정 시간 동안 진행되지 않으면 해당 상태를 실패로 표시할 수 있습니다.

`progressDeadlineSeconds`는 Deployment가 업데이트 진행 상황을 확인하는 최대 시간을 초 단위로 지정하는 필드입니다.

```yaml
spec:
  progressDeadlineSeconds: 600
```

기본값은 600초입니다.

설정된 시간 동안 업데이트가 진행되지 않으면 Deployment 상태에 `ProgressDeadlineExceeded` 조건이 표시됩니다.

이 설정은 업데이트 실패를 감지하는 기준이며, 실패한 Deployment를 자동으로 이전 버전으로 롤백하는 기능은 아닙니다. 문제의 원인을 확인하고 필요하면 직접 롤백해야 합니다.

업데이트 진행 중 여러 설정을 변경한 뒤 한 번에 반영하려면 Deployment의 Rollout을 일시 중지하고 재개할 수도 있습니다.

```bash
kubectl rollout pause deployment/nginx-deployment

# Deployment 설정 변경

kubectl rollout resume deployment/nginx-deployment
```

## **12. 핵심 정리**

Deployment는 Kubernetes에서 애플리케이션을 선언적으로 배포하고 관리하기 위한 Workload 리소스입니다.

- Deployment는 ReplicaSet을 생성하고 관리합니다.
- ReplicaSet은 지정된 개수의 Pod 복제본을 유지합니다.
- `replicas`를 통해 원하는 Pod 개수를 지정할 수 있습니다.
- `selector`를 통해 관리할 Pod를 식별합니다.
- `template`을 통해 생성할 Pod의 구성을 정의합니다.
- RollingUpdate를 통해 기존 Pod를 점진적으로 교체할 수 있습니다.
- Recreate를 통해 기존 Pod를 모두 종료한 후 새로운 Pod를 생성할 수 있습니다.
- Revision History를 활용하여 이전 버전으로 롤백할 수 있습니다.
- HorizontalPodAutoscaler와 연동하여 Pod 개수를 자동으로 조절할 수 있습니다.
- Service와 함께 사용하여 Pod의 IP 변경과 관계없이 안정적인 서비스 접근 경로를 제공할 수 있습니다.
- `progressDeadlineSeconds`를 통해 업데이트 진행이 지연되는 상황을 감지할 수 있습니다.

**Deployment는 Pod를 직접 관리하는 대신 ReplicaSet을 통해 Pod의 복제본과 배포 상태를 관리하는 상위 Workload 리소스입니다.**

Kubernetes에서 일반적인 웹 애플리케이션이나 API 서버를 배포할 때 가장 기본적으로 사용하는 리소스이며, ReplicaSet, StatefulSet, DaemonSet, Job 등의 다른 Workload 리소스를 이해하는 데 중요한 기반이 됩니다.

## **참고자료**

- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Deployment API Reference](https://kubernetes.io/docs/reference/kubernetes-api/apps/deployment-v1/)
- [Update a Deployment Without Downtime](https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/)
- [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

