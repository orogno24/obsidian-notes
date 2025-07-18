### Kong이란?

Kong은 **NGINX 기반의 오픈소스 API Gateway**입니다. 마이크로서비스 아키텍처에서 API 트래픽을 중앙 집중적으로 관리하고, 인증, 로깅, 속도 제한 등의 기능을 제공합니다.

### 핵심 특징

- **NGINX + Lua 기반**: 고성능 처리
- **플러그인 시스템**: 다양한 기능 확장 가능
- **RESTful Admin API**: 프로그래밍 방식 관리
- **오픈소스**: 무료 사용 가능 + Enterprise 버전 선택적 사용
- **클라우드 네이티브**: Kubernetes와 완벽 통합

### Kong 구성 요소

- **Kong Gateway**: 실제 API 프록시 역할
- **Kong Admin API**: 관리 인터페이스
- **PostgreSQL/Cassandra**: 설정 저장소
- **Konga**: 웹 기반 관리 UI (선택사항)

---

## ⚙️ Kong 설치

### 1. Helm 저장소 추가 및 Values 파일 준비

```bash
# Kong Helm 저장소 추가
helm repo add kong https://charts.konghq.com
helm repo update

# 기본 values.yaml 파일 생성
helm show values kong/kong > values.yaml
```

### 2. Kong Values 설정

```yaml
# values.yaml 주요 설정
env:
  database: "postgres"  # PostgreSQL 사용

ingressController:
  enabled: false        # Kubernetes Ingress Controller 비활성화
  ingressClass: nginx   # 기존 NGINX Ingress 사용

manager:
  enabled: false        # Kong Manager (Enterprise) 비활성화

admin:
  enabled: true         # Admin API 활성화
  http:
    enabled: true

# PostgreSQL 내장 설정
postgresql:
  enabled: true
  auth:
    username: kong
    database: kong
  image:
    tag: 11.21.0-debian-11-r12  # ★ PostgreSQL 11 버전 사용 (중요)
```

> 💡 **PostgreSQL 11 버전 사용 이유**: Kong과의 호환성을 위해 안정된 버전 사용 권장

### 3. Kong 설치 실행

```bash
# Kong 설치
helm install kong kong/kong -n op-common -f values.yaml

# 설치 확인
kubectl get pods -n op-common
kubectl get svc -n op-common
```

---

## 🎛️ Konga 관리 UI 설치

### 1. PostgreSQL에 Konga용 데이터베이스 생성

#### PostgreSQL 접속

```bash
# PostgreSQL Pod 확인
kubectl get pod -n op-common | grep postgresql

# PostgreSQL에 접속
kubectl exec -it kong-postgresql-0 -n op-common -- psql -U postgres
```

#### PostgreSQL 비밀번호 확인

```bash
# Secret에서 비밀번호 확인
kubectl get secret kong-postgresql -n op-common -o yaml

# Base64 디코딩하여 실제 비밀번호 확인
echo "<base64-encoded-password>" | base64 -d
```

#### Konga용 사용자 및 데이터베이스 생성

```sql
-- PostgreSQL에서 실행
CREATE USER konga WITH PASSWORD 'konga';
CREATE DATABASE konga_database OWNER konga;
GRANT ALL PRIVILEGES ON DATABASE konga_database TO konga;

-- 확인
\l  -- 데이터베이스 목록 확인
\q  -- 종료
```

### 2. Konga Chart 준비

```bash
# Konga Chart 다운로드
git clone https://github.com/pantsel/konga.git
mv konga/charts/konga ./konga-chart
cp konga-chart/values.yaml konga-values.yaml
```

### 3. Konga Values 설정

```yaml
# konga-values.yaml
service:
  type: NodePort      # 또는 ClusterIP (Ingress 사용 시)
  port: 80
  targetPort: 80

config:
  port: 1337
  node_env: development
  konga_hook_timeout: 60000
  db_adapter: postgres
  db_uri: postgres://konga:konga@kong-postgresql:5432/konga_database
  db_pg_schema: public
```

### 4. Konga 설치

```bash
# Konga 설치
helm install konga ./konga-chart -n op-common -f konga-values.yaml

# 설치 확인
kubectl get pods -n op-common | grep konga
```

### 5. Konga Ingress 설정

#### TLS 인증서 생성

```bash
# Self-signed 인증서 생성
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout konga.key -out konga.crt \
  -subj "/CN=konga.dev.gcp.go.kr/O=konga"

# Secret 생성
kubectl create secret tls konga-tls \
  --cert=konga.crt \
  --key=konga.key \
  -n op-common
```

#### Ingress 리소스 생성

```yaml
# konga-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: konga-ingress
  namespace: op-common
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "1024m"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: konga.dev.gcp.go.kr
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: konga
            port:
              number: 80
  tls:
  - hosts:
    - konga.dev.gcp.go.kr
    secretName: konga-tls
```

```bash
kubectl apply -f konga-ingress.yaml
```

### 6. Konga 초기 설정

