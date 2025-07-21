Velero는 쿠버네티스 클러스터의 데이터를 백업하고 복구하는 오픈소스 도구입니다. 주로 다음 두 가지를 백업합니다:

1. **쿠버네티스 리소스(메타데이터)**: 네임스페이스, 디플로이먼트, 서비스, PVC 정의 등 YAML 리소스들
2. **볼륨의 실제 데이터**: Persistent Volume에 저장된 애플리케이션 데이터

## MinIO란?

MinIO는 **S3 호환 오브젝트 스토리지**를 제공하는 오픈소스 프로그램입니다. AWS S3와 유사한 방식으로 동작하며, 파일(Object)을 버킷(Bucket) 단위로 저장합니다.

### 특징
- AWS S3와 호환되는 API 제공
- 가볍고 빠르며 어디든 설치 가능
- 개발, 테스트, 실제 서비스 환경에 모두 적합

### 사용 사례
- 사내 시스템에 프라이빗 클라우드 저장소 구축
- AWS S3 대체로 비용 절감
- 데이터 백업 및 대용량 파일 저장

## Velero와 MinIO의 관계

Velero가 백업한 데이터를 저장할 곳이 필요한데, MinIO가 바로 그 저장소 역할을 합니다. MinIO는 S3 API와 호환되므로 Velero가 백업 파일을 안전하게 보관할 수 있습니다.

## Velero 동작 원리

### 1. 설치 단계
- Velero 서버(컨트롤러)가 클러스터에 설치
- 백업/복원 명령을 처리할 준비 완료

### 2. 스토리지 설정
- MinIO 버킷 생성 및 S3 호환 엔드포인트 등록
- Velero가 이 버킷에 백업 파일 저장

### 3. 백업 실행
- Velero가 네임스페이스, 리소스 정의, 볼륨 스냅샷 정보 수집
- tar.gz 형태로 압축하여 MinIO에 업로드

### 4. 복원 과정
- MinIO에서 백업 데이터를 가져와서 리소스 정의 복원
- 볼륨 스냅샷도 함께 복원

## 실제 사용 예시

### 백업 명령
```bash
velero backup create my-app-backup --include-namespaces my-app
```

이 명령은 다음과 같이 동작합니다:
1. `my-app` 네임스페이스의 모든 리소스를 YAML로 추출
2. PVC와 연결된 볼륨의 스냅샷 생성 (스토리지 클래스가 지원하는 경우)
3. MinIO 버킷에 백업 파일 저장

### 복원 명령
```bash
velero restore create --from-backup my-app-backup
```

이 명령은 다음과 같이 동작합니다:
1. MinIO에서 백업 데이터를 가져옴
2. 리소스를 클러스터에 다시 생성
3. 볼륨 데이터도 함께 복원

## Velero 주요 구성요소

- **Backup**: 지정된 리소스와 볼륨을 저장
- **Restore**: 백업에서 데이터 복원
- **Volume Snapshot**: 볼륨의 상태 저장
- **Backup Storage Location**: MinIO 같은 저장소 위치
- **Volume Snapshot Location**: 스냅샷 제공자 (클라우드, CSI 등)

---

# MinIO + Velero 설치 및 클러스터 간 리소스 이관

## 1단계: MinIO 설치 및 설정

### 1.1 MinIO 바이너리 설치
```bash
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/
```

### 1.2 데이터 디렉토리 및 사용자 생성
```bash
sudo mkdir -p /data/minio
sudo useradd -r minio-user -s /sbin/nologin
sudo chown -R minio-user:minio-user /data/minio
```

### 1.3 환경 변수 설정
```bash
sudo mkdir -p /etc/default

sudo tee /etc/default/minio <<EOF
MINIO_VOLUMES="/data/minio"
MINIO_OPTS="--console-address :9001"
MINIO_ROOT_USER="minioadmin"
MINIO_ROOT_PASSWORD="minioadmin"
EOF
```

### 1.4 systemd 서비스 등록 및 실행
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

### 1.5 MinIO Client (mc) 설치 및 설정
```bash
wget https://dl.min.io/client/mc/release/linux-amd64/mc
sudo mv mc /usr/local/bin/
sudo chmod +x /usr/local/bin/mc
```

