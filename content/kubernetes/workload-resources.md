# Kubernetes Workload Resources

Workload 리소스는 Kubernetes에서 애플리케이션을 실행하는 Pod의 생성, 복제, 배포 및 생명주기를 관리합니다.

## 1. Workload 리소스 구성

| 리소스 | 주요 역할 | 주요 특징 |
|---|---|---|
| **Pod** | 컨테이너 실행 단위 | 하나 이상의 컨테이너를 포함하는 최소 실행 단위 |
| **Deployment** | Stateless 애플리케이션 배포 | ReplicaSet 관리, Rolling Update 및 Rollback |
| **ReplicaSet** | Pod 복제본 관리 | 지정된 개수의 Pod 유지 |
| **StatefulSet** | Stateful 애플리케이션 배포 | 안정적인 식별자, 순서 및 영속 스토리지 |
| **DaemonSet** | Node별 Pod 실행 | 각 Node에 지정된 Pod 배치 |
| **Job** | 일회성 작업 실행 | 작업 완료 여부를 기준으로 Pod 관리 |
| **CronJob** | 주기적인 작업 실행 | Cron 스케줄에 따라 Job 생성 |

## 2. 리소스 관계

```text
Workload Resources
│
├── Pod
│   └── 컨테이너 실행 단위
│
├── Deployment
│   └── ReplicaSet
│       └── Pod
│
├── StatefulSet
│   └── Pod
│       └── PVC
│
├── DaemonSet
│   └── Node별 Pod
│
├── Job
│   └── 완료를 목표로 하는 Pod
│
└── CronJob
    └── Job
        └── Pod
```

## 3. 사용 목적에 따른 분류

| 목적 | 사용 리소스 | 대표적인 사용 사례 |
|---|---|---|
| Stateless 애플리케이션 | Deployment | REST API, 웹 서버 |
| Stateful 애플리케이션 | StatefulSet | MySQL, PostgreSQL, Kafka |
| Node별 에이전트 | DaemonSet | Fluent Bit, Node Exporter |
| 일회성 작업 | Job | 데이터 마이그레이션 |
| 주기적 작업 | CronJob | 정기 백업, 데이터 정산 |

## 4. 핵심 정리

- Deployment는 일반적인 Stateless 애플리케이션 배포에 사용합니다.
- StatefulSet은 안정적인 식별자와 스토리지가 필요한 애플리케이션에 사용합니다.
- DaemonSet은 Node마다 실행해야 하는 에이전트에 사용합니다.
- Job과 CronJob은 작업 완료를 목적으로 하는 배치성 애플리케이션에 사용합니다.
- ReplicaSet은 Deployment가 Pod 복제본을 관리하기 위해 사용하는 리소스입니다.

참고: https://kubernetes.io/docs/concepts/workloads/