# Helm Values 재사용 가이드

## 3가지 방법 비교

| 방법 | 난이도 | 유지보수성 | 유연성 | 추천도 |
|------|--------|-----------|--------|--------|
| 1. YAML Anchors | ⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 2. 구조화된 설정 | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 3. 통합 설정 + 자동 생성 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

---

## 방법 1: YAML Anchors (가장 간단!)

### 장점
- ✅ 템플릿 수정 불필요
- ✅ 표준 YAML 기능
- ✅ 즉시 적용 가능
- ✅ 이해하기 쉬움

### 단점
- ⚠️ 같은 파일 내에서만 참조 가능
- ⚠️ 복잡한 구조에는 제한적

### 사용법

```yaml
# values.yaml
proxyNames:
  external: &externalProxyName custom-proxy-config
  internal: &internalProxyName custom-proxy-config-internal

envoyProxies:
  - name: *externalProxyName  # ← 앵커 참조
    provider:
      type: Kubernetes

gatewayClasses:
  - name: eg
    parametersRef:
      name: *externalProxyName  # ← 동일한 값 재사용!
```

### 앵커 문법

```yaml
# 앵커 정의
key: &anchorName value

# 앵커 참조
otherKey: *anchorName

# 결과
key: value
otherKey: value  # 동일한 값
```

### 실전 예시

```yaml
# 공통 annotation 정의
commonAnnotations: &awsNLBAnnotations
  service.beta.kubernetes.io/aws-load-balancer-type: external
  service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip

envoyProxies:
  - name: proxy-1
    provider:
      kubernetes:
        envoyService:
          annotations:
            <<: *awsNLBAnnotations  # ← 객체 병합
            service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
  
  - name: proxy-2
    provider:
      kubernetes:
        envoyService:
          annotations:
            <<: *awsNLBAnnotations  # ← 동일한 base annotations
            service.beta.kubernetes.io/aws-load-balancer-scheme: internal
```

---

## 방법 2: 구조화된 설정

### 장점
- ✅ 설정이 논리적으로 그룹화
- ✅ 문서화가 쉬움
- ✅ 관련 설정을 한 곳에서 관리

### 단점
- ⚠️ 여전히 값을 여러 곳에 입력해야 함
- ⚠️ 완전한 자동화는 아님

### 사용법

```yaml
# values.yaml
# 프록시 설정을 객체로 구조화
proxies:
  external:
    name: custom-proxy-config
    gatewayClassName: eg
    scheme: internet-facing
  
  internal:
    name: custom-proxy-config-internal
    gatewayClassName: eg-internal
    scheme: internal

# 실제 리소스 정의 (위 설정 참고)
envoyProxies:
  - name: custom-proxy-config  # proxies.external.name
    provider:
      kubernetes:
        envoyService:
          annotations:
            scheme: internet-facing  # proxies.external.scheme

gatewayClasses:
  - name: eg  # proxies.external.gatewayClassName
    parametersRef:
      name: custom-proxy-config  # proxies.external.name
```

### 장점: 문서화 효과

```yaml
# 한눈에 전체 구조 파악 가능
proxies:
  external:
    name: custom-proxy-config
    gatewayClassName: eg
    gateway: eg
    scheme: internet-facing
  
  internal:
    name: custom-proxy-config-internal
    gatewayClassName: eg-internal
    gateway: eg-internal
    scheme: internal
```

---

## 방법 3: 통합 설정 + 자동 생성 (최고!)

### 장점
- ✅ 한 곳에서 모든 설정 관리
- ✅ 중복 제거
- ✅ 실수 방지
- ✅ 확장성 최고

### 단점
- ⚠️ 템플릿 수정 필요
- ⚠️ 초기 설정이 복잡

### 구조

```yaml
# values.yaml - 단일 소스
proxyConfigs:
  - name: custom-proxy-config
    gatewayClassName: eg
    provider:
      type: Kubernetes
      kubernetes:
        envoyService:
          type: LoadBalancer
          annotations:
            service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
```

```yaml
# templates/envoyproxy.yaml - 자동 생성
{{- range .Values.proxyConfigs }}
---
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: {{ .name }}  # ← proxyConfigs에서 자동으로 가져옴
spec:
  provider:
    {{- toYaml .provider | nindent 4 }}
{{- end }}
```

