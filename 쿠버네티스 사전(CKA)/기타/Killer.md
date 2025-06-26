## 📋 시험 안내사항

- 각 문제는 특정 인스턴스에서 해결해야 함
- 다른 인스턴스로 이동하려면 `exit` 명령어로 메인 터미널로 돌아간 후 ssh 연결
- 필요시 `sudo -i`로 root 권한 획득

---

## 🔧 Question 1 | Contexts

**인스턴스**: `ssh cka9412`

### 📝 문제

`/opt/course/1/kubeconfig` 파일에서 다음 정보를 추출하여 저장하세요:

1. 모든 kubeconfig context 이름을 `/opt/course/1/contexts`에 한 줄씩 저장
2. 현재 context 이름을 `/opt/course/1/current-context`에 저장
3. `account-0027` 사용자의 client-certificate를 base64 디코딩하여 `/opt/course/1/cert`에 저장

### 💡 해답

**Step 1: Context 이름들 추출**

```bash
ssh cka9412

# 모든 context 확인
k --kubeconfig /opt/course/1/kubeconfig config get-contexts

# context 이름만 추출하여 저장
k --kubeconfig /opt/course/1/kubeconfig config get-contexts -oname > /opt/course/1/contexts
```

**Step 2: 현재 context 확인**

```bash
k --kubeconfig /opt/course/1/kubeconfig config current-context > /opt/course/1/current-context
```

**Step 3: 인증서 추출 및 디코딩**

```bash
# 방법 1: 수동으로 확인 후 디코딩
k --kubeconfig /opt/course/1/kubeconfig config view -o yaml --raw
echo LS0tLS1CRUdJTi... | base64 -d > /opt/course/1/cert

# 방법 2: 자동화
k --kubeconfig /opt/course/1/kubeconfig config view --raw -ojsonpath="{.users[0].user.client-certificate-data}" | base64 -d > /opt/course/1/cert
```

---

## 🔧 Question 2 | MinIO Operator, CRD Config, Helm Install

**인스턴스**: `ssh cka7968`

### 📝 문제

Helm을 사용하여 MinIO Operator를 설치하고 Tenant CRD를 구성하세요:

1. `minio` Namespace 생성
2. `minio/operator` Helm chart를 `minio-operator`라는 이름으로 설치
3. `/opt/course/2/minio-tenant.yaml`에서 `features` 하위에 `enableSFTP: true` 추가
4. Tenant 리소스 생성

### 💡 해답

**Step 1: Namespace 생성**

```bash
ssh cka7968
k create ns minio
```

**Step 2: Helm Chart 설치**

```bash
# 사용 가능한 chart 확인
helm repo list
helm search repo

# MinIO operator 설치
helm -n minio install minio-operator minio/operator

# 설치 확인
helm -n minio ls
k -n minio get pod
```

**Step 3: Tenant YAML 수정**

```yaml
# /opt/course/2/minio-tenant.yaml
apiVersion: minio.min.io/v2
kind: Tenant
metadata:
  name: tenant
  namespace: minio
  labels:
    app: minio
spec:
  features:
    bucketDNS: false
    enableSFTP: true                     # 이 줄 추가
  image: quay.io/minio/minio:latest
  # ... 나머지 설정
```

**Step 4: Tenant 리소스 생성**

```bash
k -f /opt/course/2/minio-tenant.yaml apply
k -n minio get tenant
```

### 📚 개념 정리

- **Helm Chart**: Kubernetes YAML 템플릿 파일들의 패키지
- **Helm Release**: Chart가 설치된 인스턴스
- **Operator**: Kubernetes API와 통신하며 CRD와 함께 동작하는 Pod
- **CRD**: Kubernetes API의 커스텀 확장

---

## 🔧 Question 3 | Scale down StatefulSet

**인스턴스**: `ssh cka3962`

### 📝 문제

`project-h800` Namespace에 있는 `o3db-*` Pod들을 1개 replica로 스케일 다운하세요.

### 💡 해답

```bash
ssh cka3962

# Pod 확인
k -n project-h800 get pod | grep o3db

# StatefulSet 확인
k -n project-h800 get deploy,ds,sts | grep o3db

# 스케일 다운
k -n project-h800 scale sts o3db --replicas 1

# 결과 확인
k -n project-h800 get sts o3db
```

---

## 🔧 Question 4 | Find Pods first to be terminated

**인스턴스**: `ssh cka2556`

### 📝 문제

`project-c13` Namespace에서 리소스 부족 시 가장 먼저 종료될 Pod들을 찾아 `/opt/course/4/pods-terminated-first.txt`에 저장하세요.

