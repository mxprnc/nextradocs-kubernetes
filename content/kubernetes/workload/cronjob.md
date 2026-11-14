# CronJob

## 1. CronJob이란?

**CronJob은 Kubernetes에서 지정된 일정에 따라 Job을 반복적으로 생성하고 실행하는 Workload 리소스입니다.**

Job은 특정 작업을 완료할 때까지 Pod를 실행하고 관리합니다. 반면 CronJob은 작업을 실행할 시점을 정의하고, 해당 일정이 도래하면 Job을 생성합니다.

CronJob 자체가 컨테이너를 직접 실행하는 것은 아닙니다. CronJob이 Job을 생성하고, Job이 Pod를 생성하여 실제 작업을 수행하는 구조입니다.

예를 들어 다음과 같은 작업에 CronJob을 사용할 수 있습니다.

- 매일 새벽 데이터베이스 백업
- 매시간 로그 및 임시 파일 정리
- 매일 특정 시간에 통계 데이터 집계
- 정기적인 데이터 동기화
- 주기적인 리포트 생성
- 만료된 데이터 삭제 및 정리

CronJob의 주요 특징은 다음과 같습니다.

- Cron 표현식을 사용하여 작업 실행 일정을 지정합니다.
- 지정된 일정에 따라 Job을 생성합니다.
- Job Template을 통해 실행할 작업의 구성을 정의합니다.
- 동시 실행 정책을 통해 이전 작업이 완료되지 않았을 때의 동작을 제어할 수 있습니다.
- 실행 시작 기한을 설정하여 일정이 누락된 작업의 실행 여부를 제어할 수 있습니다.
- 성공 및 실패한 Job의 보관 개수를 설정할 수 있습니다.
- 특정 시간대를 지정하여 작업 실행 일정을 관리할 수 있습니다.

CronJob은 정기적으로 반복 실행해야 하는 작업을 Kubernetes에서 선언적으로 관리하기 위한 리소스입니다.

## 2. CronJob의 구조

CronJob은 지정된 일정에 따라 Job을 생성하고, Job은 Pod를 생성하여 실제 작업을 수행합니다.

```text
Kubernetes Cluster
│
└── CronJob: daily-backup
    │
    ├── Schedule: 매일 02:00
    │
    ├── Job: daily-backup-001
    │   └── Pod
    │       └── Container
    │           └── 백업 작업 실행
    │
    ├── Job: daily-backup-002
    │   └── Pod
    │       └── Container
    │           └── 백업 작업 실행
    │
    └── Job: daily-backup-003
        └── Pod
            └── Container
                └── 백업 작업 실행
```

각 실행 시점마다 새로운 Job이 생성되며, 해당 Job이 완료되면 실행 결과에 따라 성공 또는 실패 상태로 기록됩니다.

CronJob은 Job을 생성하는 역할을 담당하고, 실제 작업의 실행 및 재시도는 Job Controller가 관리합니다.

## 3. CronJob YAML 구성

다음은 매일 새벽 2시에 데이터 백업 작업을 실행하는 CronJob 예제입니다.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"
  timeZone: "Asia/Seoul"

  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 300

  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1

  jobTemplate:
    spec:
      backoffLimit: 3
      activeDeadlineSeconds: 1800

      template:
        spec:
          containers:
            - name: backup
              image: busybox:1.36
              command:
                - sh
                - -c
                - "echo 'Backup started'; sleep 10; echo 'Backup completed'"

          restartPolicy: Never
