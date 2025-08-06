## 🎯 문제 요구사항

**john이라는 사용자를 만들어서 development 네임스페이스에서 Pod를 관리할 수 있게 하기**

## 📋 해결 과정 (3단계)

### 1단계: 사용자 인증 설정 (CSR)

1. **이미 주어진 것들:**
```bash
ls /root/CKA/
john.key  # john의 개인키 (이미 있음)
john.csr  # john의 인증서 요청서 (이미 있음)
```

2. **CSR을 쿠버네티스가 이해할 수 있는 형태로 변환 🔄**
```bash
# john.csr을 Base64로 인코딩
cat /root/CKA/john.csr | base64 -w 0
# 결과: LS0tLS1CRUdJTi... (긴 문자열)
```

3. **쿠버네티스 CSR 객체 생성**

```yaml
# csr.yaml - 인증서 서명 요청
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john-developer
spec:
  signerName: kubernetes.io/kube-apiserver-client  # 중요!
  usages: ["digital signature", "key encipherment", "client auth"]
  request: [Base64로 인코딩된 CSR 내용]
```

```bash
# 1. 원본 CSR 파일 확인
cat /root/CKA/john.csr
# 결과: -----BEGIN CERTIFICATE REQUEST----- 로 시작하는 텍스트

# 2. Base64로 인코딩 (한 줄로 출력)
cat /root/CKA/john.csr | base64 -w 0
# 결과: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0K... (긴 문자열)
```

4. **관리자가 승인**

```bash
kubectl apply -f csr.yaml

# 승인 (이 순간 john 사용자 생성!)
kubectl certificate approve john-developer
```

### 2단계: 권한 정의 (Role)

**목적**: development 네임스페이스에서 Pod에 대한 권한 정의

```yaml
# role.yaml - 권한 규칙
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development  # 이 네임스페이스에서만 유효
  name: developer
rules:
- apiGroups: [""]  # Pod는 core API 그룹
  resources: ["pods"]
  verbs: ["create", "get", "update", "delete", "list"]  # 허용할 작업들
```

### 3단계: 사용자와 권한 연결 (RoleBinding)

**목적**: john 사용자에게 developer 역할 부여

```yaml
# rolebinding.yaml - 사용자와 역할 연결
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: development
subjects:
- kind: User
  name: john  # 연결할 사용자
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer  # 부여할 역할
  apiGroup: rbac.authorization.k8s.io
```

## 🔍 각 파일의 역할

|파일|역할|비유|
|---|---|---|
|**csr.yaml**|신분증 발급 신청|"john이라는 사람이 회사에 들어오고 싶어요"|
|**role.yaml**|직책 정의|"개발자는 개발실에서 컴퓨터를 쓸 수 있어요"|
|**rolebinding.yaml**|직책 임명|"john을 개발자로 임명합니다"|

## ✅ 확인 방법

```bash
# john이 development 네임스페이스에서 pod를 만들 수 있는지 확인
kubectl auth can-i create pods --as=john -n development
```

## 💡 핵심 포인트

1. **CSR의 signerName**: Kubernetes 1.19부터 필수로 지정해야 함(request, signerName, usages 구조)
2. **Base64 인코딩**: CSR 내용을 `cat john.csr | base64 -w 0`로 인코딩
3. **네임스페이스**: Role과 RoleBinding 모두 development 네임스페이스에 생성
4. **검증**: `kubectl auth can-i`로 권한 확인

**결과**: john 사용자가 development 네임스페이스에서만 Pod를 생성/조회/수정/삭제할 수 있게 됨! 🎉