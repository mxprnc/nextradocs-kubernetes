# IngressClass

## 1. IngressClass란?

IngressClass는 Kubernetes에서 Ingress 리소스를 처리할 Ingress Controller의 종류와 관련 설정을 정의하는 리소스입니다.

Kubernetes 클러스터에는 여러 종류의 Ingress Controller가 설치될 수 있습니다. 예를 들어 NGINX 기반 Controller와 Traefik 기반 Controller를 동시에 운영하거나, 내부 서비스와 외부 서비스를 서로 다른 Controller로 처리할 수 있습니다.

이러한 환경에서는 각 Ingress 리소스를 어떤 Ingress Controller가 처리해야 하는지 구분할 필요가 있습니다.

IngressClass는 이러한 연결 관계를 정의합니다.

Ingress 리소스의 `spec.ingressClassName` 필드에 IngressClass의 이름을 지정하면 해당 IngressClass를 참조하게 됩니다.

예를 들어 다음과 같이 설정할 수 있습니다.

```yaml
spec:
  ingressClassName: nginx
```

위 설정은 `nginx`라는 이름의 IngressClass를 사용하도록 지정합니다.

IngressClass는 `networking.k8s.io/v1` API 그룹에 속하며, 일반적인 Kubernetes 리소스와 달리 클러스터 범위(Cluster-scoped)로 관리됩니다.

