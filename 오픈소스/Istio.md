## 서비스 메시란?

서비스 메시는 마이크로서비스 아키텍처에서 서비스 간 통신을 제어하고 관리하는 인프라 계층입니다. 각 서비스에 사이드카 프록시를 배치하여 모든 네트워크 트래픽을 가로채고 제어하며, 복잡한 마이크로서비스 환경에서 일관성 있는 관리를 제공합니다.

### 핵심 기능

- **트래픽 관리**: 서비스 간의 트래픽을 제어하고 정책을 적용하여 안정성을 보장
- **보안**: 서비스 간의 통신 암호화 및 인증, 인가를 통해 보안을 강화
- **장애 처리**: 서비스 실패 시 자동으로 대체 경로를 설정하여 가용성을 확보
- **관측성**: 서비스 간의 호출 정보를 수집하여 성능 모니터링 및 문제 분석에 활용
- **서비스 디스커버리**: 쿠버네티스의 기본 서비스 디스커버리를 확장
### 기본 쿠버네티스만 있을 때:

```
App A → Service → Pod B1, B2, B3 (라운드로빈)
```

### 이스티오가 있을 때:

```
App A → Envoy → 고급 라우팅 로직 → Pod B1(v1), B2(v2) 
                                  ↑
                            (카나리 배포: v1=90%, v2=10%)
```

## 핵심 차이점

| 기능               | 쿠버네티스 기본        | 이스티오 추가   |
| ---------------- | --------------- | --------- |
| **기본 서비스 디스커버리** | ✅ DNS + Service | ✅ 동일 + 확장 |
| **로드밸런싱**        | 라운드로빈           | 다양한 알고리즘  |
| **트래픽 분할**       | ❌               | ✅ 가중치 기반  |
| **회로 차단기**       | ❌               | ✅         |
| **재시도 정책**       | ❌               | ✅         |
| **mTLS**         | ❌               | ✅         |
| **트래픽 관측성**      | 기본              | 상세한 메트릭   |

- **쿠버네티스**: "어디로 갈지 찾아주는" 기본 내비게이션
- **이스티오**: "어떤 경로로, 얼마나 빠르게, 안전하게 갈지" 결정하는 고급 내비게이션

---

## 통신 흐름

### 1. 파드 생성 및 초기화

- 쿠버네티스가 파드 생성 시 이스티오가 자동으로 사이드카 컨테이너 주입
- init 컨테이너가 먼저 실행되어 iptables 규칙 설정
- 모든 네트워크 트래픽이 Envoy 프록시로 리다이렉트되도록 구성

### 2. 설정 정보 동기화

- Envoy 프록시가 istiod(컨트롤 플레인)에 연결
- 서비스 정보, 라우팅 규칙, 엔드포인트 목록, 인증서 수신
- 변경사항 실시간 업데이트

### 3. 서비스 A → 서비스 B 통신 과정

1. **요청 시작**: 서비스 A의 애플리케이션이 서비스 B로 요청
2. **트래픽 인터셉트**: iptables가 요청을 A의 Envoy로 리다이렉트
3. **라우팅 결정**: Envoy가 서비스 레지스트리 조회 및 라우팅 규칙 적용
4. **보안 통신**: A와 B의 Envoy 간 mTLS 암호화 연결 수립
5. **정책 적용**: B의 Envoy가 인증/권한 정책 적용
6. **요청 전달**: B의 Envoy가 요청을 B의 애플리케이션으로 전달
7. **응답 반환**: 응답이 동일한 경로로 반대 방향으로 이동

### 4. Kong + Istio 통합 구성

#### 트래픽 흐름

```
외부 요청 → API Gateway 파드 → Kong 라우팅 → Envoy 인터셉트 → 최종 라우팅
```

- **Kong 라우팅** (첫 번째 결정):
    - URL `/users`를 확인 → `user-service`로 라우팅 결정
    - Kong이 `http://user-service:8080/users`로 요청 전달 시도

- **Envoy 인터셉트** (두 번째 결정):
    - Kong의 아웃바운드 요청을 인터셉트
    - `user-service`에 대한 Istio 라우팅 규칙 적용
    - 예: `user-service` v1에 80%, v2에 20% 트래픽 분배

- **최종 라우팅**: Envoy가 선택한 구체적인 엔드포인트로 요청 전달

#### 역할 분담

- **Kong**: 어떤 서비스로 갈지 결정 (서비스 선택)
- **Envoy**: 선택된 서비스의 어떤 인스턴스로 갈지 결정 (인스턴스 선택)

#### Kong의 역할

- **외부 API 관리 담당**
    - 사용자 인증/인가 (JWT, OAuth, 기본 인증 등)
    - API 키 관리 및 할당량 제어
    - 속도 제한 및 트래픽 조절
    - API 라이프사이클 관리
- **라우팅 특징**
    - URL 경로 기반 라우팅 (예: `/users/*` → user-service)
    - 외부 트래픽의 첫 진입점 역할
