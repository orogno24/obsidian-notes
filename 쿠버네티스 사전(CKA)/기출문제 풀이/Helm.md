cl## 🎯 문제 요구사항

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


rfp -> 제안서 -> 요구사항 정의서/요구사항 명세서(이어서 씀) -> UI 있으면 화면설계서 및 UI 가이드(이거 다 상세설계서), 없으면 개발계획서, 사업수행계획서 -> 상세설계서(클래스 다이어그램 등)

프로젝트 순서: 분석, 설계, 구축, 시험



One application, `webpage-server-01`, is deployed on the Kubernetes cluster by the Helm tool. Now, the team wants to deploy a new version of the application by replacing the existing one. A new version of the helm chart is given in the `/root/new-version` directory on the terminal. Validate the chart before installing it on the Kubernetes cluster.   
  
  
Use the `helm` command to validate and install the chart. After successfully installing the newer version, uninstall the older version.

controlplane ~ ✖ helm lint .
==> Linting .
Error unable to check Chart.yaml file in chart: stat Chart.yaml: no such file or directory

Error: 1 chart(s) linted, 1 chart(s) failed

controlplane ~ ➜  helm install --generate-name /root/new-version
NAME: new-version-1748237422
LAST DEPLOYED: Mon May 26 05:30:22 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None

controlplane ~ ➜  helm list
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                       APP VERSION
new-version-1748237422  default         1               2025-05-26 05:30:22.545857437 +0000 UTC deployed        webpage-server-02-0.1.1     v2         
webpage-server-01       default         1               2025-05-26 05:28:34.975342474 +0000 UTC deployed        webpage-server-01-0.1.0     v1         

controlplane ~ ➜  helm uninstall webpage-server-01
release "webpage-server-01" uninstalled

controlplane ~ ➜  helm list
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                       APP VERSION
new-version-1748237422  default         1               2025-05-26 05:30:22.545857437 +0000 UTC deployed        webpage-server-02-0.1.1     v2         

controlplane ~ ➜  
