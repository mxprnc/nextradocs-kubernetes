# ConfigMap

## 1. ConfigMap이란?

ConfigMap은 애플리케이션에서 사용하는 **비기밀 설정 데이터(non-confidential configuration data)를 Key-Value 형태로 저장하는 Kubernetes API 리소스**입니다.

컨테이너 이미지 내부에 애플리케이션 설정을 포함하는 대신 ConfigMap을 통해 설정을 외부에서 관리할 수 있습니다. 이를 통해 애플리케이션 코드와 실행 환경에 종속적인 설정을 분리하고, 동일한 컨테이너 이미지를 개발(Development), 테스트(Test), 운영(Production) 환경에서 재사용할 수 있습니다.

예를 들어 데이터베이스 주소, 애플리케이션 실행 모드, 로그 레벨, 기능 활성화 여부 등의 설정을 ConfigMap에 저장할 수 있습니다.

```text
                    Kubernetes Cluster
┌───────────────────────────────────────────────────────┐
│                                                       │
│  ┌──────────────────────┐                             │
│  │      ConfigMap       │                             │
│  │                      │                             │
│  │  DATABASE_HOST       │                             │
│  │  LOG_LEVEL           │                             │
│  │  APP_MODE            │                             │
│  └──────────┬───────────┘                             │
│             │                                         │
│             │ Configuration Data                      │
│             ▼                                         │
│  ┌───────────────────────────────┐                    │
│  │             Pod               │                    │
│  │                               │                    │
│  │  ┌─────────────────────────┐  │                    │
│  │  │      Container          │  │                    │
│  │  │                         │  │                    │
│  │  │  Application            │  │                    │
│  │  │  - DATABASE_HOST        │  │                    │
│  │  │  - LOG_LEVEL            │  │                    │
│  │  │  - APP_MODE             │  │                    │
│  │  └─────────────────────────┘  │                    │
│  └───────────────────────────────┘                    │
│                                                       │
└───────────────────────────────────────────────────────┘
```

ConfigMap은 설정 데이터를 저장하고 Pod는 해당 데이터를 참조하여 컨테이너를 구성합니다. 하나의 ConfigMap을 여러 Pod에서 공유하여 사용할 수도 있습니다.

다만 ConfigMap은 비기밀 데이터 저장을 목적으로 설계되었으며, 비밀번호, 인증 토큰, 개인 키 등의 민감한 정보는 Secret을 사용해야 합니다.

## 2. ConfigMap의 주요 특징

### 2.1. 애플리케이션과 설정의 분리

ConfigMap을 사용하면 컨테이너 이미지와 환경별 설정을 독립적으로 관리할 수 있습니다.

예를 들어 동일한 애플리케이션 이미지를 사용하면서 환경에 따라 서로 다른 데이터베이스 주소를 지정할 수 있습니다.

| 환경 | DATABASE_HOST |
|---|---|
| Development | localhost |
| Staging | staging-db |
| Production | production-db |

환경마다 컨테이너 이미지를 새로 빌드할 필요 없이 ConfigMap의 설정만 변경하여 동일한 애플리케이션을 서로 다른 환경에서 실행할 수 있습니다.

### 2.2. Key-Value 기반 데이터 저장

ConfigMap은 `data`와 `binaryData` 필드를 통해 설정 데이터를 저장합니다.

- `data`: UTF-8 문자열 형태의 데이터를 저장합니다.
- `binaryData`: 바이너리 데이터를 Base64 인코딩 형태로 저장합니다.

일반적인 애플리케이션 설정은 대부분 `data` 필드를 사용합니다.

### 2.3. Namespace 범위

ConfigMap은 Namespace 범위의 리소스입니다.

Pod에서 ConfigMap을 참조하려면 두 리소스가 동일한 Namespace에 존재해야 합니다.

서로 다른 Namespace에 있는 ConfigMap을 일반적인 Pod 설정 참조 방식으로 직접 사용할 수는 없습니다.

### 2.4. 데이터 크기 제한

ConfigMap에 저장할 수 있는 데이터의 크기는 1MiB로 제한됩니다.

