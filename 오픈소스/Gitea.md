### Gitea란?

Gitea는 **Go 언어로 작성된 가벼운 자체 호스팅 Git 서비스**입니다. GitHub이나 GitLab과 유사한 기능을 제공하지만, 단일 바이너리로 배포되어 설치가 간편하고 시스템 리소스 사용량이 적습니다.

### 핵심 특징

- **경량성**: 최소한의 시스템 요구사항
- **단일 바이너리**: Go 언어 특성으로 배포 간편
- **완전 오픈소스**: MIT 라이선스
- **자체 호스팅**: 완전한 데이터 주권
- **빠른 성능**: Git 작업에 최적화
- **심플한 UI**: 필수 기능 중심의 직관적 인터페이스

### 적합한 사용 사례

- **기업 내부 코드 관리**
- **프라이빗 팀 프로젝트**
- **제한된 네트워크 환경에서의 개발**
- **데이터 주권/규제 요구사항이 있는 상황**
- **리소스가 제한된 환경** (라즈베리 파이 등)
- **간단한 Git 서버가 필요한 경우**

---

## ⚖️ GitHub vs GitLab vs Gitea 비교

### 상세 기능 비교

|항목|**GitHub**|**GitLab**|**Gitea**|
|---|---|---|---|
|**소유/운영**|Microsoft (클라우드 중심)|GitLab Inc. (독립 운영)|**커뮤니티 기반 (비영리적)**|
|**설치형 지원**|제한적 (Enterprise Server만)|**완벽 지원 (CE/EE)**|**매우 가볍고 쉬운 설치**|
|**CI/CD 내장**|GitHub Actions (강력)|**GitLab CI/CD (완전 통합)**|없음 (외부 연동 필요)|
|**속도/경량성**|무난한 성능|기능이 많아 무거움|**매우 가볍고 빠름**|
|**UI/UX**|직관적이고 세련됨|기능 많아 복잡할 수 있음|**심플한 UI (필수 기능만)**|
|**오픈소스**|❌ (부분만 공개)|✅ (CE 오픈소스)|✅ **완전 오픈소스**|
|**기능 확장성**|Marketplace 앱 다양|**DevOps 기능 전반 내장**|제한적, 외부 통합 권장|
|**접근성**|전 세계 클라우드|클라우드/온프레미스|**온프레미스 전용**|
|**보안 수준**|클라우드 의존|높은 보안 기능|**완전 격리 가능**|
|**리소스 사용량**|클라우드라 무관|높음 (RAM 4GB+ 권장)|**낮음 (RAM 512MB도 가능)**|

### 선택 가이드

#### GitHub 선택 시

- **오픈소스 프로젝트** 공개
- **협업과 커뮤니티** 중요
- **GitHub Actions** 활용 원함
- **빠른 시작**이 우선

#### GitLab 선택 시

- **완전한 DevOps 파이프라인** 필요
- **기업급 기능** 요구
- **이슈 관리, 프로젝트 관리** 통합 필요
- **보안 스캔, 컴플라이언스** 중요

#### Gitea 선택 시

- **완전한 프라이빗** 환경 필요
- **경량 솔루션** 선호
- **단순한 Git 서버**만 필요
- **리소스 제약** 환경
- **데이터 주권** 확보 필요

---

## 🚀 Gitea 설치

### 1. Helm을 통한 설치

#### Helm 저장소 추가

```bash
# Gitea Helm 저장소 추가
helm repo add gitea-charts https://dl.gitea.io/charts/

# 저장소 업데이트
helm repo update

# Values 파일 확인 (선택사항)
helm show values gitea-charts/gitea > gitea-values.yaml
```

#### Gitea 설치

```bash
# 기본 설정으로 설치
helm install gitea gitea-charts/gitea --namespace op-gitops

# 커스텀 values 사용 시
helm install gitea gitea-charts/gitea --namespace op-gitops -f gitea-values.yaml
```

#### 설치 확인

