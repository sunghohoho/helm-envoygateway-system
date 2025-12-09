# Helm의 with 이해하기

## 1. with의 두 가지 역할

### 역할 1: 값이 있을 때만 실행 (if와 비슷)
### 역할 2: 컨텍스트(.) 변경 (range와 비슷)

## 2. 기본 문법

```yaml
{{- with .parametersRef }}
  # parametersRef가 존재하면 이 블록 실행
  # 그리고 . = parametersRef로 컨텍스트 변경
{{- end }}
```

## 3. 실제 예시로 이해하기

### values.yaml
```yaml
gatewayClasses:
  # parametersRef가 있는 경우
  - name: eg
    controllerName: gateway.envoyproxy.io/gatewayclass-controller
    parametersRef:
      group: gateway.envoyproxy.io
      kind: EnvoyProxy
      name: custom-proxy-config
  
  # parametersRef가 없는 경우
  - name: simple-gateway
    controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

### 템플릿 (with 사용)
```yaml
{{- range .Values.gatewayClasses }}
---
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: {{ .name }}
spec:
  controllerName: {{ .controllerName }}
  {{- with .parametersRef }}
  parametersRef:
    group: {{ .group }}
    kind: {{ .kind }}
    name: {{ .name }}
  {{- end }}
{{- end }}
```

### 결과

#### 첫 번째 GatewayClass (parametersRef 있음)
```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  parametersRef:                    # ✅ 생성됨
    group: gateway.envoyproxy.io
    kind: EnvoyProxy
    name: custom-proxy-config
```

#### 두 번째 GatewayClass (parametersRef 없음)
```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: simple-gateway
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  # parametersRef 블록이 아예 생성되지 않음 ✅
```

## 4. with의 컨텍스트 변경

### with 사용 전
```yaml
{{- range .Values.gatewayClasses }}
spec:
  controllerName: {{ .controllerName }}
  parametersRef:
    group: {{ .parametersRef.group }}           # 반복적으로 .parametersRef 작성
    kind: {{ .parametersRef.kind }}
    name: {{ .parametersRef.name }}
{{- end }}
```

### with 사용 후 (간결함!)
```yaml
{{- range .Values.gatewayClasses }}
spec:
  controllerName: {{ .controllerName }}
  {{- with .parametersRef }}
  parametersRef:
    group: {{ .group }}                         # .parametersRef.group → .group
    kind: {{ .kind }}                           # .parametersRef.kind → .kind
    name: {{ .name }}                           # .parametersRef.name → .name
  {{- end }}
{{- end }}
```

## 5. with 안에서 컨텍스트 변화

```yaml
{{- range .Values.gatewayClasses }}
  # 현재 컨텍스트: . = 현재 gatewayClass 항목
  
  name: {{ .name }}                             # ✅ 현재 gatewayClass의 name
  
  {{- with .parametersRef }}
    # 컨텍스트 변경: . = parametersRef 객체
    
    group: {{ .group }}                         # ✅ parametersRef.group
    kind: {{ .kind }}                           # ✅ parametersRef.kind
    
    # 주의! 이제 .name은 parametersRef.name을 의미
    name: {{ .name }}                           # ✅ parametersRef.name
    
    # gatewayClass의 name이 필요하면?
    # name: {{ .name }}                         # ❌ 에러! parametersRef.name이 됨
    # name: {{ $.name }}                        # ❌ 에러! $ = 최상위 루트
  {{- end }}
  
  # with 블록 밖: 다시 원래 컨텍스트로 복귀
  name: {{ .name }}                             # ✅ 다시 gatewayClass의 name
{{- end }}
```

## 6. with vs if 비교

### if 사용 (컨텍스트 변경 없음)
```yaml
{{- if .parametersRef }}
parametersRef:
  group: {{ .parametersRef.group }}             # 반복적으로 .parametersRef 필요
  kind: {{ .parametersRef.kind }}
  name: {{ .parametersRef.name }}
{{- end }}
```

### with 사용 (컨텍스트 변경)
```yaml
{{- with .parametersRef }}
parametersRef:
  group: {{ .group }}                           # 간결함!
  kind: {{ .kind }}
  name: {{ .name }}
{{- end }}
```

## 7. 실전 예시

### annotations가 있을 때만 추가
```yaml
# values.yaml
envoyProxies:
  - name: proxy-1
    provider:
      kubernetes:
        envoyService:
          type: LoadBalancer
          annotations:
            key1: value1
            key2: value2
  
  - name: proxy-2
    provider:
      kubernetes:
        envoyService:
          type: LoadBalancer
          # annotations 없음
