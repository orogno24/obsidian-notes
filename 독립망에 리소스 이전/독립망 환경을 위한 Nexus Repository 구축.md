## 📋 목표

인터넷이 차단된 독립망 환경에서도 Docker 이미지와 오픈소스를 설치할 수 있도록 Nexus Repository를 구축하고 백업/복원하는 방법

## 🏗️ 전체 프로세스

1. 운영 환경의 Nexus에 필요한 이미지 Push
2. Nexus PV 데이터 백업 (tar 압축)
3. VirtualBox 클러스터에 Nexus 설치
4. 백업 데이터 복원
5. 인터넷 차단 상태에서 이미지 Pull 테스트

---

## 1. 운영 환경 Nexus 백업

### 1.1 Nexus Pod 및 PV 확인

```bash
# Nexus Pod 확인
kubectl get pods -n cicd | grep nexus

# PV 마운트 경로 확인
kubectl describe pod -n cicd <nexus-pod-name> | grep -A 5 "Mounts:"
# 결과: /nexus-data

# PV/PVC 확인
kubectl get pv,pvc -n cicd
```

### 1.2 백업 실행

```bash
# Nexus Pod 접속
kubectl exec -it -n cicd <nexus-pod-name> -- /bin/bash

# 데이터 용량 확인
du -sh /nexus-data
df -h

# tar로 압축 (gzip이 없는 경우)
cd /
tar -cpf /tmp/nexus-data-backup.tar nexus-data/
exit

# 로컬로 복사
kubectl cp cicd/<nexus-pod-name>:/tmp/nexus-data-backup.tar ./nexus-data-backup.tar
```

### 1.3 백업용 Pod 사용 (권장)

```yaml
# backup-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nexus-backup-pod
  namespace: cicd
spec:
  containers:
  - name: backup
    image: ubuntu:22.04
    command: ['sleep', '7200']
    volumeMounts:
    - name: nexus-data
      mountPath: /nexus-data
  volumes:
  - name: nexus-data
    nfs:
      server: <nfs-server-ip>
      path: /nas/cicd/nexus-nexus-repository-manager-data
```

```bash
kubectl apply -f backup-pod.yaml
kubectl exec -it -n cicd nexus-backup-pod -- /bin/bash
apt update && apt install -y tar gzip
cd /
tar -czpf /tmp/nexus-data-backup.tar.gz nexus-data/
exit
kubectl cp cicd/nexus-backup-pod:/tmp/nexus-data-backup.tar.gz ./nexus-data-backup.tar.gz
kubectl delete pod -n cicd nexus-backup-pod
```

---

## 2. VirtualBox 클러스터에 Nexus 설치

### 2.1 values.yaml 준비

```yaml
# nexus-values-virtualbox.yaml
image:
  repository: sonatype/nexus3
  tag: 3.64.0

ingress:
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
  enabled: true
  hostPath: /
  hostRepo: nexus.virtualbox.local
  ingressClassName: nginx
  tls:
  - hosts:
    - nexus.virtualbox.local
    secretName: nexus-tls-secret

nexus:
  env:
  - name: INSTALL4J_ADD_VM_PARAMS
    value: |-
      -Xms4096M -Xmx4096M
      -XX:MaxDirectMemorySize=4096M
      -XX:+UnlockExperimentalVMOptions
      -XX:+UseCGroupMemoryLimitForHeap
      -Djava.util.prefs.userRoot=/nexus-data/javaprefs
  - name: NEXUS_SECURITY_RANDOMPASSWORD
    value: "true"
  nexusPort: 8081
  securityContext:
    fsGroup: 200
    runAsGroup: 200
    runAsUser: 200

persistence:
  accessMode: ReadWriteOnce
  enabled: true
  storageSize: 10Gi
  storageClass: ""

service:
  enabled: true
  name: nexus3
  type: NodePort
  nodePort: 30081
```

### 2.2 PV 생성

```yaml
# nexus-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nexus-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  hostPath:
    path: /data/nexus
```

```bash
# 디렉토리 생성 (모든 노드)
sudo mkdir -p /data/nexus
sudo chmod 777 /data/nexus

# PV 생성
kubectl apply -f nexus-pv.yaml
```

### 2.3 Nexus 설치

