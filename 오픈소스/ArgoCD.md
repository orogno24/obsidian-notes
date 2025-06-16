## 🔐 Argo CD Git 저장소 URL 문법 및 인증 방식

### 📌 1. **기본 형태**

```bash
http://<username>:<password_or_token>@<git-server>/<owner>/<repo>.git
```

예시:

```bash
http://jenkins:d50f8213e4bd18fca3b99b2ecc65f6fa0d656ea8@gitea-http:3000/git-ops/frontendservice-cicd.git
```

- `jenkins`: Git 사용자 이름
- `d50f...`: Access Token (또는 비밀번호)
- `gitea-http:3000`: Git 서버 주소 (내부 도메인 또는 서비스명)
- 뒤에 `.git`까지 반드시 포함

> ✅ **토큰을 사용하는 것이 일반적인 보안 관행**입니다. 비밀번호 대신 토큰을 쓰면 사용자 비밀번호 노출 리스크를 줄일 수 있습니다.

---

### 📌 2. **Argo CD에서 Git 인증 구성 방법**

#### 🛠️ 방법 1: 저장소 URL에 직접 포함 (간편하지만 비추천)

- 실습, 개발 환경에서는 빠르게 설정 가능
    
- 하지만 **URL에 토큰이 그대로 노출**되므로 보안상 좋지 않음
    

#### 🛡️ 방법 2: Argo CD Credentials 설정 (권장)

Argo CD UI 또는 CLI에서 **저장소 인증 정보를 등록**:

```bash
argocd repo add http://gitea-http:3000/git-ops/easycook-cicd.git \
  --username jenkins \
  --password <your_token> \
  --insecure
```

- UI에서도 Settings → Repositories → Add → 인증 방식 선택 가능
- 등록 후에는 `Git URL`에 사용자명/토큰을 넣지 않아도 됨

---

### 📌 3. **Private Git 저장소 인증 종류**

|인증 방식|설명|
|---|---|
|HTTPS + 사용자/토큰|Gitea, GitHub, GitLab 등에서 일반적으로 사용|
|SSH 키|`.ssh/id_rsa` 같은 개인 키로 인증 (보안성이 높고 실무에서 많이 사용)|
|Anonymous (비공개 아님)|공개 저장소에서 인증 없이 접근 가능|

---

### 🧩 실무 적용 팁

- **인증 URL은 GitOps에 명시하지 않는 것이 좋음** → Argo CD에 등록된 credential을 활용하도록 분리
- `.argocd/config` 혹은 UI에서 **SSH 키 등록**도 고려
- 비공개 Git 서버는 **`--insecure` 플래그** 필요할 수 있음 (특히 자체 인증서 사용 시)

## ✅ Argo CD 실무 구성 및 설정 경험 요약

### 📌 1. **Git 저장소 구조**

- 하나의 모노레포(monorepo)에 여러 마이크로서비스를 디렉터리별로 구성 (`FrontendService`, `UserService` 등)
- GitOps용 리포지토리 별도 운영 (`easycook-cicd`)
- Argo CD는 각 마이크로서비스의 디렉터리 경로를 이용해 개별 `Application`으로 관리

### 📌 2. **Application 구성**

다음과 같이 각 서비스에 대한 Argo CD `Application`을 구성:

- **Git URL**: `http://gitea-http:3000/git-ops/easycook-cicd.git`
- **Target Revision**: `HEAD` (또는 브랜치명 사용 가능)
- **Path**: `FrontendService` / `UserService` 등
- **Sync Options**:
    - `CreateNamespace`: ✅
    - `Validate`: ❌
    - `PruneLast`: ✅
    - `ApplyOutOfSyncOnly`: ✅

- **Sync Policy**: `Automated`

    - `Prune`: ✅
    - `Self Heal`: ✅
    - `Allow Empty`: ❌

> 🔎 `Self Heal`: 실시간으로 쿠버네티스 리소스 상태가 Git과 달라졌을 때 자동으로 되돌리는 기능

### 📌 3. **Application 삭제 시 Propagation Policy**

- `Foreground`: 선택 (리소스를 다 지운 뒤 Application 삭제 → 종속 리소스도 모두 제거)
- `Background`, `Non-cascading`은 실무에서는 잘 쓰지 않음

---

## 🛠️ 4. CI/CD 파이프라인과 GitOps 연계

### 변경 감지 및 배포 흐름

1. **태그 기반 변경 감지**: `git describe` + `git diff` 조합으로 변경된 마이크로서비스 디렉터리만 감지
2. **서비스별 build → docker push → GitOps 매니페스트 이미지 태그 변경**
3. Argo CD가 GitOps 리포지토리의 변경을 감지하여 자동으로 `sync` 수행

### 예시:

```bash
git describe --tags --abbrev=0 HEAD~1  # 이전 태그 감지
git diff --name-only $LAST_TAG HEAD   # 변경된 서비스 디렉토리 확인
```

---

## 📄 5. GitOps 매니페스트 업데이트 방식

- 각 서비스별로 `/userservice-cicd/kustomization.yaml` 등에서 `image.tag`만 변경
- Jenkins 파이프라인 내에서 GitOps 리포지토리에 커밋 → Argo CD가 자동 감지

