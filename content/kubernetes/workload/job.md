# Job

## 1. Job이란?

Job은 Kubernetes에서 특정 작업을 완료할 때까지 Pod를 실행하고 관리하는 Workload 리소스입니다.

Deployment나 StatefulSet은 애플리케이션이 지속적으로 실행되도록 Pod를 유지하는 데 목적이 있습니다. 반면 Job은 정해진 작업을 수행하고 정상적으로 완료되면 실행을 종료하는 데 목적이 있습니다.

예를 들어 다음과 같은 작업에 Job을 사용할 수 있습니다.

- 데이터베이스 초기 데이터 적재
- 데이터 마이그레이션
- 대량 데이터 처리
- 이미지 변환 및 파일 처리
- 일회성 배치 작업
- 정기 작업을 수행하기 위한 개별 실행 단위

Job은 Pod를 생성하고 작업이 완료될 때까지 실행 상태를 관리합니다. Pod가 정상적으로 작업을 완료하면 Job도 완료 상태가 됩니다.

작업 도중 Pod가 실패하면 Job은 설정된 재시도 정책에 따라 새로운 Pod를 생성하여 작업을 다시 시도할 수 있습니다.

Job의 주요 특징은 다음과 같습니다.

- 작업이 완료될 때까지 Pod를 실행합니다.
- Pod가 정상적으로 종료되면 Job을 완료 상태로 전환합니다.
- Pod가 실패하면 설정된 정책에 따라 재시도할 수 있습니다.
- 여러 Pod를 병렬로 실행하여 작업을 처리할 수 있습니다.
- 작업의 성공 및 실패 횟수를 추적합니다.
- 완료된 Job과 Pod를 일정 기간 유지하거나 정리할 수 있습니다.

Job은 지속적으로 실행되는 서비스가 아니라 완료가 필요한 작업을 실행하고 관리하기 위한 Workload 리소스입니다.

## 2. Job의 구조

Job은 Pod Template을 기반으로 Pod를 생성하고 관리합니다.

가장 기본적인 Job은 하나의 Pod를 실행하여 작업을 완료하는 구조입니다.

```
Kubernetes Cluster
│
└── Job: data-processing
    │
    └── Pod
        │
        └── Container
            │
            └── 작업 실행
                 │
                 ▼
              작업 완료
                 │
                 ▼
              Job Complete
```

Job은 Pod가 정상적으로 종료되었는지 확인하여 작업 완료 여부를 판단합니다.

예를 들어 데이터 처리 프로그램이 실행을 마치고 종료 코드 `0`을 반환하면 Pod는 성공적으로 종료되고 Job은 완료 상태가 됩니다.

반면 컨테이너가 오류로 종료되거나 Pod가 실패하면 Job은 설정된 재시도 정책에 따라 작업을 다시 실행할 수 있습니다.

## 3. Job YAML 구성

다음은 간단한 데이터 처리 작업을 수행하는 Job 예제입니다.

```
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processing
spec:
  template:
    spec:
      containers:
        - name: processor
          image: busybox:1.36
          command:
            - sh
            - -c
            - "echo 'Processing data'; sleep 10; echo 'Processing completed'"

      restartPolicy: Never
```

각 필드의 역할은 다음과 같습니다.

| 필드              | 설명                                                |
| --------------- | ------------------------------------------------- |
| `apiVersion`    | Kubernetes API 버전을 지정합니다. Job은 `batch/v1`을 사용합니다. |
| `kind`          | 생성할 리소스의 종류를 지정합니다.                               |
| `metadata`      | Job의 이름과 메타데이터를 정의합니다.                            |
| `spec`          | Job의 원하는 실행 상태를 정의합니다.                            |
| `template`      | Job이 생성할 Pod의 구성을 정의합니다.                          |
| `containers`    | 실제 작업을 수행할 컨테이너를 정의합니다.                           |
| `restartPolicy` | 컨테이너가 종료되었을 때 Pod 내부에서 재시작할지 여부를 지정합니다.           |