```bash
# namespace 생성
kubectl create namespace cicd

# TLS Secret 생성
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nexus-tls.key -out nexus-tls.crt \
  -subj "/CN=nexus.virtualbox.local"
kubectl create secret tls nexus-tls-secret \
  --cert=nexus-tls.crt --key=nexus-tls.key -n cicd

# Helm 설치
helm repo add sonatype https://sonatype.github.io/helm3-charts/
helm repo update
helm install nexus sonatype/nexus-repository-manager \
  --namespace cicd \
  --version 64.2.0 \
  -f nexus-values-virtualbox.yaml

# Pod가 Running 상태가 될 때까지 대기
kubectl get pods -n cicd -w

# 초기 비밀번호 확인
kubectl exec -n cicd nexus-nexus-repository-manager-65f7d7ddd4-r7l6g -- cat /nexus-data/admin.password
```

---

## 3. 백업 데이터 복원

### 3.1 백업 파일 업로드

```bash
# VirtualBox 노드로 파일 복사
/tmp 경로로 버추얼박스 UI를 통해 파일 이동
```

### 3.2 데이터 복원

```bash
# Nexus 중지
kubectl scale deployment -n cicd nexus-nexus-repository-manager --replicas=0

# 복원용 Pod 생성
cat <<EOF > restore-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nexus-restore-pod
  namespace: cicd
spec:
  containers:
  - name: restore
    image: ubuntu:22.04
    command: ['sleep', '3600']
    volumeMounts:
    - name: nexus-data
      mountPath: /nexus-data
  volumes:
  - name: nexus-data
    persistentVolumeClaim:
      claimName: nexus-nexus-repository-manager-data
EOF

kubectl apply -f restore-pod.yaml

# 백업 파일 복사 및 복원
kubectl cp /tmp/nexus-data-backup.tar.gz cicd/nexus-restore-pod:/tmp/
kubectl exec -it -n cicd nexus-restore-pod -- bash
cd /
tar -xzpf /tmp/nexus-data-backup.tar.gz
chown -R 200:200 /nexus-data/
exit

# 정리 및 재시작
kubectl delete pod -n cicd nexus-restore-pod
kubectl scale deployment -n cicd nexus-nexus-repository-manager --replicas=1
```

---

## 4. Docker 이미지 Pull 설정

### 4.1 서비스 확인

```bash
kubectl get svc -n cicd nexus-nexus-repository-manager
# 결과: 8081:30081/TCP,9900:32415/TCP
# 8081 = Nexus UI (NodePort: 30081)
# 9900 = Docker Registry (NodePort: 32415)
```

### 4.2 Docker 설정 (모든 노드)

```bash
# insecure registry 추가
sudo vi /etc/docker/daemon.json
```

```json
{
  "insecure-registries": ["<node-ip>:32415", "localhost:32415"]
}
```

```bash
sudo systemctl restart docker
```

### 4.3 Docker 로그인

```bash
# 노드 IP 확인
kubectl get nodes -o wide

# Docker 로그인
docker login <node-ip>:32415
Username: admin
Password: <admin-password>
```

### 4.4 이미지 Pull

```bash
# 이미지 구조: <registry>/<path>:<tag>
docker pull <node-ip>:32415/my-docker-image/jaeger-agent:1.62.0

# 확인
docker images | grep jaeger
```

---

## 5. 인터넷 차단 테스트

### 5.1 인터넷 차단

```bash
# iptables로 외부 트래픽 차단
sudo iptables -A OUTPUT -o <external-interface> -j DROP

# 또는 네트워크 케이블 제거
```

### 5.2 오프라인 상태에서 이미지 Pull

```bash
# Nexus에서 이미지 Pull (인터넷 없이)
docker pull <node-ip>:32415/my-docker-image/jaeger-agent:1.62.0

# Kubernetes Deployment 예시
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: jaeger-agent
        image: <node-ip>:32415/my-docker-image/jaeger-agent:1.62.0
        imagePullPolicy: IfNotPresent
```

---

## 📝 주요 포인트

### ✅ 성공 요인

1. **포트 구분**: Nexus UI(30081)와 Docker Registry(32415) 포트가 다름
2. **경로 구조**: `<registry>/<image-path>:<tag>` 형식 준수
3. **권한 설정**: 복원 시 200:200 (nexus 사용자) 권한 설정 필수
4. **insecure-registry**: Docker daemon.json에 등록 필요

### ⚠️ 주의사항

1. **디스크 공간**: 최소 4GB 이상 여유 공간 필요
2. **메모리**: Nexus 실행에 최소 2.7GB RAM 필요
3. **네트워크**: 클러스터 내부 통신은 인터넷 차단과 무관
4. **백업 크기**: 대용량 백업 시 split 명령어 활용

### 🔧 트러블슈팅

- **manifest unknown**: 이미지 경로나 태그 확인
- **disk space**: Docker system prune, journal 로그 정리
- **memory error**: JVM 힙 메모리 증가
- **permission denied**: chown -R 200:200 실행
