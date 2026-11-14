# Kubernetes Configuration Resources

Configuration 리소스는 Kubernetes에서 애플리케이션 실행에 필요한 설정 데이터와 민감한 정보를 관리하기 위한 리소스입니다.

컨테이너 이미지와 애플리케이션 설정을 분리하여 동일한 이미지를 개발, 테스트 및 운영 환경에서 재사용할 수 있도록 지원합니다.

## 1. Configuration 리소스 구성

| 리소스 | 주요 역할 | 주요 특징 |
|---|---|---|
| **ConfigMap** | 일반 설정 데이터 관리 | 환경 변수, 설정 파일 등을 Pod에 전달 |
| **Secret** | 민감한 정보 관리 | 비밀번호, 토큰, 인증서 등을 Pod에 전달 |

## 2. 리소스 관계

### 2.1 ConfigMap 기반 설정 전달

```text
ConfigMap
    │
    │
    ▼
   Pod
    │
    ▼
 Container
    │
    ├── Environment Variables
    │
    └── Volume Mount
         └── Configuration Files
```

ConfigMap은 애플리케이션에서 사용하는 일반적인 설정 데이터를 저장합니다.

Pod는 ConfigMap의 데이터를 환경 변수 또는 Volume 형태로 전달받아 사용할 수 있습니다.

주요 사용 사례는 다음과 같습니다.

- 데이터베이스 Host 및 Port
- 애플리케이션 실행 환경
- 외부 API Endpoint
- 애플리케이션 설정 파일

### 2.2 Secret 기반 민감 정보 전달

```text
Secret
   │
   │
   ▼
  Pod
   │
   ▼
Container
   │
   ├── Environment Variables
   │
   └── Volume Mount
        └── Secret Files
```

Secret은 비밀번호, 인증 토큰, 인증서 등 민감한 정보를 저장합니다.

ConfigMap과 마찬가지로 환경 변수 또는 Volume 형태로 Pod에 전달할 수 있습니다.

주요 사용 사례는 다음과 같습니다.

- 데이터베이스 계정 정보
- API 인증 토큰
- TLS 인증서 및 개인키
- 외부 서비스 인증 정보

## 3. 데이터 전달 방식

ConfigMap과 Secret은 Pod에 데이터를 전달하는 방식이 유사합니다.

| 전달 방식 | 설명 |
|---|---|
| Environment Variables | 컨테이너의 환경 변수로 설정값 전달 |
| Volume Mount | 파일 형태로 컨테이너 내부에 마운트 |
| `envFrom` | ConfigMap 또는 Secret의 여러 항목을 환경 변수로 일괄 전달 |
| `projected` Volume | ConfigMap, Secret, ServiceAccountToken 등의 데이터를 하나의 Volume으로 결합 |

환경 변수로 전달된 값은 실행 중인 컨테이너에서 자동으로 갱신되지 않습니다.

Volume으로 마운트한 ConfigMap과 Secret은 일반적으로 데이터 변경 시 갱신될 수 있지만, `subPath`로 마운트한 경우에는 자동 갱신되지 않습니다.

## 4. ConfigMap과 Secret 비교

| 구분 | ConfigMap | Secret |
|---|---|---|
| 목적 | 일반 설정 데이터 관리 | 민감한 정보 관리 |
| 데이터 예시 | Host, Port, 환경 설정 | Password, Token, 인증서 |
| 환경 변수 전달 | 지원 | 지원 |
| Volume 마운트 | 지원 | 지원 |
| 데이터 인코딩 | 일반 문자열 | Base64 인코딩이 일반적 |
| 암호화 | 기본적으로 암호화하지 않음 | 기본적으로 Base64는 암호화가 아님 |

Secret은 민감한 정보를 관리하기 위한 리소스이지만, Base64 인코딩만으로 데이터가 보호되는 것은 아닙니다.

운영 환경에서는 Encryption at Rest, RBAC 및 외부 Secret 관리 시스템 등을 함께 고려해야 합니다.

## 5. 핵심 정리

- ConfigMap은 애플리케이션의 일반적인 설정 데이터를 관리합니다.
- Secret은 비밀번호, 토큰 및 인증서 등의 민감한 정보를 관리합니다.
- 두 리소스 모두 환경 변수 또는 Volume 형태로 Pod에 전달할 수 있습니다.
- 애플리케이션 이미지와 설정을 분리하여 환경별 설정을 독립적으로 관리할 수 있습니다.
- Secret을 사용할 때는 별도의 접근 제어 및 암호화 정책을 고려해야 합니다.

참고:
- https://kubernetes.io/docs/concepts/configuration/configmap/
- https://kubernetes.io/docs/concepts/configuration/secret/