### 3.1. restartPolicy

Job의 Pod Template에서는 `restartPolicy`를 `Never` 또는 `OnFailure`로 설정해야 합니다.

```
spec:
  template:
    spec:
      restartPolicy: Never
```

두 정책의 차이는 다음과 같습니다.

| 정책          | 설명                                                                          |
| ----------- | --------------------------------------------------------------------------- |
| `Never`     | 컨테이너가 실패하면 기존 Pod를 재시작하지 않습니다. Job Controller가 필요에 따라 새로운 Pod를 생성하여 재시도합니다. |
| `OnFailure` | 컨테이너가 실패하면 동일한 Pod 내부에서 컨테이너를 재시작할 수 있습니다.                                  |

`Always`는 Job의 Pod Template에서 사용할 수 없습니다.

Job은 작업이 완료되면 Pod를 종료해야 하므로 지속적으로 컨테이너를 재시작하는 `Always` 정책과 맞지 않습니다.

### 3.2. completions

`completions`는 Job이 성공적으로 완료해야 하는 작업의 횟수를 지정합니다.

```
spec:
  completions: 3
```

위 설정은 성공적으로 완료된 Pod가 총 3개가 되어야 Job이 완료된다는 의미입니다.

예를 들어 다음과 같이 동작할 수 있습니다.

```
Job: data-processing
│
├── Pod 1 ── Success
├── Pod 2 ── Success
└── Pod 3 ── Success
     │
     ▼
  3 Completions
     │
     ▼
  Job Complete
```

`completions`를 지정하지 않으면 기본적으로 하나의 성공적인 완료를 목표로 합니다.

### 3.3. parallelism

`parallelism`은 동시에 실행할 수 있는 Pod의 최대 개수를 지정합니다.

```
spec:
  completions: 6
  parallelism: 3
```

위 설정은 총 6개의 작업을 성공적으로 완료해야 하며, 동시에 최대 3개의 Pod를 실행할 수 있다는 의미입니다.

```
Job: data-processing
│
├── Batch 1
│   ├── Pod 1 ── Running
│   ├── Pod 2 ── Running
│   └── Pod 3 ── Running
│
├── Batch 2
│   ├── Pod 4 ── Running
│   ├── Pod 5 ── Running
│   └── Pod 6 ── Running
│
└── Job Complete
```

실제 실행 순서와 동시 실행 개수는 Pod의 생성 및 종료 시점, 노드의 리소스 상황 등에 따라 달라질 수 있습니다.

### 3.4. backoffLimit

`backoffLimit`은 Job이 실패한 경우 재시도할 수 있는 횟수를 지정합니다.

```
spec:
  backoffLimit: 4
```

기본값은 `6`입니다.

Job은 실패한 Pod의 수 등을 기준으로 재시도 정책을 적용하며, 실패가 허용된 한도를 초과하면 Job을 실패 상태로 처리합니다.

다음과 같은 흐름으로 동작할 수 있습니다.

```
Job 실행
   │
   ▼
Pod 생성
   │
   ▼
작업 실패
   │
   ▼
재시도 가능 여부 확인
   │
   ├── 재시도 가능
   │      │
   │      ▼
   │   새로운 Pod 생성
   │
   └── 재시도 한도 초과
          │
          ▼
       Job Failed
```

`restartPolicy: Never`를 사용하면 실패한 Pod가 종료된 상태로 남고, Job Controller가 새로운 Pod를 생성하여 재시도할 수 있습니다.

`restartPolicy: OnFailure`를 사용하면 동일한 Pod 내부에서 컨테이너가 재시작될 수 있으므로 재시도 횟수와 관련된 동작이 달라집니다.

따라서 재시도 횟수를 정확하게 제어해야 한다면 Pod 재시작 정책과 Job의 실패 처리 정책을 함께 이해해야 합니다.

## 4. Job의 실행 방식