대용량 설정 파일이나 대량의 데이터를 저장하기 위한 용도로는 적합하지 않습니다. 이러한 데이터는 별도의 Volume, 데이터베이스 또는 파일 스토리지 서비스를 사용하는 것이 적절합니다.

### 2.5. 비기밀 데이터 저장

ConfigMap은 데이터를 암호화하거나 보호하는 보안 기능을 제공하지 않습니다.

따라서 비밀번호, API Key, 인증서의 개인 키와 같은 민감한 정보는 ConfigMap에 저장하지 않아야 합니다.

민감한 설정 데이터는 Secret을 사용해야 합니다.

## 3. ConfigMap의 구조

ConfigMap은 `apiVersion: v1`을 사용하는 Kubernetes Core API 리소스입니다.

다음은 ConfigMap의 기본적인 YAML 구조입니다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: application-config
  namespace: default
data:
  APP_MODE: production
  LOG_LEVEL: INFO
  DATABASE_HOST: database-service
```

주요 필드는 다음과 같습니다.

| 필드 | 설명 |
|---|---|
| `apiVersion` | Kubernetes API 버전을 지정합니다. ConfigMap은 `v1`을 사용합니다. |
| `kind` | 리소스 종류를 지정합니다. `ConfigMap`을 사용합니다. |
| `metadata.name` | ConfigMap의 이름을 지정합니다. |
| `metadata.namespace` | ConfigMap이 생성될 Namespace를 지정합니다. |
| `data` | UTF-8 문자열 형태의 설정 데이터를 저장합니다. |
| `binaryData` | Base64 인코딩된 바이너리 데이터를 저장합니다. |
| `immutable` | ConfigMap의 데이터 변경을 방지할지 여부를 지정합니다. |

`data`와 `binaryData`는 선택 필드이며, 두 필드에 동일한 Key를 중복해서 정의할 수 없습니다.

또한 ConfigMap의 이름은 유효한 DNS Subdomain 형식이어야 하며, 각 Key는 영숫자 문자, 하이픈(`-`), 언더스코어(`_`), 마침표(`.`)로 구성해야 합니다.

### 3.1. 여러 줄의 설정 데이터 저장

ConfigMap은 단일 문자열뿐 아니라 여러 줄로 구성된 설정 파일의 내용도 저장할 수 있습니다.

YAML의 Literal Block Scalar(`|`)를 사용하면 여러 줄의 문자열을 하나의 Key에 저장할 수 있습니다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: application-config
data:
  application.properties: |
    server.port=8080
    logging.level.root=INFO
    spring.application.name=sample-service

  nginx.conf: |
    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://application:8080;
        }
    }
```

위 예제에서는 `application.properties`와 `nginx.conf`라는 두 개의 Key에 각각 설정 파일의 내용을 저장합니다.

ConfigMap 자체는 각 Key의 값이 단일 설정 값인지 파일 형태의 데이터인지 구분하지 않습니다. 실제 데이터가 어떻게 사용되는지는 Pod에서 ConfigMap을 참조하는 방식에 따라 결정됩니다.

## 4. ConfigMap 생성 및 관리

ConfigMap은 YAML Manifest를 사용하거나 `kubectl create configmap` 명령어를 사용하여 생성할 수 있습니다.

### 4.1. YAML Manifest를 이용한 생성

다음과 같이 ConfigMap Manifest를 작성합니다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: application-config
data:
  APP_MODE: production
  LOG_LEVEL: INFO
  DATABASE_HOST: database-service
```

Manifest를 적용합니다.

```bash
kubectl apply -f configmap.yaml
```

ConfigMap이 정상적으로 생성되었는지 확인합니다.

```bash
kubectl get configmaps
```

특정 ConfigMap의 상세 정보를 확인하려면 다음 명령어를 사용합니다.

```bash
kubectl describe configmap application-config
```

YAML 형식으로 전체 리소스 정보를 확인하려면 다음 명령어를 사용합니다.

```bash
kubectl get configmap application-config -o yaml
```

### 4.2. 명령어를 이용한 생성

`kubectl create configmap` 명령어를 사용하면 별도의 YAML Manifest 없이 ConfigMap을 생성할 수 있습니다.

**Literal 값을 이용한 생성**

```bash
kubectl create configmap application-config \
  --from-literal=APP_MODE=production \
  --from-literal=LOG_LEVEL=INFO \
  --from-literal=DATABASE_HOST=database-service