```

각 필드의 역할은 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `apiVersion` | Kubernetes API 버전을 지정합니다. CronJob은 `batch/v1`을 사용합니다. |
| `kind` | 생성할 리소스의 종류를 지정합니다. |
| `metadata` | CronJob의 이름과 메타데이터를 정의합니다. |
| `spec` | CronJob의 실행 일정과 동작 방식을 정의합니다. |
| `schedule` | Job을 생성할 실행 일정을 Cron 표현식으로 지정합니다. |
| `timeZone` | Cron 표현식을 해석할 시간대를 지정합니다. |
| `concurrencyPolicy` | 이전 Job이 실행 중일 때 새로운 Job의 실행 방식을 지정합니다. |
| `startingDeadlineSeconds` | 예정된 실행 시점을 놓쳤을 때 Job을 시작할 수 있는 최대 지연 시간을 지정합니다. |
| `successfulJobsHistoryLimit` | 보관할 성공한 Job의 최대 개수를 지정합니다. |
| `failedJobsHistoryLimit` | 보관할 실패한 Job의 최대 개수를 지정합니다. |
| `jobTemplate` | CronJob이 생성할 Job의 구성을 정의합니다. |
| `backoffLimit` | Job이 실패했을 때 재시도할 수 있는 횟수를 지정합니다. |
| `activeDeadlineSeconds` | Job의 최대 실행 시간을 초 단위로 지정합니다. |
| `template` | Job이 생성할 Pod의 구성을 정의합니다. |

`jobTemplate` 내부의 구성은 일반적인 Job의 `spec`과 동일한 구조를 사용합니다. 다만 CronJob 내부에 중첩되는 형태이므로 `apiVersion`과 `kind`는 지정하지 않습니다.

## 4. CronJob의 주요 특징

### 4.1. Cron 표현식 (Schedule)

CronJob은 `schedule` 필드에 Cron 표현식을 지정하여 Job의 실행 일정을 정의합니다.

```yaml
spec:
  schedule: "0 2 * * *"
```

위 설정은 매일 오전 2시에 Job을 생성하도록 지정합니다.

Cron 표현식은 공백으로 구분되는 5개의 필드로 구성됩니다.

```text
# ┌───────────── 분 (0 - 59)
# │ ┌───────────── 시 (0 - 23)
# │ │ ┌───────────── 일 (1 - 31)
# │ │ │ ┌───────────── 월 (1 - 12)
# │ │ │ │ ┌───────────── 요일 (0 - 6, 일요일부터 토요일)
# │ │ │ │ │
# * * * * *
```

각 필드의 의미는 다음과 같습니다.

| 필드 | 허용 범위 | 설명 |
|---|---|---|
| 첫 번째 | `0-59` | 분 |
| 두 번째 | `0-23` | 시 |
| 세 번째 | `1-31` | 일 |
| 네 번째 | `1-12` | 월 |
| 다섯 번째 | `0-6` | 요일 (0은 일요일) |

Cron 표현식에서 자주 사용하는 문자는 다음과 같습니다.

| 문자 | 의미 | 예시 |
|---|---|---|
| `*` | 모든 값 | `* * * * *` |
| `,` | 여러 값 지정 | `0,30 * * * *` |
| `-` | 범위 지정 | `0 9-18 * * *` |
| `/` | 일정한 간격 지정 | `*/10 * * * *` |

대표적인 Cron 표현식은 다음과 같습니다.

| Cron 표현식 | 실행 일정 |
|---|---|
| `* * * * *` | 매분 실행 |
| `*/5 * * * *` | 5분마다 실행 |
| `0 * * * *` | 매시간 정각에 실행 |
| `0 2 * * *` | 매일 오전 2시에 실행 |
| `0 9 * * 1-5` | 평일 오전 9시에 실행 |
| `0 0 * * 0` | 매주 일요일 자정에 실행 |
| `0 0 1 * *` | 매월 1일 자정에 실행 |

CronJob은 지정된 일정에 따라 Job을 생성하지만, 실제 Job의 생성 및 실행 시점이 Cron 표현식에 지정된 시각과 완전히 일치하는 것을 보장하지는 않습니다.

Controller의 처리 주기, 클러스터 상태 및 리소스 상황 등에 따라 실제 실행이 지연될 수 있습니다.

### 4.2. 시간대 설정 (Time Zone)

CronJob의 `timeZone` 필드는 Cron 표현식을 해석할 시간대를 지정합니다.

```yaml
spec:
  schedule: "0 2 * * *"
  timeZone: "Asia/Seoul"