Job은 작업의 특성에 따라 단일 작업, 순차 작업, 병렬 작업 등으로 구성할 수 있습니다.

### 4.1. 단일 작업 (Single Completion)

가장 기본적인 Job 실행 방식입니다.

하나의 Pod가 작업을 실행하고 정상적으로 완료되면 Job도 완료됩니다.

```
apiVersion: batch/v1
kind: Job
metadata:
  name: single-job
spec:
  template:
    spec:
      containers:
        - name: worker
          image: busybox:1.36
          command:
            - sh
            - -c
            - "echo 'Job completed'"

      restartPolicy: Never
```

별도로 `completions`를 지정하지 않았으므로 하나의 성공적인 완료를 목표로 합니다.

다음과 같은 작업에 적합합니다.

- 데이터베이스 초기화
- 일회성 데이터 처리
- 애플리케이션 초기 설정
- 단일 파일 변환

### 4.2. 순차 작업 (Sequential Processing)

여러 작업을 순차적으로 처리해야 한다면 `completions`를 지정하고 `parallelism`을 `1`로 설정할 수 있습니다.

```
apiVersion: batch/v1
kind: Job
metadata:
  name: sequential-job
spec:
  completions: 5
  parallelism: 1

  template:
    spec:
      containers:
        - name: worker
          image: busybox:1.36
          command:
            - sh
            - -c
            - "echo 'Processing task'; sleep 5"

      restartPolicy: Never
```

위 설정은 성공적으로 완료된 Pod가 총 5개가 될 때까지 한 번에 하나씩 작업을 실행합니다.

```
Job
 │
 ▼
Pod 1 실행
 │
 ▼
Pod 1 완료
 │
 ▼
Pod 2 실행
 │
 ▼
Pod 2 완료
 │
 ▼
Pod 3 실행
 │
 ▼
Pod 3 완료
 │
 ▼
Pod 4 실행
 │
 ▼
Pod 4 완료
 │
 ▼
Pod 5 실행
 │
 ▼
Job Complete
```

다만 이 방식은 각각의 Pod가 독립적인 작업 단위를 수행하는 구조입니다.

앞선 Pod가 처리한 결과를 다음 Pod가 반드시 이어서 처리해야 하는 작업이라면 애플리케이션 차원에서 작업 순서와 상태를 관리해야 합니다.

### 4.3. 병렬 작업 (Parallel Processing)

여러 작업을 동시에 처리해야 한다면 `parallelism`을 1보다 큰 값으로 설정할 수 있습니다.

```
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
spec:
  completions: 10
  parallelism: 3

  template:
    spec:
      containers:
        - name: worker
          image: busybox:1.36
          command:
            - sh
            - -c
            - "echo 'Processing task'; sleep 10"

      restartPolicy: Never
```

위 설정은 총 10개의 작업을 완료해야 하며, 동시에 최대 3개의 Pod를 실행할 수 있습니다.

```
Job: parallel-job
│
├── Worker 1 ── Task 1
├── Worker 2 ── Task 2
├── Worker 3 ── Task 3
│
├── Worker 4 ── Task 4
├── Worker 5 ── Task 5
├── Worker 6 ── Task 6
│
├── Worker 7 ── Task 7
├── Worker 8 ── Task 8
├── Worker 9 ── Task 9
│
└── Worker 10 ── Task 10
```

실제로는 앞선 Pod가 완료되는 대로 새로운 Pod가 실행되어 최대 병렬 실행 개수를 유지합니다.

이러한 방식은 독립적인 데이터 처리 작업을 여러 개의 Pod로 분산할 때 유용합니다.

## 5. Indexed Job

Indexed Job은 각 작업에 고유한 인덱스를 부여하여 병렬 작업을 수행하는 방식입니다.

일반적인 병렬 Job에서는 각 Pod가 어떤 작업을 담당하는지 별도로 구분하기 어려울 수 있습니다.

Indexed Job을 사용하면 각 Pod에 고유한 완료 인덱스를 부여할 수 있습니다.

