### ArgoCD란?

ArgoCD는 **Kubernetes를 위한 선언적 GitOps 지속적 배포 도구**입니다. Git 저장소를 단일 진실 소스로 사용하여 애플리케이션을 자동으로 배포하고 관리합니다.

### 핵심 특징

- **GitOps 패턴**: Git을 단일 진실 소스로 사용
- **선언적 배포**: 원하는 상태를 선언하면 자동으로 동기화
- **다중 테넌시**: 여러 클러스터와 애플리케이션 관리
- **웹 UI 제공**: 직관적인 관리 인터페이스
- **RBAC 지원**: 세밀한 권한 제어
- **자동 동기화**: Git 변경사항 감지 시 자동 배포

### GitOps 워크플로우

```
개발자 코드 변경 → Git Push → ArgoCD 감지 → 자동 배포 → Kubernetes 클러스터
```

---

## ⚙️ ArgoCD 설치

### 1. Helm 저장소 추가 및 Values 준비

```bash
# ArgoCD Helm 저장소 추가
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# 기본 values.yaml 생성
helm show values argo/argo-cd > values.yaml
```

### 2. ArgoCD Values 설정

```yaml
# values.yaml 주요 설정
global:
  domain: argocd.dev.gcp.go.kr

server:
  service:
    type: ClusterIP   # Ingress 사용으로 NodePort 불필요
  ingress:
    enabled: false    # 수동으로 Ingress 생성 예정
  
  # 추가 보안 설정
  config:
    url: https://argocd.dev.gcp.go.kr
    application.instanceLabelKey: argocd.argoproj.io/instance
    
  # 리소스 제한 설정
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 250m
      memory: 256Mi

# Redis 설정 (고가용성을 위한 외부 Redis 사용 시)
redis-ha:
  enabled: false

redis:
  enabled: true
```

### 3. ArgoCD 설치

```bash
# 네임스페이스가 없는 경우 생성
kubectl create namespace op-gitops

# ArgoCD 설치
helm install argocd argo/argo-cd -n op-gitops -f values.yaml

# 설치 확인
kubectl get pods -n op-gitops
kubectl get svc -n op-gitops
```

### 4. 초기 관리자 비밀번호 확인

```bash
# 초기 admin 비밀번호 확인
kubectl -n op-gitops get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# 결과 예시: IG4NxwFZZtw-OV6m
```

### 5. TLS 인증서 생성

```bash
# Self-Signed 인증서 생성
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout argocd-tls.key -out argocd-tls.crt \
  -subj "/CN=argocd.dev.gcp.go.kr/O=argocd"

# Kubernetes Secret 생성
kubectl create secret tls argocd-tls \
  --cert=argocd-tls.crt \
  --key=argocd-tls.key \
  -n op-gitops
```

### 6. Ingress 설정

```yaml
# argocd-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: op-gitops
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
spec:
  ingressClassName: nginx
  rules:
  - host: argocd.dev.gcp.go.kr
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
  tls:
  - hosts:
    - argocd.dev.gcp.go.kr
    secretName: argocd-tls
```

```bash
kubectl apply -f argocd-ingress.yaml
```

### 7. 접속 확인

```bash
# 웹 브라우저에서 접속
https://argocd.dev.gcp.go.kr

# 또는 NodePort로 접속 (개발환경)
https://argocd.dev.gcp.go.kr:30191

# 로그인 정보
Username: admin
Password: <위에서 확인한 초기 비밀번호>
```

---

## 💻 ArgoCD CLI 설정

### 1. ArgoCD CLI 설치

#### Linux/WSL 설치

```bash
# 최신 버전 확인 및 다운로드
VERSION=$(curl -s https://api.github.com/repos/argoproj/argo-cd/releases/latest | jq -r .tag_name)
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/download/${VERSION}/argocd-linux-amd64

# 실행 권한 부여 및 PATH 추가
chmod +x argocd-linux-amd64
sudo mv argocd-linux-amd64 /usr/local/bin/argocd

# 버전 확인
argocd version
```

#### Windows 설치