### 1.6 MinIO 연결 및 사용자 생성
```bash
mc alias set minio http://localhost:9000 minioadmin minioadmin
mc ls minio
mc mb minio/velero

mc admin user add minio veleroaccess veleropass123
mc admin policy attach minio readwrite --user veleroaccess
```

## 2단계: Velero 설치 및 설정

### 2.1 Velero 바이너리 설치
```bash
VERSION=v1.16.1

curl -L -o velero.tar.gz https://github.com/vmware-tanzu/velero/releases/download/${VERSION}/velero-${VERSION}-linux-amd64.tar.gz

tar -xvzf velero.tar.gz
sudo mv velero-${VERSION}-linux-amd64/velero /usr/local/bin/
rm -rf velero-${VERSION}-linux-amd64 velero.tar.gz
```

### 2.2 자격 증명 파일 생성 및 Secret 등록
```bash
cat <<EOF > minio.credentials
[default]
aws_access_key_id = veleroaccess
aws_secret_access_key = veleropass123
EOF

kubectl create namespace op-inspection
kubectl create secret generic minio.credentials --namespace op-inspection --from-file=./minio.credentials
```

### 2.3 Velero 설치
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

설치 확인:
```bash
kubectl get all -n op-inspection | grep velero
```

## 3단계: 백업 준비 및 수행

### 3.1 백업 대상 리소스 준비
백업할 애플리케이션에 적절한 레이블과 어노테이션을 추가합니다.

**Deployment에 레이블 추가:**
```yaml
metadata:
  labels:
    app.kubernetes.io/instance: nexus # 넥서스에게 인식시키기 위한 라벨
```

**PV 백업용 어노테이션 추가 (선택사항):**
```yaml
spec:
  template:
    metadata:
      annotations:
        backup.velero.io/backup-volumes: nexus-repository-manager-data # 볼륨 이름
    volumes:
    - name: nexus-repository-manager-data
```

### 3.2 백업 실행
```bash
velero backup create nexus-common \
--include-namespaces op-common \
--selector "app.kubernetes.io/instance=nexus" \
--namespace op-inspection \
--default-volumes-to-fs-backup
```

**명령어 옵션 설명:**
- `velero backup create nexus-common`: 백업 이름 지정
- `--include-namespaces op-common`: 백업할 네임스페이스 지정
- `--selector "app.kubernetes.io/instance=nexus"`: 특정 라벨을 가진 리소스만 백업
- `--namespace op-inspection`: Velero가 설치된 네임스페이스
- `--default-volumes-to-fs-backup`: PVC를 파일시스템 기반으로 백업 (hostPath, NFS 등에 유용)

### 3.3 백업 상태 확인
```bash
velero backup get -n op-inspection
velero backup describe nexus-common --namespace op-inspection
velero backup logs nexus-common -n op-inspection
```

### 3.4 MinIO 백업 파일 확인
```bash
mc ls minio/velero/backups
```

만약 파일이 보이지 않는다면:
```bash
# MinIO alias 확인
mc alias list

# alias 재등록
mc alias set minio http://<MinIO IP>:9000 minioadmin minioadmin

# 버킷 확인
mc ls minio
mc ls minio/velero
```

### 3.5 백업 데이터 압축
```bash
cd /data/minio
sudo tar -czvf velero.tar.gz ./velero
```

이 `velero.tar.gz` 파일을 대상 클러스터로 전송합니다.

## 4단계: 대상 클러스터에서 복원

### 4.1 사전 준비사항 확인
- MinIO Secret 확인
- Velero 설치 확인
- MinIO 설치 확인(독립망일 경우)
- PV 연결을 위한 NFS 설치 확인
- 백업 대상의 NFS명, StorageClass명, 네임스페이스 확인
- velero.tar.gz 파일 복사 확인

### 4.2 백업 데이터 복원
```bash
cd /data/minio
sudo tar -xzvf velero.tar.gz
```

### 4.3 백업 파일 확인
```bash
velero backup get -n op-inspection
mc ls minio/velero/backups
```

### 4.4 Velero 복원 실행
```bash
velero restore create nexus --from-backup nexus-common -n op-inspection
```

### 4.5 복원 상태 확인
```bash
# 복원 상태 확인
velero restore get -n op-inspection

# 복원 상세정보
velero restore describe nexus -n op-inspection

# 복원 로그
velero restore logs nexus -n op-inspection
```

