물론입니다. 방금 진행하신 것처럼 **"인터넷망 또는 VPN으로 연결된 클러스터 간 Velero 백업/복원"** 환경에 적합한 **📦 일반망 기반 Velero + MinIO 이관 가이드**를 아래에 구성해드릴게요.

---

# 🌐 일반망 기반 Velero-MinIO 리소스 이관 가이드

> **두 클러스터 간 네트워크 통신 가능(VPN, 사설망, 또는 공인망)**  
> ✅ MinIO는 **한쪽(보통 백업 클러스터)에만 설치**됨  
> ✅ 다른 클러스터는 **MinIO에 원격으로 연결하여 복원만 수행**

---

## 📌 구성 개요

|구성 요소|위치|역할|
|---|---|---|
|MinIO|📍 원본 클러스터 (예: KTC)|백업 데이터 저장소|
|Velero|양쪽 모두 (KTC + PPP)|백업 및 복원 도구|
|통신|두 클러스터 간 통신 가능해야 함|MinIO에 접근 가능해야 함|

---

## ✅ 1. 원본 클러스터(KTC)에 MinIO 설치

> [이미 진행한 MinIO 설치 방법과 동일]

```bash
# MinIO binary 설치
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio && sudo mv minio /usr/local/bin/

# MinIO 데이터 디렉토리
sudo mkdir -p /data/minio
sudo useradd -r minio-user -s /sbin/nologin
sudo chown -R minio-user:minio-user /data/minio

# 환경변수 설정
sudo tee /etc/default/minio <<EOF
MINIO_VOLUMES="/data/minio"
MINIO_OPTS="--console-address :9001"
MINIO_ROOT_USER="minioadmin"
MINIO_ROOT_PASSWORD="minioadmin"
EOF

# systemd 등록
sudo tee /etc/systemd/system/minio.service <<EOF
[Unit]
Description=MinIO
After=network.target
[Service]
User=minio-user
Group=minio-user
EnvironmentFile=/etc/default/minio
ExecStart=/usr/local/bin/minio server \$MINIO_VOLUMES \$MINIO_OPTS
Restart=always
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now minio
```

---

## ✅ 2. MinIO Client(mc) 및 Velero 설치

```bash
# mc 설치
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc && sudo mv mc /usr/local/bin/mc

# MinIO 연결
mc alias set minio http://localhost:9000 minioadmin minioadmin
mc mb minio/velero

# 사용자 및 권한
mc admin user add minio veleroaccess veleropass123
mc admin policy attach minio readwrite --user veleroaccess
```

---

## ✅ 3. 원본 클러스터(KTC) Velero 설치

```bash
# Velero binary
VERSION=v1.16.1
curl -L -o velero.tar.gz https://github.com/vmware-tanzu/velero/releases/download/${VERSION}/velero-${VERSION}-linux-amd64.tar.gz
tar -xvzf velero.tar.gz && sudo mv velero-${VERSION}-linux-amd64/velero /usr/local/bin/
```

### Velero 자격증명 파일

```bash
cat <<EOF > minio.credentials
[default]
aws_access_key_id = veleroaccess
aws_secret_access_key = veleropass123
EOF

kubectl create namespace op-inspection
kubectl create secret generic minio.credentials --namespace op-inspection --from-file=./minio.credentials
```

### Velero 설치

```bash
velero install \
--provider aws \
--plugins velero/velero-plugin-for-aws \
--bucket velero \
--backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://<MINIO IP>:9000 \
--secret-file ./minio.credentials \
--namespace op-inspection \
--use-node-agent \
--default-volumes-to-fs-backup
```

---

## ✅ 4. 백업 수행

```bash
velero backup create nexus \
--include-namespaces cicd \
--selector "app.kubernetes.io/instance=nexus" \
--namespace op-inspection \
--default-volumes-to-fs-backup
```

---

## ✅ 5. 복원 대상 클러스터(PPP)에 Velero 설치만 수행

> MinIO는 설치 ❌, **MinIO의 IP를 직접 사용해서 Velero 설치**

```bash
# velero binary 설치
# 동일하게 설치

# minio.credentials 작성
cat <<EOF > minio.credentials
[default]
aws_access_key_id = veleroaccess
aws_secret_access_key = veleropass123
EOF

kubectl create namespace op-inspection
kubectl create secret generic minio.credentials --namespace op-inspection --from-file=./minio.credentials
```

### 📌 **주의:** MinIO IP는 반드시 KTC의 MinIO 서버 주소여야 함

```bash
velero install \
--provider aws \
--plugins velero/velero-plugin-for-aws \
--bucket velero \
--backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://<KTC의 MinIO IP>:9000 \
--secret-file ./minio.credentials \
--namespace op-inspection \
--use-node-agent \
--default-volumes-to-fs-backup
```

---

## ✅ 6. 복원 수행 (PPP)

```bash
velero backup get -n op-inspection        # 백업 목록 확인
velero restore create nexus --from-backup nexus
```

---

## ✅ 7. 복원 상태 확인

```bash
velero restore get -n op-inspection
velero restore logs nexus -n op-inspection
```

---

## ✅ 8. 데이터 확인

```bash
kubectl get pv                            # PV 바운드 여부
kubectl get pods -n <네임스페이스>
kubectl exec -n <네임스페이스> -it pod/<pod명> -- /bin/bash
du -sh /nexus-data                        # 실제 복원된 데이터 크기 확인
```

---

## 🧩 정리

|구분|설치 위치|설명|
|---|---|---|
|**MinIO**|KTC (한쪽만)|백업 저장소 역할|
|**Velero**|양쪽 모두|백업/복원 실행|
|**백업 전송**|필요 없음|PPP는 직접 KTC MinIO에 접근하여 복원|
|**전제 조건**|네트워크 통신 가능해야 함|사설망, VPN, 또는 공인망으로 가능|

---

필요하시면 이걸 **Markdown 파일이나 PDF 포맷**으로 내보내드릴 수도 있어요.  
더 세분화된 NFS 설정이나 ingress 연동도 추가 가능하니 말씀 주세요!