### Harbor란?

Harbor는 **엔터프라이즈급 프라이빗 컨테이너 레지스트리**입니다. Docker Hub와 달리 사내 네트워크에 직접 설치하여 완전한 보안과 제어를 제공합니다.

### 주요 특징

- **완전한 프라이빗 운영**: 외부 접근 차단 가능
- **강력한 보안**: RBAC, 이미지 서명, 취약점 스캔
- **정책 관리**: 복제, 감사 로그, 정책 제어
- **CI/CD 통합**: Jenkins, GitLab 등과 유연한 연동

---

## 🔧 사전 준비사항

### 1. 네임스페이스 생성

```bash
kubectl create namespace op-common
```

### 2. NFS 스토리지 구성

#### NFS 서버 설치 (Ubuntu)

```bash
# NFS 서버 패키지 설치
sudo apt-get update
sudo apt-get install -y nfs-kernel-server

# 공유 폴더 구성
sudo mkdir -p /srv/nfs/k8s
sudo chmod -R 777 /srv/nfs/k8s
sudo chown -R $USER:$USER /srv/nfs/k8s

# NFS 공유 설정
echo "/srv/nfs/k8s *(rw,sync,no_root_squash,no_subtree_check)" | sudo tee -a /etc/exports
sudo exportfs -rav
sudo systemctl restart nfs-kernel-server
```

#### 방화벽 설정 (GCP)

```bash
gcloud compute firewall-rules create allow-nfs \
  --allow tcp:2049,tcp:111,udp:111,udp:2049 \
  --target-tags=k8s-master \
  --source-ranges=10.0.0.0/8
```

#### NFS Provisioner 설치

```bash
# Helm 저장소 추가
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update

# NFS Provisioner values 파일 생성
cat > nfs-values.yaml <<EOF
nfs:
  server: 10.178.0.3  # NFS 서버 IP 주소
  path: /srv/nfs/k8s
storageClass:
  name: nfs
  defaultClass: true
  reclaimPolicy: Retain
EOF

# NFS Provisioner 설치
helm install nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  -f nfs-values.yaml -n kube-system
```

### 3. 스토리지 구조 이해

|구성 요소|역할|
|---|---|
|**StorageClass**|"NFS 방식으로 만들고, 경로는 `/storage` 밑으로!" 라고 규칙을 정함|
|**PVC**|"저장소 주세요!" 라고 요청함|
|**PV**|"그 저장소는 여기 있어요 → `/storage/pvc-xyz`" 라고 위치 안내함|
|**Pod**|PVC를 통해 `/storage/pvc-xyz`를 실제 마운트해서 데이터 저장함|
|**NFS 서버**|그 모든 데이터가 진짜로 저장되는 물리적인 장소|

---

## 🔐 Harbor 설치

### 1. 인증서 생성

#### v3.ext 파일 작성

```bash
cat <<EOF | tee v3.ext
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1=harbor.oke.com
EOF
```

#### 인증기관(CA) 인증서 생성

```bash
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650 \
  -subj "/C=KR/ST=Seoul/L=Seoul/O=Example/OU=Cloud Native/CN=Example Root CA" \
  -key ca.key -out ca.crt
```

#### Harbor 서버 인증서 생성

```bash
openssl genrsa -out registry.example.com.key 4096
openssl req -sha512 -new \
  -subj "/C=KR/ST=Seoul/L=Seoul/O=Example/OU=Cloud Native/CN=registry.example.com" \
  -key registry.example.com.key -out registry.example.com.csr
openssl x509 -req -sha512 -days 3650 -extfile v3.ext \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -in registry.example.com.csr -out registry.example.com.crt
```

#### Kubernetes Secret 생성

```bash
kubectl create secret tls harbor-tls \
  --cert=registry.example.com.crt \
  --key=registry.example.com.key \
  -n op-common
```

### 2. Harbor Helm 차트 준비

```bash
# Helm 저장소 추가
helm repo add harbor https://helm.goharbor.io
helm repo update

# Harbor values 파일 다운로드
helm show values harbor/harbor > values.yaml
```

### 3. Harbor Values 설정