```
apiVersion: batch/v1
kind: Job
metadata:
  name: indexed-job
spec:
  completions: 5
  parallelism: 2
  completionMode: Indexed

  template:
    spec:
      containers:
        - name: worker
          image: busybox:1.36
          command:
            - sh
            - -c
            - "echo Processing index: ${JOB_COMPLETION_INDEX}"

      restartPolicy: Never
```

위 설정에서는 다음과 같은 인덱스가 각 작업에 할당됩니다.

```
Job: indexed-job
│
├── Index 0
├── Index 1
├── Index 2
├── Index 3
└── Index 4
```

Pod 내부에서는 `JOB_COMPLETION_INDEX` 환경 변수를 통해 자신의 작업 인덱스를 확인할 수 있습니다.

예를 들어 데이터 파일을 5개로 나누어 처리한다면 각 Pod가 담당할 파일을 인덱스를 기준으로 결정할 수 있습니다.

```
Index 0 ── data-0.csv
Index 1 ── data-1.csv
Index 2 ── data-2.csv
Index 3 ── data-3.csv
Index 4 ── data-4.csv
```

Indexed Job은 작업 단위가 명확하게 구분되는 병렬 처리에 적합합니다.

## 6. Job의 주요 설정

Job은 작업의 실행 시간, 재시도, 병렬성 및 완료 후 정리 방식 등을 제어할 수 있습니다.

### 6.1. activeDeadlineSeconds

`activeDeadlineSeconds`는 Job이 실행될 수 있는 최대 시간을 초 단위로 지정합니다.

```
spec:
  activeDeadlineSeconds: 300
```

위 설정은 Job이 실행된 후 최대 300초 동안만 작업을 수행하도록 제한합니다.

지정한 시간이 초과되면 Kubernetes는 실행 중인 Job Pod를 종료하고 Job을 실패 상태로 처리합니다.

`backoffLimit`보다 우선적으로 적용되므로 재시도를 계속하더라도 전체 실행 시간이 제한됩니다.

### 6.2. ttlSecondsAfterFinished

`ttlSecondsAfterFinished`는 Job이 완료되거나 실패한 이후 리소스를 자동으로 삭제하기까지 대기할 시간을 지정합니다.

```
spec:
  ttlSecondsAfterFinished: 100
```

위 설정은 Job이 완료되거나 실패한 시점부터 100초가 지난 후 Job을 정리하도록 요청합니다.

Job이 삭제되면 해당 Job이 소유한 Pod도 함께 정리될 수 있습니다.

이 설정은 완료된 Job이 클러스터에 계속 남아 리소스를 차지하거나 관리 대상이 증가하는 것을 방지하는 데 유용합니다.

단, 실제 삭제 시점은 TTL Controller의 처리 상황에 따라 달라질 수 있습니다.

### 6.3. suspend

`suspend`는 Job의 실행을 일시 중지할지 여부를 지정합니다.

```
spec:
  suspend: true
```

`true`로 설정하면 Job의 새로운 Pod 생성을 중지하고, 실행 중인 Job Pod를 종료하도록 처리합니다.

다시 실행하려면 다음과 같이 설정합니다.

```
spec:
  suspend: false
```

이를 통해 Job의 실행을 일시적으로 중단하거나 재개할 수 있습니다.

### 6.4. backoffLimitPerIndex

Indexed Job에서는 `backoffLimitPerIndex`를 사용하여 각 인덱스별 재시도 허용 횟수를 지정할 수 있습니다.

```
spec:
  completionMode: Indexed
  completions: 5
  parallelism: 2
  backoffLimitPerIndex: 2
```

위 설정은 각 인덱스 작업마다 최대 2회의 실패를 허용하도록 구성합니다.

일반적인 `backoffLimit`이 Job 전체의 실패 처리에 적용되는 것과 달리, `backoffLimitPerIndex`는 개별 인덱스의 실패 횟수를 기준으로 관리합니다.