```

위 설정은 한국 시간(KST)을 기준으로 매일 오전 2시에 Job을 생성합니다.

`timeZone`을 지정하지 않으면 `kube-controller-manager` 프로세스의 로컬 시간대를 기준으로 일정을 해석합니다.

따라서 운영 환경에서 시간대에 따른 실행 오류를 방지하려면 `timeZone`을 명시적으로 설정하는 것이 좋습니다.

`timeZone` 기능은 Kubernetes v1.27부터 Stable 상태로 제공됩니다.

시간대를 지정할 때는 IANA Time Zone 데이터베이스에서 사용하는 이름을 사용합니다.

대표적인 예시는 다음과 같습니다.

```yaml
# 한국 표준시
timeZone: "Asia/Seoul"

# 협정 세계시
timeZone: "Etc/UTC"

# 미국 동부 시간
timeZone: "America/New_York"
```

시간대는 `schedule` 문자열 내부에 `TZ` 또는 `CRON_TZ`를 포함하는 방식으로 지정하지 않습니다. Kubernetes에서는 `.spec.timeZone` 필드를 사용해야 합니다.

### 4.3. Job Template

`jobTemplate`은 CronJob이 실행 시점마다 생성할 Job의 구성을 정의합니다.

```yaml
spec:
  schedule: "0 2 * * *"

  jobTemplate:
    spec:
      backoffLimit: 3

      template:
        spec:
          containers:
            - name: backup
              image: busybox:1.36
              command:
                - sh
                - -c
                - "echo 'Backup started'"

          restartPolicy: Never
```

`jobTemplate` 내부에는 일반적인 Job에서 사용하는 설정을 정의할 수 있습니다.

- `completions`
- `parallelism`
- `backoffLimit`
- `activeDeadlineSeconds`
- `completionMode`
- Pod Template 및 컨테이너 설정

예를 들어 데이터베이스 백업을 수행하는 Job이라면 컨테이너 이미지, 실행 명령어, 환경 변수, Secret, Volume 등을 정의할 수 있습니다.

CronJob의 `jobTemplate`을 변경하면 변경 이후 새롭게 생성되는 Job부터 수정된 구성이 적용됩니다.

이미 생성되어 실행 중인 Job과 Pod에는 변경 사항이 소급 적용되지 않습니다.

## 5. CronJob의 동시 실행 정책 (Concurrency Policy)

CronJob은 이전 Job이 아직 실행 중인 상태에서 다음 실행 일정이 도래할 수 있습니다.

예를 들어 매시간 실행되는 작업이 90분 동안 수행된다면 이전 작업이 완료되기 전에 다음 실행 시간이 도래할 수 있습니다.

이러한 상황을 제어하기 위해 `concurrencyPolicy`를 제공합니다.

```yaml
spec:
  concurrencyPolicy: Forbid
```

지원하는 정책은 다음과 같습니다.

| 정책 | 설명 |
|---|---|
| `Allow` | 기본값입니다. 이전 Job의 실행 여부와 관계없이 새로운 Job을 생성할 수 있습니다. |
| `Forbid` | 이전 Job이 실행 중이면 새로운 Job의 실행을 건너뜁니다. |
| `Replace` | 이전 Job이 실행 중이면 기존 Job을 종료하도록 처리하고 새로운 Job을 생성합니다. |

### 5.1. Allow

`Allow`는 이전 Job이 실행 중이더라도 새로운 Job을 생성할 수 있도록 허용합니다.

```yaml
spec:
  concurrencyPolicy: Allow
```

예를 들어 10분마다 실행되는 작업이 30분 동안 수행된다면 여러 Job이 동시에 실행될 수 있습니다.

```text
시간       Job 실행 상태

10:00      Job-1 시작
10:10      Job-1 실행 중 / Job-2 시작
10:20      Job-1 실행 중 / Job-2 실행 중 / Job-3 시작
10:30      Job-1 완료 / Job-2 실행 중 / Job-3 실행 중 / Job-4 시작
```

동일한 작업을 여러 번 동시에 수행해도 문제가 없는 경우에 사용할 수 있습니다.

다만 중복 실행으로 인해 데이터 충돌이나 리소스 사용량 증가가 발생하지 않도록 주의해야 합니다.

### 5.2. Forbid

`Forbid`는 이전 Job이 완료되지 않은 상태에서 새로운 실행 일정이 도래하면 해당 실행을 건너뜁니다.

```yaml
spec:
  concurrencyPolicy: Forbid
