# etcd

## 1. etcd란?

etcd는 분산 환경에서 데이터를 안정적으로 저장하고 관리하기 위한 **분산 Key-Value 저장소(Distributed Key-Value Store)**입니다.

Kubernetes에서는 클러스터의 상태와 설정 정보를 저장하는 기본 데이터 저장소로 사용됩니다. Pod, Deployment, Service, ConfigMap, Secret 등 Kubernetes API를 통해 관리되는 대부분의 리소스 정보가 etcd에 저장됩니다.

Kubernetes Control Plane에서 etcd는 다음과 같은 역할을 담당합니다.

- Kubernetes 리소스의 영구 저장소 역할을 수행합니다.
- 분산 합의 알고리즘을 통해 여러 노드에 데이터를 복제하고 일관성을 유지합니다.
- 데이터의 변경 이력을 관리하고 Watch 메커니즘을 통해 변경 사항을 전달합니다.
- 장애 상황에서도 저장된 데이터를 복구할 수 있도록 백업 및 복구 기능을 제공합니다.

Kubernetes API Server는 etcd에 직접 접근하는 대신 etcd와 통신하는 역할을 담당합니다. 따라서 Kubernetes 리소스에 대한 읽기 및 쓰기 요청은 일반적으로 API Server를 거쳐 처리됩니다.

## 2. 주요 역할

### 2.1. Kubernetes 클러스터 상태 저장

etcd는 Kubernetes 클러스터의 원하는 상태(Desired State)와 실제 리소스 관리에 필요한 정보를 저장합니다.

예를 들어 다음과 같은 리소스 정보가 저장됩니다.

- Pod
- Deployment
- ReplicaSet
- Service
- ConfigMap
- Secret
- Namespace
- Node

사용자가 Deployment를 생성하면 API Server는 요청을 처리한 후 해당 리소스의 상태를 etcd에 저장합니다.

이후 kube-controller-manager와 kube-scheduler 등의 Control Plane 컴포넌트는 API Server를 통해 리소스 상태를 조회하고, 저장된 상태를 기반으로 클러스터를 관리합니다.

### 2.2. 데이터 영속성 보장

etcd는 메모리에만 데이터를 보관하지 않고 디스크에 데이터를 기록하여 영속성을 보장합니다.

데이터 변경 사항은 WAL(Write-Ahead Log)에 기록되며, 주기적으로 Snapshot을 생성하여 데이터 복구와 저장 공간 관리를 지원합니다.

이러한 구조를 통해 etcd 프로세스가 재시작되거나 노드에 장애가 발생하더라도 디스크에 기록된 데이터를 기반으로 상태를 복구할 수 있습니다.

### 2.3. 분산 환경에서 데이터 일관성 유지

etcd는 여러 노드로 구성된 클러스터 환경에서 Raft 합의 알고리즘을 사용합니다.

Raft는 여러 노드가 동일한 데이터 상태를 유지하도록 합의하는 분산 합의 알고리즘입니다.

이를 통해 특정 노드에 장애가 발생하더라도 나머지 노드가 정상적으로 동작하고 Quorum을 유지한다면 데이터의 읽기 및 쓰기 작업을 계속 처리할 수 있습니다.

### 2.4. 데이터 변경 사항 감시

etcd는 Watch API를 제공하여 특정 Key 또는 Key 범위의 데이터 변경 사항을 감시할 수 있습니다.

Kubernetes API Server는 etcd의 Watch 기능을 활용하여 리소스 변경 사항을 감지하고 Kubernetes API의 Watch 기능을 통해 클라이언트에 변경 사항을 전달할 수 있습니다.

이를 통해 Kubernetes 컴포넌트와 클라이언트는 지속적으로 전체 리소스를 조회하지 않고도 변경 사항을 확인할 수 있습니다.

## 3. 데이터 저장 구조

etcd는 Key-Value 구조를 기반으로 데이터를 저장하며, 내부적으로 MVCC와 WAL 등의 메커니즘을 사용하여 데이터의 일관성과 영속성을 관리합니다.

### 3.1. Key-Value 구조

etcd는 데이터를 Key와 Value의 쌍으로 저장합니다.

- Key: 데이터를 식별하는 고유한 경로입니다.
- Value: Key에 대응하는 실제 데이터입니다.

Kubernetes 리소스는 일반적으로 다음과 같은 형태의 Key를 사용하여 etcd에 저장됩니다.