```powershell
# PowerShell에서 실행
$version = (Invoke-RestMethod https://api.github.com/repos/argoproj/argo-cd/releases/latest).tag_name
Invoke-WebRequest -Uri "https://github.com/argoproj/argo-cd/releases/download/$version/argocd-windows-amd64.exe" -OutFile "argocd.exe"
```

### 2. ArgoCD CLI 로그인

#### 일반적인 로그인

```bash
# HTTPS 인그레스를 통한 로그인
argocd login argocd.dev.gcp.go.kr \
  --username admin \
  --password <초기-비밀번호> \
  --insecure \
  --grpc-web
```

#### GCP 환경에서 NodePort 로그인

```bash
# NodePort를 사용하는 경우
argocd login argocd.dev.gcp.go.kr:30191 \
  --username admin \
  --password IG4NxwFZZtw-OV6m \
  --insecure \
  --grpc-web
```

#### 포트포워딩을 통한 로그인 (개발환경)

```bash
# 별도 터미널에서 포트포워딩
kubectl port-forward svc/argocd-server -n op-gitops 8080:443

# 다른 터미널에서 로그인
argocd login localhost:8080 \
  --username admin \
  --password <초기-비밀번호> \
  --insecure
```

### 3. 비밀번호 변경

```bash
# 초기 비밀번호를 새 비밀번호로 변경
argocd account update-password \
  --current-password <초기-비밀번호> \
  --new-password <새-비밀번호>

# 예시
argocd account update-password \
  --current-password IG4NxwFZZtw-OV6m \
  --new-password MyNewSecurePassword123!
```

### 4. 기본 CLI 명령어

```bash
# 클러스터 정보 확인
argocd cluster list

# 애플리케이션 목록 확인
argocd app list

# 저장소 목록 확인
argocd repo list

# 계정 정보 확인
argocd account get-user-info

# 로그아웃
argocd logout argocd.dev.gcp.go.kr
```

---

## 🔗 Gitea 연동

### 1. Git 저장소 URL 문법 및 인증 방식

#### 📌 기본 URL 형태

```bash
http://<username>:<password_or_token>@<git-server>/<owner>/<repo>.git
```

**실제 예시:**

```bash
http://jenkins:d50f8213e4bd18fca3b99b2ecc65f6fa0d656ea8@gitea-http:3000/git-ops/frontendservice-cicd.git
```

**URL 구성 요소:**

- `jenkins`: Git 사용자 이름
- `d50f...`: Access Token (또는 비밀번호)
- `gitea-http:3000`: Git 서버 주소 (내부 도메인 또는 서비스명)
- 뒤에 `.git`까지 반드시 포함

> ✅ **보안 권장사항**: 비밀번호 대신 Access Token 사용으로 보안 리스크 최소화

#### 📌 인증 방법 비교

|방법|설명|장점|단점|
|---|---|---|---|
|**URL 직접 포함**|URL에 인증 정보 포함|빠른 설정|토큰 노출 위험|
|**ArgoCD Credentials**|별도 인증 정보 등록|보안성 우수|초기 설정 복잡|
|**SSH 키**|공개/개인 키 쌍 사용|높은 보안성|키 관리 필요|

### 2. ArgoCD Credentials를 통한 안전한 인증 (권장)

#### CLI를 통한 저장소 추가

```bash
# 안전한 방법: Credentials 별도 등록
argocd repo add http://gitea-http:3000/git-ops/easycook-cicd.git \
  --username jenkins \
  --password d50f8213e4bd18fca3b99b2ecc65f6fa0d656ea8 \
  --insecure-skip-server-verification

# 저장소 연결 확인
argocd repo list
```

#### UI를 통한 저장소 추가

```
ArgoCD UI → Settings → Repositories → Connect Repo using HTTPS
```

**저장소 설정 정보:**

- **Type**: `git`
- **Repository URL**: `http://gitea-http:3000/git-ops/easycook-cicd.git`
- **Username**: `jenkins`
- **Password**: `d50f8213e4bd18fca3b99b2ecc65f6fa0d656ea8`

**고급 설정:**

- ✅ **Skip server verification**: 체크 (self-signed 인증서 사용 시)
- ✅ **Force HTTP basic auth**: 체크 (기본 인증 강제)
- **TLS client certificate**: 필요 시 설정

### 3. SSH 키 인증 설정 (고보안 환경)