```bash
# Pod 상태 확인
kubectl get pods -n op-gitops

# 서비스 확인
kubectl get svc -n op-gitops

# Gitea 로그 확인
kubectl logs -l app.kubernetes.io/name=gitea -n op-gitops
```

### 2. TLS 인증서 생성

```bash
# 개인 키 생성
openssl genrsa -out gitea-tls.key 2048

# 인증서 서명 요청(CSR) 생성
openssl req -new -key gitea-tls.key -out gitea-tls.csr \
  -subj "/CN=gitea.dev.gcp.go.kr"

# 자체 서명 인증서 생성
openssl x509 -req -days 365 -in gitea-tls.csr \
  -signkey gitea-tls.key -out gitea-tls.crt

# Kubernetes Secret 생성
kubectl create secret tls gitea-tls -n op-gitops \
  --cert=gitea-tls.crt \
  --key=gitea-tls.key
```

### 3. Ingress 설정

```yaml
# gitea-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitea-ingress
  namespace: op-gitops
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "1024m"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: gitea.dev.gcp.go.kr
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gitea-http
            port:
              number: 3000
  tls:
  - hosts:
    - gitea.dev.gcp.go.kr
    secretName: gitea-tls
```

```bash
kubectl apply -f gitea-ingress.yaml
```

### 4. 초기 접속 및 설정

```bash
# Gitea 접속 URL
https://gitea.dev.gcp.go.kr:30191/
```

**초기 설정 과정:**

1. 데이터베이스 설정 (SQLite/PostgreSQL/MySQL)
2. 관리자 계정 생성
3. SSH 서버 설정
4. 메일 서버 설정 (선택사항)

---

## 🔑 액세스 토큰 관리

### 액세스 토큰이 필요한 이유

- **보안 강화**: 비밀번호보다 안전
- **권한 세분화**: 특정 작업만 허용 가능
- **취소 가능**: 언제든지 토큰 무효화 가능
- **CI/CD 연동**: 자동화된 시스템에서 안전한 인증

### 액세스 토큰 생성 방법

#### 1. Gitea UI에서 생성

```
Gitea 로그인 → 우측 상단 프로필 → 설정(Settings) → 애플리케이션(Applications)
```

#### 2. 새 토큰 생성

- **토큰 이름**: 용도별 구분 (예: `jenkins-ci`, `argocd-sync`)
- **만료 시간**: 보안 정책에 따라 설정
- **권한 설정**:
    - **Repository Access**: "All (public, private, and limited)"
    - **Specific Permissions**:
        - `repository`: "Read and Write" (필수)
        - `organization`: "Read" (조직 저장소 사용 시)
        - `package`: "Read" (패키지 사용 시)

#### 3. 토큰 보안 관리

```bash
# 생성된 토큰 예시
ghp_1234567890abcdefghijklmnopqrstuvwxyz

# Kubernetes Secret으로 저장
kubectl create secret generic gitea-token \
  --from-literal=token=ghp_1234567890abcdefghijklmnopqrstuvwxyz \
  -n op-gitops

# Jenkins Credentials로 저장
# Dashboard → Manage Jenkins → Credentials → Add Credentials
# Kind: Secret text
# Secret: <토큰 값>
# ID: gitea-token
```

### 토큰 사용 예시

#### Git 클론 시 토큰 사용

```bash
# HTTPS 클론에서 토큰 사용
git clone https://<username>:<token>@gitea.dev.gcp.go.kr/user/repo.git

# 또는 Git credential helper 설정
git config --global credential.helper store
# 이후 첫 푸시/풀 시 username과 token 입력
```

#### Jenkins에서 토큰 사용

```groovy
// Jenkinsfile에서 Git 체크아웃
checkout([
    $class: 'GitSCM',
    userRemoteConfigs: [[
        url: 'https://gitea.dev.gcp.go.kr/user/repo.git',
        credentialsId: 'gitea-token'
    ]],
    branches: [[name: 'main']]
])
```

---

## 💻 Git 작업 절차