```text
/registry/pods/default/nginx
/registry/deployments/default/nginx
/registry/services/default/nginx
/registry/secrets/default/my-secret
```

위 경로는 Kubernetes가 etcd에 리소스를 저장할 때 사용하는 대표적인 Key 형태입니다.

각 Key의 Value에는 해당 리소스의 직렬화된 데이터가 저장됩니다.

### 3.2. MVCC (Multi-Version Concurrency Control)

MVCC는 하나의 Key에 대한 데이터 변경 이력을 여러 버전으로 관리하는 방식입니다.

일반적인 Key-Value 저장소에서는 기존 데이터를 새로운 값으로 덮어쓰는 방식으로 관리할 수 있지만, etcd는 데이터가 변경될 때마다 새로운 Revision을 부여하여 이전 버전과 변경 이력을 관리합니다.

예를 들어 다음과 같이 데이터가 변경되었다고 가정할 수 있습니다.

```text
Key: /config/app

Revision 10: version = "1.0"
Revision 15: version = "1.1"
Revision 20: version = "1.2"
```

이 경우 현재 값은 Revision 20에 해당하지만, 이전 Revision을 기준으로 데이터를 조회할 수도 있습니다.

MVCC를 통해 다음과 같은 기능을 지원합니다.

- 데이터 변경 이력 관리
- 특정 Revision을 기준으로 한 데이터 조회
- 동시 접근 상황에서 일관된 데이터 조회
- Watch 메커니즘을 위한 변경 이력 제공

단, 이전 버전의 데이터가 영구적으로 보존되는 것은 아닙니다. etcd의 Compaction 기능을 통해 오래된 Revision을 제거할 수 있습니다.

### 3.3. Revision

Revision은 etcd에서 데이터 변경을 식별하기 위해 사용하는 논리적인 버전 번호입니다.

etcd는 클러스터 전체에서 변경 사항을 구분할 수 있도록 Revision을 관리합니다.

Revision은 다음과 같은 특징을 가집니다.

- 데이터 변경이 발생하면 Revision이 증가합니다.
- 하나의 트랜잭션에서 여러 Key가 변경되더라도 해당 변경 사항은 동일한 Revision에 포함될 수 있습니다.
- Revision을 통해 특정 시점의 데이터 상태를 조회할 수 있습니다.
- Watch 기능은 특정 Revision 이후에 발생한 변경 사항을 확인하는 데 활용됩니다.

Revision은 etcd의 데이터 변경 이력을 관리하는 핵심 개념이며, MVCC와 Watch 메커니즘의 기반이 됩니다.

Kubernetes의 Resource Version 역시 리소스 변경을 식별하는 데 사용되지만, etcd Revision과 동일한 개념으로 취급해서는 안 됩니다. Kubernetes Resource Version은 API 사용자가 불투명한 식별자로 취급해야 하는 값이며, 내부 저장소의 Revision과 직접적인 일대일 대응을 가정해서는 안 됩니다.

### 3.4. WAL (Write-Ahead Log)

WAL은 실제 데이터 변경 사항을 저장하기 전에 로그에 먼저 기록하는 방식입니다.

etcd는 데이터 변경 사항을 WAL에 기록하여 장애가 발생했을 때 기록된 로그를 기반으로 데이터를 복구할 수 있도록 합니다.

주요 특징은 다음과 같습니다.

- 데이터 변경 사항을 로그 형태로 기록합니다.
- 로그 기록을 통해 장애 발생 시 데이터 복구를 지원합니다.
- Raft 로그와 연계하여 클러스터 내 데이터 변경 사항을 관리합니다.

WAL은 etcd의 데이터 영속성과 복구 가능성을 뒷받침하는 중요한 구성 요소입니다.

### 3.5. Snapshot

etcd는 저장된 데이터를 Snapshot 형태로 관리하여 데이터 복구와 저장 공간 관리를 지원합니다.

Snapshot은 WAL에 기록된 변경 사항을 일정 시점의 상태로 정리하는 데 사용됩니다.

etcd의 Snapshot은 크게 다음 두 가지 관점에서 구분할 수 있습니다.

- **내부 Snapshot:** etcd가 자체적으로 데이터 상태를 관리하고 WAL을 정리하기 위해 사용하는 Snapshot입니다.
- **운영 백업 Snapshot:** 관리자가 `etcdctl snapshot save` 등을 사용하여 장애 복구 목적으로 별도 생성하는 백업 데이터입니다.