#### SSH 키 쌍 생성

```bash
# SSH 키 생성
ssh-keygen -t rsa -b 4096 -C "argocd@company.com" -f argocd_rsa

# 공개 키를 Gitea에 등록
cat argocd_rsa.pub
```

#### ArgoCD에 SSH 키 등록

```bash
# SSH 키로 저장소 추가
argocd repo add git@gitea-http:3000:git-ops/easycook-cicd.git \
  --ssh-private-key-path ./argocd_rsa \
  --insecure-ignore-host-key
```

### 3. 네트워크 고려사항

#### 내부 DNS 사용

```bash
# ArgoCD가 Gitea에 접근할 때 내부 서비스 DNS 사용
http://gitea-http.op-gitops.svc:3000/user/repo.git

# 네임스페이스가 다른 경우
http://gitea-http.op-gitops.svc.cluster.local:3000/user/repo.git
```

#### 외부 URL 사용 (선택사항)

```bash
# 외부 도메인을 통한 접근 (네트워크 정책에 따라)
https://gitea.dev.gcp.go.kr/user/repo.git
```

### 4. Gitea에서 웹훅 설정

#### ArgoCD 웹훅 URL

```
https://argocd.dev.gcp.go.kr/api/webhook
```

#### 웹훅 설정 방법

```
Gitea → Repository → Settings → Webhooks → Add Webhook
```

**설정 값:**

- **Payload URL**: `https://argocd.dev.gcp.go.kr/api/webhook`
- **Content Type**: `application/json`
- **Secret**: (ArgoCD 웹훅 시크릿 설정 시)
- **Trigger Events**:
    - ✅ Push events
    - ✅ Pull request events

---

## 📱 애플리케이션 관리

### 1. GitOps 저장소 구조 설계

#### 모노레포(Monorepo) 패턴

```
easycook-cicd/
├── FrontendService/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── UserService/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── OrderService/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── README.md
```

#### 장점

- **단일 저장소 관리**: 모든 마이크로서비스 매니페스트를 한 곳에서 관리
- **버전 동기화**: 전체 시스템의 일관된 버전 관리 가능
- **권한 관리 단순화**: 하나의 저장소에 대한 권한만 관리

### 2. 실무 중심 애플리케이션 생성

#### ArgoCD UI에서 마이크로서비스별 애플리케이션 생성

```
ArgoCD UI → Applications → New App
```

**FrontendService 애플리케이션 설정:**

- **Application Name**: `frontendservice`
- **Project**: `default`
- **Repository URL**: `http://gitea-http:3000/git-ops/easycook-cicd.git`
- **Revision**: `HEAD`
- **Path**: `FrontendService`
- **Cluster URL**: `https://kubernetes.default.svc`
- **Namespace**: `frontend`

**Sync Options (권장 설정):**

- ✅ **CreateNamespace**: 네임스페이스 자동 생성
- ❌ **Validate**: 검증 비활성화 (복잡한 매니페스트에서 오류 방지)
- ✅ **PruneLast**: 리소스 삭제를 마지막에 수행
- ✅ **ApplyOutOfSyncOnly**: 변경된 리소스만 적용

**Sync Policy (자동화 설정):**

- **Policy**: `Automated`
- ✅ **Prune**: 삭제된 리소스 자동 정리
- ✅ **Self Heal**: 수동 변경사항 자동 복구
- ❌ **Allow Empty**: 빈 디렉토리 허용 안함

> 💡 **Self Heal**: 실시간으로 Kubernetes 리소스 상태가 Git과 달라졌을 때 자동으로 되돌리는 기능

#### CLI를 통한 여러 마이크로서비스 애플리케이션 생성

```bash
# FrontendService 애플리케이션 생성
argocd app create frontendservice \
  --repo http://gitea-http:3000/git-ops/easycook-cicd.git \
  --path FrontendService \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace frontend \
  --sync-policy automated \
  --auto-prune \
  --self-heal \
  --sync-option CreateNamespace=true \
  --sync-option PruneLast=true \
  --sync-option ApplyOutOfSyncOnly=true

# UserService 애플리케이션 생성
argocd app create userservice \
  --repo http://gitea-http:3000/git-ops/easycook-cicd.git \
  --path UserService \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace backend \
  --sync-policy automated \
  --auto-prune \
  --self-heal \
  --sync-option CreateNamespace=true

# OrderService 애플리케이션 생성
argocd app create orderservice \
  --repo http://gitea-http:3000/git-ops/easycook-cicd.git \
  --path OrderService \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace backend \
  --sync-policy automated \
  --auto-prune \
  --self-heal \
  --sync-option CreateNamespace=true

# 모든 애플리케이션 상태 확인
argocd app list
```