```

예를 들어 10분마다 실행되는 작업이 30분 동안 수행된다면 다음과 같이 동작합니다.

```text
시간       Job 실행 상태

10:00      Job-1 시작
10:10      Job-1 실행 중 / Job-2 실행 건너뜀
10:20      Job-1 실행 중 / Job-3 실행 건너뜀
10:30      Job-1 완료
10:40      Job-4 시작
```

이전 작업이 완료되지 않은 경우 새로운 작업을 시작하지 않으므로 중복 실행을 방지하는 데 유용합니다.

단, 건너뛴 실행이 항상 영구적으로 누락되는 것은 아닙니다. `startingDeadlineSeconds` 등의 설정과 Controller의 다음 처리 과정에 따라 실행 가능 여부가 다시 판단될 수 있습니다.

### 5.3. Replace

`Replace`는 이전 Job이 실행 중인 상태에서 새로운 실행 일정이 도래하면 기존 Job을 대체하도록 동작합니다.

```yaml
spec:
  concurrencyPolicy: Replace
```

예를 들어 10분마다 실행되는 작업이 30분 동안 수행된다면 다음과 같이 동작합니다.

```text
시간       Job 실행 상태

10:00      Job-1 시작
10:10      Job-1 종료 처리 / Job-2 시작
10:20      Job-2 종료 처리 / Job-3 시작
10:30      Job-3 종료 처리 / Job-4 시작
```

Replace는 이전 작업의 결과가 불필요하거나 최신 작업이 우선되어야 하는 경우에 사용할 수 있습니다.

다만 기존 Job을 종료하는 과정에서 작업이 중간에 중단될 수 있으므로, 실행 중인 작업의 데이터 정합성과 종료 처리 방식을 고려해야 합니다.

### 5.4. 동시 실행 정책 비교

| 구분 | Allow | Forbid | Replace |
|---|---|---|---|
| 이전 Job 실행 중 새 일정 도래 | 새 Job 생성 | 새 Job 실행 건너뜀 | 기존 Job을 대체하고 새 Job 생성 |
| 동시 실행 | 허용 | 방지 | 방지 |
| 기존 Job 유지 | 유지 | 유지 | 종료 처리 |
| 주요 사용 사례 | 병렬 실행이 가능한 작업 | 중복 실행을 방지해야 하는 작업 | 최신 실행이 이전 실행보다 우선하는 작업 |

`concurrencyPolicy`는 동일한 CronJob이 생성한 Job 사이에만 적용됩니다. 서로 다른 CronJob이 생성한 Job 사이의 동시 실행은 제어하지 않습니다.

## 6. CronJob의 실행 기한 (startingDeadlineSeconds)

`startingDeadlineSeconds`는 예정된 실행 시점을 놓쳤을 때 Job을 시작할 수 있는 최대 지연 시간을 초 단위로 지정합니다.

```yaml
spec:
  startingDeadlineSeconds: 300
```

위 설정은 예정된 실행 시점으로부터 최대 300초 이내에 Job을 시작할 수 있도록 허용합니다.

예를 들어 오전 2시에 실행되어야 하는 Job이 Controller의 일시적인 문제로 실행되지 못했다면 다음과 같이 동작할 수 있습니다.

```text
02:00  예정된 실행 시점
  │
  ▼
Job 생성 실패 또는 지연
  │
  ▼
02:03  Controller가 실행 누락 확인
  │
  ▼
02:05  실행 기한 이내
  │
  ▼
Job 생성 가능

02:06  실행 기한 초과
  │
  ▼