## 7. Job의 실패 처리와 재시도

Job은 Pod가 실패했다고 해서 즉시 전체 작업을 실패 상태로 처리하지 않습니다.

설정된 재시도 정책에 따라 새로운 Pod를 생성하여 작업을 다시 수행할 수 있습니다.

예를 들어 `restartPolicy: Never`를 사용하면 다음과 같은 흐름으로 동작합니다.

```
Job 생성
  │
  ▼
Pod 생성
  │
  ▼
작업 실행
  │
  ├── 성공
  │    │
  │    ▼
  │  Job Complete
  │
  └── 실패
       │
       ▼
   실패 횟수 확인
       │
       ├── 재시도 가능
       │    │
       │    ▼
       │  새로운 Pod 생성
       │
       └── 재시도 한도 초과
            │
            ▼
         Job Failed
```

Job은 기본적으로 실패한 Pod를 다시 실행할 수 있도록 설계되어 있습니다.

따라서 Job에서 실행하는 애플리케이션은 재시도 상황을 고려해야 합니다.

예를 들어 다음과 같은 상황을 고려해야 합니다.

- 작업이 일부 완료된 상태에서 Pod가 실패하는 경우
- 동일한 데이터가 여러 번 처리되는 경우
- 외부 API 호출이 성공했지만 응답을 받기 전에 Pod가 종료되는 경우
- 데이터베이스에 반영된 작업을 다시 수행하는 경우

이러한 문제를 방지하려면 애플리케이션에서 멱등성(Idempotency)을 확보하거나 작업 진행 상태를 별도로 관리하는 것이 중요합니다.

## 8. Job과 CronJob의 관계

Job은 작업을 실행하고 완료 여부를 관리하는 리소스입니다.

CronJob은 지정된 일정에 따라 Job을 생성하는 리소스입니다.

즉, CronJob은 직접 컨테이너를 실행하는 것이 아니라 실행 시점에 Job을 생성하고, Job이 실제 Pod를 실행합니다.

```
CronJob
  │
  │ 일정에 따라 실행
  ▼
Job
  │
  │ 작업 수행
  ▼
Pod
  │
  ▼
Container
  │
  ▼
작업 완료
```

예를 들어 매일 새벽 2시에 데이터베이스 백업을 수행하려면 CronJob을 사용하여 해당 시간에 Job을 생성하도록 구성할 수 있습니다.

반면 일회성 데이터 마이그레이션처럼 특정 시점에 한 번만 실행하면 되는 작업에는 Job을 직접 사용할 수 있습니다.

| 구분       | Job               | CronJob               |
| -------- | ----------------- | --------------------- |
| 주요 목적    | 작업 실행 및 완료 관리     | 일정에 따른 Job 생성         |
| 실행 방식    | 생성된 시점에 작업 수행     | Cron 표현식에 따라 Job 생성   |
| 반복 실행    | 기본적으로 반복 일정 기능 없음 | 반복 실행 지원              |
| 실제 실행 단위 | Pod               | Job을 생성하며 Job이 Pod 실행 |
| 주요 사용 사례 | 일회성 데이터 처리        | 정기적인 백업, 정기 배치 작업     |

## 9. Job과 다른 Workload 리소스의 관계

Kubernetes의 Workload 리소스는 애플리케이션의 실행 목적에 따라 구분됩니다.

| 리소스         | 주요 역할                     | 주요 사용 사례         |
| ----------- | ------------------------- | ---------------- |
| Pod         | 컨테이너 실행 단위                | 단일 애플리케이션 실행     |
| Deployment  | Stateless 애플리케이션 배포 및 관리  | 웹 애플리케이션, API 서버 |
| ReplicaSet  | 지정된 Pod 복제본 개수 유지         | Pod 복제본 관리       |
| StatefulSet | 안정적인 식별자와 스토리지를 가진 Pod 관리 | 데이터베이스, 분산 시스템   |
| DaemonSet   | 대상 노드마다 Pod 배치            | 로그 수집, 모니터링 에이전트 |
| Job         | 작업이 완료될 때까지 Pod 실행        | 일회성 배치 작업        |
| CronJob     | 일정에 따라 Job 실행             | 정기적인 배치 작업       |