### 3. YAML을 통한 애플리케이션 생성

#### 마이크로서비스별 Application 매니페스트

```yaml
# frontendservice-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontendservice
  namespace: op-gitops
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: http://gitea-http:3000/git-ops/easycook-cicd.git
    targetRevision: HEAD
    path: FrontendService
  destination:
    server: https://kubernetes.default.svc
    namespace: frontend
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    - Validate=false
    - PruneLast=true
    - ApplyOutOfSyncOnly=true
---
# userservice-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: userservice
  namespace: op-gitops
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: http://gitea-http:3000/git-ops/easycook-cicd.git
    targetRevision: HEAD
    path: UserService
  destination:
    server: https://kubernetes.default.svc
    namespace: backend
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    - PruneLast=true
```

```bash
# 모든 애플리케이션 한 번에 생성
kubectl apply -f frontendservice-app.yaml
kubectl apply -f userservice-app.yaml
kubectl apply -f orderservice-app.yaml
```

### 4. 애플리케이션 삭제 시 Propagation Policy

#### 삭제 정책 옵션

|정책|설명|사용 시나리오|
|---|---|---|
|**Foreground**|종속 리소스를 먼저 삭제 후 애플리케이션 삭제|**권장**: 완전한 정리|
|**Background**|애플리케이션을 먼저 삭제 후 종속 리소스 삭제|빠른 삭제 필요 시|
|**Non-cascading**|애플리케이션만 삭제, 리소스는 유지|리소스 보존 필요 시|

#### CLI를 통한 애플리케이션 삭제

```bash
# Foreground 정책으로 완전 삭제 (권장)
argocd app delete frontendservice --cascade --propagation-policy foreground

# Background 정책으로 빠른 삭제
argocd app delete userservice --cascade --propagation-policy background

# 애플리케이션만 삭제 (리소스 유지)
argocd app delete orderservice --cascade=false
```

## 🔄 CI/CD 파이프라인과 GitOps 연계

### 1. 변경 감지 기반 빌드 전략

#### Git 태그 기반 변경 감지

```bash
# Jenkins 파이프라인에서 변경된 서비스 감지
#!/bin/bash

# 이전 태그 확인
LAST_TAG=$(git describe --tags --abbrev=0 HEAD~1)
echo "마지막 태그: $LAST_TAG"

# 변경된 파일 목록 확인
CHANGED_FILES=$(git diff --name-only $LAST_TAG HEAD)
echo "변경된 파일들:"
echo "$CHANGED_FILES"

# 변경된 서비스 디렉토리 추출
CHANGED_SERVICES=$(echo "$CHANGED_FILES" | cut -d'/' -f1 | sort | uniq)
echo "변경된 서비스들:"
echo "$CHANGED_SERVICES"
```

#### 마이크로서비스별 조건부 빌드