### 4.6 복원 후 확인
```bash
# PV 바운드 상태 확인
kubectl get pv

# 각 서비스 동작 확인
kubectl get pods -n <복원된 네임스페이스>
```

---

# 클러스터 간 백업/복원 방식 비교

## 일반망 환경 (네트워크 연결 가능)

클러스터 A와 B가 서로 네트워크로 통신 가능한 환경에서는 다음과 같이 구성합니다:

### 구성
- **클러스터 A**: Velero + MinIO 설치
- **클러스터 B**: Velero만 설치
- **전송 방식**: 네트워크 통신 (HTTP/S3 API)

### 흐름
```plaintext
클러스터 B의 Velero → 클러스터 A의 MinIO 접속 → 백업 데이터 직접 읽음
```

### S3 URL 설정
```bash
--s3Url=http://<클러스터 A의 IP>:9000
```

### 장점
- 간편함 (MinIO를 한 곳에만 설치)
- 실시간 전송 가능

### 조건
- 두 클러스터가 서로 통신 가능한 망에 있어야 함

## 독립망 환경 (망분리)

클러스터 A와 B가 물리적으로 분리된 망(인터넷 불가)일 때는 다음과 같이 구성합니다:

### 구성
- **클러스터 A**: Velero + MinIO 설치 → 백업 → 압축(tar.gz)
- **클러스터 B**: Velero + MinIO 설치 → tar.gz 복사 후 압축 해제 → 복원

### 전송 방식
- SFTP, 보안 USB, 망연계 시스템 등을 통해 압축 파일(`velero.tar.gz`)을 오프라인 전송

### 흐름
```plaintext
클러스터 A에서 백업 → MinIO 데이터 압축 → 전송 → PPP 클러스터에서 압축 해제 → 복원
```

### 이유
- 독립망이라 네트워크 직접 연결이 불가능
- Velero가 외부 MinIO에 접근할 수 없음

## 방식 비교 요약

| 항목           | 일반망 (연결 가능)       | 독립망 (PPP 등)          |
| ------------ | ----------------- | -------------------- |
| MinIO 설치 위치  | A 클러스터에만 설치       | A, B 클러스터 모두 설치      |
| Velero 설치 위치 | A, B 모두           | A, B 모두              |
| 데이터 전송 방식    | 네트워크 전송 (`s3Url`) | tar.gz로 오프라인 전송      |
| 연결 조건        | IP 통신 가능해야 함      | 통신 안 됨 (망분리)         |
| 복원 방법        | Velero가 직접 읽음     | 압축 풀고 MinIO에 복사 후 복원 |

---

# Velero와 Nexus의 차이점

## Velero - "이삿짐 포장, 운반, 설치 담당자"

Velero는 **클러스터 안에 있는 리소스(앱, 설정, 볼륨 데이터 등)를 통째로 백업하고, 복원하는 도구**입니다.

### Velero의 역할
- **백업**: 네임스페이스, Deployment, PVC, Secret 등 쿠버네티스 리소스를 통째로 저장
- **복원**: 백업한 데이터를 그대로 다시 클러스터에 설치

즉, Velero는 **앱 구성 + 데이터 전체를 통째로 복제하는 도구**입니다.

## Nexus - "앱 설치에 필요한 프로그램 창고"

Nexus는 **앱이 실행될 때 필요한 컨테이너 이미지를 저장하는 '이미지 저장소(Private Docker Registry)'**입니다.

### Nexus의 역할
- 개발자가 만든 이미지를 저장 (`docker push`)
- 쿠버네티스에서 앱 실행 시 필요한 이미지 제공 (`docker pull`)
- 클러스터 간 이관 시 Nexus 자체도 함께 옮겨야 할 수 있음

즉, Nexus는 **앱이 실행되기 위해 필요한 재료(이미지)를 보관하는 창고**입니다.

## 요약

| 도구         | 실무에서 하는 일                 | 비유             |
| ---------- | ------------------------- | -------------- |
| **Velero** | 앱 구성, 설정, 데이터 백업/복원       | 이삿짐 포장 + 설치 기사 |
| **Nexus**  | 앱 실행에 필요한 컨테이너 이미지 저장소 역할 | 프로그램 창고, 자재 창고 |