```

**파일을 이용한 생성**

```bash
kubectl create configmap application-config \
  --from-file=application.properties
```

파일을 이용하면 파일 이름이 ConfigMap의 Key가 되고 파일 내용이 Value가 됩니다.

예를 들어 `application.properties` 파일에 다음과 같은 내용이 있다면,

```properties
server.port=8080
logging.level.root=INFO
```

ConfigMap에는 다음과 같은 형태로 저장됩니다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: application-config
data:
  application.properties: |
    server.port=8080
    logging.level.root=INFO
```

**디렉터리를 이용한 생성**

```bash
kubectl create configmap application-config \
  --from-file=./config/
```

디렉터리를 지정하면 디렉터리 내부의 적절한 파일들이 각각 별도의 Key-Value 항목으로 ConfigMap에 저장됩니다. 파일 이름이 Key가 되고 파일 내용이 Value가 됩니다.

### 4.3. ConfigMap 수정 및 삭제

기존 ConfigMap을 수정하려면 다음 명령어를 사용할 수 있습니다.

```bash
kubectl edit configmap application-config
```

또는 YAML Manifest를 수정한 다음 다시 적용할 수 있습니다.

```bash
kubectl apply -f configmap.yaml
```

ConfigMap을 삭제하려면 다음 명령어를 사용합니다.

```bash
kubectl delete configmap application-config
```

## 5. Pod에서 ConfigMap 사용하기

Pod는 ConfigMap에 저장된 데이터를 다양한 방식으로 사용할 수 있습니다.

대표적인 사용 방식은 다음과 같습니다.

1. 환경 변수(Environment Variables)로 주입
2. Volume을 통해 설정 파일로 마운트
3. 컨테이너의 Command 및 Args에서 사용
4. Kubernetes API를 통해 직접 조회

### 5.1. 환경 변수로 주입하기

ConfigMap의 특정 Key를 컨테이너의 환경 변수로 주입할 수 있습니다.

Pod Manifest에서 `env[].valueFrom.configMapKeyRef`를 사용합니다.

먼저 다음과 같은 ConfigMap이 생성되어 있다고 가정합니다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: application-config
data:
  APP_MODE: production
  LOG_LEVEL: INFO
  DATABASE_HOST: database-service
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
        - name: APP_MODE
          valueFrom:
            configMapKeyRef:
              name: application-config
              key: APP_MODE

        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: application-config
              key: LOG_LEVEL
```

`configMapKeyRef`는 특정 ConfigMap의 특정 Key를 참조합니다.

위 예제에서는 `application-config`의 `APP_MODE`와 `LOG_LEVEL`을 각각 컨테이너의 환경 변수로 설정합니다.

컨테이너 내부에서는 일반적인 환경 변수와 동일하게 사용할 수 있습니다.

```bash
echo $APP_MODE
echo $LOG_LEVEL
```

출력 결과는 다음과 같습니다.

```text
production
INFO
```

#### optional 설정

`configMapKeyRef`에는 `optional` 필드를 지정할 수 있습니다.

```yaml
env:
  - name: APP_MODE
    valueFrom:
      configMapKeyRef:
        name: application-config
        key: APP_MODE
        optional: true
```

`optional: true`로 설정하면 참조하는 ConfigMap이나 Key가 존재하지 않더라도 해당 참조로 인해 컨테이너 시작이 실패하지 않습니다.

이 경우 해당 환경 변수는 설정되지 않습니다.

반대로 `optional`을 생략하거나 `false`로 설정하면 필수 참조로 처리됩니다. 참조 대상이 존재하지 않으면 컨테이너를 정상적으로 시작할 수 없습니다.

### 5.2. envFrom을 이용한 환경 변수 일괄 주입

ConfigMap의 모든 Key-Value 항목을 환경 변수로 주입하려면 `envFrom`을 사용할 수 있습니다.

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
        - configMapRef:
            name: application-config
```

위 예제에서는 `application-config`에 정의된 모든 Key가 환경 변수로 등록됩니다.

예를 들어 ConfigMap이 다음과 같다면,

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: application-config
data:
  APP_MODE: production
  LOG_LEVEL: INFO
  DATABASE_HOST: database-service