### 💡 해답

**개념**: 리소스 요청이 없는 Pod들이 먼저 종료됨 (QoS Class: BestEffort)

```bash
ssh cka2556

kubectl get pod -n project-c13 -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass

# 방법 1: 수동 확인
k -n project-c13 describe pod | grep -A 3 -E 'Requests|^Name:'

# 방법 2: 자동화
k -n project-c13 get pod -o jsonpath="{range .items[*]} {.metadata.name}{.spec.containers[*].resources}{'\n'}"

# 방법 3: QoS Class 확인
k get pods -n project-c13 -o jsonpath="{range .items[*]}{.metadata.name} {.status.qosClass}{'\n'}"
```

**결과 파일**:

```text
# /opt/course/4/pods-terminated-first.txt
c13-3cc-runner-heavy-65588d7d6-djtv9map
c13-3cc-runner-heavy-65588d7d6-v8kf5map
c13-3cc-runner-heavy-65588d7d6-wwpb4map
```

### 📚 QoS Classes

- **Guaranteed**: requests = limits
- **Burstable**: requests < limits
- **BestEffort**: requests/limits 없음 (먼저 종료됨)

---

## 🔧 Question 5 | Kustomize configure HPA Autoscaler

**인스턴스**: `ssh cka5774`

### 📝 문제

Kustomize를 사용하여 HPA(HorizontalPodAutoscaler)를 구성하세요:

1. `horizontal-scaling-config` ConfigMap 완전 제거
2. `api-gateway` Deployment용 HPA 추가 (min: 2, max: 4, CPU: 50%)
3. prod 환경에서는 max 6 replicas
4. staging과 prod에 변경사항 적용

### 💡 해답

**Step 1: ConfigMap 제거**

```bash
ssh cka5774
cd /opt/course/5/api-gateway

# base, staging, prod의 api-gateway.yaml에서 ConfigMap 섹션 제거
vim base/api-gateway.yaml
vim staging/api-gateway.yaml  
vim prod/api-gateway.yaml
```

**Step 2: Base에 HPA 추가**

```yaml
# base/api-gateway.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 2
  maxReplicas: 4
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
---
# ... 기존 ServiceAccount와 Deployment
```

**Step 3: Prod에서 maxReplicas 오버라이드**

```yaml
# prod/api-gateway.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway
spec:
  maxReplicas: 6
---
# ... 기존 Deployment
```

**Step 4: 변경사항 적용**

```bash
# Staging 적용
k kustomize staging | kubectl diff -f -
k kustomize staging | kubectl apply -f -

# Prod 적용  
k kustomize prod | kubectl apply -f -

# 기존 ConfigMap 수동 삭제
k -n api-gateway-staging delete cm horizontal-scaling-config
k -n api-gateway-prod delete cm horizontal-scaling-config
```

---

## 🔧 Question 6 | Storage, PV, PVC, Pod volume

**인스턴스**: `ssh cka7968`

### 📝 문제

스토리지 리소스를 생성하세요:

1. PV `safari-pv`: 2Gi, ReadWriteOnce, hostPath `/Volumes/Data`
2. PVC `safari-pvc`: `project-t230` namespace에 2Gi 요청
3. Deployment `safari`: PVC를 `/tmp/safari-data`에 마운트

### 💡 해답

**Step 1: PersistentVolume 생성**

```yaml
# 6_pv.yaml
kind: PersistentVolume
apiVersion: v1
metadata:
 name: safari-pv
spec:
 capacity:
  storage: 2Gi
 accessModes:
  - ReadWriteOnce
 hostPath:
  path: "/Volumes/Data"
```

**Step 2: PersistentVolumeClaim 생성**

```yaml
# 6_pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: safari-pvc
  namespace: project-t230
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
     storage: 2Gi
```

**Step 3: Deployment 생성**

```yaml
# 6_dep.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: safari
  name: safari
  namespace: project-t230
spec:
  replicas: 1
  selector:
    matchLabels:
      app: safari
  template:
    metadata:
      labels:
        app: safari
    spec:
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: safari-pvc
      containers:
      - image: httpd:2-alpine
        name: container
        volumeMounts:
        - name: data
          mountPath: /tmp/safari-data
```

**적용 및 확인**:

```bash
k -f 6_pv.yaml create
k -f 6_pvc.yaml create
k -f 6_dep.yaml create

# Bound 상태 확인
k -n project-t230 get pv,pvc
```

---

## 🔧 Question 7 | Node and Pod Resource Usage

**인스턴스**: `ssh cka5774`

### 📝 문제

kubectl을 사용하는 bash 스크립트 2개를 작성하세요:

1. `/opt/course/7/node.sh`: Node 리소스 사용량 표시
2. `/opt/course/7/pod.sh`: Pod와 컨테이너 리소스 사용량 표시

### 💡 해답

```bash
# /opt/course/7/node.sh
kubectl top node

# /opt/course/7/pod.sh  
kubectl top pod --containers=true
```

---

## 🔧 Question 8 | Update Kubernetes Version and join cluster

**인스턴스**: `ssh cka3962`

### 📝 문제

`cka3962-node1` 노드를 업데이트하고 클러스터에 조인하세요:

1. controlplane과 동일한 Kubernetes 버전으로 업데이트
2. kubeadm을 사용하여 클러스터에 노드 추가

### 💡 해답

**Step 1: 버전 확인**

```bash
ssh cka3962
k get node  # controlplane 버전 확인 (v1.32.1)

ssh cka3962-node1
sudo -i
kubelet --version  # 현재 버전 확인
```

**Step 2: kubelet과 kubectl 업데이트**

```bash
# worker 노드에서
apt update
apt install kubectl=1.32.1-1.1 kubelet=1.32.1-1.1
service kubelet restart
```

**Step 3: 조인 토큰 생성**

```bash
# controlplane에서
sudo -i
kubeadm token create --print-join-command
```

**Step 4: 클러스터 조인**

```bash
# worker 노드에서 위에서 출력된 명령어 실행
kubeadm join 192.168.100.31:6443 --token xxx --discovery-token-ca-cert-hash sha256:xxx

# 결과 확인
k get node
```

---

## 🔧 Question 9 | Contact K8s Api from inside Pod

**인스턴스**: `ssh cka9412`

### 📝 문제

`project-swan` Namespace의 `secret-reader` ServiceAccount을 사용하는 Pod를 생성하고, Pod 내부에서 curl로 Kubernetes API를 쿼리하여 모든 Secret을 조회하세요.

### 💡 해답

**Step 1: Pod 생성**

```yaml
# 9.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: api-contact
  name: api-contact
  namespace: project-swan
spec:
  serviceAccountName: secret-reader
  containers:
  - image: nginx:1-alpine
    name: api-contact
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

**Step 2: API 호출**

```bash
k -n project-swan exec api-contact -it -- sh

# ServiceAccount 토큰 사용
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -k https://kubernetes.default/api/v1/secrets -H "Authorization: Bearer ${TOKEN}"

# 결과를 파일로 저장
curl -k https://kubernetes.default/api/v1/secrets -H "Authorization: Bearer ${TOKEN}" > result.json
exit

# 결과 복사
k -n project-swan exec api-contact -it -- cat result.json > /opt/course/9/result.json
```

**보안 개선 (CA 인증서 사용)**:

```bash
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
curl --cacert ${CACERT} https://kubernetes.default/api/v1/secrets -H "Authorization: Bearer ${TOKEN}"
```

---

## 🔧 Question 10 | RBAC ServiceAccount Role RoleBinding

**인스턴스**: `ssh cka3962`

### 📝 문제

`project-hamster` Namespace에 RBAC 설정을 생성하세요:

1. ServiceAccount `processor` 생성
2. Role `processor`: Secret과 ConfigMap 생성 권한만 허용
3. RoleBinding `processor`: SA와 Role 연결

### 💡 해답

**Step 1: ServiceAccount 생성**

```bash
ssh cka3962
k -n project-hamster create sa processor
```

**Step 2: Role 생성**

```bash
k -n project-hamster create role processor --verb=create --resource=secret --resource=configmap
```

**Step 3: RoleBinding 생성**

```bash
k -n project-hamster create rolebinding processor --role processor --serviceaccount project-hamster:processor
```

**Step 4: 권한 테스트**

```bash
# 허용되는 작업
k -n project-hamster auth can-i create secret --as system:serviceaccount:project-hamster:processor  # yes
k -n project-hamster auth can-i create configmap --as system:serviceaccount:project-hamster:processor  # yes

# 허용되지 않는 작업
k -n project-hamster auth can-i create pod --as system:serviceaccount:project-hamster:processor  # no
k -n project-hamster auth can-i delete secret --as system:serviceaccount:project-hamster:processor  # no
```

### 📚 RBAC 조합

- **Role + RoleBinding**: 단일 Namespace에서 사용 가능하고 적용
- **ClusterRole + ClusterRoleBinding**: 클러스터 전체에서 사용 가능하고 적용
- **ClusterRole + RoleBinding**: 클러스터 전체에서 사용 가능하지만 단일 Namespace에 적용
- **Role + ClusterRoleBinding**: 불가능

---
