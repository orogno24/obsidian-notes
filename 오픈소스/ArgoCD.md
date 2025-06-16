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
    

---

필요하다면 이 내용을 문서용 포맷이나 Wiki에 붙이기 좋은 형식으로도 정리해드릴 수 있어요.