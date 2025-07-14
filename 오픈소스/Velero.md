Velero는 Kubernetes 클러스터의 데이터를 백업하고 복구하는 오픈소스 도구

**백업 목록:**

1. **쿠버네티스 리소스(메타데이터)**
    - 예: 네임스페이스, 디플로이먼트, 서비스, PVC 정의 등 YAML 리소스들
    - 즉, 클러스터의 "구성"

2. **볼륨의 실제 데이터(스냅샷 또는 파일백업)**
    - 예: Persistent Volume에 저장된 애플리케이션 데이터

---
## 🧩 **Velero에 MinIO는 왜 필요한가?**

Velero가 백업 데이터를 어딘가에 보관해야 하는데, 이걸 _오브젝트 스토리지_에 저장함

- MinIO는 S3 API와 호환되는 자체 호스팅 스토리지

즉, MinIO는 **백업 파일을 담아두는 저장소 역할**

---
## ⚙️ **Velero의 동작 흐름 요약**

1. **Velero 설치**
    - Velero 서버(컨트롤러)가 클러스터에 설치된다.
    - 백업/복원 명령을 받는다.

2. **스토리지 위치 설정**
    - MinIO 버킷을 만들어서 S3 호환 엔드포인트 등록
    - Velero가 이 버킷에 파일 저장한다.

3. **백업 실행**
    - Velero가 네임스페이스, 리소스 정의, 볼륨 스냅샷 정보를 수집한다.
    - tar.gz 형태로 묶어서 MinIO에 업로드한다.

4. **복원 시**
    - MinIO에서 백업 데이터를 가져와서 리소스 정의 복원
    - 볼륨 스냅샷도 되돌린다.

---

## 💡 **조금 더 구체적인 예시**

예를 들어 네임스페이스 `my-app`에 워크로드와 PVC가 있다고 가정

👉 **백업 명령**

```
velero backup create my-app-backup --include-namespaces my-app
```

1. Velero가 `my-app`에 존재하는 리소스들을 YAML로 추출
2. PVC와 연결된 볼륨의 스냅샷 생성 (스토리지 클래스가 지원하면)
3. MinIO 버킷에 백업 파일을 저장

👉 **복원 명령**

```
velero restore create --from-backup my-app-backup
```

1. MinIO에서 백업 데이터를 가져온다.
2. 리소스를 클러스터에 다시 생성한다.
3. 볼륨 데이터도 복원한다.

---

## 📦 **Velero의 주요 구성요소**

아주 간단히 요약:

- **Backup**: 지정 리소스와 볼륨을 저장
- **Restore**: Backup에서 복원
- **Volume Snapshot**: 볼륨의 상태 저장
- **Backup Storage Location**: MinIO 같은 저장소
- **Volume Snapshot Location**: 스냅샷 제공자 (클라우드, CSI 등)

## 🔑 **핵심 포인트**

- Velero는 _리소스 정의 + 볼륨 데이터_ 백업
- MinIO는 S3 호환 저장소 역할
- 복원 시 클러스터를 동일 상태로 되돌린다

---

# 🌟 MinIO + Velero 설치 및 백업/복원 가이드

## 🚀 1. MinIO 설치 및 구성

### 1.1 MinIO 바이너리 다운로드/설치

```bash
sudo apt update
sudo apt install -y wget curl
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/
```

> **MinIO 실행 파일 설치**

---

### 1.2 MinIO 사용자 및 데이터 디렉토리 준비

```bash
sudo useradd -r minio-user -s /sbin/nologin
sudo mkdir -p /mnt/data
sudo chown minio-user:minio-user /mnt/data
```

> **백업 데이터를 저장할 디렉토리 생성**

---

### 1.3 MinIO 서비스 설정

환경 변수 파일 생성:

```bash
sudo mkdir -p /etc/default
sudo tee /etc/default/minio <<EOF
MINIO_VOLUMES="/mnt/data"
MINIO_OPTS="--address 0.0.0.0 --console-address :9001"
MINIO_ROOT_USER="minioadmin"
MINIO_ROOT_PASSWORD="minioadmin"
EOF
```

서비스 파일 생성:

```bash
sudo tee /etc/systemd/system/minio.service <<EOF
[Unit]
Description=MinIO Object Storage
After=network.target

[Service]
User=minio-user
Group=minio-user
EnvironmentFile=/etc/default/minio
ExecStart=/usr/local/bin/minio server \$MINIO_VOLUMES \$MINIO_OPTS
Restart=always
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

서비스 실행 및 활성화:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now minio
```

> **MinIO를 시스템 서비스로 실행**

---

### 1.4 MinIO Client(mc) 설치

```bash
wget https://dl.min.io/client/mc/release/linux-amd64/mc
sudo mv mc /usr/local/bin/
sudo chmod +x /usr/local/bin/mc
```

> **MinIO CLI 유틸리티 설치**

---

### 1.5 MinIO 연결 및 버킷 생성

MinIO 서버 연결:

```bash
mc alias set minio http://172.27.0.31:9000 minioadmin minioadmin
```

버킷 생성:

```bash
mc mb minio/velero
```

> **백업 데이터를 저장할 버킷 준비**

---

### 1.6 Kubernetes Secret 생성

```bash
kubectl create secret generic minio-secret \
  --from-literal=accessKey=yYib3R7eoMVnOcqrD49x \
  --from-literal=secretKey=YYPBj04QeP48S10jnKDRlUFmmKCQKZPosNap9HN5
```

> **Velero에서 사용할 Access Key/Secret Key 등록**

---

### 1.7 S3 Driver 설치 (EBS CSI 예시)

```bash
helm repo add k8s-sigs https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm install s3-csi k8s-sigs/aws-ebs-csi-driver \
  --namespace kube-system
```

> **(옵션) CSI Driver 설치 - 실제 스냅샷 사용 시 필요**

---

## ☁️ 2. Velero 설치 및 구성

---

### 2.1 Velero 바이너리 다운로드/설치

```bash
VERSION=v1.12.0
curl -L -o velero.tar.gz https://github.com/vmware-tanzu/velero/releases/download/${VERSION}/velero-${VERSION}-linux-amd64.tar.gz
tar -xvzf velero.tar.gz
sudo mv velero-${VERSION}-linux-amd64/velero /usr/local/bin/
rm -rf velero-${VERSION}-linux-amd64 velero.tar.gz
```

> **Velero CLI 설치**

---

### 2.2 MinIO Credentials 파일 생성

```bash
cat <<EOF > minio.credentials
[default]
aws_access_key_id = <Access Key>
aws_secret_access_key = <Secret Key>
EOF
```

Kubernetes Secret으로 등록:

```bash
kubectl create namespace opo-inspection

kubectl create secret generic minio.credentials \
  -n op-inspection \
  --from-file=./minio.credentials
```

> **Velero가 MinIO에 인증하기 위한 자격 증명**

---

### 2.3 Velero 설치

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.6.0 \
  --bucket velero \
  --backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://172.27.0.31:9000 \
  --secret-file ./minio.credentials \
  --namespace op-inspection
```

> **MinIO를 S3 호환 스토리지로 사용하는 Velero 설치**

---

## 🔄 3. Velero 백업 및 복원

---

### 3.1 백업 생성

```bash
velero backup create backup-name --include-namespaces <namespace>
```

> **특정 네임스페이스를 백업**

백업 상태 확인:

```bash
velero backup describe backup-name
```

> **백업 상세 정보 확인**

---

### 3.2 복원 진행

복원 가능한 백업 목록 조회:

```bash
velero restore get
```

복원 실행:

```bash
velero restore create restore-name --from-backup backup-name
```

복원 상세 정보 확인:

```bash
velero restore describe restore-name
```

> **백업에서 리소스 및 볼륨 복원**

---

✅ **Tip**

- `mc ls minio/velero` 명령으로 실제 저장된 백업 파일을 볼 수 있습니다.
    
- `velero backup logs backup-name` 명령으로 로그를 확인할 수 있습니다.
    
- 네임스페이스, 라벨, 리소스 종류 등으로 백업 대상을 세부 조정할 수 있습니다.
    

---

필요하시면 **YAML 파일 예시나 Helm 설치 방법**, 혹은 **볼륨 스냅샷 연동**도 추가로 알려드리겠습니다! 🌿