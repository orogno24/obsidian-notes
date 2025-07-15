## 🔍 서비스 메시 개념

### 서비스 메시란?

서비스 메시는 마이크로서비스 아키텍처에서 **서비스 간 통신을 제어하고 관리하는 인프라 계층**으로,
각 서비스에 사이드카 프록시(Envoy)를 배치하여 모든 네트워크 트래픽을 가로채고 제어함.

### 쿠버네티스 vs Istio 비교

#### 기본 쿠버네티스만 있을 때:

```
App A → Service → Pod B1, B2, B3 (라운드로빈)
```

#### Istio가 있을 때:

```
App A → Envoy → 고급 라우팅 로직 → Pod B1(v1), B2(v2) 
                                  ↑
                            (카나리 배포: v1=90%, v2=10%)
```

### 핵심 차이점

|기능|쿠버네티스 기본|Istio 추가|
|---|---|---|
|**서비스 디스커버리**|✅ DNS + Service|✅ 동일 + 확장|
|**로드밸런싱**|라운드로빈|다양한 알고리즘|
|**트래픽 분할**|❌|✅ 가중치 기반|
|**회로 차단기**|❌|✅|
|**재시도 정책**|❌|✅|
|**mTLS**|❌|✅|
|**트래픽 관측성**|기본|상세한 메트릭|

### Istio 핵심 기능

- **트래픽 관리**: 서비스 간의 트래픽을 제어하고 정책을 적용하여 안정성을 보장
- **보안**: 서비스 간의 통신 암호화 및 인증, 인가를 통해 보안을 강화
- **장애 처리**: 서비스 실패 시 자동으로 대체 경로를 설정하여 가용성을 확보
- **관측성**: 서비스 간의 호출 정보를 수집하여 성능 모니터링 및 문제 분석에 활용
- **고급 서비스 디스커버리**: 쿠버네티스의 기본 서비스 디스커버리를 확장

> 💡 **핵심 아이디어**:
> 
> - **쿠버네티스**: "어디로 갈지 찾아주는" 기본 내비게이션
> - **Istio**: "어떤 경로로, 얼마나 빠르게, 안전하게 갈지" 결정하는 고급 내비게이션

### Istio 통신 흐름

#### 1. 파드 생성 및 초기화

- 쿠버네티스가 파드 생성 시 Istio가 자동으로 사이드카 컨테이너 주입
- init 컨테이너가 먼저 실행되어 iptables 규칙 설정
- 모든 네트워크 트래픽이 Envoy 프록시로 리다이렉트되도록 구성

#### 2. 설정 정보 동기화

- Envoy 프록시가 istiod(컨트롤 플레인)에 연결
- 서비스 정보, 라우팅 규칙, 엔드포인트 목록, 인증서 수신
- 변경사항 실시간 업데이트

#### 3. 서비스 A → 서비스 B 통신 과정

1. **요청 시작**: 서비스 A의 애플리케이션이 서비스 B로 요청
2. **트래픽 인터셉트**: iptables가 요청을 A의 Envoy로 리다이렉트
3. **라우팅 결정**: Envoy가 서비스 레지스트리 조회 및 라우팅 규칙 적용
4. **보안 통신**: A와 B의 Envoy 간 mTLS 암호화 연결 수립
5. **정책 적용**: B의 Envoy가 인증/권한 정책 적용
6. **요청 전달**: B의 Envoy가 요청을 B의 애플리케이션으로 전달
7. **응답 반환**: 응답이 동일한 경로로 반대 방향으로 이동

### Kong + Istio 통합 구성

#### 트래픽 흐름

```
외부 요청 → API Gateway → Kong 라우팅 → Envoy 인터셉트 → 최종 라우팅
```

#### 역할 분담

- **Kong**: 어떤 서비스로 갈지 결정 (서비스 선택)
    - URL 경로 기반 라우팅 (`/users/*` → user-service)
    - 사용자 인증/인가 (JWT, OAuth 등)
    - API 키 관리 및 할당량 제어
    - 속도 제한 및 트래픽 조절
    
