# Kubernetes Dashboard

## 1. Kubernetes Dashboard란?

Kubernetes Dashboard는 Kubernetes 클러스터의 리소스를 웹 브라우저에서 조회하고 관리할 수 있도록 제공하는 웹 기반 사용자 인터페이스(Web UI)입니다.

Kubernetes는 기본적으로 kubectl과 Kubernetes API를 통해 클러스터를 관리합니다. Kubernetes Dashboard는 이러한 작업을 GUI 환경에서도 수행할 수 있도록 지원하며, 클러스터의 상태를 시각적으로 확인하고 Kubernetes 리소스를 생성하거나 수정할 수 있는 기능을 제공합니다.

주요 기능은 다음과 같습니다.

- Node, Namespace, Pod 등 클러스터 리소스 조회
- Deployment, StatefulSet, DaemonSet 등 Workload 관리
- Service 및 Ingress 등 네트워크 리소스 조회
- ConfigMap 및 Secret 관리
- Pod 로그 조회 및 리소스 상태 확인
- YAML 또는 JSON Manifest를 통한 리소스 생성 및 수정
- Deployment의 Scale 및 Rolling Update 관리
- CPU 및 메모리 사용량 조회

Kubernetes Dashboard는 Kubernetes의 핵심 Control Plane 구성 요소가 아닌 별도로 설치해야 하는 Addon입니다.

**참고:** Kubernetes Dashboard 프로젝트는 공식적으로 Deprecated 및 Archived 상태이며 현재 유지보수가 종료되었습니다. 기존 환경에 대한 이해를 위해 개념을 정리하되, 신규 환경에서는 공식적으로 대안으로 안내되는 Headlamp 등의 도구를 고려해야 합니다.

## 2. 주요 역할

### 2.1. 클러스터 리소스 시각화

Kubernetes Dashboard는 Kubernetes API를 통해 클러스터의 리소스 정보를 조회하고 이를 웹 UI에 표시합니다.

관리자는 개별 리소스의 상태와 구성 정보를 확인할 수 있으며, Namespace를 기준으로 리소스를 구분하여 조회할 수도 있습니다.

주요 조회 대상은 다음과 같습니다.

| 리소스 분류 | 주요 리소스 |
|---|---|
| Cluster | Node, Namespace, PersistentVolume |
| Workloads | Pod, Deployment, ReplicaSet, StatefulSet, DaemonSet, Job, CronJob |
| Services | Service, Ingress |
| Storage | PersistentVolumeClaim |
| Configuration | ConfigMap, Secret |
| Autoscaling | HorizontalPodAutoscaler |

이를 통해 여러 Kubernetes 리소스의 상태를 개별적으로 조회하지 않고 웹 인터페이스에서 통합적으로 확인할 수 있습니다.

### 2.2. Kubernetes 리소스 관리

Dashboard는 리소스 조회뿐 아니라 Kubernetes 리소스를 생성하거나 수정하는 기능도 제공합니다.

대표적인 작업은 다음과 같습니다.

- Deployment 생성 및 삭제
- Pod 및 Workload 상태 확인
- Deployment 복제본 수 조정
- Rolling Update 실행
- YAML Manifest 업로드를 통한 리소스 배포
- ConfigMap 및 Secret 수정
- Namespace 및 Node 정보 조회

Dashboard에서 수행하는 작업은 Kubernetes API를 통해 처리됩니다.

따라서 실제 리소스 변경은 Kubernetes API Server를 통해 이루어지며, Dashboard 자체가 별도의 리소스 관리 시스템을 운영하는 것은 아닙니다.

### 2.3. 애플리케이션 트러블슈팅

Dashboard는 애플리케이션의 실행 상태와 관련 정보를 확인할 수 있는 기능을 제공합니다.

대표적인 기능은 다음과 같습니다.