```bash
# Konga Pod 재시작 (필요 시)
kubectl delete pod -n op-common -l app.kubernetes.io/name=konga

# Konga 접속: https://konga.dev.gcp.go.kr
# 첫 접속 시 관리자 계정 생성 및 Kong 연결 설정
```

---

## 🏗️ 네트워킹 아키텍처

### Kong 네트워크 플로우

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   외부 클라이언트  │────│  NGINX Ingress  │────│   Kong Gateway  │
│                 │    │   Controller    │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                ↓                       ↓
                       ┌─────────────────┐    ┌─────────────────┐
                       │  Kubernetes     │    │  Backend        │
                       │  Services       │    │  Services       │
                       └─────────────────┘    └─────────────────┘
```

### 단일 진입점 패턴 구현

#### 1. 와일드카드 Ingress 설정

```yaml
# kong-unified-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kong-unified-ingress
  namespace: op-common
spec:
  ingressClassName: nginx
  rules:
  - host: "*.dev.gcp.go.kr"  # 모든 하위 도메인을 Kong으로 라우팅
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kong-proxy  # Kong 서비스 이름
            port:
              number: 80
```

#### 2. Kong 내부 라우팅 설정 (Konga UI에서)

**Grafana 서비스 예시:**

```json
// Kong Service 설정
{
  "name": "grafana-service",
  "host": "grafana-service.istio-system.svc.cluster.local",
  "port": 3000,
  "protocol": "http"
}

// Kong Route 설정
{
  "name": "grafana-route",
  "hosts": ["grafana.dev.gcp.go.kr"],
  "service": {
    "id": "grafana-service-id"
  }
}
```

**Kiali 서비스 예시:**

```json
// Kong Service 설정
{
  "name": "kiali-service", 
  "host": "kiali.istio-system.svc.cluster.local",
  "port": 20001,
  "protocol": "http"
}

// Kong Route 설정
{
  "name": "kiali-route",
  "hosts": ["kiali.dev.gcp.go.kr"],
  "service": {
    "id": "kiali-service-id"
  }
}
```

### 트래픽 플로우 예시

```
1. 사용자 접속: grafana.dev.gcp.go.kr
         ↓
2. DNS 해석: 10.107.221.158 (Ingress Controller IP)
         ↓
3. NGINX Ingress: *.dev.gcp.go.kr 규칙에 따라 Kong으로 라우팅
         ↓
4. Kong Gateway: Host 헤더 확인 후 grafana-service로 라우팅
         ↓
5. Backend Service: grafana-service.istio-system.svc.cluster.local:3000
```

---

## 🔄 Ingress vs Kong 역할

### 각 구성 요소의 역할

|구성 요소|역할|책임 범위|
|---|---|---|
|**NGINX Ingress**|클러스터 진입점|• 외부 → 내부 트래픽 라우팅<br>• TLS 종료<br>• 기본 로드밸런싱|
|**Kong Gateway**|API 관리|• 고급 라우팅<br>• 인증/인가<br>• 속도 제한<br>• 로깅/모니터링|
|**Kubernetes Service**|서비스 디스커버리|• Pod 추상화<br>• 내부 로드밸런싱|

### 왜 둘 다 필요한가?

#### 1. Ingress 없이는 불가능

- 외부에서 Kubernetes 클러스터 내부로 접근하는 **유일한 표준 방법**
- 클러스터 외부 IP와 내부 서비스 간의 브릿지 역할
- DNS 매핑의 기준점 제공

#### 2. Kong 없이는 제한적

- 기본 HTTP 라우팅만 가능
- 고급 API 관리 기능 부재
- 마이크로서비스별 정책 적용 어려움

### 협력 구조의 장점

```
외부 요청 → Ingress (진입) → Kong (관리) → Backend (처리)
```

**Ingress**: "문지기" 역할 - 들어올 수 있는 트래픽 구분  
**Kong**: "안내데스크" 역할 - 세부적인 방향 안내 및 서비스 제공  
**Backend**: "실제 업무 담당자" 역할 - 요청 처리

---

## ⚖️ Kong vs 일반 API Gateway 비교

### 상세 기능 비교

|항목|일반 API Gateway|**Kong 특징**|
|---|---|---|
|**배포 형태**|대부분 상용/폐쇄형<br>(AWS API Gateway, Azure APIM)|**오픈소스** + Enterprise 선택 가능|
|**플러그인 시스템**|제한적 또는 비공개|**다양한 공식/커스텀 플러그인**<br>(JWT, Rate Limiting, CORS 등)|
|**확장성**|특정 플랫폼 의존<br>(클라우드 등)|**NGINX 기반, 수평 확장 용이**<br>(Kubernetes 연동 쉬움)|
|**성능**|보통|**고성능 처리**<br>(NGINX + Lua 기반)|
|**관리 편의성**|벤더 종속적인 UI/CLI|**RESTful Admin API**<br>GUI 대시보드 (Konga 등)|
|**인증 및 보안**|OAuth/JWT 제한적 지원|**OAuth 2.0, JWT, ACL, Key Auth**<br>풍부한 보안 기능 내장|
|**멀티테넌시**|보통 별도 설정 필요|Enterprise 에디션에서<br>멀티테넌시 지원|
|**커뮤니티**|제한적|**강력한 오픈소스 커뮤니티**<br>지속적 업데이트|
|**비용**|라이선스/사용량 기반|**무료 오픈소스** + 선택적 Enterprise|

### Kong의 주요 플러그인

#### 인증 플러그인

- **JWT**: JSON Web Token 인증
- **OAuth 2.0**: OAuth 2.0 인증 플로우
- **Key Authentication**: API 키 기반 인증
- **Basic Authentication**: 기본 인증
- **LDAP**: LDAP 통합 인증

#### 보안 플러그인

- **Rate Limiting**: 속도 제한
- **IP Restriction**: IP 기반 접근 제어
- **CORS**: Cross-Origin Resource Sharing
- **Bot Detection**: 봇 탐지 및 차단

#### 트래픽 제어 플러그인

- **Load Balancing**: 로드 밸런싱
- **Canary Release**: 카나리 배포
- **Request/Response Transformer**: 요청/응답 변환

#### 관측성 플러그인

- **Logging**: 다양한 로깅 대상 지원
- **Monitoring**: 메트릭 수집
- **Tracing**: 분산 추적

### 사용 시나리오별 권장

#### Kong 권장 상황

- **마이크로서비스 아키텍처**
- **쿠버네티스 환경**
- **오픈소스 기반 스택 선호**
- **세밀한 커스터마이징 필요**
- **높은 성능 요구사항**

#### 일반 API Gateway 권장 상황

- **단일 클라우드 플랫폼 전용**
- **빠른 초기 구축 필요**
- **관리 복잡도 최소화 우선**
- **벤더 지원 중요**

---

## 🔧 트러블슈팅

### 일반적인 문제들

#### 1. Kong Admin API 접근 불가

**증상**: Kong Admin API에 연결할 수 없음

**원인 및 해결책**:

```bash
# Kong Admin Service 확인
kubectl get svc -n op-common | grep kong