```

컨테이너 내부에서는 다음과 같이 사용할 수 있습니다.

```bash
echo $APP_MODE
echo $LOG_LEVEL
echo $DATABASE_HOST
```

`env`와 `envFrom`의 차이점은 다음과 같습니다.

| 구분 | env | envFrom |
|---|---|---|
| 주입 방식 | 특정 Key를 선택하여 주입 | ConfigMap의 모든 Key를 일괄 주입 |
| 참조 필드 | `configMapKeyRef` | `configMapRef` |
| 환경 변수 이름 | 직접 지정 가능 | ConfigMap의 Key를 사용 |
| 주요 용도 | 필요한 설정만 선택적으로 사용 | 여러 설정을 일괄적으로 사용 |

`envFrom`에서는 `prefix`를 지정하여 주입되는 환경 변수 이름에 접두사를 추가할 수도 있습니다.

```yaml
envFrom:
  - prefix: APP_
    configMapRef:
      name: application-config
```

이 경우 ConfigMap의 `LOG_LEVEL`은 `APP_LOG_LEVEL`이라는 환경 변수로 주입됩니다.

### 5.3. Volume을 이용한 설정 파일 마운트

ConfigMap은 Volume으로 마운트하여 컨테이너 내부에서 파일 형태로 사용할 수도 있습니다.

이 방식은 애플리케이션이 환경 변수 대신 설정 파일을 읽는 구조일 때 유용합니다.

다음과 같은 ConfigMap을 사용한다고 가정합니다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: application-config
data:
  application.properties: |
    server.port=8080
    logging.level.root=INFO

  logging.properties: |
    logging.level=INFO
    logging.file=application.log
```

Pod에서는 `spec.volumes`에 ConfigMap을 지정하고, 컨테이너의 `volumeMounts`를 통해 마운트합니다.

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
        - name: config-volume
          mountPath: /etc/application-config
          readOnly: true

  volumes:
    - name: config-volume
      configMap:
        name: application-config
```

위 설정에 따라 컨테이너 내부에는 다음과 같은 파일이 생성됩니다.

```text
/etc/application-config/
├── application.properties
└── logging.properties
```

각 파일의 내용은 ConfigMap에 저장된 해당 Key의 Value와 동일합니다.

즉, ConfigMap의 Key가 파일 이름이 되고 Value가 파일의 내용이 됩니다.

#### 특정 Key만 마운트하기

ConfigMap의 모든 항목이 아니라 특정 Key만 파일로 마운트하려면 `items`를 사용할 수 있습니다.

```yaml
volumes:
  - name: config-volume
    configMap:
      name: application-config
      items:
        - key: application.properties
          path: application.properties
```

위 예제에서는 `application.properties`만 Volume에 포함됩니다.

`key`는 ConfigMap에서 가져올 Key를 지정하고, `path`는 마운트된 Volume 내부에서 사용할 파일 경로를 지정합니다.

예를 들어 다음과 같이 설정할 수도 있습니다.

```yaml
volumes:
  - name: config-volume
    configMap:
      name: application-config
      items:
        - key: application.properties
          path: config/application.properties
```

이 경우 컨테이너 내부의 파일 경로는 다음과 같습니다.

```text
/etc/application-config/config/application.properties
```

#### subPath를 이용한 단일 파일 마운트

기존 디렉터리 전체를 마운트하는 대신 특정 파일 하나만 컨테이너의 기존 디렉터리 구조에 연결하려면 `subPath`를 사용할 수 있습니다.

```yaml
volumeMounts:
  - name: config-volume
    mountPath: /app/config/application.properties
    subPath: application.properties
    readOnly: true

volumes:
  - name: config-volume
    configMap:
      name: application-config
