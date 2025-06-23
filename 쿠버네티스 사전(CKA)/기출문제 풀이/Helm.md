## 🎯 문제 요구사항

1. **Helm 리포지토리 업데이트** (새로운 차트 버전 가져오기)
2. **kk-mock1 차트를 18.1.15 버전으로 업그레이드**

## 📋 해결 과정

### 1단계: 현재 상황 파악

```bash
helm list -A
```

**확인 결과**: `kk-mock1` nginx 차트가 `kk-ns` 네임스페이스에 18.1.0 버전으로 설치되어 있음

### 2단계: Helm 리포지토리 업데이트 ⭐

```bash
helm repo update
```

**목적**: 리포지토리에서 최신 차트 버전들을 가져오기
**결과**: "Successfully got an update from the kk-mock1 chart repository"

### 3단계: 사용 가능한 버전 확인

```bash
helm search repo nginx --versions
```

**확인**: 18.1.15 버전이 사용 가능함을 확인

### 4단계: 차트 업그레이드 ⭐

```bash
helm upgrade kk-mock1 kk-mock1/nginx --version=18.1.15 -n kk-ns
```

**결과**: 18.1.0 → 18.1.15로 성공적으로 업그레이드

### 5단계: 업그레이드 확인

```bash
helm list -A
```

**확인**: 차트 버전이 18.1.15로 변경되고, REVISION이 2로 증가함

## 🔑 핵심 명령어 2개

### 1. 리포지토리 업데이트

```bash
helm repo update
```

- **목적**: 새로운 차트 버전 정보를 가져옴
- **비유**: 앱스토어에서 "새로고침" 버튼 누르기

### 2. 차트 업그레이드

```bash
helm upgrade [릴리스명] [차트명] --version=[버전] -n [네임스페이스]
```

- **실제**: `helm upgrade kk-mock1 kk-mock1/nginx --version=18.1.15 -n kk-ns`

## 💡 쉬운 이해

**상황**: 동료가 nginx를 설치해뒀는데, 새 버전이 나왔으니 업데이트해달라고 함

**과정**:

1. **업데이트 확인** (`helm repo update`) - "새 버전 있나 확인"
2. **업그레이드** (`helm upgrade`) - "실제로 새 버전으로 교체"

**결과**: nginx 18.1.0 → 18.1.15로 성공적으로 업그레이드! 🎉

**핵심**: 문제에서 요구한 "helm repository update"와 "upgrade to version 18.1.15" 모두 완료!

---

`webpage-server-01` 애플리케이션이 Helm 도구를 통해 Kubernetes 클러스터에 배포되어 있습니다. 현재 팀은 이 애플리케이션의 새 버전을 기존 버전과 교체하여 배포하려고 합니다. 새 버전의 Helm 차트는 터미널의 `/root/new-version` 디렉터리에 있습니다. 클러스터에 설치하기 전에 차트를 먼저 검증하세요.

`helm` 명령어를 사용해 차트를 검증하고 설치하세요. 새 버전이 성공적으로 설치되면, 이전 버전은 제거하세요.

## 📱 앱스토어 비유로 이해하는 Helm

|Helm 개념|앱스토어 비유|설명|
|---|---|---|
|**Chart**|앱 (APK 파일)|쿠버네티스 애플리케이션 패키지|
|**Repository**|앱스토어|차트들이 저장된 곳|
|**Release**|설치된 앱|실제로 배포된 애플리케이션 인스턴스|

---

## 🔧 핵심 명령어 체계

### 1️⃣ **저장소(앱스토어) 관리**

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami  # 새 스토어 추가
helm repo list                                           # 연결된 스토어들 확인
helm repo update                                         # 스토어 정보 업데이트
helm repo remove bitnami                                 # 스토어 삭제
```

### 2️⃣ **앱 검색 및 정보 확인**

```bash
helm search repo nginx                    # 설치 가능한 앱들 검색
helm search repo nginx --versions         # 모든 버전 보기
helm show values bitnami/nginx            # 앱 설정 정보 보기
helm show chart bitnami/nginx             # 앱 기본 정보 보기
```

### 3️⃣ **앱 설치**

```bash
helm install my-nginx bitnami/nginx                          # 기본 설치
helm install my-nginx bitnami/nginx --set replicaCount=3     # 설정 변경하며 설치
helm install my-nginx bitnami/nginx -f values.yaml          # 파일로 설정하며 설치
helm install my-nginx bitnami/nginx --dry-run --debug       # 설치 전 테스트
```

### 4️⃣ **설치된 앱 관리**

```bash
helm list                                 # 설치된 앱들 확인
helm list -n kube-system                  # 특정 네임스페이스의 앱들
helm status my-nginx                      # 앱 상태 확인
helm get values my-nginx                  # 앱 설정값 확인
```

### 5️⃣ **앱 업데이트 및 관리**

```bash
helm upgrade my-nginx bitnami/nginx --set replicaCount=5  # 앱 업그레이드
helm history my-nginx                                     # 업데이트 내역 확인
helm rollback my-nginx 1                                  # 이전 버전으로 롤백
helm uninstall my-nginx                                   # 앱 삭제
```

---

## 🎯 CKA 시험 관점에서 중요한 포인트

### **반드시 알아야 할 시나리오**

1. **기본 설치**: `helm install app-name chart-name`
2. **설정 변경**: `--set key=value` 또는 `-f values.yaml`
3. **업그레이드**: `helm upgrade` 후 `helm rollback`
4. **문제 해결**: `helm status`, `helm get values`, `kubectl` 조합 사용

### **실습해볼 애플리케이션**

- **nginx**: 웹서버 (가장 기본)
- **mysql**: 데이터베이스 (상태가 있는 앱)
- **prometheus**: 모니터링 (복잡한 설정)

---

## ⚡ 빠른 치트시트

```bash
# 📋 기본 플로우
helm repo add bitnami https://charts.bitnami.com/bitnami
helm search repo nginx
helm install my-app bitnami/nginx
helm list
helm upgrade my-app bitnami/nginx --set replicaCount=2
helm rollback my-app 1
helm uninstall my-app

# 🔍 문제 해결
helm status my-app                    # 상태 확인
helm get values my-app                # 설정 확인  
kubectl get pods -l app=my-app        # 실제 파드 확인
helm template my-app bitnami/nginx    # 실제 배포될 yaml 미리보기
```

이제 Helm의 전체 그림이 보이시나요? CKA 실습하실 때 이 순서대로 해보시면 됩니다! 🚀


























프로젝트 순서: 분석, 설계, 구축, 시험

기획안(ppt) -> wbs -> 요구사항 명세서 -> UI 가이드 -> 클래스 다이어그램(Doxygen) -> 마이크로서비스(3개 이상) -> API 명세서 -> 단위테스트(JUnit)

rfp -> 제안서 -> 요구사항 정의서/요구사항 명세서(이어서 씀) -> UI 있으면 화면설계서 및 UI 가이드(이거 다 상세설계서), 없으면 개발계획서, 사업수행계획서 -> 상세설계서(클래스 다이어그램 등)