```

```yaml
# 템플릿
{{- range .Values.envoyProxies }}
spec:
  provider:
    kubernetes:
      envoyService:
        type: {{ .provider.kubernetes.envoyService.type }}
        {{- with .provider.kubernetes.envoyService.annotations }}
        annotations:
          {{- toYaml . | nindent 10 }}
        {{- end }}
{{- end }}
```

```yaml
# 결과 (proxy-1)
spec:
  provider:
    kubernetes:
      envoyService:
        type: LoadBalancer
        annotations:                            # ✅ 생성됨
          key1: value1
          key2: value2

# 결과 (proxy-2)
spec:
  provider:
    kubernetes:
      envoyService:
        type: LoadBalancer
        # annotations 블록이 아예 없음 ✅
```

## 8. with에서 상위 컨텍스트 접근

```yaml
{{- range .Values.gatewayClasses }}
metadata:
  name: {{ .name }}                             # gatewayClass의 name
spec:
  {{- with .parametersRef }}
  parametersRef:
    name: {{ .name }}                           # parametersRef의 name
    namespace: {{ $.Values.namespace }}         # ✅ 최상위 namespace
  {{- end }}
{{- end }}
```

## 9. 중첩 with

```yaml
{{- with .provider }}
  {{- with .kubernetes }}
    {{- with .envoyService }}
      type: {{ .type }}
      {{- with .annotations }}
      annotations:
        {{- toYaml . | nindent 2 }}
      {{- end }}
    {{- end }}
  {{- end }}
{{- end }}
```

하지만 이렇게 중첩하면 복잡하므로, 적절히 사용하세요!

## 10. 정리

| 특징 | if | with |
|------|----|----|
| 조건 체크 | ✅ | ✅ |
| 컨텍스트 변경 | ❌ | ✅ |
| 코드 간결성 | 보통 | 높음 |
| 사용 시점 | 단순 조건 | 중첩 객체 접근 |

### 언제 with를 사용하나?

1. **값이 선택적(optional)일 때**
   ```yaml
   {{- with .annotations }}
   annotations:
     {{- toYaml . | nindent 2 }}
   {{- end }}
   ```

2. **중첩된 객체를 반복 접근할 때**
   ```yaml
   {{- with .provider.kubernetes.envoyService }}
   type: {{ .type }}
   port: {{ .port }}
   {{- end }}
   ```

3. **코드를 간결하게 만들고 싶을 때**
   ```yaml
   # Before
   {{ .parametersRef.group }}
   {{ .parametersRef.kind }}
   {{ .parametersRef.name }}
   
   # After
   {{- with .parametersRef }}
   {{ .group }}
   {{ .kind }}
   {{ .name }}
   {{- end }}
   ```

## 11. 프로그래밍 언어 비유

```javascript
// JavaScript
if (gatewayClass.parametersRef) {
  const ref = gatewayClass.parametersRef;  // with와 비슷
  console.log(ref.group);
  console.log(ref.kind);
  console.log(ref.name);
}
```

```python
# Python
if parametersRef := gatewayClass.get('parametersRef'):  # with와 비슷
    print(parametersRef['group'])
    print(parametersRef['kind'])
    print(parametersRef['name'])
```

```yaml
# Helm
{{- with .parametersRef }}
group: {{ .group }}
kind: {{ .kind }}
name: {{ .name }}
{{- end }}
```