두 Snapshot은 목적이 다르므로 구분해서 이해해야 합니다.

## 4. Raft 합의 알고리즘

### 4.1. Raft란?

Raft는 분산 환경에서 여러 노드가 동일한 데이터 상태를 유지하도록 합의하는 분산 합의 알고리즘입니다.

etcd는 Raft를 사용하여 클러스터 내 데이터 변경 사항을 복제하고, 여러 노드가 동일한 변경 사항을 적용하도록 보장합니다.

Raft는 다음과 같은 역할을 수행합니다.

- 클러스터의 Leader 선출
- 데이터 변경 사항의 복제
- 노드 간 데이터 일관성 유지
- 장애 발생 시 새로운 Leader 선출

etcd 클러스터에서는 일반적으로 하나의 Leader와 여러 Follower가 존재합니다.

### 4.2. Leader, Follower, Candidate

Raft에서는 노드의 상태를 다음 세 가지로 구분합니다.

| 상태 | 역할 |
| --- | --- |
| Leader | 클라이언트의 쓰기 요청을 처리하고 변경 사항을 다른 노드에 복제합니다. |
| Follower | Leader로부터 로그를 전달받아 복제하고 상태를 유지합니다. |
| Candidate | Leader가 없거나 기존 Leader에 장애가 발생했을 때 새로운 Leader 선출에 참여합니다. |

정상적인 클러스터에서는 하나의 Leader가 선출되며 나머지 노드는 Follower 상태로 동작합니다.

### 4.3. Leader Election

Leader Election은 Raft 클러스터에서 Leader를 선출하는 과정입니다.

각 노드는 일정 시간 동안 Leader의 Heartbeat를 받지 못하면 Leader에 장애가 발생했다고 판단하고 새로운 Leader 선출을 시작할 수 있습니다.

일반적인 과정은 다음과 같습니다.

1. Follower가 일정 시간 동안 Leader의 Heartbeat를 받지 못합니다.
2. 해당 노드는 Candidate 상태로 전환됩니다.
3. 새로운 Term을 시작하고 다른 노드에 투표를 요청합니다.
4. 과반수 이상의 투표를 확보한 노드가 새로운 Leader로 선출됩니다.
5. 새로운 Leader는 Follower에 Heartbeat를 전달하며 클러스터를 관리합니다.

Leader Election이 진행되는 동안에는 일시적으로 쓰기 요청 처리가 중단될 수 있습니다.

### 4.4. Quorum

Quorum은 Raft 클러스터에서 합의를 성립시키기 위해 필요한 최소 노드 수를 의미합니다.

etcd는 과반수 이상의 노드가 동의해야 데이터 변경 사항을 커밋할 수 있습니다.

필요한 Quorum은 다음과 같이 계산합니다.

```text
Quorum = floor(N / 2) + 1
```

- N: etcd 클러스터의 전체 노드 수
- Quorum: 합의를 위해 필요한 최소 노드 수

예를 들어 다음과 같습니다.

| 노드 수 | Quorum | 허용 가능한 장애 노드 수 |
| --- | --- | --- |
| 1 | 1 | 0 |
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

3개 노드로 구성된 etcd 클러스터에서는 2개 노드가 정상적으로 동작하면 Quorum을 유지할 수 있습니다.

반대로 2개 이상의 노드에 장애가 발생하면 Quorum을 확보할 수 없으므로 새로운 데이터 변경 사항을 커밋할 수 없습니다.

Quorum은 etcd의 고가용성을 이해하는 핵심 개념입니다. 노드 수가 많다고 무조건 고가용성이 높아지는 것은 아니며, 과반수 합의를 유지할 수 있는 노드 구성과 장애 도메인을 고려해야 합니다.

### 4.5. 홀수 개 노드 구성

etcd 클러스터는 일반적으로 홀수 개의 노드로 구성합니다.

예를 들어 3개 노드와 4개 노드의 Quorum을 비교하면 다음과 같습니다.

| 노드 수 | Quorum | 허용 가능한 장애 노드 수 |
| --- | --- | --- |
| 3 | 2 | 1 |
| 4 | 3 | 1 |

