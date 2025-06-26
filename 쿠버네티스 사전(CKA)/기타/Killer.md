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
## 🔧 Question 11 | DaemonSet on all Nodes

**인스턴스**: `ssh cka2556`

### 📝 문제

`project-tiger` Namespace에서 다음을 생성하세요:

- **DaemonSet 이름**: `ds-important`
- **이미지**: `httpd:2-alpine`
- **레이블**: `id=ds-important`, `uuid=18426a0b-5f59-4e10-923f-c0e078e82462`
- **리소스 요청**: CPU 10m, 메모리 10Mi
- **스케줄링**: 모든 노드(controlplane 포함)에서 실행

### 💡 해답

```bash
ssh cka2556

# Deployment 템플릿으로 시작하여 DaemonSet으로 변경
k -n project-tiger create deployment --image=httpd:2-alpine ds-important --dry-run=client -o yaml > 11.yaml
```

**DaemonSet YAML 수정**:

```yaml
# 11.yaml
apiVersion: apps/v1
kind: DaemonSet                                     # Deployment에서 변경
metadata:
  labels:                                           # 추가
    id: ds-important                                # 추가
    uuid: 18426a0b-5f59-4e10-923f-c0e078e82462      # 추가
  name: ds-important
  namespace: project-tiger                          # 중요
spec:
  #replicas: 1                                      # 제거 (DaemonSet은 replicas 없음)
  selector:
    matchLabels:
      id: ds-important                              # 추가
      uuid: 18426a0b-5f59-4e10-923f-c0e078e82462    # 추가
  #strategy: {}                                     # 제거
  template:
    metadata:
      labels:
        id: ds-important                            # 추가
        uuid: 18426a0b-5f59-4e10-923f-c0e078e82462  # 추가
    spec:
      containers:
      - image: httpd:2-alpine
        name: ds-important
        resources:
          requests:                                 # 추가
            cpu: 10m                                # 추가
            memory: 10Mi                            # 추가
      tolerations:                                  # 추가 (controlplane에서 실행하기 위함)
      - effect: NoSchedule                          # 추가
        key: node-role.kubernetes.io/control-plane  # 추가
```

**적용 및 확인**:

```bash
k -f 11.yaml create

# DaemonSet 상태 확인
k -n project-tiger get ds
k -n project-tiger get pod -l id=ds-important -o wide
```

### 📚 개념 정리

- **DaemonSet**: 각 노드에 정확히 하나의 Pod를 실행
- **Toleration**: Taint가 있는 노드에서도 Pod 실행을 허용
- **controlplane Taint**: `node-role.kubernetes.io/control-plane:NoSchedule`

---

## 🔧 Question 12 | Deployment on all Nodes

**인스턴스**: `ssh cka2556`

### 📝 문제

`project-tiger` Namespace에서 다음을 구현하세요:

- **Deployment 이름**: `deploy-important` (3 replicas)
- **레이블**: `id=very-important`
- **컨테이너 1**: `container1` (nginx:1-alpine)
- **컨테이너 2**: `container2` (google/pause)
- **제약조건**: 하나의 worker 노드에는 하나의 Pod만 실행 (`topologyKey: kubernetes.io/hostname`)

### 💡 해답

**템플릿 생성**:

```bash
ssh cka2556
k -n project-tiger create deployment --image=nginx:1-alpine deploy-important --dry-run=client -o yaml > 12.yaml
```

**TopologySpreadConstraints 사용**

```yaml
# 12.yaml (대안)
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    id: very-important
  name: deploy-important
  namespace: project-tiger
spec:
  replicas: 3
  selector:
    matchLabels:
      id: very-important
  template:
    metadata:
      labels:
        id: very-important
    spec:
      containers:
      - image: nginx:1-alpine
        name: container1
        resources: {}
      - image: google/pause
        name: container2
      topologySpreadConstraints:                 # 추가
      - maxSkew: 1                               # 추가
        topologyKey: kubernetes.io/hostname      # 추가
        whenUnsatisfiable: DoNotSchedule         # 추가
        labelSelector:                           # 추가
          matchLabels:                           # 추가
            id: very-important                   # 추가
```

**적용 및 확인**:

```bash
k -f 12.yaml create

# 결과 확인 (2/3 ready - 하나는 pending)
k -n project-tiger get deploy -l id=very-important
k -n project-tiger get pod -o wide -l id=very-important
```

### 📚 개념 정리

- **PodAntiAffinity**: 특정 레이블의 Pod와 같은 노드에 스케줄링 방지
- **TopologySpreadConstraints**: Pod를 토폴로지 도메인에 균등하게 분산
- **topologyKey**: 노드 레이블 기준으로 토폴로지 도메인 정의

