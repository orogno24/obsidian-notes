Velero는 Kubernetes 클러스터의 데이터를 백업하고 복구하는 오픈소스 도구

**백업 목록:**

1. **쿠버네티스 리소스(메타데이터)**
    - 예: 네임스페이스, 디플로이먼트, 서비스, PVC 정의 등 YAML 리소스들
    - 즉, 클러스터의 "구성"

2. **볼륨의 실제 데이터(스냅샷 또는 파일백업)**
    - 예: Persistent Volume에 저장된 애플리케이션 데이터

### **MinIO란?**

MinIO(미니오)는 **파일을 저장하고 꺼낼 수 있는 공간(저장소)을 만드는 오픈소스 프로그램** 

사진, 동영상, 문서 등을 클라우드(예: Google Drive, Dropbox)에 저장하듯,  
MinIO는 기업이나 개발자가 **자기 서버에 그런 클라우드 저장 공간을 만들 수 있게 해주는 도구**

- **AWS S3**와 **비슷한 방식**으로 동작
- **파일(Object)을 버킷(bucket)이라는 단위로 저장
- 아주 가볍고 빠르며, 어디든 설치할 수 있어서 **개발, 테스트, 실제 서비스** 어디에나 적합함

---
### **사용 사례**

- 사내 시스템이나 로컬 환경에 **프라이빗 클라우드 저장소**를 만들고 싶을 때
- AWS S3 같은 퍼블릭 클라우드를 쓰기 부담스럽거나, **비용 절감**이 필요할 때
- 데이터 백업이나 대용량 파일 저장이 필요한 프로젝트에서

### **비유로 이해해 보기**

- 📦 MinIO = 내 사무실에 설치하는 **작은 Dropbox 서버**
- 📁 Bucket = 큰 폴더
- 🖼️ Object = 그 폴더 안에 들어 있는 파일(예: 사진, 문서 등)

---

## 🧩 **Velero에 MinIO는 왜 필요한가?**

Velero가 백업 데이터를 어딘가에 보관해야 하는데, 이걸 오브젝트 스토리지에 저장함

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

1. MinIO에서 백업 데이터를 가져온다。
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
# 천안KTC-대구PPP 리소스 이관 가이드

## 천안 KTC 백업 준비
## ✅ 1. MinIO 설치 및 설정

### ① MinIO 설치

```bash
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/
```

### ② MinIO 데이터 디렉토리 및 사용자 생성

```bash
sudo mkdir -p /data/minio
sudo useradd -r minio-user -s /sbin/nologin
sudo chown -R minio-user:minio-user /data/minio
```

### ③ 환경 변수 파일 설정

```bash
sudo mkdir -p /etc/default

sudo tee /etc/default/minio <<EOF
MINIO_VOLUMES="/data/minio"
MINIO_OPTS="--console-address :9001"
MINIO_ROOT_USER="minioadmin"
MINIO_ROOT_PASSWORD="minioadmin"
EOF
```

### ④ systemd 서비스 등록 및 실행

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

sudo systemctl daemon-reload
sudo systemctl enable --now minio
sudo systemctl status minio
```

---

## ✅ 1-2. MinIO Client (mc) 설치 및 설정

```bash
wget https://dl.min.io/client/mc/release/linux-amd64/mc
sudo mv mc /usr/local/bin/
sudo chmod +x /usr/local/bin/mc
```

### MinIO 연결 및 사용자 생성

```bash
mc alias set minio http://localhost:9000 minioadmin minioadmin
mc ls minio
mc mb minio/velero

mc admin user add minio veleroaccess veleropass123
mc admin policy attach minio readwrite --user veleroaccess
```

---

## ✅ 2. Velero 설치 및 설정

### ① Velero 바이너리 설치

```bash
VERSION=v1.16.1

curl -L -o velero.tar.gz https://github.com/vmware-tanzu/velero/releases/download/${VERSION}/velero-${VERSION}-linux-amd64.tar.gz

tar -xvzf velero.tar.gz

sudo mv velero-${VERSION}-linux-amd64/velero /usr/local/bin/

rm -rf velero-${VERSION}-linux-amd64 velero.tar.gz
```

### ② 자격 증명 파일 생성 및 Secret 등록

```bash
cat <<EOF > minio.credentials
[default]
aws_access_key_id = veleroaccess
aws_secret_access_key = veleropass123
EOF
```

```
kubectl create namespace op-inspection

kubectl create secret generic minio.credentials --namespace op-inspection --from-file=./minio.credentials
```

### ③ Velero 설치

> **MinIO가 설치된 노드의 IP 주소**로 `s3Url`을 수정하세요. 예: `http://10.100.0.101:9000`