4개 노드로 구성하더라도 3개 노드의 동의가 필요하므로 3개 노드 구성보다 장애 허용 범위가 증가하지 않습니다.

반면 5개 노드로 구성하면 2개 노드까지 장애를 허용할 수 있습니다.

따라서 일반적으로 3개 또는 5개 노드 구성을 사용하며, 비용과 장애 허용 범위를 함께 고려하여 클러스터 크기를 결정합니다.

## 5. etcd 클러스터 구성

etcd는 단일 노드 또는 여러 노드로 구성할 수 있습니다.

### 5.1. 단일 노드 구성

단일 노드 etcd는 하나의 노드에서 etcd 프로세스를 실행하는 구성입니다.

```text
+----------------------+
| Kubernetes Control   |
| Plane                |
|                      |
|  +----------------+  |
|  | kube-apiserver |  |
|  +----------------+  |
|          |           |
+----------|-----------+
           |
           v
+----------------------+
|        etcd          |
|                      |
|      Single Node     |
+----------------------+
```

주요 특징은 다음과 같습니다.

- 구성이 단순합니다.
- 운영 및 관리 비용이 낮습니다.
- 단일 노드 장애가 발생하면 etcd를 사용할 수 없습니다.
- 고가용성이 필요한 운영 환경에는 적합하지 않을 수 있습니다.

개발 및 테스트 환경에서 주로 사용됩니다.

### 5.2. 다중 노드 구성

다중 노드 etcd는 여러 노드에 etcd를 배치하여 데이터를 복제하고 고가용성을 확보하는 구성입니다.

```text
              +----------------+
              | kube-apiserver |
              +----------------+
                      |
                      v
             +-------------------+
             |    etcd Cluster   |
             |                   |
             |  +-------------+  |
             |  | etcd Node 1 |  |
             |  |   Leader    |  |
             |  +-------------+  |
             |        |          |
             |  +-------------+  |
             |  | etcd Node 2 |  |
             |  |  Follower   |  |
             |  +-------------+  |
             |        |          |
             |  +-------------+  |
             |  | etcd Node 3 |  |
             |  |  Follower   |  |
             |  +-------------+  |
             +-------------------+
```

주요 특징은 다음과 같습니다.

- 여러 노드가 데이터를 복제하여 저장합니다.
- 일부 노드에 장애가 발생해도 Quorum을 유지하면 서비스를 계속할 수 있습니다.
- Leader 장애 발생 시 새로운 Leader를 선출합니다.
- 노드 간 네트워크 통신과 데이터 복제로 인해 운영 복잡도가 증가합니다.

운영 환경에서는 일반적으로 3개 이상의 노드로 구성하여 고가용성을 확보합니다.

### 5.3. Stacked etcd와 External etcd

Kubernetes Control Plane에서 etcd를 배치하는 방식은 크게 Stacked etcd와 External etcd로 구분할 수 있습니다.

#### Stacked etcd

Stacked etcd는 etcd 인스턴스를 각 Control Plane 노드에 함께 배치하는 구성입니다.

일반적으로 kubeadm의 기본적인 고가용성 Control Plane 구성에서 사용됩니다.

```text
+-----------------------------+
|       Control Plane 1       |
|                             |
|  +-----------------------+  |
|  |    kube-apiserver     |  |
|  +-----------------------+  |
|  +-----------------------+  |
|  |        etcd           |  |
|  +-----------------------+  |
+-----------------------------+

+-----------------------------+
|       Control Plane 2       |
|                             |
|  +-----------------------+  |
|  |    kube-apiserver     |  |
|  +-----------------------+  |
|  +-----------------------+  |
|  |        etcd           |  |
|  +-----------------------+  |
+-----------------------------+

+-----------------------------+
|       Control Plane 3       |
|                             |
|  +-----------------------+  |
|  |    kube-apiserver     |  |
|  +-----------------------+  |
|  +-----------------------+  |
|  |        etcd           |  |
|  +-----------------------+  |
+-----------------------------+
```

주요 특징은 다음과 같습니다.

- Control Plane 노드와 etcd 노드가 함께 배치됩니다.
- 별도의 etcd 서버를 구축하지 않아도 됩니다.
- Control Plane 노드 장애가 etcd 멤버 장애로도 이어질 수 있습니다.
- Control Plane과 etcd의 리소스 사용량이 같은 노드에 영향을 줍니다.

#### External etcd