---

## 🔧 Question 13 | Gateway API Ingress

**인스턴스**: `ssh cka7968`

### 📝 문제

`project-r500` Namespace에서 기존 Ingress를 Gateway API로 교체하세요:

1. 기존 Ingress(`/opt/course/13/ingress.yaml`)와 동일한 라우팅을 하는 HTTPRoute `traffic-director` 생성
2. `/auto` 경로 추가: User-Agent가 `mobile`이면 mobile로, 아니면 desktop으로 라우팅

**테스트 명령어**:

```bash
curl r500.gateway:30080/desktop
curl r500.gateway:30080/mobile
curl r500.gateway:30080/auto -H "User-Agent: mobile" 
curl r500.gateway:30080/auto
```

### 💡 해답

**Step 1: 기존 Ingress 분석**

```yaml
# /opt/course/13/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: traffic-director
spec:
  ingressClassName: nginx
  rules:
    - host: r500.gateway
      http:
        paths:
          - backend:
              service:
                name: web-desktop
                port:
                  number: 80
            path: /desktop
            pathType: Prefix
          - backend:
              service:
                name: web-mobile
                port:
                  number: 80
            path: /mobile
            pathType: Prefix
```

**Step 2: HTTPRoute 생성**

```yaml
# traffic-director.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: traffic-director
  namespace: project-r500
spec:
  parentRefs:
    - name: main   # 기존 Gateway 이름
  hostnames:
    - "r500.gateway"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /desktop
      backendRefs:
        - name: web-desktop
          port: 80
    - matches:
        - path:
            type: PathPrefix
            value: /mobile
      backendRefs:
        - name: web-mobile
          port: 80
    # /auto 경로 - User-Agent: mobile인 경우
    - matches:
        - path:
            type: PathPrefix
            value: /auto
          headers:
          - type: Exact
            name: user-agent
            value: mobile
      backendRefs:
        - name: web-mobile
          port: 80
    # /auto 경로 - 기본값 (desktop)
    - matches:
        - path:
            type: PathPrefix
            value: /auto
      backendRefs:
        - name: web-desktop
          port: 80
```

**적용 및 테스트**:

```bash
ssh cka7968
k -n project-r500 apply -f traffic-director.yaml

# 테스트
curl r500.gateway:30080/desktop
curl r500.gateway:30080/mobile
curl -H "User-Agent: mobile" r500.gateway:30080/auto
curl r500.gateway:30080/auto
```

### 📚 개념 정리

- **Gateway API**: Ingress의 차세대 대안, 더 유연하고 확장 가능
- **HTTPRoute**: HTTP 트래픽 라우팅 규칙 정의
- **Rule 순서**: 규칙 순서가 중요 (첫 번째 매치되는 규칙 적용)

---

## 🔧 Question 14 | Check Certificate Validity

**인스턴스**: `ssh cka9412`

### 📝 문제

클러스터 인증서 관련 작업을 수행하세요:

1. `openssl` 또는 `cfssl`로 kube-apiserver 인증서 만료일 확인하여 `/opt/course/14/expiration`에 저장
2. `kubeadm` 명령어로 만료일 확인하여 두 방법이 같은 결과를 보이는지 확인
3. kube-apiserver 인증서 갱신 명령어를 `/opt/course/14/kubeadm-renew-certs.sh`에 저장

### 💡 해답

**Step 1: 인증서 파일 찾기**

```bash
ssh cka9412
sudo -i

# apiserver 관련 인증서 확인
find /etc/kubernetes/pki | grep apiserver
```

**Step 2: openssl로 만료일 확인**

```bash
# 인증서 정보 확인
openssl x509 -noout -text -in /etc/kubernetes/pki/apiserver.crt | grep Validity -A2

# 결과 예시:
#         Validity
#             Not Before: Oct 29 14:14:27 2024 GMT
#             Not After : Oct 29 14:19:27 2025 GMT
```

**만료일 저장**:

```text
# /opt/course/14/expiration
Oct 29 14:19:27 2025 GMT
```

**Step 3: kubeadm으로 확인**

```bash
# 모든 인증서 만료일 확인
kubeadm certs check-expiration | grep apiserver

# 결과 예시:
# apiserver                  Oct 29, 2025 14:19 UTC   356d    ca         no      
# apiserver-etcd-client      Oct 29, 2025 14:19 UTC   356d    etcd-ca    no      
# apiserver-kubelet-client   Oct 29, 2025 14:19 UTC   356d    ca         no 
```