- **Envoy**: 선택된 서비스의 어떤 인스턴스로 갈지 결정 (인스턴스 선택)
    - 트래픽 분할 (v1=80%, v2=20%)
    - 로드밸런싱 알고리즘 적용
    - 회로 차단기, 재시도 정책

---

## ⚙️ Istio 설치 및 설정

### 1. 기본 설치

```bash
# Istio 최신 버전 다운로드
curl -L https://istio.io/downloadIstio | sh -

# Istio 디렉토리로 이동
cd istio-<version>/

# istioctl을 PATH에 추가
export PATH=$PWD/bin:$PATH

# 설치 확인
istioctl version
```

### 2. IstioOperator 설정 (Jaeger 트레이싱 포함)

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
# Istio 설치
istioctl install -f ./tracing.yaml --skip-confirmation

# 기본 애드온 설치
kubectl apply -f samples/addons
```

### 3. Telemetry 설정

```yaml
# telemetry.yaml
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

```bash
kubectl apply -f telemetry.yaml
```

### 4. 사이드카 주입 설정

```bash
# 네임스페이스에 사이드카 자동 주입 활성화
kubectl label namespace default istio-injection=enabled

# 확인
kubectl get namespace -L istio-injection
```

### 5. Gateway 및 VirtualService 예제

```yaml
# gateway-virtualservice.yaml
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

## 📊 관측성 도구 구성

### Jaeger (분산 트레이싱)

#### 1. 사전 요구사항: cert-manager 설치

```bash
# CRD 설치
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.crds.yaml

# cert-manager 설치
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml

# 설치 확인
kubectl get pods -n cert-manager
```

#### 2. Jaeger Operator 설치

```bash
# 네임스페이스 생성
kubectl create ns observability

# Jaeger Operator 설치
kubectl apply -f https://github.com/jaegertracing/jaeger-operator/releases/latest/download/jaeger-operator.yaml
```

#### 3. Elasticsearch 설치 (저장소)

```bash
# ECK Operator 설치
kubectl apply -f https://download.elastic.co/downloads/eck/2.12.1/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.12.1/operator.yaml

# 스토리지 준비
sudo mkdir -p /mnt/data/elasticsearch
sudo chown -R 1000:1000 /mnt/data/elasticsearch
```

```yaml
# elasticsearch-storage.yaml
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
---
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

#### 4. Jaeger 프로덕션 인스턴스 생성

```bash
# Elasticsearch 패스워드 확인
kubectl get secret quickstart-es-elastic-user -n observability -o jsonpath="{.data.elastic}" | base64 -d
```

```yaml
# jaeger-production.yaml
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
        password: <YOUR_ELASTICSEARCH_PASSWORD>
        tls:
          skip-host-verify: true
```

```
kubectl create -f jaeger-production.yaml
```

#### 5. Jaeger 접근을 위한 Ingress

```yaml
# jaeger-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jaeger-query-ingress
  namespace: observability
spec:
  ingressClassName: nginx
  rules:
  - host: jaeger.your-domain.com
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

#### 6. 배포 상태 확인

```
kubectl get pods -n cert-manager
kubectl get pods -n observability
kubectl get ingress -n observability
```


---

### Kiali (서비스 메시 시각화)

#### 1. Kiali Operator 설치

```bash
# Helm 저장소 추가
helm repo add kiali https://kiali.org/helm-charts
helm repo update kiali

# Kiali Operator 설치
helm install --namespace kiali-operator --create-namespace kiali-operator kiali/kiali-operator
```

#### 2. Kiali 인스턴스 생성

```yaml
# kiali-instance.yaml
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

#### 3. Kiali Ingress

```yaml
# kiali-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kiali-ingress
  namespace: istio-system
spec:
  ingressClassName: nginx
  rules:
  - host: kiali.your-domain.com
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

## 🚀 실습 및 예제

### Spring Boot 애플리케이션 트레이싱 설정

#### 1. 의존성 추가

**Gradle:**

```gradle
dependencyManagement {
    imports {
        mavenBom("io.opentelemetry.instrumentation:opentelemetry-instrumentation-bom:2.14.0")
    }
}