External etcd는 etcd를 Kubernetes Control Plane 노드와 분리된 별도의 노드에 배치하는 구성입니다.

```text
+-----------------------------+
|       Control Plane 1       |
|                             |
|  +-----------------------+  |
|  |    kube-apiserver     |  |
|  +-----------------------+  |
+-----------------------------+

+-----------------------------+
|       Control Plane 2       |
|                             |
|  +-----------------------+  |
|  |    kube-apiserver     |  |
|  +-----------------------+  |
+-----------------------------+

+-----------------------------+
|       Control Plane 3       |
|                             |
|  +-----------------------+  |
|  |    kube-apiserver     |  |
|  +-----------------------+  |
+-----------------------------+

             |
             v

+-----------------------------+
|        External etcd        |
|                             |
|  +-----------------------+  |
|  |      etcd Node 1      |  |
|  +-----------------------+  |
|  |      etcd Node 2      |  |
|  +-----------------------+  |
|  |      etcd Node 3      |  |
|  +-----------------------+  |
+-----------------------------+
```

주요 특징은 다음과 같습니다.

- etcd와 Control Plane의 운영 및 리소스를 분리할 수 있습니다.
- etcd를 독립적으로 확장하고 관리할 수 있습니다.
- Control Plane 노드 장애와 etcd 노드 장애의 영향 범위를 분리할 수 있습니다.
- 별도의 etcd 노드 운영과 네트워크 구성이 필요합니다.

#### Stacked etcd와 External etcd 비교

| 구분 | Stacked etcd | External etcd |
| --- | --- | --- |
| 배치 방식 | Control Plane 노드에 함께 배치 | 별도의 노드에 배치 |
| 인프라 구성 | 상대적으로 단순 | 상대적으로 복잡 |
| 리소스 관리 | Control Plane과 리소스 공유 | 독립적인 리소스 관리 |
| 장애 영향 | Control Plane과 etcd 장애가 연결될 수 있음 | 장애 영역을 분리할 수 있음 |
| 운영 복잡도 | 상대적으로 낮음 | 상대적으로 높음 |
| 주요 고려 사항 | 노드 장애 시 Control Plane과 etcd의 영향 | 네트워크, 인증서, etcd 별도 운영 |

두 구성 모두 etcd를 여러 노드로 구성하여 고가용성을 확보할 수 있습니다. 핵심 차이는 etcd가 Control Plane 노드에 함께 배치되는지, 별도의 노드에 분리되는지에 있습니다.

## 6. API Server와 etcd의 관계

### 6.1. 데이터 접근 구조

Kubernetes에서 etcd는 클러스터 상태를 저장하는 데이터 저장소이며, kube-apiserver는 Kubernetes API 요청을 처리하면서 etcd에 데이터를 읽고 쓰는 역할을 담당합니다.

일반적인 데이터 접근 흐름은 다음과 같습니다.

```text
       Kubernetes Client
                |
                v
        +---------------+
        | kube-apiserver|
        +---------------+
                |
                v
        +---------------+
        |     etcd      |
        +---------------+
```

예를 들어 사용자가 `kubectl apply` 명령어로 Deployment를 생성하면 다음과 같은 과정으로 처리됩니다.

1. kubectl이 kube-apiserver에 API 요청을 전달합니다.
2. kube-apiserver가 인증, 인가, Admission Control 등의 요청 처리 과정을 수행합니다.
3. 요청이 정상적으로 처리되면 kube-apiserver가 etcd에 리소스 데이터를 저장합니다.
4. 이후 kube-scheduler와 kube-controller-manager가 API Server를 통해 리소스 상태를 확인하고 필요한 작업을 수행합니다.

### 6.2. 책임 구분

kube-apiserver와 etcd는 서로 다른 책임을 담당합니다.

| 구성 요소 | 주요 책임 |
| --- | --- |
| kube-apiserver | Kubernetes API 제공, 요청 검증, 인증 및 인가, 리소스 관리 |
| etcd | 데이터 영속성, 데이터 복제, 분산 합의, 데이터 일관성 유지 |

etcd는 Kubernetes API나 Pod, Deployment와 같은 Kubernetes 리소스의 개념을 직접 이해하지 않습니다.

etcd는 Key-Value 저장소로서 데이터를 저장하고 관리하며, Kubernetes 리소스의 의미와 API 동작은 kube-apiserver가 담당합니다.

