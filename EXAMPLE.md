# Helm 템플릿 변수 참조 가이드

## values.yaml 구조

```yaml
# 최상위 레벨
namespace: envoy-gateway-system
createNamespace: true

# 배열 (여러 개의 EnvoyProxy)
envoyProxies:
  - name: custom-proxy-config        # 첫 번째 항목
    provider:
      type: Kubernetes
      kubernetes:
        envoyService:
          type: LoadBalancer
          annotations:
            key1: value1
            key2: value2
  
  - name: custom-proxy-config-internal  # 두 번째 항목
    provider:
      type: Kubernetes
      kubernetes:
        envoyService:
          type: LoadBalancer
          annotations:
            key3: value3
```

## 템플릿에서 참조하는 방법

### 1. 최상위 값 참조

```yaml
# 템플릿
namespace: {{ .Values.namespace }}

# 결과
namespace: envoy-gateway-system
```

### 2. 배열 반복 (range)

```yaml
# 템플릿
{{- range .Values.envoyProxies }}
---
name: {{ .name }}
{{- end }}

# 결과
---
name: custom-proxy-config
---
name: custom-proxy-config-internal
```

### 3. 중첩된 값 참조

```yaml
# 템플릿
{{- range .Values.envoyProxies }}
metadata:
  name: {{ .name }}
spec:
  provider:
    type: {{ .provider.type }}
    kubernetes:
      envoyService:
        type: {{ .provider.kubernetes.envoyService.type }}
{{- end }}

# 결과 (첫 번째 항목)
metadata:
  name: custom-proxy-config
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyService:
        type: LoadBalancer
```

### 4. range 안에서 최상위 값 참조

```yaml
# 템플릿
{{- range .Values.envoyProxies }}
metadata:
  name: {{ .name }}                      # 현재 항목의 name
  namespace: {{ $.Values.namespace }}    # 최상위 namespace ($ 사용!)
{{- end }}

# 결과
metadata:
  name: custom-proxy-config
  namespace: envoy-gateway-system
---
metadata:
  name: custom-proxy-config-internal
  namespace: envoy-gateway-system
```

## 변수 참조 규칙

| 위치 | 참조 방법 | 설명 |
|------|----------|------|
| 최상위 | `.Values.xxx` | 최상위 레벨 값 |
| range 안 | `.xxx` | 현재 반복 항목의 값 |
| range 안에서 최상위 | `$.Values.xxx` | $ = 루트 컨텍스트 |
| 중첩 객체 | `.parent.child` | 점(.)으로 연결 |
| 배열 | `range .Values.array` | range로 반복 |

## 실전 예시

### values.yaml
```yaml
namespace: my-namespace

envoyProxies:
  - name: proxy-1
    provider:
      type: Kubernetes
      kubernetes:
        envoyService:
          type: LoadBalancer
          annotations:
            key1: value1
            key2: value2
```

### 템플릿
```yaml
{{- range .Values.envoyProxies }}
---
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: {{ .name }}                           # proxy-1
  namespace: {{ $.Values.namespace }}         # my-namespace
spec:
  provider:
    type: {{ .provider.type }}                # Kubernetes
    {{- if eq .provider.type "Kubernetes" }}
    kubernetes:
      envoyService:
        type: {{ .provider.kubernetes.envoyService.type }}  # LoadBalancer
        {{- with .provider.kubernetes.envoyService.annotations }}
        annotations:
          {{- toYaml . | nindent 10 }}        # key1: value1, key2: value2
        {{- end }}
    {{- end }}
{{- end }}
```

### 최종 결과
```yaml
---
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: proxy-1
  namespace: my-namespace
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyService:
        type: LoadBalancer
        annotations:
          key1: value1
          key2: value2
```

## 자주 사용하는 패턴

### 1. with 블록 (값이 있을 때만 렌더링)
```yaml
{{- with .provider.kubernetes.envoyService.annotations }}
annotations:
  {{- toYaml . | nindent 2 }}
{{- end }}
```

### 2. if 조건문
```yaml
{{- if eq .provider.type "Kubernetes" }}
kubernetes:
  # ...
{{- end }}
```

### 3. 기본값 설정
```yaml
type: {{ .provider.type | default "Kubernetes" }}
```

### 4. 여러 레벨 중첩
```yaml
{{- range .Values.gateways }}
  {{- range .listeners }}
    - name: {{ .name }}
      port: {{ .port }}
  {{- end }}
{{- end }}
```
