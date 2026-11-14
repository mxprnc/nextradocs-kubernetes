# Secret

## 1. Secret이란?

Secret은 Kubernetes에서 **비밀번호, 인증 토큰, SSH 개인 키, TLS 인증서 등 민감한 정보를 저장하고 관리하기 위한 API 리소스**입니다.

애플리케이션의 민감한 정보를 컨테이너 이미지나 애플리케이션 코드에 직접 포함하면 정보가 노출될 위험이 있습니다. Secret을 사용하면 이러한 정보를 애플리케이션과 분리하여 관리하고, Pod에서 필요한 시점에 환경 변수나 Volume 등의 방식으로 전달할 수 있습니다.

예를 들어 데이터베이스 비밀번호를 컨테이너 이미지에 포함하는 대신 Secret에 저장하고, Pod가 해당 Secret을 참조하도록 구성할 수 있습니다.

```text
                    Kubernetes Cluster
┌───────────────────────────────────────────────────────┐
│                                                       │
│  ┌──────────────────────┐                             │
│  │        Secret        │                             │
│  │                      │                             │
│  │  DB_USERNAME         │                             │
│  │  DB_PASSWORD         │                             │
│  │  API_TOKEN           │                             │
│  └──────────┬───────────┘                             │
│             │                                         │
│             │ Sensitive Data                          │
│             ▼                                         │
│  ┌───────────────────────────────┐                    │
│  │             Pod               │                    │
│  │                               │                    │
│  │  ┌─────────────────────────┐  │                    │
│  │  │      Container          │  │                    │
│  │  │                         │  │                    │
│  │  │  Application            │  │                    │
│  │  │  - DB_USERNAME          │  │                    │
│  │  │  - DB_PASSWORD          │  │                    │
│  │  │  - API_TOKEN            │  │                    │
│  │  └─────────────────────────┘  │                    │
│  └───────────────────────────────┘                    │
│                                                       │
└───────────────────────────────────────────────────────┘
```

Secret은 민감한 데이터를 저장하기 위한 리소스이지만, Secret을 생성하는 것만으로 데이터가 자동으로 암호화되는 것은 아닙니다.

Kubernetes는 기본적으로 Secret의 `data` 필드에 저장되는 값을 Base64 인코딩 형태로 표현합니다. Base64는 암호화 방식이 아니므로 실제 보안을 위해서는 Encryption at Rest, RBAC, 최소 권한 원칙 등의 보안 설정을 함께 적용해야 합니다.

## 2. Secret의 주요 특징

### 2.1. 민감한 데이터의 분리 관리

Secret을 사용하면 애플리케이션 코드와 민감한 데이터를 분리할 수 있습니다.

예를 들어 다음과 같은 정보를 Secret으로 관리할 수 있습니다.

- 데이터베이스 사용자 이름 및 비밀번호
- 외부 API 인증 토큰
- TLS 인증서 및 개인 키
- SSH 인증 키
- 레지스트리 인증 정보
- 애플리케이션에서 사용하는 기타 인증 정보

이러한 데이터를 컨테이너 이미지에 직접 포함하지 않으면 이미지가 외부에 배포되거나 공유되더라도 민감한 정보가 함께 노출되는 위험을 줄일 수 있습니다.

### 2.2. Key-Value 기반 데이터 저장

Secret은 `data`와 `stringData` 필드를 사용하여 데이터를 저장합니다.

- `data`: Base64로 인코딩된 데이터를 저장합니다.
- `stringData`: 평문 문자열을 입력할 수 있는 편의 필드입니다.

`stringData`에 입력한 값은 Kubernetes API Server가 처리하여 `data`에 저장합니다.

Base64 인코딩은 데이터를 암호화하는 기능이 아니므로 인코딩된 값만으로 보안이 확보되는 것은 아닙니다.

### 2.3. Namespace 범위

Secret은 Namespace 범위의 리소스입니다.

Pod에서 Secret을 참조하려면 두 리소스가 동일한 Namespace에 존재해야 합니다.

서로 다른 Namespace에 존재하는 Secret을 일반적인 Pod 설정 참조 방식으로 직접 사용할 수는 없습니다.

### 2.4. 데이터 크기 제한

Secret에 저장할 수 있는 데이터의 크기는 1MiB로 제한됩니다.

대용량 파일이나 일반적인 데이터 저장소로 사용하는 것은 적절하지 않습니다.

### 2.5. 민감한 데이터에 대한 접근 제어

Secret은 Kubernetes RBAC을 통해 접근 권한을 제어할 수 있습니다.

예를 들어 특정 ServiceAccount에 특정 Namespace의 Secret을 조회할 수 있는 권한만 부여할 수 있습니다.

Secret에 대한 접근 권한은 최소한으로 제한하는 것이 중요합니다. 특히 Secret을 조회할 수 있는 권한이 있는 사용자는 해당 데이터의 실제 내용을 확인할 수 있으므로 권한 부여에 주의해야 합니다.

### 2.6. Immutable Secret