- **부가 기능**
    - API 문서화 및 개발자 포털
    - Kong Manager를 통한 GUI 관리 인터페이스

---

## Istio 설치

### 기본 설치

```bash
# Istio 최신 버전 다운로드 및 설치
curl -L https://istio.io/downloadIstio | sh -

# 다운로드된 Istio 디렉토리로 이동 (버전 확인 후 수정)
cd istio-<해당 istio 버전>/

# istioctl 명령어를 PATH에 등록
export PATH=$PWD/bin:$PATH
```

### IstioOperator 설정 (Jaeger 트레이싱 포함)

```yaml
# tracing.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    enableTracing: true
    defaultConfig:
      tracing: {}
    extensionProviders:
    - name: jaeger
      opentelemetry:
        port: 4317
        service: jaeger-collector.istio-system.svc.cluster.local
```

```bash
# IstioOperator 설정으로 Istio 설치
istioctl install -f ./tracing.yaml --skip-confirmation

# 기본 애드온 설치 (Kiali, Jaeger 등)
kubectl apply -f samples/addons
```

### Telemetry 설정

```yaml
# Telemetry CR 생성 - 트레이싱 provider로 jaeger 설정
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  tracing:
    - providers:
        - name: jaeger
```

### 샘플 애플리케이션 배포 (Bookinfo)

```bash
# Gateway API CRD 설치 (필요 시)
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
{ kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.2.1" | kubectl apply -f -; }

# 사이드카 주입 설정
kubectl label namespace default istio-injection=enabled

# Bookinfo 샘플 애플리케이션 배포
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

### Gateway 및 VirtualService 설정

```yaml
# bookinfo.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 8080
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```

---

## Jaeger Production 구성

### 1. cert-manager 설치

```bash
# CRD 설치
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.crds.yaml

# cert-manager 구성 요소 설치
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml

# 설치 확인
kubectl get pods -n cert-manager
```

### 2. Jaeger Operator 설치

```bash
kubectl create ns observability
kubectl apply -f https://github.com/jaegertracing/jaeger-operator/releases/latest/download/jaeger-operator.yaml
```

### 3. Elasticsearch 설치 (ECK)

```bash
kubectl apply -f https://download.elastic.co/downloads/eck/2.12.1/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.12.1/operator.yaml
```

#### 스토리지 준비

```bash
# 디렉토리 생성 및 권한 설정
sudo mkdir -p /mnt/data/elasticsearch
sudo chown -R 1000:1000 /mnt/data/elasticsearch
```

```yaml
# local-es-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-es-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  hostPath:
    path: /mnt/data/elasticsearch
```

#### Elasticsearch 인스턴스 생성

```yaml
# elasticsearch.yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart
  namespace: observability
spec:
  version: 8.12.2
  nodeSets:
  - name: default
    count: 1
    config:
      node.store.allow_mmap: false
```

### 4. Jaeger 인스턴스 생성

#### Elasticsearch 패스워드 확인

```bash
kubectl get secret quickstart-es-elastic-user -n observability -o jsonpath="{.data.elastic}" | base64 -d
```

```yaml
# jaeger-prod.yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-prod
  namespace: observability
spec:
  strategy: production
  collector:
    replicas: 2
  query:
    replicas: 2
  agent:
    strategy: DaemonSet
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: https://quickstart-es-http.observability.svc.cluster.local:9200
        username: elastic
        password: <복호화한_패스워드>
        tls:
          skip-host-verify: true
```

### 5. Jaeger 인그레스 설정

```yaml
# jaeger-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jaeger-query-ingress
  namespace: observability
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  ingressClassName: nginx
  rules:
  - host: jaeger.dev.eris.go.kr
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: jaeger-prod-query
            port:
              number: 16686
```

---

## Kiali 설치 및 구성

### 1. Kiali Operator 설치

```bash
# Kiali Helm 저장소 추가
helm repo add kiali https://kiali.org/helm-charts
helm repo update kiali

# Kiali Operator 설치
helm install --namespace kiali-operator --create-namespace kiali-operator kiali/kiali-operator
```

### 2. Kiali 서버 배포

```yaml
# kiali.yaml
apiVersion: kiali.io/v1alpha1
kind: Kiali
metadata:
  name: kiali
  namespace: istio-system
spec:
  auth:
    strategy: anonymous
  deployment:
    cluster_wide_access: false
    discovery_selectors:
      default:
      - matchLabels:
          kubernetes.io/metadata.name: default
    view_only_mode: false
  server:
    web_root: "/kiali"
```

### 3. Kiali 인그레스 설정

```yaml
# kiali-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kiali-ingress
  namespace: istio-system
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  ingressClassName: nginx
  rules:
  - host: kiali.dev.gcp.go.kr
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kiali
            port:
              number: 20001
```

---

## 애플리케이션 트레이싱 설정

### Spring Boot 애플리케이션 설정

#### application.yml

```yaml
otel:
  exporter:
    otlp:
      protocol: grpc
      endpoint: http://jaeger-prod-collector.observability.svc.cluster.local:4317
  traces:
    exporter: otlp
