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
# MinIO + Velero 설치 및 천안KTC-대구PPP 리소스 이관 

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
velero backup create nexus-common \
--include-namespaces op-common \
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
```
mc ls minio/velero # velero 버킷 루트
mc ls minio/velero/backups   # 백업 목록
```
    ➡︎ 가이드에 있던
    `[2025-07-04 18:04:51 KST] 4.0KiB nexus/`
    
    같은 엔트리가 여기서 보여야 함

② MinIO 디렉토리 이동
```
$ cd /data/minio

$ ls -la
.
..
.minio.sys
velero
```

③ velero 저장소 압축
```
sudo tar -czvf velero.tar.gz ./velero
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
velero backup get -n op-inspection
mc ls minio/velero/backups 
[2025-07-04 18:04:51 KST] 4.0KiB nexus/
```

## 대구PPP 복원

① Velero 복원
```
velero restore create nexus --from-backup nexus -n op-inspection
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

| 단계                             | 설명                              |
| ------------------------------ | ------------------------------- |
| **1. 천안 KTC에 MinIO 설치**        | 백업 파일을 저장할 오브젝트 스토리지 역할 (S3 대용) |
| **2. Velero 설치**               | 쿠버네티스 리소스 + 볼륨 데이터를 백업하는 도구     |
| **3. Velero로 백업 수행**           | 지정된 네임스페이스, PVC 등을 tar.gz로 저장   |
| **4. MinIO에 저장된 백업 압축**        | `.tar.gz`로 만들어 물리 전송 또는 네트워크 전송 |
| **5. 대구 PPP로 압축파일 복사**         | SFTP로 전달                        |
| **6. 대구에 MinIO + Velero 설치**   | 복원을 위한 동일한 구조 구축                |
| **7. 압축 해제 + Velero 복원 명령 실행** | 리소스 + PV 복원                     |
물론이죠! 방금 말씀하신 내용을 **정리해서 깔끔하게 설명해주는 문서 형식**으로 정리해 드릴게요. 누구나 쉽게 이해할 수 있도록 풀어쓴 버전입니다 👇

---

## 📦 클러스터 간 Velero 백업/복원 방식 비교

### ✅ [1] **일반망 (인터넷 or 사설망 연결 가능)**

> 클러스터 A와 B가 **서로 네트워크로 통신 가능**한 환경

#### 구성

- **클러스터 A**: Velero + MinIO 설치
- **클러스터 B**: Velero만 설치
- **전송 방식**: 네트워크 통신 (HTTP/S3 API)

#### 흐름

```plaintext
클러스터 B의 Velero → 클러스터 A의 MinIO 접속 → 백업 데이터 직접 읽음
```

#### S3 URL 예시:

```bash
--s3Url=http://<클러스터 A의 IP>:9000
```

#### 장점

- 간편함 (MinIO를 한 곳에만 설치)
- 실시간 전송 가능

#### 조건

- 두 클러스터가 **서로 통신 가능한 망**에 있어야 함


### 🚫 [2] **독립망 (공공망, 외부망과 완전 분리된 환경)**

> 클러스터 A와 B가 **물리적으로 분리된 망**(인터넷 불가)일 때

#### 구성

- **클러스터 A**: Velero + MinIO 설치 → 백업 → 압축(tar.gz)
- **클러스터 B (PPP)**: Velero + MinIO 설치 → tar.gz 복사 후 압축 해제 → 복원

#### 전송 방식

- **SFTP**, **보안 USB**, **망연계 시스템** 등을 통해  
    압축 파일(`velero.tar.gz`)을 오프라인 전송

#### 흐름

```plaintext
클러스터 A에서 백업 → MinIO 데이터 압축 → 전송 → PPP 클러스터에서 압축 해제 → 복원
```

#### 이유

- **독립망이라 네트워크 직접 연결이 불가능**
- Velero가 외부 MinIO에 접근할 수 없음

---

## 🧠 핵심 정리 요약표

|항목|일반망 (연결 가능)|독립망 (PPP 등)|
|---|---|---|
|MinIO 설치 위치|A 클러스터에만 설치|A, B 클러스터 모두 설치|
|Velero 설치 위치|A, B 모두|A, B 모두|
|데이터 전송 방식|네트워크 전송 (`s3Url`)|tar.gz로 오프라인 전송|
|연결 조건|IP 통신 가능해야 함|통신 안 됨 (망분리)|
|복원 방법|Velero가 직접 읽음|압축 풀고 MinIO에 복사 후 복원|

## 🎯 Nexus와의 차이점

클러스터 A에는 이런 리소스들이 존재:

- 앱(Pod, Deployment)
- 설정(ConfigMap, Secret)
- 저장된 데이터(PV)
- 이미지(컨테이너 이미지)

이걸 전부 B 클러스터로 옮겨야 하는데,  
각 도구가 **자기 역할**을 수행:

---

## 🚚 1. Velero – "이삿짐 포장, 운반, 설치 담당자"

> **클러스터 안에 있는 리소스(앱, 설정, 볼륨 데이터 등)를 통째로 백업하고, 복원하는 도구**

### 🧰 Velero의 역할은?

- **A에서 백업**
    - 네임스페이스, Deployment, PVC, Secret 등 쿠버네티스 리소스를 통째로 저장
- **PV 데이터도 같이 백업** (Restic 방식이면 파일 단위)
    
- **B에서 복원**
    - A에서 포장한 데이터를 그대로 다시 깔아줌


👉 즉, Velero는 **앱 구성 + 데이터 전체를 통째로 복제하는 도구**

---

## 📦 2. Nexus – "앱 설치에 필요한 프로그램 창고"

> **앱이 실행될 때 필요한 컨테이너 이미지를 저장하는 '이미지 저장소(Private Docker Registry)'**

### 🧰 Nexus의 역할은?

- 개발자가 만든 이미지를 여기 저장함 (`docker push`)
    
- 쿠버네티스에서 앱이 실행될 때 이 이미지가 필요하니까,  
    클러스터 B에서도 **Nexus에서 이미지를 받아야 함** (`docker pull`)
    
- 따라서 클러스터 B에서도 Nexus에 접근할 수 있어야 하고,  
    필요하다면 Nexus 자체도 같이 옮겨야 함
    

👉 즉, Nexus는 **앱이 실행되기 위해 필요한 재료(이미지)를 보관하는 창고**

---
## ✅ 요약 정리

|도구|실무에서 하는 일|비유|
|---|---|---|
|**Velero**|앱 구성, 설정, 데이터 백업/복원|이삿짐 포장 + 설치 기사|
|**Nexus**|앱 실행에 필요한 컨테이너 이미지 저장소 역할|프로그램 창고, 자재 창고|