![Kubernetes Ingress와 IngressClass의 관계](https://images.openai.com/static-rsc-4/dz_r2ZWf3Kzv4qI8rR15NJJZmjyMp7t6b_6GxERNAO3nbfzd3iK2GXnif4dMnWC4G5Dv1qBLrmuezytNObJrpmdsTo4d86hg_GL4C_Fgz0dEIOOZty67mQlj29bT7FLw897-27tpDPC9eCwdRFHt-aD0XUEj3bv1hAsch9D5GNM?purpose=inline)

## 2. IngressClass의 주요 역할

IngressClass는 다음과 같은 역할을 수행합니다.

- **Controller 식별:** Ingress를 처리할 Ingress Controller의 종류를 지정합니다.
- **Controller 선택:** 여러 Ingress Controller가 설치된 환경에서 Ingress 처리 주체를 구분합니다.
- **Controller 설정 연결:** Ingress Controller가 사용할 추가 설정 리소스를 참조할 수 있습니다.
- **정책 분리:** 내부 및 외부 트래픽 등 서로 다른 네트워크 환경에 맞는 Ingress 처리를 구성할 수 있습니다.
- **기본 IngressClass 지정:** 별도의 Class를 지정하지 않은 Ingress에 적용할 기본 IngressClass를 설정할 수 있습니다.

## 3. IngressClass의 동작 구조

IngressClass는 Ingress와 Ingress Controller 사이의 연결 관계를 정의합니다.

일반적인 동작 과정은 다음과 같습니다.

1. IngressClass 리소스를 생성하고 사용할 Controller를 지정합니다.
2. Ingress 리소스의 `spec.ingressClassName`에 IngressClass 이름을 지정합니다.
3. Ingress Controller가 Kubernetes API Server를 감시합니다.
4. Controller는 자신이 처리하도록 설정된 IngressClass를 확인합니다.
5. 해당 IngressClass를 참조하는 Ingress를 대상으로 실제 라우팅 설정을 구성합니다.
6. 외부 요청이 들어오면 Ingress Controller가 정의된 규칙에 따라 Backend Service로 트래픽을 전달합니다.

예를 들어 다음과 같이 구성할 수 있습니다.

| IngressClass | Controller | 처리 대상 |
|---|---|---|
| `nginx` | NGINX Ingress Controller | 일반 웹 서비스 |
| `traefik` | Traefik Controller | 별도 웹 서비스 |
| `internal-nginx` | NGINX Ingress Controller | 내부 전용 서비스 |

IngressClass를 사용하면 여러 Controller가 하나의 클러스터에서 서로 다른 Ingress 리소스를 독립적으로 처리할 수 있습니다.

참고: IngressClass 자체가 Ingress Controller를 설치하거나 실행하는 것은 아닙니다. 실제 Controller는 별도로 배포되어 실행되어야 하며, IngressClass의 `spec.controller` 값은 해당 Controller 구현체를 식별하기 위한 값입니다.

## 4. IngressClass 리소스 구조

IngressClass는 다음과 같은 주요 필드로 구성됩니다.

| 필드 | 설명 |
|---|---|
| `metadata.name` | IngressClass의 이름입니다. |
| `metadata.annotations` | 기본 IngressClass 여부 등의 추가 메타데이터를 지정합니다. |
| `spec.controller` | IngressClass를 처리하는 Controller 구현체를 식별합니다. |
| `spec.parameters` | Controller가 참조할 추가 설정 리소스를 지정합니다. |

### 4.1 metadata.name

IngressClass의 이름을 지정합니다.

Ingress 리소스는 이 이름을 `spec.ingressClassName`에서 참조합니다.

```yaml
metadata:
  name: nginx
```

다음과 같이 Ingress에서 참조할 수 있습니다.

```yaml
spec:
  ingressClassName: nginx
```

IngressClass는 Cluster-scoped 리소스이므로 Namespace를 지정하지 않습니다.

### 4.2 spec.controller

`spec.controller`는 해당 IngressClass를 처리할 Controller 구현체를 식별하는 문자열입니다.

```yaml
spec:
  controller: k8s.io/ingress-nginx
```

대표적인 값으로는 다음과 같은 형태가 있습니다.

| Controller 구현체 | Controller 식별자 예시 |
|---|---|
| Community ingress-nginx | `k8s.io/ingress-nginx` |
| Traefik | `traefik.io/ingress-controller` |

이 값은 Kubernetes가 직접 Controller를 찾아 실행하는 데 사용하는 것이 아니라, 각 Ingress Controller가 자신이 처리할 IngressClass를 식별하는 데 사용됩니다.

따라서 실제 Controller 구현체가 기대하는 식별자와 일치해야 합니다.

### 4.3 spec.parameters

`spec.parameters`는 IngressClass와 연결할 추가 설정 리소스를 지정하는 필드입니다.

Controller 구현체가 제공하는 설정 리소스를 참조할 수 있으며, 이를 통해 여러 Ingress에서 공통으로 사용할 설정을 관리할 수 있습니다.

```yaml
spec:
  controller: k8s.io/ingress-nginx
  parameters:
    apiGroup: example.com
    kind: IngressParameters
    name: nginx-config
    scope: Cluster
```

주요 필드는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `apiGroup` | 참조할 리소스의 API 그룹입니다. |
| `kind` | 참조할 리소스의 종류입니다. |
| `name` | 참조할 리소스의 이름입니다. |
| `scope` | 참조 리소스의 범위를 지정합니다. |
| `namespace` | Namespace 범위 리소스를 참조할 때 사용할 Namespace입니다. |

`scope`는 `Cluster` 또는 `Namespace`를 사용할 수 있습니다.

- `Cluster`: 클러스터 범위의 설정 리소스를 참조합니다.
- `Namespace`: 특정 Namespace에 속한 설정 리소스를 참조합니다.

`scope`를 생략하면 기본적으로 `Cluster` 범위로 해석됩니다.

`scope: Namespace`를 사용하는 경우 `namespace`를 지정해야 합니다.

참고: `parameters`는 선택적 필드이며 모든 Ingress Controller가 이를 지원하는 것은 아닙니다. 실제 지원 여부와 참조할 리소스의 종류는 Controller 구현체에 따라 달라집니다.

## 5. IngressClass 예제

### 5.1 기본 IngressClass 생성

다음은 NGINX Ingress Controller를 사용하기 위한 IngressClass 예제입니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
```

각 필드의 의미는 다음과 같습니다.

- `metadata.name`: Ingress에서 참조할 IngressClass의 이름입니다.
- `spec.controller`: 해당 IngressClass를 처리할 Controller의 식별자입니다.

위 리소스를 생성하면 `nginx`라는 IngressClass를 사용할 수 있습니다.

```bash
kubectl apply -f ingress-class.yaml
```

생성한 IngressClass는 다음과 같이 Ingress에서 참조합니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80
```

이 경우 `nginx` IngressClass를 처리하도록 구성된 Ingress Controller가 해당 Ingress를 관리합니다.

### 5.2 기본 IngressClass 지정

클러스터에서 가장 일반적으로 사용할 IngressClass를 기본값으로 지정할 수 있습니다.

이를 위해 다음 Annotation을 사용합니다.

```yaml
ingressclass.kubernetes.io/is-default-class: "true"
```

예제는 다음과 같습니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx
```

기본 IngressClass가 지정되면 `spec.ingressClassName`을 생략한 신규 Ingress 리소스에 해당 Class가 자동으로 할당됩니다.

예를 들어 다음 Ingress는 `ingressClassName`을 명시하지 않았습니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: default-ingress
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

클러스터에 기본 IngressClass가 하나만 지정되어 있다면 Kubernetes Admission Controller가 신규 Ingress 생성 과정에서 기본 Class를 할당합니다.

주의사항:

- 기본 IngressClass는 클러스터에 최대 하나만 지정하는 것이 바람직합니다.
- 기본 IngressClass가 두 개 이상 지정되어 있으면 Class를 명시하지 않은 신규 Ingress 생성이 거부됩니다.
- 기본 IngressClass 설정은 기존 Ingress 리소스에 소급 적용되지 않습니다.
- 운영 환경에서는 기본값에 의존하기보다 `ingressClassName`을 명시하는 것이 명확합니다.

### 5.3 Parameters를 사용하는 IngressClass

IngressClass는 Controller가 제공하는 추가 설정 리소스를 참조할 수 있습니다.

다음은 Cluster 범위의 설정 리소스를 참조하는 예제입니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx-custom
spec:
  controller: k8s.io/ingress-nginx
  parameters:
    apiGroup: example.com
    kind: IngressParameters
    name: nginx-config
    scope: Cluster
```

위 예제에서는 `example.com` API 그룹의 `IngressParameters` 리소스 중 `nginx-config`라는 리소스를 참조합니다.

Namespace 범위의 설정 리소스를 참조하려면 다음과 같이 구성할 수 있습니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx-custom
spec:
  controller: k8s.io/ingress-nginx
  parameters:
    apiGroup: example.com
    kind: IngressParameters
    name: nginx-config
    scope: Namespace
    namespace: ingress-config
```

이 경우 `ingress-config` Namespace에 존재하는 `nginx-config` 설정 리소스를 참조합니다.

단, 위 예제의 `IngressParameters`는 설명을 위한 가상의 Custom Resource입니다. 실제로 사용하려면 해당 Controller가 제공하는 CRD와 설정 방식을 확인해야 합니다.

## 6. 여러 Ingress Controller 운영

IngressClass를 사용하면 하나의 Kubernetes 클러스터에서 여러 Ingress Controller를 독립적으로 운영할 수 있습니다.

예를 들어 다음과 같은 구성을 생각할 수 있습니다.

| IngressClass | Controller | 용도 |
|---|---|---|
| `nginx-external` | NGINX Ingress Controller | 외부 사용자 요청 |
| `nginx-internal` | NGINX Ingress Controller | 내부 서비스 요청 |
| `traefik` | Traefik Controller | 별도 애플리케이션 요청 |

이러한 구조에서는 각 IngressClass가 서로 다른 Controller를 지정하거나, 동일한 Controller 구현체의 서로 다른 인스턴스를 구분할 수 있습니다.

### 6.1 서로 다른 Controller 사용

NGINX와 Traefik을 함께 사용하는 예제입니다.

![여러 Ingress Controller와 IngressClass 구성 예시](https://images.openai.com/static-rsc-4/dZz4flxzuoCakojm16AHgzsAoagF2p_3uLI5fm4_bJIx9PCEJQck3vupcDQwQBrx2TyAyZqDKWZ6UasMY_I1LYgC2sEREtcfd4nnAvEd6TfdhRKMOwxF8Dn1vmpPzQm2r2DjEw5oCplL7PqqMZ770o19rlrQo2jwScdU406itJY?purpose=inline)

먼저 NGINX용 IngressClass를 생성합니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
```

다음으로 Traefik용 IngressClass를 생성합니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: traefik
spec:
  controller: traefik.io/ingress-controller
```

각 Ingress 리소스에서 사용할 IngressClass를 지정합니다.

NGINX를 사용하는 Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: nginx.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-backend
                port:
                  number: 80
```

Traefik을 사용하는 Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: traefik-ingress
spec:
  ingressClassName: traefik
  rules:
    - host: traefik.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: traefik-backend
                port:
                  number: 80
```

각 Controller는 자신에게 지정된 IngressClass를 참조하는 Ingress를 처리합니다.

### 6.2 동일한 Controller의 여러 인스턴스 사용

동일한 NGINX Ingress Controller를 여러 개 실행하면서 각각 다른 IngressClass를 사용하도록 구성할 수도 있습니다.

예를 들어 다음과 같이 구분할 수 있습니다.

- **External Controller:** 인터넷에서 접근하는 외부 서비스 담당
- **Internal Controller:** 사내 네트워크에서 접근하는 내부 서비스 담당

이 경우 두 Controller 인스턴스가 서로 다른 Controller 식별자를 사용하도록 구성해야 합니다.

예를 들어 다음과 같이 설정할 수 있습니다.

| 구분 | External | Internal |
|---|---|---|
| IngressClass 이름 | `nginx-external` | `nginx-internal` |
| Controller 식별자 | `k8s.io/external-ingress-nginx` | `k8s.io/internal-ingress-nginx` |
| 담당 트래픽 | 외부 트래픽 | 내부 트래픽 |

각 IngressClass의 설정은 다음과 같습니다.

External IngressClass:

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx-external
spec:
  controller: k8s.io/external-ingress-nginx
```

Internal IngressClass:

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx-internal
spec:
  controller: k8s.io/internal-ingress-nginx
```

각 NGINX Controller 인스턴스도 자신의 IngressClass에 대응하는 Controller 식별자를 사용하도록 설정해야 합니다.

참고: IngressClass를 두 개 생성하는 것만으로 Controller 인스턴스가 자동으로 분리되는 것은 아닙니다. 각 Controller의 실행 옵션, IngressClass 감시 설정, Leader Election ID 및 외부 노출 방식 등을 함께 구성해야 합니다.

## 7. IngressClass와 기존 Ingress Annotation의 차이

Kubernetes 1.18 이전에는 Ingress Controller를 지정하기 위해 다음 Annotation이 널리 사용되었습니다.

```yaml
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
```

이 방식은 Ingress 리소스에 Controller 이름을 직접 지정하는 구조였습니다.

Kubernetes 1.18부터는 IngressClass 리소스와 `spec.ingressClassName` 필드가 도입되어 기존 Annotation을 대체하는 표준 방식으로 사용됩니다.

### 7.1 기존 방식

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: api.example.com
```

### 7.2 현재 권장 방식

먼저 IngressClass를 생성합니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
```

이후 Ingress에서 IngressClass를 참조합니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
```

두 방식의 차이는 다음과 같습니다.

| 구분 | 기존 Annotation | IngressClass |
|---|---|---|
| 설정 위치 | Ingress Metadata | 별도 IngressClass 리소스 |
| Controller 지정 | Annotation 값 | IngressClass의 `spec.controller` |
| 추가 설정 | Controller별 Annotation | `spec.parameters` 등 활용 가능 |
| 표준화 | 공식 API 필드가 아닌 관행적인 방식 | Kubernetes 표준 API |
| 상태 | Deprecated | 권장 방식 |

기존 Annotation인 `kubernetes.io/ingress.class`는 Deprecated 상태이므로 신규 구성에서는 `spec.ingressClassName`을 사용하는 것이 바람직합니다.

또한 `ingressClassName`은 기존 Annotation과 완전히 동일한 필드가 아닙니다. 기존 방식이 Controller를 직접 가리켰다면, 새로운 방식은 IngressClass 리소스를 참조하고 해당 리소스가 Controller를 지정하는 구조입니다.

## 8. IngressClass 확인 및 관리 명령어

IngressClass를 조회하고 관리할 때 사용하는 대표적인 kubectl 명령어는 다음과 같습니다.

### IngressClass 목록 조회

```bash
kubectl get ingressclass
```

또는 다음과 같이 축약할 수 있습니다.

```bash
kubectl get ingressclasses
```

### IngressClass 상세 조회

```bash
kubectl describe ingressclass nginx
```

### IngressClass YAML 조회

```bash
kubectl get ingressclass nginx -o yaml
```

### 모든 IngressClass 조회

IngressClass는 Cluster-scoped 리소스이므로 Namespace 옵션 없이 조회합니다.

```bash
kubectl get ingressclass -o wide
```

### IngressClass가 연결된 Ingress 확인

```bash
kubectl get ingress -A
```

특정 IngressClass를 사용하는 Ingress만 조회하려면 다음과 같이 실행할 수 있습니다.

```bash
kubectl get ingress -A \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,CLASS:.spec.ingressClassName'
```

### IngressClass 생성

```bash
kubectl apply -f ingress-class.yaml
```

### IngressClass 삭제

```bash
kubectl delete ingressclass nginx
```

IngressClass를 삭제하더라도 해당 Class를 참조하던 Ingress 리소스가 자동으로 삭제되지는 않습니다. 따라서 IngressClass를 삭제하기 전에 해당 Class를 사용하는 Ingress가 있는지 확인해야 합니다.

## 9. IngressClass 사용 시 주의사항

IngressClass를 운영 환경에서 사용할 때는 다음 사항을 고려해야 합니다.

1. **Controller 설치 여부:** IngressClass를 생성하는 것만으로 Ingress Controller가 설치되거나 실행되지는 않습니다.
2. **Controller 식별자 일치:** `spec.controller` 값은 실제 Ingress Controller가 처리하도록 설정된 식별자와 일치해야 합니다.
3. **기본 IngressClass 중복 방지:** 기본 IngressClass는 클러스터에 하나만 지정해야 합니다.
4. **명시적인 Class 지정:** 운영 환경에서는 Ingress의 `spec.ingressClassName`을 명시하는 것이 관리와 추적에 유리합니다.
5. **Controller 설정 확인:** 동일한 Controller 구현체를 여러 인스턴스로 실행할 때는 각 인스턴스의 감시 범위와 Controller 식별자를 구분해야 합니다.
6. **Parameters 지원 여부:** `spec.parameters`는 Controller 구현체에 따라 지원 여부와 참조 가능한 리소스가 다릅니다.
7. **기존 Annotation 마이그레이션:** 기존 `kubernetes.io/ingress.class` Annotation을 사용하는 환경은 `spec.ingressClassName` 기반으로 전환하는 것이 바람직합니다.

## 10. 정리

IngressClass는 Kubernetes에서 Ingress 리소스를 처리할 Ingress Controller를 지정하는 리소스입니다.

하나의 클러스터에서 여러 Ingress Controller를 운영하거나, 동일한 Controller를 서로 다른 용도로 분리하여 운영할 때 사용합니다.

IngressClass의 주요 특징은 다음과 같습니다.

- Ingress와 Ingress Controller 사이의 연결 관계를 정의합니다.
- `spec.controller`를 통해 Controller 구현체를 식별합니다.
- Ingress의 `spec.ingressClassName`에서 참조합니다.
- `spec.parameters`를 통해 Controller의 추가 설정 리소스를 연결할 수 있습니다.
- `ingressclass.kubernetes.io/is-default-class` Annotation으로 기본 IngressClass를 지정할 수 있습니다.
- Cluster-scoped 리소스로 관리됩니다.
- 여러 Ingress Controller가 하나의 클러스터에서 독립적으로 동작하도록 구성할 수 있습니다.

Ingress와 IngressClass의 관계는 다음과 같습니다.

| 리소스 | 역할 |
|---|---|
| Ingress | 외부 HTTP/HTTPS 트래픽의 라우팅 규칙을 정의합니다. |
| IngressClass | Ingress를 처리할 Controller의 종류와 설정을 정의합니다. |
| Ingress Controller | Ingress와 IngressClass를 감시하고 실제 트래픽 처리를 수행합니다. |

Ingress가 외부 트래픽을 어디로 전달할지 정의한다면, IngressClass는 해당 규칙을 어떤 Controller가 처리할지 결정하는 역할을 수행합니다.

## 공식 문서 및 참고 자료

- [Kubernetes 공식 문서 - Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kubernetes API Reference - IngressClass v1](https://kubernetes.io/docs/reference/kubernetes-api/networking/ingress-class-v1/)
- [Kubernetes 공식 문서 - Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
- [What is Ingress Class Name in Kubernetes? (Explained With Examples)](https://blog.techiescamp.com/ingress-class-name/)
- [Kubernetes Ingress: Controllers, Routing, And Security](https://www.wiz.io/academy/container-security/kubernetes-ingress)