### 6.3. etcd 장애가 Kubernetes에 미치는 영향

etcd는 Kubernetes 클러스터 상태를 저장하는 핵심 구성 요소이므로 장애가 발생하면 클러스터의 관리 기능에 영향을 미칩니다.

다만 etcd 장애가 발생했다고 해서 이미 실행 중인 모든 Pod가 즉시 종료되는 것은 아닙니다.

| 장애 상황 | 영향 |
| --- | --- |
| etcd 단일 노드 장애 | 단일 노드 구성에서는 API Server의 데이터 접근이 불가능해집니다. |
| 다중 노드 중 일부 장애 | Quorum을 유지하면 클러스터가 정상적으로 동작할 수 있습니다. |
| Quorum 상실 | 새로운 데이터 변경 사항을 커밋할 수 없습니다. |
| etcd 전체 장애 | API Server의 정상적인 리소스 읽기 및 쓰기가 불가능해집니다. |

이미 Worker Node에서 실행 중인 Pod는 kubelet과 컨테이너 런타임에 의해 계속 동작할 수 있습니다.

그러나 API Server를 통한 리소스 관리, 스케줄링, 컨트롤러의 상태 조정 등 Control Plane의 정상적인 동작은 제한됩니다.

따라서 etcd의 고가용성 구성과 정기적인 백업은 Kubernetes 클러스터 운영에서 매우 중요합니다.

## 7. Watch 메커니즘

### 7.1. Watch란?

Watch는 etcd에서 특정 Key 또는 Key 범위의 데이터 변경 사항을 실시간으로 감시하는 기능입니다.

클라이언트는 특정 Revision 이후에 발생하는 변경 사항을 구독할 수 있으며, 데이터가 변경되면 etcd가 변경 이벤트를 전달합니다.

일반적인 조회 방식과 Watch 방식은 다음과 같이 비교할 수 있습니다.

| 구분 | 일반 조회 | Watch |
| --- | --- | --- |
| 동작 방식 | 클라이언트가 직접 데이터를 조회 | 변경 이벤트를 구독 |
| 데이터 확인 | 반복적인 조회 필요 | 변경 발생 시 이벤트 전달 |
| 네트워크 요청 | 주기적으로 요청 발생 | 연결을 유지하며 변경 사항 수신 |
| 주요 목적 | 특정 시점의 데이터 확인 | 지속적인 변경 사항 감시 |

### 7.2. Revision과 Watch의 관계

etcd의 Watch는 Revision을 기반으로 동작합니다.

클라이언트가 특정 Revision을 기준으로 Watch를 생성하면 해당 Revision 이후에 발생한 데이터 변경 사항을 전달받을 수 있습니다.

예를 들어 다음과 같은 변경 이력이 있다고 가정하겠습니다.

```text
Revision 10: ConfigMap 생성
Revision 15: ConfigMap 수정
Revision 20: ConfigMap 수정
Revision 25: ConfigMap 삭제
```

클라이언트가 Revision 15부터 Watch를 시작하면 이후 발생한 Revision 20과 Revision 25의 변경 이벤트를 확인할 수 있습니다.

이러한 방식은 클라이언트가 매번 전체 데이터를 조회하지 않고도 변경 사항을 추적할 수 있도록 합니다.

단, 요청한 시작 Revision이 이미 Compaction된 상태라면 해당 Revision부터의 변경 이력을 제공할 수 없습니다. 이 경우 최신 상태를 다시 조회한 뒤 새로운 Revision을 기준으로 Watch를 시작해야 합니다.

### 7.3. Kubernetes에서의 Watch 활용

Kubernetes API Server는 etcd의 Watch 기능을 활용하여 리소스 변경 사항을 감지합니다.

이후 Kubernetes API의 Watch 기능을 통해 클라이언트에 변경 사항을 전달할 수 있습니다.

예를 들어 kube-controller-manager는 API Server를 통해 리소스 변경 사항을 감지하고, 해당 상태를 기반으로 컨트롤러의 조정 작업(Reconciliation)을 수행합니다.

전체 흐름은 다음과 같습니다.

```text
+----------------+
|      etcd      |
+----------------+
        |
        | 데이터 변경 이벤트
        v
+----------------+
| kube-apiserver |
+----------------+
        |
        | Kubernetes API Watch
        v
+----------------+
|  Controller    |
+----------------+
        |
        | 상태 조정 요청
        v
+----------------+
| kube-apiserver |
+----------------+
```