**Step 4: 갱신 명령어 저장**

```bash
# /opt/course/14/kubeadm-renew-certs.sh
kubeadm certs renew apiserver
```

### 📚 인증서 관리 명령어

```bash
# 모든 인증서 만료일 확인
kubeadm certs check-expiration

# 모든 인증서 갱신
kubeadm certs renew all

# 특정 인증서만 갱신
kubeadm certs renew apiserver
kubeadm certs renew apiserver-kubelet-client
```

---

## 🔧 Question 15 | NetworkPolicy

**인스턴스**: `ssh cka7968`

### 📝 문제

보안 사고 방지를 위해 NetworkPolicy를 생성하세요:

**정책 이름**: `np-backend` (`project-snake` Namespace)
**허용 규칙**: `backend-*` Pod들이 다음으로만 연결 가능

- `db1-*` Pod의 1111 포트
- `db2-*` Pod의 2222 포트

**차단 대상**: `backend-*` Pod에서 `vault-*` Pod의 3333 포트 접속

### 💡 해답

**Step 1: 현재 상태 확인**

```bash
ssh cka7968

# Pod 및 레이블 확인
k -n project-snake get pod -L app
k -n project-snake get pod -o wide

# 현재 연결 테스트
k -n project-snake exec backend-0 -- curl -s 10.44.0.25:1111  # db1
k -n project-snake exec backend-0 -- curl -s 10.44.0.23:2222  # db2
k -n project-snake exec backend-0 -- curl -s 10.44.0.22:3333  # vault (차단되어야 함)
```

**Step 2: NetworkPolicy 생성**

```yaml
# 15_np.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: np-backend
  namespace: project-snake
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress                    # Egress만 제어
  egress:
    -                           # 첫 번째 규칙
      to:                           # 목적지 조건
      - podSelector:
          matchLabels:
            app: db1
      ports:                        # 포트 조건
      - protocol: TCP
        port: 1111
    -                           # 두 번째 규칙
      to:                           # 목적지 조건
      - podSelector:
          matchLabels:
            app: db2
      ports:                        # 포트 조건
      - protocol: TCP
        port: 2222
```

**적용 및 테스트**:

```bash
k -f 15_np.yaml create

# 허용된 연결 테스트
k -n project-snake exec backend-0 -- curl -s 10.44.0.25:1111  # 성공
k -n project-snake exec backend-0 -- curl -s 10.44.0.23:2222  # 성공

# 차단된 연결 테스트
k -n project-snake exec backend-0 -- curl -s 10.44.0.22:3333  # 타임아웃
```

### 📚 NetworkPolicy 이해하기

**올바른 정책 (AND 조건)**:

```yaml
egress:
  - to:
    - podSelector:
        matchLabels:
          app: db1
    ports:
    - protocol: TCP
      port: 1111
```

→ `app=db1` **AND** `port=1111`

**잘못된 정책 (OR 조건)**:

```yaml
egress:
  - to:
    - podSelector:
        matchLabels:
          app: db1
    - podSelector:
        matchLabels:
          app: db2
    ports:
    - protocol: TCP
      port: 1111
    - protocol: TCP
      port: 2222
```

→ `(app=db1 OR app=db2)` **AND** `(port=1111 OR port=2222)`

---

## 🔧 Question 16 | Update CoreDNS Configuration

**인스턴스**: `ssh cka5774`

### 📝 문제

CoreDNS 설정을 업데이트하세요:

1. 기존 설정을 `/opt/course/16/coredns_backup.yaml`에 백업
2. `SERVICE.NAMESPACE.custom-domain`이 `SERVICE.NAMESPACE.cluster.local`과 동일하게 작동하도록 설정
3. 테스트: `nslookup kubernetes.default.svc.cluster.local`과 `nslookup kubernetes.default.svc.custom-domain` 모두 성공해야 함

### 💡 해답

**Step 1: 백업 생성**

```bash
ssh cka5774

# CoreDNS ConfigMap 백업
k -n kube-system get cm coredns -oyaml > /opt/course/16/coredns_backup.yaml
```

**Step 2: 설정 확인**

```yaml
# 기존 CoreDNS 설정
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        # ... 나머지 설정
    }
```

**Step 3: 설정 업데이트**

```bash
k -n kube-system edit cm coredns
```

**수정된 설정**:

```yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes custom-domain cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        # ... 나머지 설정
    }
```

**Step 4: CoreDNS 재시작 및 테스트**