해당 실행 건너뜀
```

`startingDeadlineSeconds`를 지정하지 않으면 누락된 실행에 별도의 시작 기한을 적용하지 않습니다.

다만 누락된 실행이 무조건 모두 실행되는 것은 아닙니다. Controller는 누락된 실행 횟수를 확인하며, 100회를 초과하는 누락이 발생한 경우 해당 시점의 Job 생성을 중단하고 오류를 기록할 수 있습니다.

또한 이 값은 10초 미만으로 설정하지 않는 것이 좋습니다. CronJob Controller는 일반적으로 10초 주기로 실행 일정을 확인하므로 너무 짧은 시작 기한을 설정하면 Job이 생성되지 않을 수 있습니다.

특정 실행 시점이 지나면 작업의 의미가 없어지는 경우 `startingDeadlineSeconds`를 활용할 수 있습니다.

## 7. CronJob의 실행 일시 중지 (Suspend)

`suspend`는 CronJob이 새로운 Job을 생성하지 않도록 일시 중지하는 설정입니다.

```yaml
spec:
  suspend: true
```

기본값은 `false`이며, `true`로 설정하면 이후 예정된 실행에서 새로운 Job을 생성하지 않습니다.

이미 생성되어 실행 중인 Job에는 영향을 주지 않습니다.

```text
CronJob 실행 상태

10:00  Job-1 생성 및 실행
       │
       ▼
10:05  suspend: true
       │
       ├── Job-1 계속 실행
       │
       ├── 10:10 실행 건너뜀
       │
       └── 10:20 실행 건너뜀
```

다시 실행하려면 `suspend`를 `false`로 변경합니다.

```yaml
spec:
  suspend: false
```

주의할 점은 중지된 기간에 예정되어 있던 실행이 누락된 실행으로 기록된다는 것입니다.

`startingDeadlineSeconds`를 설정하지 않은 상태에서 CronJob을 다시 활성화하면 중지 기간에 누락된 실행이 즉시 생성될 수 있습니다.

따라서 CronJob을 재개할 때는 누락된 실행이 한꺼번에 생성되어도 문제가 없는지 확인해야 합니다.

## 8. Job 실행 이력 관리

CronJob은 실행이 완료된 Job을 일정 개수만큼 보관할 수 있습니다.

이를 위해 다음 두 가지 설정을 제공합니다.

- `successfulJobsHistoryLimit`
- `failedJobsHistoryLimit`

```yaml
spec:
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
```

각 설정의 의미는 다음과 같습니다.

| 필드 | 설명 | 기본값 |
|---|---|---|
| `successfulJobsHistoryLimit` | 성공적으로 완료된 Job의 보관 개수 | 3 |
| `failedJobsHistoryLimit` | 실패한 Job의 보관 개수 | 1 |

예를 들어 성공한 Job을 3개, 실패한 Job을 1개 보관하도록 설정하면 다음과 같이 관리됩니다.

```text
CronJob: daily-backup
│
├── Successful Jobs
│   ├── Job-101
│   ├── Job-102
│   └── Job-103
│
└── Failed Jobs
    └── Job-104
```

보관 개수를 `0`으로 설정하면 해당 상태의 완료된 Job을 보관하지 않습니다.

```yaml
spec:
  successfulJobsHistoryLimit: 0
  failedJobsHistoryLimit: 0
```

이 설정은 CronJob Controller가 완료된 Job의 이력을 정리하는 기준입니다.

Job 자체의 `ttlSecondsAfterFinished`와는 별개의 기능이며, 두 설정을 함께 사용하는 경우 각각의 정리 정책을 고려해야 합니다.

실패한 Job을 보관하면 실행 상태와 로그를 확인하여 문제를 분석할 수 있으므로 운영 환경에서는 적절한 이력 보관 개수를 설정하는 것이 좋습니다.

## 9. CronJob의 동작 과정

CronJob은 Controller가 지정된 실행 일정을 확인하고, 일정이 도래하면 Job을 생성하는 방식으로 동작합니다.

일반적인 실행 과정은 다음과 같습니다.

```text
CronJob 생성
     │
     ▼
schedule 및 timeZone 확인
     │
     ▼
실행 일정 도래 여부 확인
     │
     ▼
동시 실행 정책 확인
     │
     ├── Allow
     │     └── 새 Job 생성 허용
     │
     ├── Forbid
     │     └── 이전 Job 실행 여부 확인
     │
     └── Replace
           └── 기존 Job 대체 처리
     │
     ▼
Job 생성
     │
     ▼
Job Controller가 Pod 생성
     │
     ▼