### 새 프로젝트 시작하기

#### 1. 로컬 Git 저장소 초기화

```bash
# 프로젝트 폴더에서 Git Bash 실행
cd /path/to/your/project

# Git 저장소 초기화
git init

# 사용자 정보 설정 (최초 1회)
git config --global user.name "Your Name"
git config --global user.email "your.email@company.com"
```

#### 2. Gitea에서 저장소 생성

```
Gitea UI → New Repository → 저장소 이름 입력 → Create Repository
```

#### 3. 원격 저장소 연결

```bash
# 원격 저장소 추가
git remote add origin https://gitea.dev.gcp.go.kr:30191/username/repository.git

# 원격 저장소 확인
git remote -v
```

#### 4. 초기 커밋 및 푸시

```bash
# 파일 스테이징
git add .

# 초기 커밋
git commit -m "Initial commit"

# 기본 브랜치를 main으로 설정 (선택사항)
git branch -M main

# 원격 저장소에 푸시
git push -u origin main
```

### 기존 저장소 작업하기

#### 일반적인 Git 워크플로우

```bash
# 1. 최신 변경사항 가져오기
git pull origin main

# 2. 새 브랜치 생성 (기능 개발 시)
git checkout -b feature/new-feature

# 3. 파일 수정 후 변경사항 확인
git status
git diff

# 4. 변경사항 스테이징
git add .
# 또는 특정 파일만
git add filename.txt

# 5. 커밋
git commit -m "Add new feature: description"

# 6. 원격 저장소에 푸시
git push origin feature/new-feature

# 7. Gitea UI에서 Pull Request 생성
```

#### 원격 저장소와 충돌 해결

```bash
# 원격 저장소에 이미 커밋이 있는 경우
git pull origin main --allow-unrelated-histories

# 충돌 발생 시 수동 해결 후
git add .
git commit -m "Merge conflict resolved"

# 최종 푸시
git push -u origin main
```

### Git 설정 최적화

#### 유용한 Git 설정

```bash
# 기본 에디터 설정
git config --global core.editor "code --wait"  # VS Code 사용 시

# 줄바꿈 설정 (Windows)
git config --global core.autocrlf true

# 줄바꿈 설정 (Linux/Mac)
git config --global core.autocrlf input

# Pull 전략 설정
git config --global pull.rebase false

# 기본 브랜치명 설정
git config --global init.defaultBranch main
```

#### .gitignore 템플릿

```bash
# .gitignore 예시 (Java 프로젝트)
*.class
*.jar
*.war
target/
.gradle/
build/
.idea/
*.iml
.vscode/
.DS_Store
```

---

## 👥 저장소 권한 관리

### 1. 협업자 관리

#### 사용자 추가

```
저장소 → Settings → Collaborators → Add Collaborator
```

**권한 레벨:**

- **Read**: 코드 읽기, 이슈 생성
- **Write**: 코드 읽기/쓰기, 브랜치 생성
- **Admin**: 모든 권한 + 저장소 설정

#### 팀 기반 권한 관리

```
Organization → Teams → Create Team → 팀에 저장소 할당
```

### 2. 브랜치 보호 규칙

#### 보호 규칙 설정

```
저장소 → Settings → Branches → Add Rule
```

**주요 설정 옵션:**

- **브랜치 패턴**: `main`, `develop`, `release/*`
- **직접 푸시 금지**: ✅ 체크
- **병합 전 검토 요구**: ✅ 체크 (Pull Request 필수)
- **상태 체크 요구**: CI/CD 성공 시에만 병합
- **최신 상태 요구**: 병합 전 최신 커밋 필요

#### 브랜치 전략 예시

```bash
# Git Flow 방식
main       (프로덕션)
develop    (개발 통합)
feature/*  (기능 개발)
release/*  (릴리스 준비)
hotfix/*   (긴급 수정)
```

### 3. 웹훅 설정

#### ArgoCD 자동 동기화 웹훅

```
저장소 → Settings → Webhooks → Add Webhook
```

