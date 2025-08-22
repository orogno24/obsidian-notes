저장소 관리:
winscp로 독립망 클러스터에 파일 전송
마스터노드로 소스코드 압축파일 옮기고 source 디렉토리에 압축 해제함
깃랩 디렉토리는 트럼본에서 등록하면 자동으로 생성됨
이후 깃랩에 있는 각 프로젝트 주소복사 후 깃 클론 <깃랩 주소> 함
이후 압축 해제한 프로젝트별 디렉토리에서 git add, commint "Initial commit"

### 🎯 작업 개요

사내 독립망 클러스터에 오픈소스 Helm 차트와 Docker 이미지를 설치하기 위해:

✅ Nexus Repository Manager에 Helm/Docker 저장소 구성  
✅ containerd에 HTTP 레포지토리 사용 설정  
✅ Helm 차트 및 이미지 로컬 업로드/등록  
✅ 클러스터에서 설치 확인
#### 🟢 1. Nexus 저장소 구성

✅ **Helm 저장소**

- Type: hosted
- Format: helm
- URL 예: `http://nexus-host:8081/repository/okesro-helm/`

✅ **Docker 저장소**

- Type: hosted
- Format: docker
- HTTP Port: `9900`
- HTTPS Port: `9901` (비활성화 가능)
- Anonymous pull 허용 설정 체크

---

#### 🟢 2. Helm 차트 다운로드 및 업로드

✅ 외부 인터넷 환경에서:

```bash
helm pull bitnami/nginx --version 15.5.2
```

→ `nginx-15.5.2.tgz` 파일 생성

✅ Nexus Helm 저장소에 업로드

- Nexus 웹 UI → Upload
- 업로드 후 **Rebuild index** 수행

✅ Helm Repo 등록

```bash
helm repo add okestro http://nexus-host:8081/repository/okesro-helm/
helm repo update
```

---

#### 🟢 3. Docker 이미지 저장 및 로드

✅ 외부 환경에서:

```bash
helm pull 이후 tgz파일 로컬에 다운로드
```

✅ 독립망으로 tar 파일 이관 후:

```bash
다운로드한 tgz파일을 넥서스에 직접 업로드하고 독립망에 설치
```

---

#### 🟢 4. containerd HTTP 레지스트리 설정

✅ containerd는 기본적으로 HTTPS만 사용 → **HTTP 명시 필요**
✅ `/etc/containerd/config.toml` 또는 hosts.toml 설정:

**예시 (config.toml):**

```toml
[plugins."io.containerd.grpc.v1.cri".registry]
  config_path = "/etc/containerd/certs.d"
```

✅ hosts.toml 생성:

```toml
server = "http://nexus-host:9900"

[host."http://nexus-host:9900"]
  capabilities = ["pull", "resolve", "push"]
```

✅ 적용:

```bash
sudo systemctl restart containerd
```

✅ 검증:

```bash
crictl pull nexus-host:9900/bitnami/nginx:1.25.3
```

---

#### 🟢 5. Helm Chart 설치

✅ 버전 확인:

```bash
helm search repo okestro/nginx
```

✅ 설치:

```bash
helm install nginx okestro/nginx --version 15.5.2
```

✅ `--version`은 Chart Version임에 주의 (App Version 아님)

---

### 🟢 개념 정리

|용어|설명|
|---|---|
|**Helm Chart**|Kubernetes 리소스 템플릿 패키지 (.tgz)|
|**Release**|Helm Chart를 클러스터에 설치한 인스턴스|
|**Values**|설치 시 변수값|
|**Chart Version**|Helm Chart 자체 버전|
|**App Version**|차트에서 배포하는 앱 버전|

---

### 🟢 자주 발생한 문제 및 원인

✅ **에러:**

```
http: server gave HTTP response to HTTPS client
```

**원인:** containerd가 HTTPS로 접속 시도. HTTP 설정 누락.

✅ **에러:**

```
connect: connection refused
```

**원인:** Nexus Pod CrashLoopBackOff 상태 / NodePort 미연결.

✅ **CrashLoopBackOff**  
**원인:** Helm Chart에 명령 미지정 (`sleep infinity` 필요).