```groovy
// Jenkinsfile 예시
pipeline {
    agent any
    
    environment {
        HARBOR_REGISTRY = 'harbor.example.com'
        GITOPS_REPO = 'http://gitea-http:3000/git-ops/easycook-cicd.git'
    }
    
    stages {
        stage('변경 감지') {
            steps {
                script {
                    // 이전 태그 대비 변경된 서비스 감지
                    def lastTag = sh(
                        script: "git describe --tags --abbrev=0 HEAD~1 2>/dev/null || echo 'v0.0.0'",
                        returnStdout: true
                    ).trim()
                    
                    def changedFiles = sh(
                        script: "git diff --name-only ${lastTag} HEAD",
                        returnStdout: true
                    ).trim()
                    
                    // 변경된 서비스 목록 추출
                    def changedServices = []
                    changedFiles.split('\n').each { file ->
                        def service = file.split('/')[0]
                        if (service && !changedServices.contains(service)) {
                            changedServices.add(service)
                        }
                    }
                    
                    env.CHANGED_SERVICES = changedServices.join(',')
                    echo "변경된 서비스: ${env.CHANGED_SERVICES}"
                }
            }
        }
        
        stage('서비스별 빌드') {
            parallel {
                stage('FrontendService') {
                    when {
                        expression { env.CHANGED_SERVICES.contains('FrontendService') }
                    }
                    steps {
                        buildAndDeploy('FrontendService', 'frontend')
                    }
                }
                
                stage('UserService') {
                    when {
                        expression { env.CHANGED_SERVICES.contains('UserService') }
                    }
                    steps {
                        buildAndDeploy('UserService', 'user')
                    }
                }
                
                stage('OrderService') {
                    when {
                        expression { env.CHANGED_SERVICES.contains('OrderService') }
                    }
                    steps {
                        buildAndDeploy('OrderService', 'order')
                    }
                }
            }
        }
    }
}

def buildAndDeploy(serviceName, imagePrefix) {
    // 서비스 디렉토리로 이동
    dir(serviceName) {
        // Docker 이미지 빌드
        script {
            def imageTag = "${BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
            def imageName = "${HARBOR_REGISTRY}/${imagePrefix}:${imageTag}"
            
            // Docker 빌드 및 푸시
            sh "docker build -t ${imageName} ."
            sh "docker push ${imageName}"
            
            // GitOps 저장소 업데이트
            updateGitOpsManifest(serviceName, imageTag)
            
            echo "✅ ${serviceName} 빌드 및 배포 완료: ${imageName}"
        }
    }
}

def updateGitOpsManifest(serviceName, imageTag) {
    // GitOps 저장소 클론
    sh """
        git clone ${GITOPS_REPO} gitops-temp
        cd gitops-temp/${serviceName}
        
        # Kustomization.yaml에서 이미지 태그 업데이트
        sed -i 's|newTag: .*|newTag: ${imageTag}|' kustomization.yaml
        
        # 변경사항 커밋 및 푸시
        git config user.email "jenkins@company.com"
        git config user.name "Jenkins CI"
        git add .
        git commit -m "Update ${serviceName} image to ${imageTag}"
        git push
        
        cd ../..
        rm -rf gitops-temp
    """
}
```

### 2. GitOps 매니페스트 업데이트 패턴

#### Kustomization 기반 이미지 관리

```yaml
# FrontendService/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml

images:
- name: frontend-app
  newName: harbor.example.com/frontend
  newTag: "123-abc1234"  # Jenkins에서 자동 업데이트

commonLabels:
  app: frontend
  version: v1.0.0
```

#### Helm Values 기반 이미지 관리

```yaml
# UserService/values.yaml
image:
  repository: harbor.example.com/user
  tag: "124-def5678"  # Jenkins에서 자동 업데이트
  pullPolicy: IfNotPresent

replicaCount: 3

service:
  type: ClusterIP
  port: 8080
```

#### Plain YAML 기반 이미지 관리

```yaml
# OrderService/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: harbor.example.com/order:125-ghi9012  # Jenkins에서 sed로 업데이트
        ports:
        - containerPort: 8080
```

### 3. ArgoCD 자동 동기화 흐름

#### 전체 CI/CD 파이프라인 흐름

```mermaid
graph TD
    A[개발자 코드 변경] --> B[Git Push to Source Repo]
    B --> C[Jenkins 웹훅 트리거]
    C --> D[변경된 서비스 감지]
    D --> E[해당 서비스만 빌드]
    E --> F[Docker 이미지 푸시]
    F --> G[GitOps 매니페스트 업데이트]
    G --> H[GitOps Repo에 Push]
    H --> I[ArgoCD 변경 감지]
    I --> J[자동 동기화 실행]
    J --> K[Kubernetes 배포 완료]
```

#### ArgoCD 동기화 정책 세부 설정