ConfigMap과 마찬가지로 Secret도 `immutable: true`를 지정하여 생성 이후 데이터 변경을 방지할 수 있습니다.

Immutable Secret은 불필요한 데이터 변경을 방지하고 대규모 클러스터에서 Secret 변경 감시에 따른 부하를 줄이는 데 도움이 됩니다.

## 3. Secret의 구조

Secret은 `apiVersion: v1`을 사용하는 Kubernetes Core API 리소스입니다.

다음은 Secret의 기본적인 YAML 구조입니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: application-secret
  namespace: default
type: Opaque
data:
  DB_USERNAME: YWRtaW4=
  DB_PASSWORD: cGFzc3dvcmQ=
```

주요 필드는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `apiVersion` | Kubernetes API 버전을 지정합니다. Secret은 `v1`을 사용합니다. |
| `kind` | 리소스 종류를 지정합니다. `Secret`을 사용합니다. |
| `metadata.name` | Secret의 이름을 지정합니다. |
| `metadata.namespace` | Secret이 생성될 Namespace를 지정합니다. |
| `type` | Secret의 데이터 유형을 지정합니다. |
| `data` | Base64로 인코딩된 Key-Value 데이터를 저장합니다. |
| `stringData` | 평문 문자열 형태로 데이터를 입력할 수 있는 편의 필드입니다. |
| `immutable` | Secret의 데이터 변경을 방지할지 여부를 지정합니다. |

### 3.1. data

`data`는 Secret의 실제 데이터를 저장하는 필드입니다.

각 Key의 Value는 Base64로 인코딩된 문자열이어야 합니다.

예를 들어 다음과 같은 평문 데이터를 저장한다고 가정합니다.

```text
DB_USERNAME=admin
DB_PASSWORD=password
```

Base64 인코딩을 적용하면 다음과 같습니다.

```bash
echo -n 'admin' | base64
echo -n 'password' | base64
```

출력 결과는 다음과 같습니다.

```text
YWRtaW4=
cGFzc3dvcmQ=
```

이를 Secret의 `data`에 저장할 수 있습니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: application-secret
type: Opaque
data:
  DB_USERNAME: YWRtaW4=
  DB_PASSWORD: cGFzc3dvcmQ=
```

Kubernetes API Server는 Base64로 인코딩된 값을 디코딩하여 Secret 데이터를 처리합니다.

Base64 인코딩은 누구나 쉽게 디코딩할 수 있으므로 암호화와 혼동해서는 안 됩니다.

### 3.2. stringData

`stringData`는 Base64 인코딩을 직접 수행하지 않고 평문 문자열을 입력할 수 있도록 제공하는 편의 필드입니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: application-secret
type: Opaque
stringData:
  DB_USERNAME: admin
  DB_PASSWORD: password
```

위 Manifest를 적용하면 Kubernetes API Server가 `stringData`의 값을 처리하여 `data`에 저장합니다.

따라서 `kubectl get secret application-secret -o yaml` 명령어로 조회하면 일반적으로 `stringData`가 아닌 Base64로 인코딩된 `data`가 표시됩니다.

`stringData`는 API에서 읽을 수 있는 출력 필드가 아니며, 주로 Secret 생성 및 수정 시 사용합니다.

또한 동일한 Key가 `data`와 `stringData`에 모두 존재하면 `stringData`의 값이 우선합니다.

`stringData`는 서버 측 적용(Server-Side Apply)과 호환되지 않는 제약이 있으므로 선언적 방식으로 Secret을 관리할 때는 주의해야 합니다.

### 3.3. type

`type`은 Secret에 저장되는 데이터의 용도를 지정합니다.

Kubernetes는 Secret의 유형에 따라 특정 데이터 구조나 필수 Key를 검증할 수 있습니다.

대표적인 유형은 다음과 같습니다.

| Type | 설명 |
|---|---|
| `Opaque` | 일반적인 사용자 정의 Secret입니다. |
| `kubernetes.io/service-account-token` | ServiceAccount 토큰을 저장하는 Secret입니다. |
| `kubernetes.io/dockercfg` | Docker Registry 인증 정보를 저장하는 Secret입니다. |
| `kubernetes.io/dockerconfigjson` | Docker Registry 인증 정보를 JSON 형식으로 저장하는 Secret입니다. |
| `kubernetes.io/basic-auth` | HTTP Basic Authentication 정보를 저장하는 Secret입니다. |
| `kubernetes.io/ssh-auth` | SSH 인증 정보를 저장하는 Secret입니다. |
| `kubernetes.io/tls` | TLS 인증서와 개인 키를 저장하는 Secret입니다. |
| `bootstrap.kubernetes.io/token` | Bootstrap Token을 저장하는 Secret입니다. |

특별한 유형을 사용하지 않는 일반적인 애플리케이션 인증 정보는 `Opaque`를 사용합니다.

### 3.4. Opaque Secret

`Opaque`는 Secret의 기본 유형으로, 사용자가 임의의 Key-Value 데이터를 저장할 수 있습니다.

예를 들어 데이터베이스 인증 정보나 외부 API 인증 토큰 등을 저장할 때 사용할 수 있습니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-secret
type: Opaque
stringData:
  DB_USERNAME: admin
  DB_PASSWORD: secure-password
```