```yaml
# values.yaml 주요 설정
expose:
  type: ingress
  tls:
    enabled: true
    certSource: secret
    secret:
      secretName: "harbor-tls"
  ingress:
    hosts:
      core: registry.example.com
    className: "nginx"
    annotations:
      nginx.ingress.kubernetes.io/ssl-redirect: "true"
      nginx.ingress.kubernetes.io/proxy-body-size: "0"
      
# 내부 DNS 주소 사용 (중요!)
externalURL: http://harbor-core.op-common.svc.cluster.local:80

# 영구 스토리지 설정
persistence:
  enabled: true
  persistentVolumeClaim:
    registry:
      storageClass: "nfs"
      accessMode: ReadWriteOnce
      size: 5Gi
    chartmuseum:
      storageClass: "nfs"
      accessMode: ReadWriteOnce
      size: 5Gi
    jobservice:
      jobLog:
        storageClass: "nfs"
        accessMode: ReadWriteOnce
        size: 1Gi
      scanDataExports:
        storageClass: "nfs"
        accessMode: ReadWriteOnce
        size: 1Gi
    database:
      storageClass: "nfs"
      accessMode: ReadWriteOnce
      size: 1Gi
    redis:
      storageClass: "nfs"
      accessMode: ReadWriteOnce
      size: 1Gi
    trivy:
      storageClass: "nfs"
      accessMode: ReadWriteOnce
      size: 5Gi

# Redis 내부 사용 설정 (연결 오류 방지)
redis:
  type: internal
```

### 4. Harbor 설치 실행

```bash
# Harbor 설치
helm install harbor harbor/harbor -n op-common -f values.yaml

# 설치 확인
kubectl get pods -n op-common
kubectl get ingress -n op-common

# 롤아웃 상태 확인
kubectl -n op-common rollout status deploy/harbor-core
```

---

## 🌐 설치 후 구성

### 1. 호스트 설정

#### 클러스터 노드 설정

```bash
# 각 클러스터 노드에서 실행 (마스터 노드 IP 사용)
sudo sh -c "echo '10.178.0.3 registry.example.com' >> /etc/hosts"
```

#### 로컬 PC 설정 (Windows)

```bash
# C:\Windows\System32\drivers\etc\hosts 파일에 추가
34.64.207.236 registry.example.com
```

### 2. 인증서 적용 (Ubuntu 노드)

```bash
# 시스템 인증서 저장소에 추가
sudo cp registry.example.com.crt /usr/local/share/ca-certificates/
sudo cp ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates

# Containerd에 인증서 적용
sudo mkdir -p /etc/containerd/certs.d/registry.example.com:30191
sudo cp registry.example.com.crt /etc/containerd/certs.d/registry.example.com:30191/ca.crt
sudo systemctl restart containerd
```

### 3. Harbor 접속

```bash
# 웹 브라우저에서 접속
https://registry.example.com:30191/

# 기본 로그인 정보
사용자명: admin
비밀번호: Harbor12345
```

---

## ⚙️ Helm 관리 명령어

### 기본 조회 명령어

```bash
# 설치된 릴리스 목록 확인
helm list
helm list -A  # 모든 네임스페이스 포함

# 특정 릴리스의 values 확인
helm get values harbor -n op-common                 # 변경된 값만
helm get values harbor -n op-common -a              # 모든 값 포함
helm get values harbor -n op-common -o yaml > harbor_values.yaml  # 파일로 저장

# 릴리스 전체 정보 확인
helm get all harbor -n op-common                    # 매니페스트 포함 모든 정보
helm get chart harbor -n op-common                  # 차트 정보만
```

### 관리 명령어

```bash
# 릴리스 업그레이드
helm upgrade harbor harbor/harbor -n op-common -f values.yaml

# 릴리스 삭제
helm uninstall harbor -n op-common

# 특정 버전으로 롤백
helm rollback harbor 1 -n op-common
```

### 실용 예시

```bash
# Harbor의 현재 설정 백업
helm get values harbor -n op-common -o yaml > harbor-backup-$(date +%Y%m%d).yaml

# 새 설정으로 업그레이드
helm upgrade harbor harbor/harbor -n op-common -f new-values.yaml

# 업그레이드 실패 시 이전 버전으로 롤백
helm rollback harbor -n op-common
```

---

## 🔧 트러블슈팅

### 일반적인 문제들

#### 1. Redis 연결 오류

**증상**: Harbor 서비스들이 Redis에 연결하지 못함

**해결책**:

```yaml
# values.yaml에서 내부 Redis 사용 설정
redis:
  type: internal
```

#### 2. PVC 바인딩 오류

**증상**: PersistentVolumeClaim이 Pending 상태

**원인 분석**:

- StorageClass가 올바르지 않음
- NFS Provisioner가 정상 작동하지 않음
- NFS 서버 연결 문제

**해결책**:

```bash
# StorageClass 확인
kubectl get storageclass

# NFS Provisioner 상태 확인
kubectl get pods -n kube-system | grep nfs

# PVC 상태 확인
kubectl get pvc -n op-common

# PVC 재생성
helm upgrade harbor harbor/harbor -n op-common -f values.yaml
```