implementation 'io.opentelemetry.instrumentation:opentelemetry-spring-boot-starter'
implementation 'io.opentelemetry:opentelemetry-exporter-jaeger:1.34.1'
```

**Maven:**

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

#### 2. 애플리케이션 설정

**application.yml:**

```yaml
otel:
  exporter:
    otlp:
      protocol: grpc
      endpoint: http://jaeger-prod-collector.observability.svc.cluster.local:4317
  traces:
    exporter: otlp
```

**application.properties:**

```properties
otel.exporter.otlp.protocol=grpc
otel.exporter.otlp.endpoint=http://jaeger-prod-collector.observability.svc.cluster.local:4317
otel.traces.exporter=otlp
```

#### 3. RestTemplate 설정

```java
@Configuration
public class RestTemplateConfig {
    
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

### Bookinfo 샘플 애플리케이션 배포

```bash
# Gateway API CRD 설치 (필요시)
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
{ kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.2.1" | kubectl apply -f -; }

# Bookinfo 애플리케이션 배포
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

# 동작 확인
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- \
curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

---

## 🔧 트러블슈팅

### Kiali 관련 이슈

#### 1. Jaeger Collector 연결이 빨간색으로 표시되는 문제

**원인:** gRPC 연결의 정상적인 종료를 Kiali가 오류로 해석 **해결책:** "Sent Messages" 메트릭이 초록색이면 정상 동작

#### 2. Grafana 연결 오류

```bash
# Kiali ConfigMap 수정
kubectl edit configmap kiali -n istio-system

# grafana 섹션에 추가:
# url: https://grafana.your-domain.com
# internal_url: http://grafana.istio-system:3000

# Kiali 재시작
kubectl rollout restart deployment kiali -n istio-system
```

### 실무 환경 설정

#### Prometheus 연동

```yaml
# Kiali ConfigMap에서 Prometheus 설정
external_services:
  prometheus:
    url: http://prometheus-stack-kube-prom-prometheus.metrics-monitoring:9090
  custom_dashboards:
    enabled: true
  istio:
    root_namespace: istio-system
  tracing:
    enabled: false
```

#### ServiceMonitor 생성

```yaml
# prometheus-servicemonitor.yaml
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

## 📝 유용한 명령어

### 사이드카 주입 관리

```bash
# 네임스페이스에 자동 주입 활성화
kubectl label namespace default istio-injection=enabled

# 수동 사이드카 주입
istioctl kube-inject -f deployment.yaml -o deployment-injected.yaml

# 특정 파드에서 사이드카 주입 비활성화 (Deployment에 추가)
# annotations:
#   sidecar.istio.io/inject: "false"
```

### Istio 상태 확인

```bash
# Istio 버전 확인
istioctl version

# 프록시 상태 확인
istioctl proxy-status

# 구성 검증
istioctl analyze

# 트래픽 정책 확인
kubectl get virtualservice,gateway,destinationrule -A

# 특정 서비스의 프록시 설정 확인
istioctl proxy-config cluster <pod-name> -n <namespace>
```

### 네트워크 정책 변경(노드포트 접속용)

```bash
# Ingress Gateway를 NodePort로 변경
kubectl patch svc istio-ingressgateway -n istio-system \
-p '{"spec": {"type": "NodePort"}}'

# Kiali를 NodePort로 변경
kubectl patch -n istio-system svc kiali -p '{"spec": {"type": "NodePort"}}'

# Jaeger를 NodePort로 변경
kubectl patch -n istio-system svc tracing -p '{"spec": {"type": "NodePort"}}'
```

### 디버깅

```bash
# 특정 파드의 Envoy 로그 확인
kubectl logs <pod-name> -c istio-proxy -n <namespace>

# Envoy 어드민 인터페이스 접근
kubectl port-forward <pod-name> 15000:15000 -n <namespace>
# 브라우저에서 http://localhost:15000 접근

# 트레이스 생성 테스트
for i in {1..10}; do
  curl -H "Host: bookinfo.com" http://<gateway-ip>/productpage
done
```

---