`Opaque` Secret은 데이터의 구체적인 형식이나 용도를 Kubernetes가 제한하지 않으므로 다양한 애플리케이션 설정에 활용할 수 있습니다.

### 3.5. kubernetes.io/tls Secret

TLS Secret은 TLS 인증서와 개인 키를 저장하기 위한 유형입니다.

일반적으로 다음 두 개의 Key를 사용합니다.

- `tls.crt`: TLS 인증서
- `tls.key`: TLS 개인 키

다음은 TLS Secret의 예제입니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: application-tls
type: kubernetes.io/tls
data:
  tls.crt: <BASE64_ENCODED_CERTIFICATE>
  tls.key: <BASE64_ENCODED_PRIVATE_KEY>
```

TLS Secret은 Ingress Controller나 기타 TLS를 지원하는 Kubernetes 구성 요소에서 인증서와 개인 키를 참조할 때 활용할 수 있습니다.

### 3.6. kubernetes.io/dockerconfigjson Secret

Docker Registry에 접근하기 위한 인증 정보를 저장하는 Secret입니다.

Private Container Registry에서 이미지를 가져와야 하는 경우 사용할 수 있습니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: registry-secret
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <BASE64_ENCODED_DOCKER_CONFIG_JSON>
```

일반적으로 `kubectl create secret docker-registry` 명령어를 사용하면 해당 유형의 Secret을 생성할 수 있습니다.

Pod의 `imagePullSecrets`에 연결하면 Kubernetes가 Private Registry에 접근할 때 인증 정보를 사용할 수 있습니다.

## 4. Secret 생성 및 관리

Secret은 YAML Manifest 또는 `kubectl create secret` 명령어를 사용하여 생성할 수 있습니다.

### 4.1. YAML Manifest를 이용한 생성

다음과 같이 Secret Manifest를 작성합니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: application-secret
type: Opaque
stringData:
  DB_USERNAME: admin
  DB_PASSWORD: secure-password
  API_TOKEN: example-token
```

Manifest를 적용합니다.

```bash
kubectl apply -f secret.yaml
```

Secret이 정상적으로 생성되었는지 확인합니다.

```bash
kubectl get secrets
```

특정 Secret의 상세 정보를 확인하려면 다음 명령어를 사용합니다.

```bash
kubectl describe secret application-secret
```

YAML 형식으로 전체 리소스 정보를 확인하려면 다음 명령어를 사용합니다.

```bash
kubectl get secret application-secret -o yaml
```

단, `kubectl describe secret`은 일반적으로 데이터의 실제 내용을 출력하지 않고 Key와 데이터 크기 등의 정보만 표시합니다.

반면 `kubectl get secret -o yaml`은 Base64로 인코딩된 데이터를 출력하므로 민감한 정보가 노출되지 않도록 주의해야 합니다.

### 4.2. 명령어를 이용한 생성

`kubectl create secret generic` 명령어를 사용하면 일반적인 Opaque Secret을 생성할 수 있습니다.

**Literal 값을 이용한 생성**

```bash
kubectl create secret generic application-secret \
  --from-literal=DB_USERNAME=admin \
  --from-literal=DB_PASSWORD=secure-password
```

**파일을 이용한 생성**

```bash
kubectl create secret generic application-secret \
  --from-file=credentials.json
```

파일을 이용하면 파일 이름이 Key가 되고 파일 내용이 Value가 됩니다.

예를 들어 `credentials.json` 파일을 지정하면 다음과 같은 형태로 저장됩니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: application-secret
type: Opaque
data:
  credentials.json: <BASE64_ENCODED_FILE_CONTENT>
```

**TLS Secret 생성**

TLS 인증서와 개인 키를 이용하여 Secret을 생성할 수 있습니다.

```bash
kubectl create secret tls application-tls \
  --cert=tls.crt \
  --key=tls.key
```

이 명령어는 `kubernetes.io/tls` 유형의 Secret을 생성합니다.

**Docker Registry Secret 생성**

```bash
kubectl create secret docker-registry registry-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=my-user \
  --docker-password=my-password \
  --docker-email=my-email@example.com
```

이 명령어는 `kubernetes.io/dockerconfigjson` 유형의 Secret을 생성합니다.

실제 운영 환경에서는 명령어 인수에 비밀번호를 직접 입력하면 셸 히스토리나 프로세스 정보 등에 노출될 가능성이 있으므로 주의해야 합니다.

### 4.3. Secret 수정 및 삭제

기존 Secret을 수정하려면 다음 명령어를 사용할 수 있습니다.

```bash
kubectl edit secret application-secret
```

또는 YAML Manifest를 수정한 다음 다시 적용할 수 있습니다.

```bash
kubectl apply -f secret.yaml
```

Secret을 삭제하려면 다음 명령어를 사용합니다.