- Pod의 현재 상태 및 이벤트 조회
- Container 로그 확인
- Pod 및 Workload의 상태 확인
- 리소스 사용량 조회
- Service와 연결된 Pod 및 Endpoint 정보 확인

예를 들어 Pod가 `CrashLoopBackOff` 상태에 있는 경우 Dashboard에서 해당 Pod의 상태와 이벤트를 확인하고 Container 로그를 조회하여 장애 원인을 분석할 수 있습니다.

## 3. 아키텍처 및 동작 방식

### 3.1. 전체 구성

Kubernetes Dashboard는 웹 UI와 Kubernetes API를 연결하는 구조로 동작합니다.

사용자는 웹 브라우저를 통해 Dashboard에 접근하며, Dashboard는 인증된 요청을 기반으로 Kubernetes API Server와 통신합니다.

일반적인 구성 요소는 다음과 같습니다.

| 구성 요소 | 역할 |
|---|---|
| Web Browser | Dashboard UI에 접근하는 클라이언트입니다. |
| Dashboard UI | Kubernetes 리소스의 조회 및 관리를 위한 웹 인터페이스입니다. |
| Dashboard Backend | Kubernetes API Server와 통신하여 리소스 정보를 조회하고 관리 요청을 처리합니다. |
| Kubernetes API Server | Dashboard의 Kubernetes 리소스 조회 및 변경 요청을 처리합니다. |
| Authentication | 사용자 인증 정보를 검증합니다. |
| RBAC | 인증된 사용자가 수행할 수 있는 Kubernetes API 작업을 제어합니다. |
| Metrics Server | CPU 및 메모리 사용량을 제공하며, Dashboard의 관련 기능에서 활용할 수 있습니다. |

전체적인 요청 흐름은 다음과 같습니다.

```text
+---------------------------+
|       Web Browser         |
|                           |
|      Dashboard UI         |
+-------------+-------------+
              |
              | HTTPS
              v
+---------------------------+
|    Kubernetes Dashboard   |
|                           |
|      Authentication       |
|                           |
|      Backend API          |
+-------------+-------------+
              |
              | Kubernetes API Request
              v
+---------------------------+
|       kube-apiserver      |
|                           |
|   Authentication         |
|   Authorization (RBAC)    |
|   Admission Control       |
+-------------+-------------+
              |
              |
       +------+------+
       |             |
       v             v
+------------+  +------------+
| Workloads  |  | Cluster    |
|            |  | Resources  |
| Pod        |  | Node       |
| Deployment |  | Service    |
| StatefulSet|  | ConfigMap  |
+------------+  +------------+
```

Dashboard는 Kubernetes API를 사용하는 클라이언트 역할을 수행합니다. 따라서 Kubernetes 리소스의 최종 상태와 변경 사항은 Kubernetes API 및 각 리소스를 관리하는 Controller에 의해 결정됩니다.

### 3.2. 리소스 조회 과정

사용자가 Dashboard에서 특정 Namespace의 Pod 목록을 조회하는 경우 다음과 같은 과정으로 처리됩니다.

1. 사용자가 웹 브라우저에서 Dashboard에 접근합니다.
2. Dashboard는 사용자의 인증 정보를 확인합니다.
3. Dashboard Backend는 인증 정보를 포함하여 Kubernetes API Server에 요청을 전달합니다.
4. kube-apiserver는 요청의 인증 정보를 검증합니다.
5. 인증된 사용자 또는 ServiceAccount의 권한을 RBAC 등을 통해 확인합니다.
6. 권한이 확인되면 kube-apiserver는 해당 리소스 정보를 반환합니다.
7. Dashboard는 응답받은 정보를 웹 UI에 표시합니다.

```text
User
 |
 | 1. Access Dashboard
 v
Dashboard UI
 |
 | 2. Authenticated Request
 v
Dashboard Backend
 |
 | 3. Kubernetes API Request
 v
kube-apiserver
 |
 | 4. Authentication
 | 5. Authorization (RBAC)
 v
Kubernetes Resources
 |
 | 6. API Response
 v
Dashboard Backend
 |
 | 7. Render Resource Information
 v
Dashboard UI
 |
 v
User
```