```

이 방식은 기존 디렉터리의 다른 파일을 그대로 유지하면서 ConfigMap의 특정 파일만 연결할 때 유용합니다.

단, `subPath`로 마운트된 ConfigMap 파일은 ConfigMap이 변경되어도 자동으로 업데이트되지 않습니다.

### 5.4. Command와 Args에서 사용하기

ConfigMap을 환경 변수로 주입한 다음 컨테이너의 `command`와 `args`에서 사용할 수도 있습니다.

Kubernetes는 `$(VAR_NAME)` 형식의 변수 치환 문법을 지원합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: application-pod
spec:
  containers:
    - name: application
      image: busybox:latest
      command: ["/bin/sh", "-c"]
      args:
        - echo "Application Mode: $(APP_MODE)"
      env:
        - name: APP_MODE
          valueFrom:
            configMapKeyRef:
              name: application-config
              key: APP_MODE
```

위 예제에서는 ConfigMap에서 가져온 `APP_MODE` 값이 컨테이너의 실행 인수에 반영됩니다.

`$(VAR_NAME)`은 Kubernetes가 처리하는 변수 치환 문법이며, 셸의 `$VAR_NAME`과는 구분해야 합니다.

### 5.5. Kubernetes API를 통해 직접 조회하기

애플리케이션에서 Kubernetes API를 직접 호출하여 ConfigMap을 조회하는 방법도 있습니다.

이 방식에서는 애플리케이션이 Kubernetes API Server에 요청하여 ConfigMap의 데이터를 가져옵니다.

일반적으로 다음과 같은 과정을 거칩니다.

1. Pod에 ServiceAccount를 연결합니다.
2. 해당 ServiceAccount에 ConfigMap을 조회할 수 있는 RBAC 권한을 부여합니다.
3. 애플리케이션에서 Kubernetes API Server에 요청합니다.
4. API Server가 반환한 ConfigMap 데이터를 애플리케이션에서 사용합니다.

이 방식은 환경 변수나 Volume을 통해 설정 데이터를 전달하는 방식과 달리 애플리케이션이 API를 직접 사용한다는 특징이 있습니다.

특히 ConfigMap 변경 사항을 Watch API 등을 통해 감지하고, 변경된 설정에 맞춰 애플리케이션 동작을 갱신해야 하는 경우 활용할 수 있습니다.

다만 API 접근을 위해 적절한 인증 및 RBAC 권한 설정이 필요하며, 다른 Namespace의 ConfigMap을 조회하려면 해당 Namespace에 대한 접근 권한도 별도로 부여해야 합니다.

## 6. ConfigMap 변경 시 반영 방식

ConfigMap은 독립적인 Kubernetes 리소스이므로 Pod와 별도로 수정할 수 있습니다.

다만 ConfigMap을 사용하는 방식에 따라 변경 사항이 반영되는 시점과 방식이 다릅니다.

### 6.1. 환경 변수로 사용하는 경우

ConfigMap의 데이터를 환경 변수로 주입한 경우, 이미 실행 중인 컨테이너의 환경 변수는 자동으로 변경되지 않습니다.

컨테이너가 시작될 때 ConfigMap의 값을 환경 변수로 가져오기 때문입니다.

따라서 ConfigMap을 수정하더라도 기존 컨테이너에는 변경 사항이 반영되지 않습니다.

변경된 값을 적용하려면 컨테이너를 재시작하거나 Pod를 새로 생성해야 합니다.

Deployment에서 관리하는 Pod라면 다음과 같이 재시작할 수 있습니다.

```bash
kubectl rollout restart deployment application
```

### 6.2. Volume으로 마운트한 경우

ConfigMap을 일반적인 Volume으로 마운트한 경우에는 ConfigMap 변경 사항이 파일에 자동으로 반영됩니다.

다만 업데이트는 즉시 이루어지지 않으며, kubelet의 동기화 주기와 ConfigMap 캐시 정책에 따라 지연될 수 있습니다.

기본 설정에서는 다음 요소에 의해 업데이트 지연이 발생할 수 있습니다.

- kubelet의 Pod 동기화 주기: 기본 1분
- ConfigMap 캐시 TTL: 기본 1분

따라서 ConfigMap 변경 후 파일에 반영되기까지 최대 약 2분이 소요될 수 있습니다.

단, 파일이 업데이트되더라도 애플리케이션이 해당 파일을 다시 읽거나 설정을 Reload하는 기능을 갖추고 있어야 실제 동작에 변경 사항이 반영됩니다.

애플리케이션이 시작 시점에만 설정 파일을 읽는 구조라면 별도의 Reload 또는 재시작 처리가 필요합니다.