Container 작업 실행
     │
     ▼
Job 완료 또는 실패
     │
     ▼
실행 이력 관리
```

CronJob Controller는 일정에 맞춰 Job을 생성하려고 하지만, 정확히 한 번의 Job 생성이나 특정 시각에 대한 실행을 보장하지는 않습니다.

Controller의 처리 지연이나 클러스터 상태 등에 따라 특정 실행이 누락되거나 동일한 일정에 대해 Job이 중복 생성될 가능성이 있습니다.

따라서 CronJob이 실행하는 작업은 중복 실행에 안전하도록 설계해야 합니다.

## 10. CronJob과 Job의 관계

CronJob과 Job은 함께 사용되는 Workload 리소스이지만 담당하는 역할은 다릅니다.

CronJob은 작업 실행 일정을 관리하고, Job은 실제 작업의 실행과 완료 여부를 관리합니다.

```text
CronJob
│
│ schedule에 따른 실행
│
├── Job 1
│   └── Pod
│       └── Container
│
├── Job 2
│   └── Pod
│       └── Container
│
└── Job 3
    └── Pod
        └── Container
```

| 구분 | Job | CronJob |
|---|---|---|
| 주요 목적 | 작업 실행 및 완료 관리 | 일정에 따른 Job 생성 |
| 실행 방식 | 생성된 시점에 작업 수행 | Cron 표현식에 따라 반복 실행 |
| 실제 실행 단위 | Pod | Job을 생성하며 Job이 Pod 실행 |
| 재시도 관리 | `backoffLimit` 등으로 관리 | 생성한 Job의 설정을 통해 관리 |
| 동시 실행 제어 | Job 자체에는 CronJob의 동시 실행 정책이 없음 | `concurrencyPolicy`로 관리 |
| 주요 사용 사례 | 일회성 데이터 처리 | 정기적인 백업 및 배치 작업 |

예를 들어 데이터베이스를 한 번만 마이그레이션해야 한다면 Job을 직접 생성할 수 있습니다.

반면 매일 새벽 데이터베이스 백업을 수행해야 한다면 CronJob을 사용하여 일정에 따라 Job이 생성되도록 구성할 수 있습니다.

## 11. CronJob 사용 시 고려사항

### 11.1. 정확히 한 번의 실행을 보장하지 않습니다.

CronJob은 지정된 일정에 따라 Job을 생성하도록 동작하지만, 실행 시점마다 Job이 정확히 한 번만 생성된다는 것을 보장하지 않습니다.

특정 상황에서는 동일한 일정에 대해 Job이 여러 개 생성되거나, Job이 생성되지 않을 수 있습니다.

따라서 CronJob에서 실행하는 작업은 중복 실행에 안전하도록 설계해야 합니다.

예를 들어 다음과 같은 방법을 고려할 수 있습니다.

- 데이터베이스 Unique Constraint 적용
- 멱등성 키를 사용한 중복 요청 방지
- 작업 실행 이력 저장
- 분산 Lock을 활용한 중복 실행 방지
- 이미 처리한 작업인지 확인하는 로직 구현

### 11.2. 작업 실행 시간이 다음 실행 주기를 초과할 수 있습니다.

CronJob의 실행 주기보다 실제 작업 수행 시간이 길다면 이전 Job이 완료되기 전에 다음 실행 일정이 도래할 수 있습니다.

예를 들어 10분마다 실행되는 작업이 30분 동안 수행된다면 동시 실행 문제가 발생할 수 있습니다.

이러한 경우 `concurrencyPolicy`를 적절하게 설정해야 합니다.

- `Allow`: 여러 Job의 동시 실행을 허용합니다.
- `Forbid`: 이전 Job이 완료되지 않으면 새로운 실행을 건너뜁니다.
- `Replace`: 기존 Job을 대체하고 새로운 Job을 생성합니다.

작업의 특성과 데이터 정합성을 고려하여 적절한 정책을 선택해야 합니다.

### 11.3. 시간대를 명시적으로 지정해야 합니다.

`timeZone`을 지정하지 않으면 `kube-controller-manager`의 로컬 시간대를 기준으로 Cron 표현식이 해석됩니다.

클러스터 운영 환경이나 Controller 설정에 따라 예상한 실행 시간과 실제 실행 시간이 달라질 수 있습니다.

따라서 특정 지역의 시간을 기준으로 작업을 실행해야 한다면 다음과 같이 시간대를 명시하는 것이 좋습니다.

```yaml
spec:
  schedule: "0 2 * * *"
  timeZone: "Asia/Seoul"