**웹훅 설정:**

- **Payload URL**: `https://argocd.example.com/api/webhook`
- **Content Type**: `application/json`
- **Secret**: ArgoCD 웹훅 시크릿
- **Trigger Events**:
    - ✅ Push events
    - ✅ Pull request events

#### Jenkins 빌드 트리거 웹훅

```
저장소 → Settings → Webhooks → Add Webhook
```

**웹훅 설정:**

- **Payload URL**: `https://jenkins.example.com/gitea-webhook/post`
- **Content Type**: `application/json`
- **Trigger Events**: ✅ Push events

---

## 🔄 CI/CD 연동

### Jenkins 연동

#### 1. Jenkins에서 Gitea 설정

```groovy
// Jenkinsfile 예시
pipeline {
    agent any
    
    triggers {
        gitea(
            triggerOnPush: true,
            triggerOnMergeRequest: true,
            branchFilterType: 'All'
        )
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh './gradlew build'
            }
        }
        
        stage('Test') {
            steps {
                sh './gradlew test'
                publishTestResults testResultsPattern: '**/build/test-results/test/*.xml'
            }
        }
    }
}
```

#### 2. Gitea 플러그인 설정

```
Jenkins → Manage Jenkins → Configure System → Gitea Servers
```

### ArgoCD 연동

#### 1. ArgoCD에서 Git 저장소 등록

```yaml
# argocd-repository.yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitea-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  type: git
  url: https://gitea.dev.gcp.go.kr/user/manifests.git
  username: your-username
  password: your-access-token
```

#### 2. ArgoCD 애플리케이션 생성

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://gitea.dev.gcp.go.kr/user/manifests.git
    targetRevision: main
    path: k8s/
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### GitHub Actions 대안 (외부 CI/CD)

#### Drone CI 연동

```yaml
# .drone.yml
kind: pipeline
type: docker
name: default

steps:
- name: build
  image: gradle:jdk11
  commands:
  - gradle build

- name: test  
  image: gradle:jdk11
  commands:
  - gradle test

trigger:
  branch:
  - main
```

---

## 📋 활용 시나리오

### 전체 DevOps 파이프라인 구성

#### 1. 초기 설정

```bash
# Gitea 관리자 계정 설정
# 샘플 저장소 생성 (애플리케이션 코드)
# 매니페스트 저장소 생성 (Kubernetes YAML)
```

#### 2. Jenkins 파이프라인 설정

```groovy
// 소스 코드 빌드 → 테스트 → 컨테이너 이미지 생성 → Harbor 푸시
pipeline {
    agent any
    
    environment {
        HARBOR_REGISTRY = 'harbor.example.com'
        IMAGE_NAME = 'my-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://gitea.dev.gcp.go.kr/user/app.git',
                    credentialsId: 'gitea-token'
            }
        }
        
        stage('Build & Test') {
            steps {
                sh './gradlew clean build test'
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    def image = docker.build("${HARBOR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}")
                    docker.withRegistry("https://${HARBOR_REGISTRY}", 'harbor-credentials') {
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
        
        stage('Update Manifests') {
            steps {
                // GitOps: 매니페스트 저장소 업데이트
                sh """
                git clone https://gitea.dev.gcp.go.kr/user/manifests.git
                cd manifests
                sed -i 's|image: .*|image: ${HARBOR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}|' k8s/deployment.yaml
                git add .
                git commit -m 'Update image to ${IMAGE_TAG}'
                git push
                """
            }
        }
    }
}
```

#### 3. ArgoCD 자동 배포 설정

```yaml
# ArgoCD가 매니페스트 저장소를 모니터링하여 자동 배포
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-application
spec:
  source:
    repoURL: https://gitea.dev.gcp.go.kr/user/manifests.git
    path: k8s/
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 팀 협업 워크플로우

#### 1. 기능 개발 플로우

```bash
# 1. 개발자가 기능 브랜치 생성
git checkout -b feature/user-authentication