```

#### application.properties

```properties
otel.exporter.otlp.protocol=grpc
otel.exporter.otlp.endpoint=http://jaeger-prod-collector.observability.svc.cluster.local:4317
otel.traces.exporter=otlp
```

#### Gradle 의존성

```gradle
dependencyManagement {
    imports {
        mavenBom("io.opentelemetry.instrumentation:opentelemetry-instrumentation-bom:2.14.0")
    }
}

implementation 'io.opentelemetry.instrumentation:opentelemetry-spring-boot-starter'
implementation 'io.opentelemetry:opentelemetry-exporter-jaeger:1.34.1'
```

#### Maven 의존성

```xml
<dependencyManagement>
 <dependencies>
 <dependency>
 <groupId>io.opentelemetry.instrumentation</groupId>
 <artifactId>opentelemetry-instrumentation-bom</artifactId>
 <version>2.14.0</version>
 <type>pom</type>
 <scope>import</scope>
 </dependency>
 </dependencies>
</dependencyManagement>

<dependencies>
 <dependency>
 <groupId>io.opentelemetry.instrumentation</groupId>
 <artifactId>opentelemetry-spring-boot-starter</artifactId>
 </dependency>
 <dependency>
 <groupId>io.opentelemetry</groupId>
 <artifactId>opentelemetry-exporter-jaeger</artifactId>
 <version>1.34.1</version>
 </dependency>
</dependencies>
```

#### RestTemplate 설정

```java
@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

---

## 트러블슈팅

### Kiali 관련 이슈

#### Jaeger Collector 연결이 빨간색으로 표시되는 문제

**원인:**

- gRPC 통신에서 클라이언트가 데이터를 전송한 후 연결을 정상적으로 종료하면 "EOF" 메시지가 발생
- Kiali는 이러한 정상적인 연결 종료를 오류로 해석하여 빨간색으로 표시
- 실제 기능적인 문제가 아닌 Kiali의 시각화 방식에 따른 현상

**해결책:**

- "Sent Messages" 메트릭이 초록색으로 표시되면 정상 동작
- Jaeger UI에서 트레이스 데이터가 정상적으로 표시되는지 확인

#### Grafana 연결 오류 해결

```bash
# Kiali ConfigMap 수정
kubectl edit configmap kiali -n istio-system
```

```yaml
# grafana 항목에 url 추가
url: https://grafana.dev.gcp.go.kr:30191/
internal_url: http://grafana.istio-system:3000
```

```bash
# Kiali 재시작
kubectl rollout restart deployment kiali -n istio-system
kubectl rollout status deployment kiali -n istio-system
```

### 실무 환경 설정

#### Prometheus 연결

```yaml
# Kiali ConfigMap에서 Prometheus 설정
external_services:
  prometheus:
    url: http://prometheus-stack-kube-prom-prometheus.metrics-monitoring:9090
  custom_dashboards:
    enabled: true
  istio:
    root_namespace: op-trace
  tracing:
    enabled: false
  istio_namespace: istio-system
```

#### ServiceMonitor 생성

```yaml
# istio-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: istio-component-monitor
  namespace: metrics-monitoring
  labels:
    release: prometheus-stack
spec:
  selector:
    matchExpressions:
    - {key: istio, operator: In, values: [pilot]}
  namespaceSelector:
    matchNames:
    - istio-system
  endpoints:
  - port: http-monitoring
    interval: 15s
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: envoy-stats-monitor
  namespace: metrics-monitoring
  labels:
    release: prometheus-stack
spec:
  selector:
    matchExpressions:
    - {key: istio-prometheus-ignore, operator: DoesNotExist}
  namespaceSelector:
    any: true
  endpoints:
  - path: /stats/prometheus
    targetPort: 15090
    interval: 15s
```

---

## 유용한 명령어

### 사이드카 주입 관리

```bash
# 네임스페이스에 사이드카 주입 활성화
kubectl label namespace default istio-injection=enabled

# 특정 파드에 사이드카 주입
istioctl kube-inject -f deployment.yaml -o deployment-injected.yaml

# 사이드카 주입 비활성화 (파드 레벨)
# Deployment에 다음 어노테이션 추가:
# sidecar.istio.io/inject: "false"
```

### 서비스 상태 확인

```bash
# Istio 설치 확인
istioctl version

# 프록시 상태 확인
istioctl proxy-status

# 구성 검증
istioctl analyze

# 트래픽 정책 확인
kubectl get virtualservice,gateway,destinationrule -A
```

### 네트워크 정책 변경

```bash
# Ingress Gateway 서비스 타입 변경
kubectl patch svc istio-ingressgateway -n istio-system \
-p '{"spec": {"type": "NodePort"}}'

# Kiali 서비스 NodePort로 변경
kubectl patch -n istio-system svc kiali -p '{"spec": {"type": "NodePort"}}'

# Jaeger 서비스 NodePort로 변경
kubectl patch -n istio-system svc tracing -p '{"spec": {"type": "NodePort"}}'
```

---