#### 3. 인증서 문제

**증상**: HTTPS 접속 시 "인증서가 신뢰할 수 없음" 오류

**해결책**:

```bash
# 인증서 재생성 후 Secret 업데이트
kubectl delete secret harbor-tls -n op-common
kubectl create secret tls harbor-tls \
  --cert=registry.example.com.crt \
  --key=registry.example.com.key \
  -n op-common

# Harbor 재시작
kubectl rollout restart deployment/harbor-core -n op-common
```

#### 4. 네트워크 접근 문제

**증상**: 클러스터 내부에서 Harbor에 접근할 수 없음

**점검 사항**:

```bash
# Service 상태 확인
kubectl get svc -n op-common

# DNS 해결 테스트
nslookup harbor-core.op-common.svc.cluster.local

# Ingress 상태 확인
kubectl get ingress -n op-common
kubectl describe ingress harbor-ingress -n op-common
```

### 로그 확인 방법

```bash
# Harbor Core 로그
kubectl logs deployment/harbor-core -n op-common

# Harbor Registry 로그
kubectl logs deployment/harbor-registry -n op-common

# Harbor Portal 로그
kubectl logs deployment/harbor-portal -n op-common

# 모든 Harbor 관련 Pod 로그
kubectl logs -l app=harbor -n op-common --tail=100
```

---

## ⚖️ Docker Hub vs Harbor

### 기능 비교

|항목|Docker Hub|Harbor|
|---|---|---|
|**공개/비공개**|공개 기본, 비공개는 제한적|**완전한 프라이빗 레지스트리**|
|**보안 제어**|제한적 (팀 권한 관리 정도)|**LDAP/AD 인증, 세밀한 접근 제어, 이미지 서명**|
|**네트워크 위치**|클라우드에 있음|**사내 네트워크에 직접 설치**|
|**사용자/권한 관리**|단순한 팀 권한|**세분화된 역할 기반 접근 제어(RBAC)**|
|**CI/CD 통합**|지원되지만 제약 있음|**GitLab, Jenkins 등과 유연한 통합**|
|**정책 제어**|이미지 만료 정책 등 제한|**레지스트리 정책, 복제, 감사 로그**|
|**취약점 스캔**|기본적인 스캔|**Trivy 통합 고급 보안 스캔**|
|**비용**|무료/유료 플랜|**설치형으로 라이선스 비용 없음**|

### 사용 시나리오

#### Docker Hub가 적합한 경우:

- 오픈소스 프로젝트
- 개인 개발자 프로젝트
- 퍼블릭 이미지 공유
- 간단한 CI/CD 파이프라인

#### Harbor가 적합한 경우:

- **기업 환경의 프라이빗 이미지 관리**
- **보안이 중요한 프로덕션 환경**
- **세밀한 접근 권한 제어가 필요한 경우**
- **컴플라이언스 요구사항이 있는 환경**
- **내부 네트워크에서만 접근해야 하는 경우**

### 마이그레이션 고려사항

```bash
# Docker Hub에서 Harbor로 이미지 마이그레이션
docker pull your-app:latest
docker tag your-app:latest registry.example.com/library/your-app:latest
docker push registry.example.com/library/your-app:latest

# CI/CD 파이프라인에서 Harbor 사용
# Jenkinsfile 예시
stage('Push to Harbor') {
    steps {
        script {
            docker.withRegistry('https://registry.example.com', 'harbor-credentials') {
                docker.image("${IMAGE_NAME}:${BUILD_NUMBER}").push()
            }
        }
    }
}
```

---

## 📋 요약 체크리스트

### 설치 전 체크리스트

- [ ] NFS 서버 설정 완료
- [ ] NFS Provisioner 설치 확인
- [ ] 네임스페이스 생성
- [ ] 인증서 생성 및 Secret 생성

### 설치 후 체크리스트

- [ ] Harbor Pod 정상 실행 확인
- [ ] 웹 UI 접속 확인
- [ ] Docker 로그인 테스트
- [ ] 이미지 push/pull 테스트
- [ ] 백업 설정 확인

### 운영 체크리스트

- [ ] 정기적인 백업 스케줄 설정
- [ ] 모니터링 및 알림 설정
- [ ] 사용자 권한 관리
- [ ] 취약점 스캔 정책 설정
- [ ] 디스크 사용량 모니터링

---

## 📚 참고 자료

- [Harbor 공식 문서](https://goharbor.io/docs/)
- [Harbor Helm 차트](https://github.com/goharbor/harbor-helm)
- [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [NFS Provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)

---

**Tags:** #harbor #registry #kubernetes #docker #security #helm #nfs #storage #cicd