```bash
kubectl delete secret application-secret
```

Secret을 삭제하면 해당 Secret을 참조하는 Pod에서 필요한 데이터를 더 이상 정상적으로 제공받지 못할 수 있으므로 삭제 전에 참조 관계를 확인해야 합니다.

## 5. Pod에서 Secret 사용하기

Pod는 Secret에 저장된 데이터를 다양한 방식으로 사용할 수 있습니다.

대표적인 사용 방식은 다음과 같습니다.

1. 환경 변수(Environment Variables)로 주입
2. Volume을 통해 파일로 마운트
3. Image Pull Secret으로 사용
4. Kubernetes API를 통해 직접 조회

### 5.1. 환경 변수로 주입하기

Secret의 특정 Key를 컨테이너의 환경 변수로 주입할 수 있습니다.

Pod Manifest에서 `env[].valueFrom.secretKeyRef`를 사용합니다.

먼저 다음과 같은 Secret이 생성되어 있다고 가정합니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-secret
type: Opaque
stringData:
  DB_USERNAME: admin
  DB_PASSWORD: secure-password
```

Pod에서는 다음과 같이 특정 Key를 환경 변수에 연결할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: application-pod
spec:
  containers:
    - name: application
      image: nginx:latest
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: DB_USERNAME

        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: DB_PASSWORD
```

`secretKeyRef`는 특정 Secret의 특정 Key를 참조합니다.

위 예제에서는 `database-secret`의 `DB_USERNAME`과 `DB_PASSWORD`를 각각 컨테이너의 환경 변수로 설정합니다.

컨테이너 내부에서는 일반적인 환경 변수와 동일하게 사용할 수 있습니다.

```bash
echo $DB_USERNAME
echo $DB_PASSWORD
```

운영 환경에서는 비밀번호와 같은 민감한 환경 변수를 터미널이나 애플리케이션 로그에 출력하지 않도록 주의해야 합니다.

#### optional 설정

`secretKeyRef`에는 `optional` 필드를 지정할 수 있습니다.

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: database-secret
        key: DB_PASSWORD
        optional: true
```

`optional: true`로 설정하면 참조하는 Secret이나 Key가 존재하지 않더라도 해당 참조로 인해 컨테이너 시작이 실패하지 않습니다.

이 경우 해당 환경 변수는 설정되지 않습니다.

반대로 `optional`을 생략하거나 `false`로 설정하면 필수 참조로 처리됩니다. 참조 대상이 존재하지 않으면 컨테이너를 정상적으로 시작할 수 없습니다.

### 5.2. envFrom을 이용한 환경 변수 일괄 주입

Secret의 모든 Key-Value 항목을 환경 변수로 주입하려면 `envFrom`을 사용할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: application-pod
spec:
  containers:
    - name: application
      image: nginx:latest
      envFrom:
        - secretRef:
            name: database-secret
```

위 예제에서는 `database-secret`에 정의된 모든 Key가 환경 변수로 등록됩니다.

예를 들어 Secret에 다음과 같은 데이터가 있다면,

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-secret
type: Opaque
stringData:
  DB_USERNAME: admin
  DB_PASSWORD: secure-password
```

컨테이너 내부에서 다음과 같이 사용할 수 있습니다.

```bash
echo $DB_USERNAME
echo $DB_PASSWORD
```

`env`와 `envFrom`의 차이점은 다음과 같습니다.

| 구분 | env | envFrom |
|---|---|---|
| 주입 방식 | 특정 Key를 선택하여 주입 | Secret의 모든 Key를 일괄 주입 |
| 참조 필드 | `secretKeyRef` | `secretRef` |
| 환경 변수 이름 | 직접 지정 가능 | Secret의 Key를 사용 |
| 주요 용도 | 필요한 인증 정보만 선택적으로 사용 | 여러 인증 정보를 일괄적으로 사용 |

`envFrom`에서는 `prefix`를 지정하여 주입되는 환경 변수 이름에 접두사를 추가할 수도 있습니다.

```yaml
envFrom:
  - prefix: DB_
    secretRef:
      name: database-secret
```

이 경우 Secret의 `USERNAME`은 `DB_USERNAME`이라는 환경 변수로 주입됩니다.

### 5.3. Volume을 이용한 파일 마운트

Secret은 Volume으로 마운트하여 컨테이너 내부에서 파일 형태로 사용할 수도 있습니다.

이 방식은 애플리케이션이 인증서, 개인 키 또는 설정 파일을 파일 시스템을 통해 읽는 구조일 때 유용합니다.

다음과 같은 Secret을 사용한다고 가정합니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: application-secret
type: Opaque
stringData:
  username: admin
  password: secure-password
```

Pod에서는 `spec.volumes`에 Secret을 지정하고 컨테이너의 `volumeMounts`를 통해 마운트합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: application-pod
spec:
  containers:
    - name: application
      image: nginx:latest
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/application-secret
          readOnly: true

  volumes:
    - name: secret-volume
      secret:
        secretName: application-secret