etcd의 Watch와 Kubernetes API의 Watch는 서로 다른 계층의 기능입니다.

etcd의 Watch는 저장소 수준의 데이터 변경을 감지하고, Kubernetes API의 Watch는 Kubernetes 리소스 변경 사항을 API 사용자에게 전달합니다.

## 8. 보안 및 데이터 보호

### 8.1. 접근 제어

etcd는 Kubernetes 클러스터의 중요한 상태 정보를 저장하므로 접근 권한을 엄격하게 관리해야 합니다.

일반적으로 kube-apiserver만 etcd에 접근할 수 있도록 구성하고, 외부 클라이언트의 직접 접근은 제한합니다.

etcd의 접근 제어는 네트워크 정책, 방화벽, 인증서 등을 통해 관리할 수 있습니다.

### 8.2. TLS 통신

etcd는 TLS를 사용하여 클라이언트와 서버 간 통신을 암호화할 수 있습니다.

Kubernetes에서는 일반적으로 kube-apiserver와 etcd 간 통신에 TLS를 적용합니다.

주요 인증서 구성은 다음과 같습니다.

- CA 인증서: 인증서의 신뢰 체인을 구성합니다.
- Server Certificate: etcd 서버의 신원을 확인합니다.
- Client Certificate: etcd에 접근하는 클라이언트의 신원을 확인합니다.

mTLS(Mutual TLS)를 적용하면 클라이언트와 서버가 서로의 신원을 확인할 수 있습니다.

### 8.3. Encryption at Rest

Kubernetes는 Encryption at Rest 기능을 통해 etcd에 저장되는 특정 리소스 데이터를 암호화할 수 있습니다.

이 기능은 kube-apiserver가 리소스 데이터를 etcd에 저장하기 전에 암호화하는 방식으로 동작합니다.

특히 Secret과 같이 민감한 데이터를 저장하는 리소스에 적용할 수 있습니다.

Encryption at Rest는 etcd 자체의 디스크 암호화와는 다른 기능입니다.

- Kubernetes Encryption at Rest: API Server가 etcd에 데이터를 저장하기 전에 리소스 데이터를 암호화합니다.
- 디스크 암호화: 운영체제 또는 스토리지 계층에서 디스크에 기록되는 데이터를 암호화합니다.

두 방식은 서로 다른 계층에서 동작하며, 보안 요구사항에 따라 함께 적용할 수 있습니다.

## 9. 백업 및 복구

### 9.1. Snapshot Backup

etcd는 `etcdctl snapshot save` 명령어를 사용하여 현재 데이터 상태를 Snapshot으로 백업할 수 있습니다.

예를 들어 다음과 같이 백업을 수행할 수 있습니다.

```bash
ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

백업이 정상적으로 생성되었는지 확인하려면 다음 명령어를 사용할 수 있습니다.

```bash
etcdutl snapshot status snapshot.db --write-out=table
```

백업 데이터는 etcd 데이터 디렉터리와 분리된 안전한 위치에 보관하는 것이 좋습니다.

또한 백업 파일의 존재 여부만 확인하는 것이 아니라 실제 복구 절차를 정기적으로 검증해야 합니다.

### 9.2. Snapshot Restore

Snapshot Restore는 백업한 Snapshot을 사용하여 etcd 데이터를 복구하는 과정입니다.

다음과 같이 복구할 수 있습니다.

```bash
etcdutl snapshot restore snapshot.db \
  --data-dir=/var/lib/etcd-restored