### 3.3. 리소스 변경 과정

Dashboard에서 Deployment의 복제본 수를 변경하는 경우에도 Kubernetes API를 통해 요청이 처리됩니다.

예를 들어 Deployment의 `spec.replicas` 값을 3에서 5로 변경하면 다음과 같은 과정으로 동작합니다.

1. 사용자가 Dashboard에서 Deployment의 복제본 수를 변경합니다.
2. Dashboard Backend는 Deployment 변경 요청을 kube-apiserver에 전달합니다.
3. kube-apiserver는 인증 및 인가 절차를 수행합니다.
4. 권한이 확인되면 Deployment 리소스가 변경됩니다.
5. kube-controller-manager의 Deployment Controller는 변경된 상태를 감지합니다.
6. ReplicaSet의 복제본 수가 조정되고 필요한 Pod가 생성됩니다.
7. Dashboard는 Kubernetes API를 통해 변경된 상태를 다시 조회하여 표시합니다.

이처럼 Dashboard는 Kubernetes의 선언적 관리 구조를 그대로 활용합니다. Dashboard에서 리소스를 변경하더라도 실제 리소스 상태를 조정하는 역할은 Kubernetes Controller가 담당합니다.

## 4. 인증 및 권한 관리

Kubernetes Dashboard는 클러스터의 리소스를 조회하거나 변경할 수 있으므로 인증과 권한 관리가 중요합니다.

### 4.1. Authentication

Authentication은 Dashboard에 접근하는 사용자가 누구인지 확인하는 과정입니다.

기존 Kubernetes Dashboard는 Bearer Token을 사용한 로그인을 지원합니다.

Bearer Token은 Kubernetes API에 접근할 수 있는 인증 정보로, 일반적으로 ServiceAccount Token 등을 사용할 수 있습니다.

Dashboard에 로그인할 때 전달된 인증 정보는 Kubernetes API 요청에 사용되며, kube-apiserver가 해당 인증 정보를 검증합니다.

인증(Authentication)과 인가(Authorization)는 서로 다른 과정입니다.

- Authentication: 요청을 보낸 사용자가 누구인지 확인합니다.
- Authorization: 해당 사용자가 요청한 작업을 수행할 권한이 있는지 확인합니다.

### 4.2. Authorization 및 RBAC

Dashboard는 Kubernetes API Server를 통해 리소스에 접근하므로 Kubernetes의 RBAC 권한 체계를 따릅니다.

RBAC는 다음과 같은 리소스를 통해 권한을 관리합니다.

| 리소스 | 역할 |
|---|---|
| Role | 특정 Namespace 범위의 권한을 정의합니다. |
| RoleBinding | 특정 Namespace에서 Role을 사용자 또는 ServiceAccount에 연결합니다. |
| ClusterRole | 클러스터 전체 범위에서 사용할 수 있는 권한을 정의합니다. |
| ClusterRoleBinding | ClusterRole을 클러스터 범위에서 사용자 또는 ServiceAccount에 연결합니다. |

예를 들어 Dashboard에 로그인한 사용자가 특정 Namespace의 Pod를 조회할 권한만 가지고 있다면 해당 Namespace의 Pod 목록만 조회할 수 있습니다.

반대로 Deployment를 수정하거나 Secret을 조회할 권한이 없다면 Dashboard에서도 해당 작업을 수행할 수 없습니다.

따라서 Dashboard의 접근 권한은 로그인 자체가 아니라 Kubernetes API Server의 인가 정책에 의해 결정됩니다.

### 4.3. 최소 권한 원칙

Dashboard에 접근하기 위한 ServiceAccount에 불필요하게 높은 권한을 부여해서는 안 됩니다.