# Kong Pod 상태 확인
kubectl get pods -n op-common | grep kong

# Kong Admin API 포트포워딩으로 테스트
kubectl port-forward svc/kong-admin -n op-common 8001:8001

# 브라우저에서 http://localhost:8001 접속 테스트
```

#### 2. PostgreSQL 연결 오류

**증상**: Kong이 PostgreSQL에 연결하지 못함

**해결책**:

```bash
# PostgreSQL Pod 상태 확인
kubectl get pods -n op-common | grep postgresql

# PostgreSQL 로그 확인
kubectl logs kong-postgresql-0 -n op-common

# Kong 설정 확인
kubectl get configmap -n op-common | grep kong
kubectl describe configmap kong-kong -n op-common
```

#### 3. Konga 데이터베이스 연결 실패

**증상**: Konga가 시작되지 않거나 데이터베이스 오류 발생

**해결책**:

```bash
# Konga 로그 확인
kubectl logs deployment/konga -n op-common

# PostgreSQL에서 Konga 데이터베이스 확인
kubectl exec -it kong-postgresql-0 -n op-common -- psql -U postgres -c "\l"

# Konga 데이터베이스 재생성 (필요시)
kubectl exec -it kong-postgresql-0 -n op-common -- psql -U postgres -c "DROP DATABASE IF EXISTS konga_database;"
kubectl exec -it kong-postgresql-0 -n op-common -- psql -U postgres -c "CREATE DATABASE konga_database OWNER konga;"
```

#### 4. 라우팅이 작동하지 않음

**증상**: Kong을 통한 라우팅이 제대로 작동하지 않음

**진단 방법**:

```bash
# Kong Admin API를 통해 서비스 및 라우트 확인
kubectl port-forward svc/kong-admin -n op-common 8001:8001

# 별도 터미널에서 확인
curl http://localhost:8001/services
curl http://localhost:8001/routes

# Kong 프록시 로그 확인
kubectl logs deployment/kong-proxy -n op-common --tail=100
```

### 로그 분석

#### Kong 로그 레벨 설정

```yaml
# values.yaml에서 로그 레벨 설정
env:
  log_level: debug  # error, warn, notice, info, debug
```

#### 주요 로그 위치

```bash
# Kong 프록시 로그
kubectl logs deployment/kong-proxy -n op-common

# Kong Admin API 로그  
kubectl logs deployment/kong-admin -n op-common

# PostgreSQL 로그
kubectl logs kong-postgresql-0 -n op-common

# Konga 로그
kubectl logs deployment/konga -n op-common
```

### 성능 튜닝

#### Kong Worker 프로세스 설정

```yaml
# values.yaml
env:
  nginx_worker_processes: "auto"  # CPU 코어 수에 맞게 자동 설정
  nginx_worker_connections: "1024"
```

#### 리소스 제한 설정

```yaml
# values.yaml
resources:
  limits:
    cpu: 2000m
    memory: 2Gi
  requests:
    cpu: 1000m
    memory: 1Gi
```