```bash
# CoreDNS 재시작
k -n kube-system rollout restart deploy coredns

# 테스트 Pod 생성
k run bb --image=busybox:1 -- sh -c 'sleep 1d'

# DNS 해상도 테스트
k exec -it bb -- sh
nslookup kubernetes.default.svc.custom-domain
nslookup kubernetes.default.svc.cluster.local
```

### 📚 복구 방법

```bash
# 백업에서 복구
k delete -f /opt/course/16/coredns_backup.yaml
k apply -f /opt/course/16/coredns_backup.yaml
k -n kube-system rollout restart deploy coredns
```

---

## 🔧 Question 17 | Find Container Info with crictl

**인스턴스**: `ssh cka2556`

### 📝 문제

`project-tiger` Namespace에 Pod를 생성하고 컨테이너 정보를 조사하세요:

1. Pod `tigers-reunite` 생성 (httpd:2-alpine, 레이블: pod=container, container=pod)
2. Pod가 스케줄된 노드에 SSH 접속
3. `crictl`로 컨테이너 찾아서:
    - 컨테이너 ID와 info.runtimeType을 `/opt/course/17/pod-container.txt`에 저장
    - 컨테이너 로그를 `/opt/course/17/pod-container.log`에 저장

### 💡 해답

**Step 1: Pod 생성 및 노드 확인**

```bash
ssh cka2556

# Pod 생성
k -n project-tiger run tigers-reunite --image=httpd:2-alpine --labels "pod=container,container=pod"

# 스케줄된 노드 확인
k -n project-tiger get pod -o wide
# 결과 예시: tigers-reunite가 cka2556-node1에 스케줄됨
```

**Step 2: 워커 노드 접속 및 컨테이너 찾기**

```bash
# 워커 노드 접속
ssh cka2556-node1
sudo -i

# 컨테이너 ID 찾기
crictl ps | grep tigers-reunite
# 결과 예시: ba62e5d465ff0   a7ccaadd632cf   2 minutes ago   Running   tigers-reunite
```

**Step 3: 컨테이너 정보 수집**

```bash
# runtime 타입 확인
crictl inspect ba62e5d465ff0 | grep runtimeType
# 결과: "runtimeType": "io.containerd.runc.v2",

# 컨테이너 로그 확인
crictl logs ba62e5d465ff0
```

**Step 4: 결과 파일 생성**

```text
# /opt/course/17/pod-container.txt
ba62e5d465ff0 io.containerd.runc.v2
```

```text
# /opt/course/17/pod-container.log
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.44.0.29. Set the 'ServerName' directive globally to suppress this message
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.44.0.29. Set the 'ServerName' directive globally to suppress this message
[Tue Oct 29 15:12:57.211347 2024] [mpm_event:notice] [pid 1:tid 1] AH00489: Apache/2.4.62 (Unix) configured -- resuming normal operations
[Tue Oct 29 15:12:57.211841 2024] [core:notice] [pid 1:tid 1] AH00094: Command line: 'httpd -D FOREGROUND'
```

### 📚 crictl 주요 명령어

```bash
# 실행 중인 컨테이너 확인
crictl ps

# 모든 컨테이너 확인 (중지된 것 포함)
crictl ps -a

# 컨테이너 상세 정보
crictl inspect <container-id>

# 컨테이너 로그 확인
crictl logs <container-id>

# 이미지 목록
crictl images

# Pod 목록
crictl pods
```

---

## 🎯 주요 학습 포인트

### 고급 스케줄링

- **DaemonSet**: 모든 노드에 Pod 배포
- **Toleration**: Taint 무시하고 스케줄링
- **PodAntiAffinity**: Pod 간 반발력 설정
- **TopologySpreadConstraints**: Pod 분산 제어

### 네트워킹

- **Gateway API**: 차세대 Ingress 대안
- **HTTPRoute**: HTTP 라우팅 규칙
- **NetworkPolicy**: 네트워크 트래픽 제어
- **CoreDNS**: DNS 서버 설정

### 보안 및 인증서

- **TLS 인증서**: openssl로 확인
- **kubeadm**: 인증서 관리 도구
- **인증서 갱신**: 클러스터 보안 유지

### 컨테이너 런타임

- **crictl**: containerd 관리 도구
- **컨테이너 디버깅**: 로그 및 정보 수집

### 문제 해결 전략

1. **백업 먼저**: 중요한 변경 전 반드시 백업
2. **단계적 접근**: 복잡한 문제를 작은 단위로 분해
3. **테스트 반복**: 각 단계마다 결과 확인
4. **공식 문서 활용**: Kubernetes 문서에서 예제 참조