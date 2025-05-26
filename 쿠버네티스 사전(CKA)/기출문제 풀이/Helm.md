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


On the cluster, the team has installed multiple helm charts on a different namespace. By mistake, those deployed resources include one of the vulnerable images called `kodekloud/webapp-color:v1`. Find out the release name and uninstall it.

helm list -A
NAME                    NAMESPACE               REVISION        UPDATED                                 STATUS          CHART                       APP VERSION
atlanta-page-apd        atlanta-page-04         1               2025-05-26 01:10:46.245952796 +0000 UTC deployed        atlanta-page-apd-0.1.0      1.16.0     
digi-locker-apd         digi-locker-02          1               2025-05-26 01:10:45.553724078 +0000 UTC deployed        digi-locker-apd-0.1.0       1.16.0     
security-alpha-apd      security-alpha-01       1               2025-05-26 01:10:45.031610328 +0000 UTC deployed        security-alpha-apd-0.1.0    1.16.0     
web-dashboard-apd       web-dashboard-03        1               2025-05-26 01:10:45.923124126 +0000 UTC deployed        web-dashboard-apd-0.1.0     1.16.0     

cluster1-controlplane ~ ➜  helm get manifest atlanta-page-apd -n atlanta-page-04 | grep -i webapp-color:v1
          image: "kodekloud/webapp-color:v1"

cluster1-controlplane ~ ✖ helm uninstall atlanta-page-apd -n atlanta-page-04
release "atlanta-page-apd" uninstalled