# 2. 코드 작성 및 커밋
git add .
git commit -m "Add user authentication feature"

# 3. Gitea에 푸시
git push origin feature/user-authentication

# 4. Gitea UI에서 Pull Request 생성
# 5. 코드 리뷰 진행
# 6. CI/CD 파이프라인 자동 실행 (Jenkins)
# 7. 테스트 통과 후 main 브랜치에 병합
# 8. ArgoCD가 자동으로 프로덕션 배포
```

#### 2. 핫픽스 플로우

```bash
# 1. 긴급 수정 브랜치 생성
git checkout -b hotfix/security-patch

# 2. 수정 및 테스트
git commit -m "Fix security vulnerability"

# 3. 빠른 리뷰 및 병합
# 4. 자동 배포 확인
```

---

## 🔧 트러블슈팅

### 일반적인 문제들

#### 1. Git 푸시 인증 실패

**증상**: `authentication failed` 오류

**해결책**:

```bash
# 1. 액세스 토큰 재생성
# 2. Git credential 업데이트
git config --global credential.helper store
# 다시 푸시할 때 username과 새 토큰 입력

# 3. 또는 URL에 토큰 포함
git remote set-url origin https://<username>:<token>@gitea.dev.gcp.go.kr/user/repo.git
```

#### 2. Gitea Pod 시작 실패

**증상**: Gitea Pod가 CrashLoopBackOff 상태

**진단 및 해결**:

```bash
# 로그 확인
kubectl logs -l app.kubernetes.io/name=gitea -n op-gitops

# 볼륨 마운트 문제 확인
kubectl describe pod -l app.kubernetes.io/name=gitea -n op-gitops

# PVC 상태 확인
kubectl get pvc -n op-gitops

# 재시작
kubectl rollout restart deployment/gitea -n op-gitops
```

#### 3. 웹훅이 작동하지 않음

**증상**: 코드 푸시 후 Jenkins/ArgoCD가 트리거되지 않음

**해결책**:

```bash
# 1. 웹훅 URL 확인
# 2. 네트워크 연결성 테스트
curl -X POST https://jenkins.example.com/gitea-webhook/post

# 3. 웹훅 로그 확인 (Gitea UI)
# Settings → Webhooks → Recent Deliveries

# 4. 방화벽/네트워크 정책 확인
```

#### 4. HTTPS 인증서 문제

**증상**: 브라우저에서 인증서 경고

**해결책**:

```bash
# 1. 인증서 재생성
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout gitea-tls.key -out gitea-tls.crt \
  -subj "/CN=gitea.dev.gcp.go.kr"

# 2. Secret 업데이트
kubectl create secret tls gitea-tls -n op-gitops \
  --cert=gitea-tls.crt \
  --key=gitea-tls.key \
  --dry-run=client -o yaml | kubectl apply -f -

# 3. Ingress 재시작
kubectl rollout restart deployment/nginx-ingress-controller -n ingress-nginx
```

### 성능 최적화

#### Git 저장소 최적화

```bash
# 대용량 저장소 정리
git gc --aggressive --prune=now

# 불필요한 브랜치 정리
git branch -d old-feature-branch
git push origin --delete old-feature-branch
```

#### Gitea 성능 튜닝

```yaml
# gitea-values.yaml
gitea:
  config:
    cache:
      ENABLED: true
      ADAPTER: memory
      INTERVAL: 60
    session:
      PROVIDER: memory
    queue:
      TYPE: channel
```

---

## 📚 참고 자료

- [Gitea 공식 문서](https://docs.gitea.io/)
- [Gitea Helm Chart](https://gitea.com/gitea/helm-chart)
- [Git 공식 문서](https://git-scm.com/doc)
- [GitOps with ArgoCD](https://argo-cd.readthedocs.io/)
- [Jenkins Git Plugin](https://plugins.jenkins.io/git/)

---

**Tags:** #gitea #git #selfhosted #kubernetes #cicd #argocd #jenkins #golang #version-control