Deployment, StatefulSet, DaemonSet은 애플리케이션이나 에이전트가 지속적으로 실행되도록 관리하는 데 적합합니다.

반면 Job은 특정 작업이 완료되면 실행을 종료하는 데 적합합니다.

## 10. Job 사용 시 고려사항

### 10.1. 작업의 멱등성을 고려해야 합니다.

Job은 Pod가 실패하면 작업을 재시도할 수 있습니다.

따라서 이전 실행에서 일부 작업이 완료되었더라도 새로운 Pod가 동일한 작업을 다시 수행할 수 있습니다.

예를 들어 데이터베이스에 특정 레코드를 삽입하는 작업이 성공했지만 Pod가 완료 상태를 보고하기 전에 종료되면, 재시도 과정에서 동일한 데이터가 다시 삽입될 수 있습니다.

이를 방지하려면 다음과 같은 방법을 고려해야 합니다.

- 중복 요청을 방지하는 멱등성 키 사용
- 데이터베이스 Unique Constraint 적용
- 작업 진행 상태 저장
- 이미 처리된 데이터인지 확인하는 로직 구현

### 10.2. 병렬 실행 시 작업 분배를 고려해야 합니다.

`parallelism`을 설정하면 여러 Pod가 동시에 작업을 수행할 수 있습니다.

하지만 Kubernetes가 각 Pod에 실제 처리할 데이터나 작업 범위를 자동으로 분배해 주는 것은 아닙니다.

작업 분배는 애플리케이션에서 직접 구현하거나 외부 작업 큐 등을 활용해야 합니다.

예를 들어 10개의 파일을 3개의 Pod로 처리하려면 각 Pod가 담당할 파일을 구분하는 로직이 필요합니다.

Indexed Job을 사용하면 인덱스를 기반으로 작업 범위를 구분할 수 있습니다.

### 10.3. 리소스 요청량과 제한값을 설정해야 합니다.

Job도 일반적인 Pod와 마찬가지로 CPU 및 메모리 리소스를 사용합니다.

특히 대량 데이터 처리나 이미지 변환 작업은 순간적으로 많은 리소스를 사용할 수 있습니다.

따라서 컨테이너의 `resources.requests`와 `resources.limits`를 적절하게 설정해야 합니다.

```
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"

  limits:
    cpu: "1"
    memory: "1Gi"
```

병렬 Job을 실행하는 경우에는 동시에 실행되는 Pod의 전체 리소스 사용량도 고려해야 합니다.

### 10.4. Job의 실행 시간과 재시도 횟수를 제한해야 합니다.

작업이 무한정 실행되거나 실패한 작업이 반복적으로 재시도되는 상황을 방지해야 합니다.

이를 위해 다음 설정을 활용할 수 있습니다.

- `activeDeadlineSeconds`: Job의 최대 실행 시간 제한
- `backoffLimit`: 실패 재시도 허용 횟수 제한
- `ttlSecondsAfterFinished`: 완료된 Job의 자동 정리

이러한 설정을 통해 작업의 실행 범위와 리소스 사용 기간을 관리할 수 있습니다.

### 10.5. 완료된 Job과 Pod의 정리 정책을 고려해야 합니다.

Job이 완료되더라도 Job과 Pod가 즉시 삭제되는 것은 아닙니다.

완료된 Job을 유지하면 실행 결과와 로그를 확인할 수 있지만, 장기간 누적되면 Kubernetes API 객체가 증가할 수 있습니다.

`ttlSecondsAfterFinished`를 활용하면 일정 시간이 지난 Job을 자동으로 정리할 수 있습니다.

운영 환경에서는 로그와 실행 결과를 별도로 수집하고, Job 리소스의 보존 기간을 적절하게 설정하는 것이 좋습니다.