```

위 설정에 따라 컨테이너 내부에는 다음과 같은 파일이 생성됩니다.

```text
/etc/application-secret/
├── username
└── password
```

각 파일의 이름은 Secret의 Key가 되고 파일의 내용은 해당 Key의 Value가 됩니다.

애플리케이션에서는 다음과 같이 파일을 읽어 인증 정보를 사용할 수 있습니다.

```bash
cat /etc/application-secret/username
cat /etc/application-secret/password
```

실제 운영 환경에서는 민감한 파일의 내용을 터미널이나 로그에 출력하지 않도록 주의해야 합니다.

Secret Volume은 일반적으로 메모리 기반 파일 시스템을 통해 제공되므로 컨테이너의 일반적인 이미지 레이어에 민감한 데이터가 기록되는 것을 방지하는 데 도움이 됩니다.

다만 컨테이너 내부에서 데이터를 다른 위치로 복사하거나 로그에 기록하면 노출될 수 있으므로 애플리케이션 차원의 관리도 필요합니다.

### 5.4. 특정 Key만 마운트하기

Secret의 모든 항목이 아니라 특정 Key만 파일로 마운트하려면 `items`를 사용할 수 있습니다.

```yaml
volumes:
  - name: secret-volume
    secret:
      secretName: application-secret
      items:
        - key: username
          path: credentials/username
```

위 예제에서는 `username`만 Volume에 포함됩니다.

`key`는 Secret에서 가져올 Key를 지정하고, `path`는 마운트된 Volume 내부에서 사용할 파일 경로를 지정합니다.

예를 들어 다음과 같이 설정할 수도 있습니다.

```yaml
volumes:
  - name: secret-volume
    secret:
      secretName: application-secret
      items:
        - key: password
          path: database/password
```

이 경우 컨테이너 내부의 파일 경로는 다음과 같습니다.

```text
/etc/application-secret/database/password
```

### 5.5. subPath를 이용한 단일 파일 마운트

기존 디렉터리 전체를 마운트하는 대신 특정 파일 하나만 컨테이너의 기존 디렉터리 구조에 연결하려면 `subPath`를 사용할 수 있습니다.

```yaml
volumeMounts:
  - name: secret-volume
    mountPath: /app/config/password
    subPath: password
    readOnly: true

volumes:
  - name: secret-volume
    secret:
      secretName: application-secret
```

이 방식은 기존 디렉터리의 다른 파일을 그대로 유지하면서 Secret의 특정 파일만 연결할 때 유용합니다.

단, `subPath`로 마운트된 Secret 파일은 Secret이 변경되어도 자동으로 업데이트되지 않습니다.

### 5.6. Image Pull Secret으로 사용하기

Private Container Registry에서 이미지를 가져와야 하는 경우 Secret을 이미지 Pull 인증 정보로 사용할 수 있습니다.

먼저 Docker Registry 인증 정보를 저장하는 Secret을 생성합니다.

```bash
kubectl create secret docker-registry registry-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=my-user \
  --docker-password=my-password
```

이후 Pod의 `imagePullSecrets`에 Secret을 지정합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-image-pod
spec:
  containers:
    - name: application
      image: private-registry.example.com/application:1.0

  imagePullSecrets:
    - name: registry-secret
```

Kubernetes는 지정된 Secret을 사용하여 Private Registry에 인증한 뒤 컨테이너 이미지를 가져옵니다.

`imagePullSecrets`에 지정하는 Secret은 일반적으로 `kubernetes.io/dockerconfigjson` 유형이어야 하며, Pod와 동일한 Namespace에 존재해야 합니다.

여러 Pod에서 동일한 Registry 인증 정보를 사용해야 한다면 ServiceAccount의 `imagePullSecrets`에 지정하여 재사용할 수도 있습니다.

### 5.7. Kubernetes API를 통해 직접 조회하기

애플리케이션에서 Kubernetes API를 직접 호출하여 Secret을 조회하는 방법도 있습니다.

이 방식에서는 애플리케이션이 Kubernetes API Server에 요청하여 Secret의 데이터를 가져옵니다.

일반적으로 다음과 같은 과정을 거칩니다.

1. Pod에 ServiceAccount를 연결합니다.
2. 해당 ServiceAccount에 Secret을 조회할 수 있는 RBAC 권한을 부여합니다.
3. 애플리케이션에서 Kubernetes API Server에 요청합니다.
4. API Server가 반환한 Secret 데이터를 애플리케이션에서 사용합니다.

이 방식은 애플리케이션이 Secret의 변경 사항을 직접 조회하거나 Watch API 등을 통해 감지해야 하는 경우 활용할 수 있습니다.

다만 Secret을 조회할 수 있는 권한은 민감한 데이터에 대한 접근 권한과 동일하므로 최소 권한 원칙에 따라 부여해야 합니다.

## 6. Secret 변경 시 반영 방식

Secret은 독립적인 Kubernetes 리소스이므로 Pod와 별도로 수정할 수 있습니다.