```yaml
# 실무 권장 동기화 정책
syncPolicy:
  automated:
    prune: true           # 삭제된 리소스 자동 정리
    selfHeal: true        # Kubernetes에서 수동 변경 시 자동 복구
    allowEmpty: false     # 빈 매니페스트 허용 안함
  syncOptions:
  - CreateNamespace=true  # 네임스페이스 자동 생성
  - PruneLast=true        # 리소스 삭제를 마지막에 수행
  - ApplyOutOfSyncOnly=true  # 변경된 리소스만 적용
  - RespectIgnoreDifferences=true  # 무시 설정 존중
  retry:
    limit: 5              # 최대 재시도 횟수
    backoff:
      duration: 5s        # 재시도 간격
      factor: 2           # 재시도 간격 증가 비율
      maxDuration: 3m     # 최대 재시도 간격
```

### 4. 실무 팁 및 Best Practices

#### 이미지 태그 전략

```bash
# 추천하는 이미지 태그 형식
${BUILD_NUMBER}-${GIT_COMMIT_SHORT}
# 예: 123-abc1234

# 또는 시맨틱 버전과 빌드 정보 조합
${SEMANTIC_VERSION}-${BUILD_NUMBER}
# 예: v1.2.3-123
```

#### GitOps 저장소 브랜치 전략

```bash
# 환경별 브랜치 관리
main          # 프로덕션 환경
develop       # 스테이징 환경
feature/*     # 개발 환경
```

#### ArgoCD Application 환경별 구성

```bash
# 환경별 ArgoCD Application 생성
argocd app create frontend-prod \
  --repo http://gitea-http:3000/git-ops/easycook-cicd.git \
  --revision main \
  --path FrontendService \
  --dest-namespace frontend-prod

argocd app create frontend-staging \
  --repo http://gitea-http:3000/git-ops/easycook-cicd.git \
  --revision develop \
  --path FrontendService \
  --dest-namespace frontend-staging
```

### 1. 저장소 연결 실패

#### 증상

- ❌ Repository connection failed
- ❌ Unable to resolve repository

#### 진단 방법

```bash
# ArgoCD 서버 로그 확인
kubectl logs deployment/argocd-server -n op-gitops

# 네트워크 연결 테스트
kubectl exec -it deployment/argocd-server -n op-gitops -- \
  curl -v http://gitea-http.op-gitops.svc:3000

# DNS 해결 테스트
kubectl exec -it deployment/argocd-server -n op-gitops -- \
  nslookup gitea-http.op-gitops.svc
```

#### 해결 방법

```bash
# 1. 저장소 URL 수정 (내부 DNS 사용)
argocd repo add http://gitea-http.op-gitops.svc.cluster.local:3000/user/repo.git

# 2. 인증 정보 재설정
argocd repo add http://gitea-http.op-gitops.svc:3000/user/repo.git \
  --username admin \
  --password <새-액세스-토큰> \
  --insecure-skip-server-verification

# 3. 기존 저장소 삭제 후 재추가
argocd repo rm http://gitea-http.op-gitops.svc:3000/user/repo.git
```

### 2. 애플리케이션 동기화 실패

#### 증상

- 🔄 Sync failed
- ❌ ComparisonError

#### 진단 방법

```bash
# 애플리케이션 상세 상태 확인
argocd app get my-application

# 동기화 상태 확인
argocd app sync my-application --dry-run

# ArgoCD Application Controller 로그
kubectl logs deployment/argocd-application-controller -n op-gitops
```

#### 일반적인 해결 방법

```bash
# 1. 하드 새로고침
argocd app get my-application --hard-refresh

# 2. 수동 동기화
argocd app sync my-application --force

# 3. 애플리케이션 재생성
kubectl delete application my-application -n op-gitops
kubectl apply -f application.yaml

# 4. 매니페스트 검증
kubectl apply --dry-run=client -f k8s/
```

### 3. 인증서 관련 문제

#### 증상

- 🔒 TLS certificate verification failed
- ❌ x509: certificate signed by unknown authority

#### 해결 방법

```bash
# 1. Insecure 옵션 사용
argocd repo add https://gitea.dev.gcp.go.kr/user/repo.git \
  --insecure-skip-server-verification

# 2. ArgoCD 서버 설정에서 insecure 모드 활성화
kubectl patch configmap argocd-cmd-params-cm -n op-gitops \
  --patch '{"data":{"server.insecure":"true"}}'

# 3. ArgoCD 서버 재시작
kubectl rollout restart deployment/argocd-server -n op-gitops
```