### 6.3. subPath로 마운트한 경우

`subPath`를 사용하여 ConfigMap의 특정 파일을 마운트한 경우에는 ConfigMap이 변경되어도 해당 파일이 자동으로 업데이트되지 않습니다.

따라서 ConfigMap의 변경 사항을 반영하려면 Pod를 다시 생성해야 합니다.

다음 표는 각 방식의 변경 사항 반영 특성을 정리한 것입니다.

| 사용 방식 | ConfigMap 변경 시 반영 | 추가 작업 |
|---|---|---|
| 환경 변수(env) | 자동 반영되지 않음 | 컨테이너 재시작 또는 Pod 재생성 |
| Volume 마운트 | 지연 후 자동 업데이트 | 애플리케이션 Reload가 필요할 수 있음 |
| subPath 마운트 | 자동 업데이트되지 않음 | Pod 재생성 |
| Kubernetes API 직접 조회 | 애플리케이션 구현에 따라 다름 | Watch 또는 재조회 로직 필요 |

## 7. Immutable ConfigMap

Immutable ConfigMap은 생성 이후 설정 데이터의 변경을 허용하지 않는 ConfigMap입니다.

`immutable: true`를 지정하면 ConfigMap의 `data`와 `binaryData` 필드를 수정할 수 없게 됩니다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: application-config
data:
  APP_MODE: production
  LOG_LEVEL: INFO
immutable: true
```

Immutable ConfigMap은 Kubernetes v1.21부터 Stable 기능으로 제공됩니다.

### 7.1. 주요 특징

Immutable ConfigMap을 사용하면 다음과 같은 이점이 있습니다.

- 실수나 의도하지 않은 설정 변경을 방지할 수 있습니다.
- ConfigMap 변경을 감시하는 kubelet의 Watch 부하를 줄일 수 있습니다.
- 설정 데이터가 변경되지 않는다는 전제하에 보다 안정적으로 설정을 관리할 수 있습니다.

특히 ConfigMap을 대규모로 사용하는 클러스터에서 API Server와 kubelet의 부하를 줄이는 데 도움이 됩니다.

### 7.2. 변경 및 삭제

Immutable ConfigMap은 `immutable` 값을 다시 `false`로 변경할 수 없으며, `data`와 `binaryData`의 내용도 수정할 수 없습니다.

설정을 변경해야 한다면 기존 ConfigMap을 삭제하고 새 ConfigMap을 생성해야 합니다.

일반적으로는 새로운 이름을 사용하는 방식이 권장됩니다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: application-config-v2
data:
  APP_MODE: production
  LOG_LEVEL: DEBUG
immutable: true
```

새 ConfigMap을 생성한 다음 Deployment 등에서 참조하는 ConfigMap 이름을 변경하면 새로운 설정을 사용하는 Pod를 생성할 수 있습니다.

기존 ConfigMap을 삭제하고 재생성하는 경우에는 해당 ConfigMap을 참조하는 기존 Pod도 재생성하는 것이 권장됩니다.

## 8. ConfigMap 사용 시 주의사항

### 8.1. 민감한 정보 저장 금지

ConfigMap은 암호화나 기밀성 보호를 제공하지 않습니다.

비밀번호, API Key, 인증 토큰, 개인 키 등 민감한 정보는 Secret을 사용해야 합니다.

또한 Secret 역시 기본적으로 데이터가 Base64 인코딩 형태로 표현된다는 점과 별도의 암호화 설정이 필요할 수 있다는 점을 고려하여 관리해야 합니다.

### 8.2. ConfigMap의 Namespace 확인

Pod와 ConfigMap은 동일한 Namespace에 존재해야 합니다.

Pod에서 ConfigMap을 참조할 수 없는 경우 두 리소스의 Namespace가 일치하는지 확인해야 합니다.

### 8.3. 환경 변수 변경에 따른 재시작 고려

환경 변수로 ConfigMap을 주입하는 경우 설정 변경만으로 기존 컨테이너의 환경 변수가 갱신되지 않습니다.

따라서 Deployment와 같은 Workload 리소스를 사용하는 경우 설정 변경 이후 Pod를 재생성하는 배포 전략을 고려해야 합니다.