다만 Secret을 사용하는 방식에 따라 변경 사항이 반영되는 시점과 방식이 다릅니다.

### 6.1. 환경 변수로 사용하는 경우

Secret의 데이터를 환경 변수로 주입한 경우, 이미 실행 중인 컨테이너의 환경 변수는 자동으로 변경되지 않습니다.

컨테이너가 시작될 때 Secret의 값을 환경 변수로 가져오기 때문입니다.

따라서 Secret을 수정하더라도 기존 컨테이너에는 변경 사항이 반영되지 않습니다.

변경된 값을 적용하려면 컨테이너를 재시작하거나 Pod를 새로 생성해야 합니다.

Deployment에서 관리하는 Pod라면 다음과 같이 재시작할 수 있습니다.

```bash
kubectl rollout restart deployment application
```

### 6.2. Volume으로 마운트한 경우

Secret을 일반적인 Volume으로 마운트한 경우에는 Secret 변경 사항이 파일에 자동으로 반영됩니다.

다만 업데이트는 즉시 이루어지지 않으며, kubelet의 동기화 주기와 Secret 캐시 정책에 따라 지연될 수 있습니다.

기본 설정에서는 다음 요소에 의해 업데이트 지연이 발생할 수 있습니다.

- kubelet의 Pod 동기화 주기: 기본 1분
- Secret 캐시 TTL: 기본 1분

따라서 Secret 변경 후 파일에 반영되기까지 최대 약 2분이 소요될 수 있습니다.

단, 파일이 업데이트되더라도 애플리케이션이 해당 파일을 다시 읽거나 인증 정보를 Reload하는 기능을 갖추고 있어야 실제 동작에 변경 사항이 반영됩니다.

애플리케이션이 시작 시점에만 인증 정보를 읽는 구조라면 별도의 Reload 또는 재시작 처리가 필요합니다.

### 6.3. subPath로 마운트한 경우

`subPath`를 사용하여 Secret의 특정 파일을 마운트한 경우에는 Secret이 변경되어도 해당 파일이 자동으로 업데이트되지 않습니다.

따라서 Secret의 변경 사항을 반영하려면 Pod를 다시 생성해야 합니다.

다음 표는 각 방식의 변경 사항 반영 특성을 정리한 것입니다.

| 사용 방식 | Secret 변경 시 반영 | 추가 작업 |
|---|---|---|
| 환경 변수(env) | 자동 반영되지 않음 | 컨테이너 재시작 또는 Pod 재생성 |
| Volume 마운트 | 지연 후 자동 업데이트 | 애플리케이션 Reload가 필요할 수 있음 |
| subPath 마운트 | 자동 업데이트되지 않음 | Pod 재생성 |
| Image Pull Secret | 기존 실행 컨테이너에 영향 없음 | 이후 이미지 Pull 시 인증 정보 사용 |
| Kubernetes API 직접 조회 | 애플리케이션 구현에 따라 다름 | Watch 또는 재조회 로직 필요 |

## 7. Immutable Secret

Immutable Secret은 생성 이후 Secret의 데이터 변경을 허용하지 않는 Secret입니다.

`immutable: true`를 지정하면 Secret의 `data` 필드를 수정할 수 없게 됩니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: application-secret
type: Opaque
immutable: true
stringData:
  DB_USERNAME: admin
  DB_PASSWORD: secure-password
```

Immutable Secret은 Kubernetes v1.21부터 Stable 기능으로 제공됩니다.

### 7.1. 주요 특징

Immutable Secret을 사용하면 다음과 같은 이점이 있습니다.

- 실수나 의도하지 않은 민감한 데이터 변경을 방지할 수 있습니다.
- Secret 변경을 감시하는 kubelet의 Watch 부하를 줄일 수 있습니다.
- 인증 정보가 변경되지 않는다는 전제하에 보다 안정적으로 설정을 관리할 수 있습니다.

특히 Secret을 대규모로 사용하는 클러스터에서 API Server와 kubelet의 부하를 줄이는 데 도움이 됩니다.

### 7.2. 변경 및 삭제

Immutable Secret은 `immutable` 값을 다시 `false`로 변경할 수 없으며, `data`의 내용도 수정할 수 없습니다.

설정을 변경해야 한다면 기존 Secret을 삭제하고 새 Secret을 생성해야 합니다.

일반적으로는 새로운 이름을 사용하는 방식이 권장됩니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: application-secret-v2
type: Opaque
immutable: true
stringData:
  DB_USERNAME: admin
  DB_PASSWORD: new-secure-password
```

새 Secret을 생성한 다음 Deployment 등에서 참조하는 Secret 이름을 변경하면 새로운 인증 정보를 사용하는 Pod를 생성할 수 있습니다.

기존 Secret을 삭제하고 재생성하는 경우에는 해당 Secret을 참조하는 기존 Pod도 재생성하는 것이 권장됩니다.

## 8. Secret 보안 관리

Secret은 민감한 정보를 저장하는 리소스이므로 데이터 저장뿐 아니라 접근 제어, 암호화, 전달 방식까지 고려해야 합니다.