특히 `cluster-admin` ClusterRole을 부여하면 해당 ServiceAccount를 사용하는 주체가 클러스터 전체에 대해 사실상 관리자 수준의 작업을 수행할 수 있습니다.

실제 운영 환경에서는 다음과 같은 보안 원칙을 적용하는 것이 좋습니다.

- 사용자별로 별도의 인증 정보를 사용합니다.
- 필요한 Namespace와 리소스에 대해서만 권한을 부여합니다.
- 조회 권한과 변경 권한을 구분합니다.
- 불필요한 ClusterRoleBinding을 생성하지 않습니다.
- ServiceAccount Token을 안전하게 관리합니다.
- Dashboard를 인증 및 접근 제어 없이 외부에 노출하지 않습니다.

## 5. 설치 및 접근 방식

Kubernetes Dashboard는 Kubernetes 클러스터에 기본적으로 설치되지 않으며, 별도로 배포해야 합니다.

기존 공식 설치 방식은 Helm을 사용하는 방식입니다.

다만 현재 Kubernetes Dashboard 프로젝트는 Archived 상태이므로, 아래 설치 방식은 기존 환경을 이해하기 위한 참고 사항으로만 활용해야 합니다.

### 5.1. Helm을 통한 설치

기존 Kubernetes Dashboard는 다음과 같은 Helm 명령을 통해 설치할 수 있었습니다.

```bash
helm repo add kubernetes-dashboard \
  https://kubernetes.github.io/dashboard/

helm upgrade --install kubernetes-dashboard \
  kubernetes-dashboard/kubernetes-dashboard \
  --create-namespace \
  --namespace kubernetes-dashboard
```

설치가 완료되면 다음 명령으로 배포 상태를 확인할 수 있습니다.

```bash
kubectl get pods -n kubernetes-dashboard
```

### 5.2. Port Forward를 통한 접근

Dashboard를 외부에 직접 노출하지 않고 로컬 환경에서 접근하려면 kubectl의 Port Forward 기능을 사용할 수 있습니다.

```bash
kubectl -n kubernetes-dashboard port-forward \
  svc/kubernetes-dashboard-kong-proxy 8443:443
```

명령이 실행되면 다음 주소를 통해 Dashboard에 접근할 수 있습니다.

```text
https://localhost:8443
```

Port Forward는 로컬 시스템과 Kubernetes Service 사이에 임시 연결을 생성하는 방식입니다.

따라서 별도의 외부 LoadBalancer나 Ingress를 구성하지 않고도 Dashboard에 접근할 수 있습니다.

## 6. Metrics Server와의 관계

Kubernetes Dashboard는 클러스터 및 워크로드의 CPU와 메모리 사용량을 표시하는 기능을 제공할 수 있습니다.

이러한 리소스 사용량 정보는 Metrics Server가 제공하는 Metrics API를 통해 조회할 수 있습니다.

전체적인 데이터 흐름은 다음과 같습니다.

```text
+---------------------------+
|      Kubernetes Dashboard |
+-------------+-------------+
              |
              | Metrics API Request
              v
+---------------------------+
|       kube-apiserver      |
|    (Aggregation Layer)    |
+-------------+-------------+
              |
              v
+---------------------------+
|       Metrics Server      |
+-------------+-------------+
              ^
              |
              | Resource Metrics
              |
+-------------+-------------+
|          kubelet          |
+---------------------------+
```

Metrics Server는 CPU 및 메모리 사용량을 수집하여 Metrics API로 제공하고, Dashboard는 해당 API를 통해 메트릭을 조회하여 UI에 표시합니다.

따라서 Dashboard를 설치했다고 해서 리소스 사용량을 자동으로 수집할 수 있는 것은 아닙니다. 관련 정보를 조회하려면 Metrics Server와 Metrics API가 정상적으로 구성되어 있어야 합니다.

## 7. 주요 특징 및 제한 사항