```

### 11.4. Job Template 변경은 기존 실행에 영향을 주지 않습니다.

CronJob의 `jobTemplate`을 변경하면 이후 생성되는 Job부터 변경된 설정이 적용됩니다.

이미 생성되어 실행 중인 Job과 Pod는 기존 설정을 유지합니다.

따라서 실행 중인 작업까지 변경된 설정을 적용해야 한다면 해당 Job을 별도로 확인하고 처리해야 합니다.

### 11.5. Job의 실행 시간과 재시도 정책을 함께 설정해야 합니다.

CronJob은 Job을 생성하는 역할을 담당하므로 실제 작업의 실행 시간과 재시도 횟수는 Job의 설정을 통해 제어해야 합니다.

대표적으로 다음 설정을 활용할 수 있습니다.

- `backoffLimit`: 실패한 작업의 재시도 횟수 제한
- `activeDeadlineSeconds`: Job의 최대 실행 시간 제한
- `restartPolicy`: Pod 내부 컨테이너의 재시작 방식 지정

작업이 무한정 실행되거나 실패한 작업이 반복적으로 재시도되는 상황을 방지해야 합니다.

### 11.6. 실행 이력과 로그 보관 정책을 고려해야 합니다.

CronJob은 반복적으로 Job을 생성하므로 실행 이력이 지속적으로 증가할 수 있습니다.

`successfulJobsHistoryLimit`과 `failedJobsHistoryLimit`을 활용하여 완료된 Job의 보관 개수를 관리해야 합니다.

다만 Job과 Pod가 삭제되면 해당 리소스에서 직접 로그를 확인하기 어려워질 수 있습니다.

장기간 실행 이력을 분석해야 한다면 중앙 로그 수집 시스템이나 별도의 모니터링 시스템을 함께 사용하는 것이 좋습니다.

### 11.7. CronJob Controller의 실행 지연과 누락을 고려해야 합니다.

CronJob Controller는 실행 일정을 주기적으로 확인하므로 정확한 시각에 Job이 생성된다는 보장은 없습니다.

또한 Controller가 장시간 중단되거나 지나치게 많은 실행 일정이 누락되면 일부 실행이 생성되지 않을 수 있습니다.

특히 `startingDeadlineSeconds`와 누락된 실행 횟수 제한을 함께 고려해야 합니다.

작업이 반드시 특정 시점에 수행되어야 하는 시스템이라면 CronJob만으로 실행 시점을 보장하려고 해서는 안 됩니다. 작업 실행 여부를 별도로 모니터링하고 누락된 작업을 복구할 수 있는 구조를 마련해야 합니다.

---

## 12. kubectl 명령어

### 12.1. CronJob 조회

```bash
# 현재 Namespace의 CronJob 조회
kubectl get cronjobs

# 모든 Namespace의 CronJob 조회
kubectl get cronjobs -A

# 특정 CronJob 상세 조회
kubectl describe cronjob daily-backup

# CronJob의 YAML 확인
kubectl get cronjob daily-backup -o yaml
```

### 12.2. CronJob 생성 및 삭제

```bash
# YAML Manifest 적용
kubectl apply -f cronjob.yaml

# CronJob 삭제
kubectl delete cronjob daily-backup
```

CronJob을 삭제하면 해당 CronJob이 소유한 Job도 삭제될 수 있습니다. 이미 실행 중인 Job이 있는 경우 삭제 정책을 확인해야 합니다.

### 12.3. CronJob이 생성한 Job 조회

```bash
# 특정 CronJob이 생성한 Job 조회
kubectl get jobs -l app=daily-backup