### 4. 권한 관련 문제

#### 증상

- ❌ Permission denied
- ❌ RBAC access denied

#### 해결 방법

```bash
# 1. ArgoCD 서비스 어카운트 권한 확인
kubectl describe clusterrolebinding argocd-application-controller

# 2. 네임스페이스별 권한 추가
kubectl create rolebinding argocd-admin \
  --clusterrole=admin \
  --serviceaccount=op-gitops:argocd-application-controller \
  -n target-namespace

# 3. 전역 권한 부여 (주의!)
kubectl create clusterrolebinding argocd-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=op-gitops:argocd-application-controller
```

### 5. Gitea 로그 확인

#### Gitea 연동 문제 진단

```bash
# Gitea 서버 로그 확인
kubectl logs deployment/gitea -n op-gitops

# Gitea HTTP 서비스 로그
kubectl logs service/gitea-http -n op-gitops

# ArgoCD에서 Gitea 접근 로그
kubectl logs deployment/argocd-repo-server -n op-gitops | grep gitea
```

#### 웹훅 문제 진단

```bash
# Gitea 웹훅 로그 확인 (UI)
# Gitea → Repository → Settings → Webhooks → Recent Deliveries

# ArgoCD 웹훅 수신 로그
kubectl logs deployment/argocd-server -n op-gitops | grep webhook
```

---

## ⚡ 고급 설정

### 1. 다중 클러스터 관리

#### 외부 클러스터 추가

```bash
# 외부 클러스터 등록
argocd cluster add my-external-cluster \
  --server https://external-k8s-api.example.com \
  --name external-cluster

# 클러스터 목록 확인
argocd cluster list

# 외부 클러스터에 애플리케이션 배포
argocd app create external-app \
  --repo http://gitea-http.op-gitops.svc:3000/user/manifests.git \
  --path k8s \
  --dest-server https://external-k8s-api.example.com \
  --dest-namespace production
```

### 2. RBAC 설정

#### ArgoCD RBAC 정책

```yaml
# argocd-rbac-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: op-gitops
data:
  policy.default: role:readonly
  policy.csv: |
    # 개발팀 권한
    p, role:developer, applications, *, */*, allow
    p, role:developer, repositories, *, *, allow
    
    # 운영팀 권한  
    p, role:operator, applications, *, production/*, allow
    p, role:operator, clusters, *, *, allow
    
    # 그룹-역할 매핑
    g, dev-team, role:developer
    g, ops-team, role:operator
```

### 3. 알림 설정

#### Slack 알림 설정

```yaml
# argocd-notifications-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: op-gitops
data:
  service.slack: |
    token: xoxb-your-slack-bot-token
  template.app-deployed: |
    message: |
      {{if eq .serviceType "slack"}}:white_check_mark:{{end}} Application {{.app.metadata.name}} is now running new version.
  trigger.on-deployed: |
    - when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
      send: [app-deployed]
  subscriptions: |
    - recipients:
      - slack:my-channel
      triggers:
      - on-deployed
```

### 4. ApplicationSet 사용

#### 여러 환경 자동 생성

```yaml
# applicationset.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-env-apps
  namespace: op-gitops
spec:
  generators:
  - list:
      elements:
      - env: dev
        namespace: development
      - env: staging  
        namespace: staging
      - env: prod
        namespace: production
  template:
    metadata:
      name: 'my-app-{{env}}'
    spec:
      project: default
      source:
        repoURL: http://gitea-http.op-gitops.svc:3000/user/manifests.git
        targetRevision: main
        path: 'envs/{{env}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### 5. 백업 및 복구

#### ArgoCD 설정 백업

```bash
# 모든 애플리케이션 백업
argocd app list -o yaml > argocd-apps-backup.yaml

# 저장소 설정 백업
argocd repo list -o yaml > argocd-repos-backup.yaml

# 클러스터 설정 백업
argocd cluster list -o yaml > argocd-clusters-backup.yaml
```

#### 복구

```bash
# 백업된 설정 복구
kubectl apply -f argocd-apps-backup.yaml
kubectl apply -f argocd-repos-backup.yaml
```