### 8.1. Base64 인코딩과 암호화 구분

Secret의 `data` 필드는 Base64 인코딩 형태로 저장됩니다.

Base64는 데이터를 텍스트 형태로 표현하기 위한 인코딩 방식이며, 암호화 기능을 제공하지 않습니다.

예를 들어 다음과 같은 값을 저장했다고 가정합니다.

```yaml
data:
  DB_PASSWORD: cGFzc3dvcmQ=
```

해당 값은 다음 명령어로 쉽게 디코딩할 수 있습니다.

```bash
echo 'cGFzc3dvcmQ=' | base64 --decode
```

출력 결과는 다음과 같습니다.

```text
password
```

따라서 Secret의 데이터가 Base64로 표현되어 있다는 이유만으로 안전하다고 판단해서는 안 됩니다.

### 8.2. Encryption at Rest

Kubernetes는 기본적으로 Secret 데이터를 etcd에 저장할 때 암호화를 적용하지 않습니다.

따라서 별도의 설정이 없다면 etcd에 저장된 Secret 데이터를 직접 읽을 수 있는 사용자가 민감한 정보를 확인할 가능성이 있습니다.

이를 방지하려면 Encryption at Rest를 구성해야 합니다.

Kubernetes API Server의 EncryptionConfiguration을 통해 Secret 리소스에 대한 암호화 공급자(Encryption Provider)를 설정할 수 있습니다.

대표적인 공급자 유형은 다음과 같습니다.

| Provider | 설명 |
|---|---|
| `identity` | 암호화 없이 원본 데이터를 저장합니다. |
| `aescbc` | AES-CBC 알고리즘을 사용하여 데이터를 암호화합니다. |
| `secretbox` | XSalsa20과 Poly1305를 사용하는 암호화 방식을 제공합니다. |
| `aesgcm` | AES-GCM 알고리즘을 사용하여 데이터를 암호화합니다. |
| `kms` | 외부 Key Management Service를 이용하여 암호화합니다. |

운영 환경에서는 클라우드 KMS 등 외부 키 관리 시스템과 연동하여 암호화 키를 안전하게 관리하는 방식을 고려할 수 있습니다.

Encryption at Rest를 적용하면 etcd에 저장된 데이터의 기밀성을 강화할 수 있지만, API Server를 통해 Secret을 조회할 수 있는 권한까지 제한하는 것은 아닙니다.

따라서 암호화와 RBAC을 함께 적용해야 합니다.

### 8.3. RBAC을 통한 접근 제어

Secret에 대한 접근 권한은 Kubernetes RBAC을 통해 관리할 수 있습니다.

Secret을 조회할 수 있는 권한이 있다면 민감한 데이터에 접근할 수 있으므로 필요한 사용자나 ServiceAccount에만 권한을 부여해야 합니다.

예를 들어 특정 Namespace에서 `application-secret`만 조회할 수 있는 Role을 생성할 수 있습니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["application-secret"]
    verbs: ["get"]
```

이 Role을 특정 ServiceAccount에 연결하려면 RoleBinding을 생성합니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-reader-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: application-sa
    namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: secret-reader
```

위 설정에서는 `application-sa` ServiceAccount에 `application-secret`을 조회할 수 있는 권한만 부여합니다.

Secret에 대한 접근 권한을 부여할 때는 다음 사항을 고려해야 합니다.

- 불필요한 `list`, `watch`, `get` 권한을 부여하지 않습니다.
- 가능한 경우 특정 Secret만 접근할 수 있도록 `resourceNames`를 지정합니다.
- Namespace 범위의 Role을 우선적으로 활용합니다.
- ClusterRole을 사용할 경우 불필요하게 모든 Namespace의 Secret에 접근할 수 있는 권한이 부여되지 않도록 주의합니다.

### 8.4. Secret을 환경 변수로 전달할 때의 주의사항

Secret을 환경 변수로 주입하면 애플리케이션에서 일반적인 환경 변수와 동일하게 사용할 수 있습니다.

다만 환경 변수는 다음과 같은 경로를 통해 노출될 가능성이 있습니다.

- 애플리케이션 로그
- 디버깅 및 진단 도구
- 프로세스 환경 정보를 조회할 수 있는 권한
- 컨테이너 내부에서 실행되는 다른 프로세스

따라서 민감한 정보는 가능한 한 필요한 프로세스에만 제한적으로 전달해야 하며, 로그나 오류 메시지에 포함되지 않도록 관리해야 합니다.

### 8.5. 외부 Secret 관리 시스템 활용

운영 환경에서는 Kubernetes Secret만으로 모든 민감한 정보를 관리하기보다 외부 Secret 관리 시스템을 함께 활용할 수 있습니다.

대표적인 솔루션은 다음과 같습니다.

- HashiCorp Vault
- AWS Secrets Manager
- AWS Systems Manager Parameter Store
- Azure Key Vault
- Google Cloud Secret Manager