# CronJob 이름을 기준으로 Job 조회
kubectl get jobs --sort-by=.metadata.creationTimestamp
```

CronJob이 생성한 Job에는 일반적으로 `batch.kubernetes.io/cronjob-name` 레이블이 지정됩니다.

다음과 같이 해당 레이블을 이용하여 조회할 수 있습니다.

```bash
kubectl get jobs -l batch.kubernetes.io/cronjob-name=daily-backup
```

### 12.4. Job이 생성한 Pod 조회

```bash
# 특정 Job이 생성한 Pod 조회
kubectl get pods -l job-name=<job-name>

# Pod 상세 정보 확인
kubectl describe pod <pod-name>

# Pod 로그 확인
kubectl logs <pod-name>
```

### 12.5. CronJob 일시 중지 및 재개

```bash
# CronJob 일시 중지
kubectl patch cronjob daily-backup \
  -p '{"spec":{"suspend":true}}'

# CronJob 재개
kubectl patch cronjob daily-backup \
  -p '{"spec":{"suspend":false}}'
```

### 12.6. CronJob의 실행 일정 변경

```bash
# 매일 오전 3시로 실행 일정 변경
kubectl patch cronjob daily-backup \
  -p '{"spec":{"schedule":"0 3 * * *"}}'
```

시간대를 함께 변경하려면 다음과 같이 설정할 수 있습니다.

```bash
kubectl patch cronjob daily-backup \
  -p '{"spec":{"timeZone":"Asia/Seoul"}}'
```

### 12.7. CronJob을 즉시 실행

CronJob의 일정과 관계없이 즉시 작업을 실행해야 한다면 CronJob을 기반으로 별도의 Job을 생성할 수 있습니다.

```bash
kubectl create job manual-backup \
  --from=cronjob/daily-backup
```

위 명령어는 CronJob의 Job Template을 사용하여 `manual-backup`이라는 새로운 Job을 생성합니다.

이 Job은 CronJob의 다음 실행 일정을 기다리지 않고 즉시 실행됩니다.

단, 직접 생성한 Job은 CronJob Controller가 생성한 정기 실행 Job이 아니므로 CronJob의 `concurrencyPolicy`에 의해 제어되지 않습니다.

---

## 13. 핵심 정리

CronJob은 Kubernetes에서 지정된 일정에 따라 Job을 반복적으로 생성하고 실행하는 Workload 리소스입니다.

- Cron 표현식을 사용하여 Job 생성 일정을 지정합니다.
- `timeZone`을 통해 실행 일정의 시간대를 명시할 수 있습니다.
- `jobTemplate`을 통해 실행할 Job의 구성을 정의합니다.
- `concurrencyPolicy`를 통해 이전 Job이 실행 중일 때의 동작을 제어할 수 있습니다.
- `startingDeadlineSeconds`를 통해 누락된 실행의 시작 가능 시간을 제한할 수 있습니다.
- `suspend`를 사용하여 이후 Job 생성을 일시 중지하거나 재개할 수 있습니다.
- `successfulJobsHistoryLimit`과 `failedJobsHistoryLimit`을 통해 완료된 Job의 보관 개수를 관리할 수 있습니다.
- 실제 작업의 실행과 재시도는 CronJob이 생성한 Job이 담당합니다.
- CronJob은 정확히 한 번의 Job 실행을 보장하지 않으므로 작업의 멱등성을 고려해야 합니다.
- CronJob의 실행 일정은 근사적으로 처리되므로 정확한 실행 시점이 중요한 작업은 별도의 모니터링과 복구 체계를 고려해야 합니다.

**CronJob은 정기적으로 반복 실행해야 하는 작업의 일정을 관리하고, 해당 시점에 Job을 생성하는 Workload 리소스입니다.**

Job이 특정 작업의 실행과 완료를 관리한다면, CronJob은 Job이 언제 실행되어야 하는지를 관리합니다.

## 14. 참고자료

- [Kubernetes 공식 문서 - CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [Kubernetes API Reference - CronJob](https://kubernetes.io/docs/reference/kubernetes-api/batch/cron-job-v1/)
- [Kubernetes 공식 문서 - Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [Kubernetes 공식 문서 - Automatically Clean Up Finished Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)