### 7.1. 웹 기반 클러스터 관리

별도의 명령어를 직접 입력하지 않고도 웹 브라우저에서 Kubernetes 리소스를 조회하고 관리할 수 있습니다.

Kubernetes 리소스의 상태와 관계를 시각적으로 확인할 수 있어 클러스터 운영 및 학습 환경에서 유용하게 활용할 수 있습니다.

### 7.2. Kubernetes API 기반 동작

Dashboard는 자체적인 클러스터 상태 관리 시스템을 운영하지 않습니다.

Kubernetes API Server를 통해 리소스를 조회하고 변경하며, 실제 리소스의 상태와 동작은 Kubernetes의 Control Plane 및 관련 Controller에 의해 관리됩니다.

### 7.3. 권한에 따른 기능 제한

Dashboard에서 수행할 수 있는 작업은 인증된 사용자의 Kubernetes RBAC 권한에 따라 결정됩니다.

따라서 Dashboard에 로그인할 수 있다고 해서 모든 Kubernetes 리소스에 접근할 수 있는 것은 아닙니다.

### 7.4. 프로젝트 유지보수 종료

Kubernetes Dashboard 프로젝트는 Deprecated 및 Archived 상태이며, 더 이상 적극적으로 유지보수되지 않습니다.

따라서 신규 클러스터에 Kubernetes Dashboard를 도입하는 것은 권장되지 않습니다.

기존 Dashboard 환경을 운영하는 경우에는 보안 및 유지보수 측면에서 대체 도구로의 전환을 검토해야 합니다.

## 8. 대안: Headlamp

Headlamp는 Kubernetes 클러스터의 리소스를 조회하고 관리할 수 있는 웹 기반 Kubernetes UI입니다.

Kubernetes Dashboard의 대안으로 공식 Kubernetes 문서에서도 Headlamp를 안내하고 있습니다.

주요 특징은 다음과 같습니다.

- Desktop 및 In-cluster 방식 모두 지원합니다.
- kubeconfig를 통한 클러스터 인증을 지원합니다.
- 여러 Kubernetes 클러스터를 통합하여 관리할 수 있습니다.
- Kubernetes 리소스의 상태 및 YAML 정보를 조회할 수 있습니다.
- 플러그인을 통한 기능 확장이 가능합니다.
- Kubernetes RBAC 권한 체계를 활용합니다.

Headlamp는 Dashboard와 유사한 클러스터 관리 기능을 제공하지만, 실행 위치와 인증 방식 등 아키텍처 측면에서 차이가 있습니다.

| 구분 | Kubernetes Dashboard | Headlamp |
|---|---|---|
| 프로젝트 상태 | Archived / Deprecated | 유지보수 중 |
| 실행 방식 | In-cluster | Desktop / In-cluster |
| 인증 방식 | Bearer Token 중심 | kubeconfig 및 지원되는 인증 방식 |
| 멀티 클러스터 | 기본적으로 제한적 | 지원 |
| UI 확장 | 제한적 | Plugin 기반 확장 |
| 주요 목적 | 클러스터 리소스 조회 및 관리 | 클러스터 리소스 조회 및 관리 |

Headlamp 공식 문서:

- https://headlamp.dev/
- https://headlamp.dev/docs/latest/

## 9. 참고자료

- [Kubernetes Documentation - Deploy and Access the Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)
- [Kubernetes Documentation - RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes Documentation - Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [Kubernetes Documentation - API Aggregation Layer](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)
- [Kubernetes Documentation - Resource Metrics Pipeline](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)
- [Kubernetes Blog - Kubernetes Dashboard to Headlamp: A Step-by-Step Guide](https://kubernetes.io/blog/2026/07/13/kubernetes-dashboard-to-headlamp/)
- [Headlamp Official Website](https://headlamp.dev/)
- [Headlamp GitHub Repository](https://github.com/kubernetes-sigs/headlamp)