외부 시스템을 활용하면 중앙 집중식으로 인증 정보를 관리하고 접근 권한, 감사 로그, 자동 회전(Rotation) 등의 기능을 활용할 수 있습니다.

Kubernetes에서는 External Secrets Operator 또는 Secrets Store CSI Driver 등의 도구를 사용하여 외부 Secret 관리 시스템과 연동할 수 있습니다.

이러한 방식은 Kubernetes Secret에 민감한 데이터를 직접 저장하는 대신 외부 시스템에서 필요한 데이터를 가져와 사용하는 구조를 구성할 수 있도록 지원합니다.

## 9. ConfigMap과 Secret 비교

ConfigMap과 Secret은 모두 Kubernetes에서 애플리케이션 설정 데이터를 저장하고 Pod에 전달하기 위한 리소스입니다.

두 리소스는 Key-Value 형태로 데이터를 저장하고 환경 변수나 Volume을 통해 Pod에서 사용할 수 있다는 공통점이 있습니다.

가장 중요한 차이점은 저장하려는 데이터의 성격과 보안 목적입니다.

| 구분 | ConfigMap | Secret |
|---|---|---|
| 목적 | 비기밀 설정 데이터 저장 | 민감한 정보 저장 |
| 주요 데이터 | 애플리케이션 설정, 로그 레벨, 환경별 설정 | 비밀번호, 토큰, 인증서, 개인 키 |
| 데이터 구조 | Key-Value | Key-Value |
| 환경 변수 주입 | 지원 | 지원 |
| Volume 마운트 | 지원 | 지원 |
| 데이터 크기 제한 | 1MiB | 1MiB |
| Base64 인코딩 | `binaryData`에서 사용 | 일반적인 `data` 필드에서 사용 |
| 암호화 | 제공하지 않음 | 기본적으로 자동 암호화되지 않으며 별도 설정 필요 |
| Immutable | 지원 | 지원 |

일반적인 애플리케이션 설정은 ConfigMap으로 관리하고, 인증 정보와 같은 민감한 데이터는 Secret으로 관리하는 것이 적절합니다.

다만 Secret을 사용한다고 해서 자동으로 데이터가 암호화되거나 접근이 제한되는 것은 아니므로 별도의 보안 설정이 필요합니다.

## 10. 정리

Secret은 Kubernetes에서 애플리케이션의 민감한 정보를 안전하게 분리하고 관리하기 위한 핵심 리소스입니다.

주요 특징은 다음과 같습니다.

- 비밀번호, 인증 토큰, TLS 인증서, SSH 개인 키 등의 민감한 데이터를 저장합니다.
- `data`와 `stringData`를 통해 Key-Value 형태의 데이터를 관리합니다.
- `Opaque`, `kubernetes.io/tls`, `kubernetes.io/dockerconfigjson` 등 다양한 유형을 지원합니다.
- 환경 변수, Volume, Image Pull Secret, Kubernetes API를 통해 Pod에서 사용할 수 있습니다.
- 하나의 Secret을 여러 Pod에서 공유할 수 있습니다.
- Pod와 Secret은 동일한 Namespace에 존재해야 합니다.
- 데이터 크기는 1MiB로 제한됩니다.
- Base64는 인코딩 방식이며 암호화가 아닙니다.
- 기본적으로 etcd에 저장되는 Secret 데이터는 암호화되지 않으므로 Encryption at Rest 설정을 고려해야 합니다.
- RBAC을 통해 Secret에 대한 접근 권한을 최소화해야 합니다.
- 일반적인 Volume 마운트는 Secret 변경 사항을 자동으로 반영하지만, 환경 변수와 subPath 마운트는 자동 업데이트되지 않습니다.
- Immutable 설정을 통해 데이터 변경을 방지할 수 있습니다.
- 운영 환경에서는 외부 Secret 관리 시스템과의 연동도 고려할 수 있습니다.

---

## 참고자료

1. [Kubernetes 공식 문서 - Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
   - Secret의 개념, 유형, 데이터 저장 방식, Immutable Secret 및 보안 관리 방법을 설명합니다.

2. [Kubernetes 공식 문서 - Managing Secrets using kubectl](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/)
   - kubectl을 이용한 Secret 생성, 조회 및 관리 방법을 설명합니다.

3. [Kubernetes 공식 문서 - Using Secrets](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets)
   - 환경 변수, Volume, Image Pull Secret 등 다양한 Secret 활용 방법을 설명합니다.

4. [Kubernetes 공식 문서 - Encrypting Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
   - EncryptionConfiguration을 이용한 Secret 등의 리소스에 대한 저장 데이터 암호화 방법을 설명합니다.

5. [Kubernetes 공식 문서 - Good practices for Kubernetes Secrets](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)
   - Secret 접근 제어, RBAC, 저장 데이터 보호 및 운영 환경에서의 보안 관리 권장 사항을 설명합니다.

6. [Kubernetes 공식 문서 - RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
   - Role, RoleBinding, ClusterRole, ClusterRoleBinding을 이용한 Kubernetes 리소스 접근 제어 방법을 설명합니다.