### 10.6. Job은 정확히 한 번의 실행을 보장하지 않습니다.

Job은 지정된 횟수만큼 성공적인 작업 완료를 달성하도록 Pod를 관리하지만, 애플리케이션의 비즈니스 작업이 정확히 한 번만 실행되는 것을 보장하지는 않습니다.

Pod의 장애나 네트워크 문제 등으로 인해 동일한 작업이 재실행될 수 있습니다.

따라서 중복 실행에 안전한 애플리케이션 구조를 설계해야 합니다.

## 11. kubectl 명령어

### 11.1. Job 조회

```
# 현재 Namespace의 Job 조회
kubectl get jobs

# 모든 Namespace의 Job 조회
kubectl get jobs -A

# 특정 Job 상세 조회
kubectl describe job data-processing

# Job의 YAML 확인
kubectl get job data-processing -o yaml
```

### 11.2. Job 생성 및 삭제

```
# YAML Manifest 적용
kubectl apply -f job.yaml

# Job 삭제
kubectl delete job data-processing
```

Job을 삭제하면 해당 Job이 소유한 Pod도 함께 정리될 수 있습니다.

### 11.3. Job이 생성한 Pod 조회

```
# 특정 Job의 Pod 조회
kubectl get pods -l job-name=data-processing

# Pod 상세 정보 확인
kubectl describe pod <pod-name>

# Pod 로그 확인
kubectl logs <pod-name>
```

### 11.4. Job 실행 상태 확인

```
# Job 실행 상태 확인
kubectl get job data-processing

# Job의 상세 상태 확인
kubectl describe job data-processing

# Job이 완료될 때까지 대기
kubectl wait --for=condition=complete job/data-processing --timeout=300s
```

### 11.5. Job의 실행 결과 확인

```
# Job이 생성한 Pod의 상태 확인
kubectl get pods -l job-name=data-processing

# 성공적으로 완료된 Pod 확인
kubectl get pods -l job-name=data-processing \
  --field-selector=status.phase=Succeeded

# 실패한 Pod 확인
kubectl get pods -l job-name=data-processing \
  --field-selector=status.phase=Failed
```

## 12. 핵심 정리

Job은 Kubernetes에서 특정 작업을 완료할 때까지 Pod를 실행하고 관리하는 Workload 리소스입니다.

- 작업이 완료되면 Job은 Complete 상태가 됩니다.
- Pod가 실패하면 설정된 재시도 정책에 따라 작업을 다시 수행할 수 있습니다.
- `completions`를 통해 성공적으로 완료해야 하는 작업 횟수를 지정할 수 있습니다.
- `parallelism`을 통해 동시에 실행할 Pod의 최대 개수를 지정할 수 있습니다.
- Indexed Job을 사용하면 각 작업에 고유한 인덱스를 부여할 수 있습니다.
- `backoffLimit`을 통해 실패 재시도 횟수를 제한할 수 있습니다.
- `activeDeadlineSeconds`를 통해 Job의 최대 실행 시간을 제한할 수 있습니다.
- `ttlSecondsAfterFinished`를 통해 완료되거나 실패한 Job을 일정 시간이 지난 후 자동으로 정리할 수 있습니다.
- CronJob은 일정에 따라 Job을 생성하며, 실제 작업은 Job이 생성한 Pod가 수행합니다.
- Job은 애플리케이션의 작업이 정확히 한 번만 실행되는 것을 보장하지 않으므로 멱등성을 고려해야 합니다.

Job은 지속적으로 실행되는 애플리케이션이 아니라, 특정 작업을 수행하고 완료되는 실행 단위를 관리하기 위한 Workload 리소스입니다.

Deployment가 애플리케이션의 지속적인 실행 상태를 유지하는 데 적합하다면, Job은 데이터 처리, 초기화, 마이그레이션 등 완료 시점이 명확한 작업을 실행하는 데 적합합니다.