### 8.4. Volume 업데이트와 애플리케이션 Reload 구분

ConfigMap을 Volume으로 마운트하면 파일 자체는 자동으로 업데이트될 수 있습니다.

그러나 파일의 변경과 애플리케이션 내부 설정의 변경은 별개의 문제입니다.

애플리케이션이 파일 변경을 감지하고 설정을 다시 읽도록 구현되어 있지 않다면 Pod 내부의 파일이 업데이트되더라도 애플리케이션 동작은 변경되지 않을 수 있습니다.

### 8.5. 대용량 데이터 저장 지양

ConfigMap은 1MiB 이하의 설정 데이터를 저장하기 위한 리소스입니다.

대용량 파일이나 일반적인 데이터 저장소로 사용하는 것은 적절하지 않습니다.

### 8.6. ConfigMap 변경과 Deployment 배포의 연관성

ConfigMap을 수정하더라도 이를 참조하는 Deployment의 Pod Template이 자동으로 변경되는 것은 아닙니다.

따라서 Deployment의 변경 사항이 감지되지 않아 새로운 Pod가 생성되지 않을 수 있습니다.

일반적으로 다음과 같은 방법을 사용하여 설정 변경을 배포 과정에 반영할 수 있습니다.

- `kubectl rollout restart`를 통한 수동 재시작
- ConfigMap의 버전별 이름 관리
- Pod Template의 Annotation에 ConfigMap 내용의 Hash 값을 설정하여 변경 감지

이러한 방식을 사용하면 ConfigMap 변경과 애플리케이션 배포를 일관되게 관리할 수 있습니다.

## 9. ConfigMap과 Secret 비교

ConfigMap과 Secret은 모두 Kubernetes에서 애플리케이션 설정 데이터를 저장하고 Pod에 전달하기 위한 리소스입니다.

두 리소스는 데이터를 Key-Value 형태로 저장하고 환경 변수나 Volume을 통해 Pod에서 사용할 수 있다는 공통점이 있습니다.

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

ConfigMap과 Secret은 용도에 따라 구분하여 사용해야 합니다.

일반적인 애플리케이션 설정은 ConfigMap으로 관리하고, 인증 정보와 같은 민감한 데이터는 Secret으로 관리하는 것이 적절합니다.

## 10. 정리

ConfigMap은 Kubernetes에서 애플리케이션 코드와 환경별 설정 데이터를 분리하기 위한 핵심 리소스입니다.

주요 특징은 다음과 같습니다.

- 비기밀 설정 데이터를 Key-Value 형태로 저장합니다.
- 동일한 컨테이너 이미지를 서로 다른 환경에서 재사용할 수 있도록 지원합니다.
- 환경 변수, Volume, Command 및 Args, Kubernetes API를 통해 Pod에서 사용할 수 있습니다.
- 하나의 ConfigMap을 여러 Pod에서 공유할 수 있습니다.
- Pod와 ConfigMap은 동일한 Namespace에 존재해야 합니다.
- 데이터 크기는 1MiB로 제한됩니다.
- 환경 변수로 주입한 설정은 ConfigMap 변경만으로 자동 갱신되지 않습니다.
- 일반적인 Volume 마운트는 ConfigMap 변경 사항을 자동으로 반영하지만, subPath 마운트는 자동 업데이트되지 않습니다.
- Immutable 설정을 통해 데이터 변경을 방지할 수 있습니다.
- 민감한 정보는 ConfigMap이 아닌 Secret으로 관리해야 합니다.

---

## 참고자료

1. [Kubernetes 공식 문서 - ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
   - ConfigMap의 개념, 구조, 데이터 저장 방식, Immutable ConfigMap 및 Pod와의 연동 방식을 설명합니다.

2. [Kubernetes 공식 문서 - Configure a Pod to Use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
   - 환경 변수, Volume 마운트, Command 및 Args 등을 이용하여 ConfigMap을 사용하는 방법과 YAML 예제를 제공합니다.

3. [Kubernetes 공식 문서 - Configuration](https://kubernetes.io/docs/concepts/configuration/)
   - Kubernetes에서 제공하는 설정 관리 메커니즘과 ConfigMap 및 Secret의 역할을 설명합니다.