```

복구 과정에서는 기존 데이터 디렉터리와 분리된 새로운 데이터 디렉터리를 지정하는 것이 일반적입니다.

다중 노드 클러스터를 복구하는 경우에는 각 멤버의 이름, 데이터 디렉터리, 초기 클러스터 구성 등을 올바르게 지정해야 합니다.

또한 Kubernetes에서 etcd를 복구할 때는 API Server와 etcd 간의 연결 설정, 인증서, 클러스터 구성 등을 함께 확인해야 합니다.

### 9.3. etcdctl과 etcdutl

etcd 운영에서는 `etcdctl`과 `etcdutl`을 구분해서 사용합니다.

| 도구 | 주요 역할 |
| --- | --- |
| etcdctl | etcd 클러스터 상태 조회, 데이터 조회 및 변경, Snapshot 생성 등 |
| etcdutl | Snapshot 상태 확인 및 복구 등 관리 작업 |

etcd 버전에 따라 일부 명령어의 지원 여부와 사용 방식이 달라질 수 있으므로 실제 운영 환경의 etcd 버전에 맞는 공식 문서를 확인해야 합니다.

## 10. 주요 실행 옵션

etcd를 실행할 때 사용하는 주요 옵션은 다음과 같습니다.

| 옵션 | 설명 |
| --- | --- |
| `--name` | etcd 멤버의 이름을 지정합니다. |
| `--data-dir` | etcd 데이터 디렉터리를 지정합니다. |
| `--listen-client-urls` | 클라이언트 요청을 수신할 주소를 지정합니다. |
| `--advertise-client-urls` | 클라이언트에 공개할 접속 주소를 지정합니다. |
| `--listen-peer-urls` | 다른 etcd 멤버와 통신할 주소를 지정합니다. |
| `--initial-advertise-peer-urls` | 초기 클러스터 구성에서 다른 멤버에 공개할 Peer 주소를 지정합니다. |
| `--initial-cluster` | 초기 클러스터 멤버 구성을 지정합니다. |
| `--initial-cluster-state` | 초기 클러스터 구성 상태를 지정합니다. |
| `--client-cert-auth` | 클라이언트 인증서 기반 인증을 활성화합니다. |
| `--trusted-ca-file` | 클라이언트 인증서 검증에 사용할 CA 인증서를 지정합니다. |
| `--cert-file` | 클라이언트 통신에 사용할 서버 인증서를 지정합니다. |
| `--key-file` | 서버 인증서에 대응하는 개인 키를 지정합니다. |

운영 환경에서는 etcd의 데이터 디렉터리, 네트워크 주소, 인증서 및 클러스터 구성을 적절하게 설정해야 합니다.

## 11. 정리

etcd는 Kubernetes 클러스터의 상태를 저장하고 관리하는 분산 Key-Value 저장소입니다.

Kubernetes API Server는 etcd를 통해 리소스 데이터를 영구적으로 저장하고 조회하며, etcd는 분산 합의 알고리즘과 데이터 버전 관리 메커니즘을 통해 클러스터의 데이터 일관성과 안정성을 유지합니다.

etcd를 이해할 때는 다음 세 가지 개념을 특히 구분해서 이해해야 합니다.

- **Raft와 Quorum:** 여러 노드에 데이터를 복제하고 합의를 통해 일관성을 유지하는 방식입니다. Quorum은 데이터 변경 사항을 커밋하고 클러스터의 고가용성을 유지하기 위해 필요한 최소 노드 수를 결정합니다.
- **MVCC와 Revision:** 데이터 변경 이력을 여러 버전으로 관리하는 구조와 각 변경 사항을 식별하는 논리적인 버전 번호입니다. 이 구조는 특정 시점의 데이터 조회와 Watch 메커니즘의 기반이 됩니다.
- **Stacked etcd와 External etcd:** Kubernetes Control Plane에서 etcd를 배치하는 두 가지 아키텍처입니다. Control Plane 노드와 함께 배치할지, 별도의 노드에 분리할지에 따라 운영 구조와 장애 영향 범위가 달라집니다.

이 세 가지 개념을 함께 이해하면 etcd가 단순히 Kubernetes 데이터를 저장하는 데이터베이스가 아니라, **분산 합의, 데이터 일관성, 변경 이력 관리 및 고가용성을 통해 Kubernetes Control Plane의 안정성을 뒷받침하는 분산 시스템**이라는 점을 이해할 수 있습니다.

## 12. 참고자료

### Kubernetes 공식 문서

- [Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
- [Encrypting Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- [Securing a Cluster](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/)
- [Creating Highly Available Clusters with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)

### etcd 공식 문서

- [etcd Documentation](https://etcd.io/docs/)
- [etcd FAQ](https://etcd.io/docs/v3.6/faq/)
- [etcd Data Model](https://etcd.io/docs/v3.6/learning/data_model/)
- [etcd API Guarantees](https://etcd.io/docs/v3.6/learning/api_guarantees/)
- [etcd Recovery](https://etcd.io/docs/v3.6/op-guide/recovery/)
- [etcd Maintenance](https://etcd.io/docs/v3.6/op-guide/maintenance/)