```bash
velero install \
--provider aws \
--plugins velero/velero-plugin-for-aws \
--bucket velero \
--backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://<MINIO가 설치된 노드의 IP>:9000 \
--secret-file ./minio.credentials \
--namespace op-inspection \
--use-node-agent \
--default-volumes-to-fs-backup
```

```
kubectl get all -n op-inspection | grep velero
```

---
## 천안 KTC 백업
## ✅ 3. 백업 준비 및 수행

### ① Nexus 백업 레이블 확인

```yaml
# Deployment YAML
metadata:
  labels:
    app.kubernetes.io/instance: nexus # 벨레로에게 인식시키기 위해 추가
```

### ② PV 백업용 어노테이션 추가 (선택)

```yaml
spec:
  template:
    metadata:
      annotations:
        backup.velero.io/backup-volumes: nexus-repository-manager-data # 추가해야함
...
    volumes:
    - name: nexus-repository-manager-data # 위의 어노테이션은 볼륨명과 동일하게
```

---

## ✅ 4. 백업 명령어 실행

```bash
velero backup create nexus \
--include-namespaces cicd \
--selector "app.kubernetes.io/instance=nexus" \
--namespace op-inspection \
--default-volumes-to-fs-backup
```

| 옵션                               | 설명                                             |
| -------------------------------- | ---------------------------------------------- |
| `velero backup create nexus`     | 백업 이름을 `nexus`로 지정                             |
| `--include-namespaces <네임스페이스>`  | 선택한 네임스페이스를 백업 대상으로 지정                         |
| `--selector <"키=값">`             | nexus 리소스만 선택적으로 백업하기 위해 라벨을 사용                |
| `--namespace <네임스페이스>`           | Velero가 설치된 네임스페이스를 지정                         |
| `--default-volumes-to-fs-backup` | PVC를 CSI 대신 파일시스템 기반으로 백업(hostPath, NFS 등에 유용) |

```bash
velero backup get -n op-inspection # 백업 리스트 확인
velero backup describe nexus --namespace op-inspection # 백업 상세정보 확인
velero backup logs nexus -n op-inspection # 백업 로그 확인
```

① MinIO 백업 파일 확인
```
mc ls backups
[2025-07-04 18:04:51 KST] 4.0KiB nexus/
```

**만약 mc ls backup으로 보이지 않는다면?**

1 단계 : MinIO alias부터 확인

`mc alias list      # 등록된 alias 확인`

새 세션이라면 alias가 아예 없는 경우도 많으니, 없으면 다시 등록

`mc alias set minio http://<MinIO 가 동작하는 실제 IP>:9000 minioadmin minioadmin`

2 단계 : 버킷과 경로를 정확히 지정

- **버킷 목록 확인**
    `mc ls minio`
    여기서 `velero/` 버킷이 보이면 OK.

- **버킷 내부 확인**
    `mc ls minio/velero # velero 버킷 루트 mc ls minio/velero/backups   # 백업 목록`
    
    ➡︎ 가이드에 있던
    
    csharp
    
    복사편집
    
    `[2025-07-04 18:04:51 KST] 4.0KiB nexus/`
    
    같은 엔트리가 여기서 보여야 합니다.
    

> 당신이 실행한 `mc ls velero` 는 **로컬 파일시스템의 `./velero` 디렉터리**를 본 것이고,  
> `mc ls backups` 는 **alias 이름이 backups 인 것으로 착각**한 명령이었습니다.







② MinIO 디렉토리 이동
```
$ cd /data/minio

$ ls –la
.
..
.minio.sys
velero
```

③ velero 저장소 압축
```
sudo tar –czvf velero.tar.gz ./velero
```
※ velero.tar.gz 파일을 대구PPP로 전송

## 대구PPP 복원 준비

① MinIO Secret 확인

② Velero 설치 확인

③ PV 연결을 위한 NFS 설치 확인

④ 백업 대상의 nfs명, storageClass명, 네임스페이스 확인

⑤ velero.tar.gz 파일 복사 확인

⑥ MinIO의 볼륨위치에 velero.tar.gz 압축 해제

```
cd /data/minio
sudo tar –xzvf velero.tar.gz
```

⑦ MinIO 백업 파일 확인
```
$ mc ls backups
[2025-07-04 18:04:51 KST] 4.0KiB nexus/
```

## 대구PPP 복원

① Velero 복원
```
velero restore create nexus --from-backup nexus
```

② Velero 복원 확인
```
velero restore get –n op-inspection
NAME   BACKUP   STATUS
nexus   nexus        Completed
```
※ 상태가 Complete 시 완료

③ Velero 복원 상세정보
```
velero restore describe nexus –n op-inspection
```

④ Velero 복원 로그
```
velero restore logs nexus –n op-inspection
```

복원 후 확인

① PV 바운드 확인
```
kubectl get pv
```

② 각 서비스 동작 확인