```yaml
# templates/gatewayclass.yaml - 자동 생성
{{- range .Values.proxyConfigs }}
---
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: {{ .gatewayClassName }}  # ← proxyConfigs에서 자동으로 가져옴
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  parametersRef:
    name: {{ .name }}  # ← 자동으로 매칭!
{{- end }}
```

### 새 프록시 추가가 쉬움!

```yaml
# values.yaml에 하나만 추가하면
proxyConfigs:
  - name: custom-proxy-config
    gatewayClassName: eg
    provider: ...
  
  - name: custom-proxy-config-internal
    gatewayClassName: eg-internal
    provider: ...
  
  # 새 프록시 추가 - 이것만 추가하면 끝!
  - name: custom-proxy-config-dmz
    gatewayClassName: eg-dmz
    provider: ...

# EnvoyProxy, GatewayClass가 자동으로 생성됨!
```

---

## 실전 권장 사항

### 소규모 프로젝트 (1-2개 프록시)
→ **방법 1: YAML Anchors** 사용

```yaml
proxyNames:
  main: &mainProxy custom-proxy-config

envoyProxies:
  - name: *mainProxy
    provider: ...

gatewayClasses:
  - name: eg
    parametersRef:
      name: *mainProxy
```

### 중규모 프로젝트 (3-5개 프록시)
→ **방법 1 + 방법 2 조합**

```yaml
# 공통 설정
common: &commonAnnotations
  service.beta.kubernetes.io/aws-load-balancer-type: external
  service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip

# 프록시 이름
proxyNames:
  external: &externalProxy custom-proxy-config
  internal: &internalProxy custom-proxy-config-internal

envoyProxies:
  - name: *externalProxy
    provider:
      kubernetes:
        envoyService:
          annotations:
            <<: *commonAnnotations
            service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing

gatewayClasses:
  - name: eg
    parametersRef:
      name: *externalProxy
```

### 대규모 프로젝트 (6개 이상 프록시)
→ **방법 3: 통합 설정** 사용

```yaml
proxyConfigs:
  - name: proxy-1
    gatewayClassName: class-1
    provider: ...
  - name: proxy-2
    gatewayClassName: class-2
    provider: ...
  # ... 계속 추가
```

---

## 마이그레이션 가이드

### 현재 구조 → YAML Anchors

```yaml
# Before
envoyProxies:
  - name: custom-proxy-config
    provider: ...

gatewayClasses:
  - name: eg
    parametersRef:
      name: custom-proxy-config  # 중복!

# After
proxyNames:
  main: &mainProxy custom-proxy-config

envoyProxies:
  - name: *mainProxy
    provider: ...

gatewayClasses:
  - name: eg
    parametersRef:
      name: *mainProxy  # 재사용!
```

### 현재 구조 → 통합 설정

1. `values-auto-match.yaml` 참고하여 `proxyConfigs` 생성
2. `templates/envoyproxy-auto.yaml.example` → `templates/envoyproxy.yaml`로 교체
3. `templates/gatewayclass-auto.yaml.example` → `templates/gatewayclass.yaml`로 교체
4. 테스트: `helm template gateway-system ./gateway-system -f values-auto-match.yaml`

---

## 테스트 방법

```bash
# 1. 템플릿 렌더링 확인
helm template gateway-system ./gateway-system -f values-with-anchors.yaml

# 2. Lint 체크
helm lint ./gateway-system -f values-with-anchors.yaml

# 3. Dry-run 설치
helm install gateway-system ./gateway-system \
  -f values-with-anchors.yaml \
  --dry-run --debug

# 4. 실제 설치
helm install gateway-system ./gateway-system \
  -f values-with-anchors.yaml
```

---

## 추천 순서

1. **지금 당장**: YAML Anchors 적용 (5분)
2. **여유 있을 때**: 구조화된 설정 추가 (30분)
3. **장기적으로**: 통합 설정으로 마이그레이션 (2시간)

**가장 빠른 해결책: `values-with-anchors.yaml` 사용!**
