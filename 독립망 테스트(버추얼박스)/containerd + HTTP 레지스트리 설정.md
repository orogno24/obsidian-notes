## 💡 1. hosts.toml 만들기

**모든 노드에 동일하게 적용**

```bash
sudo mkdir -p /etc/containerd/certs.d/10.100.0.102:32415
```

`hosts.toml` 파일 생성:

```bash
sudo tee /etc/containerd/certs.d/10.100.0.102:32415/hosts.toml <<EOF
server = "http://10.100.0.102:32415"

[host."http://10.100.0.102:32415"]
  capabilities = ["pull", "resolve", "push"]
EOF
```

✅ 참고:

- 이 파일이 “이 레지스트리는 HTTP다”라고 containerd에 알려줍니다.
- `skip_verify` 옵션은 HTTPS에서만 필요하니 안 써도 됩니다.

---

## 💡 2. `/etc/containerd/config.toml` 초기화 및 수정

먼저 기존 파일 백업:

```bash
sudo mv /etc/containerd/config.toml /etc/containerd/config.toml.bak.$(date +%s)
```

기본 config.toml 생성:

```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

파일 열기:

```bash
sudo vi /etc/containerd/config.toml
```

**아래 부분만 찾아서 수정** (주석 참고):

```
[plugins."io.containerd.grpc.v1.cri".registry]
  config_path = "/etc/containerd/certs.d"
```

✅ **위 한 줄만 넣으면 충분**합니다.

🔍 찾기 팁 (vi 편집기에서):

- `/registry]` 로 검색 후 아래에 `config_path` 넣으면 됩니다.

---

## 💡 3. containerd 재시작

```bash
sudo systemctl restart containerd
```

---

## 💡 4. 정상 동작 확인

먼저 플러그인 상태 확인:

```bash
sudo ctr plugin ls | grep cri
```

결과가 이렇게 나오면 OK:

```
io.containerd.grpc.v1.cri   cri                   ok
```

이제 이미지 풀 테스트:

```bash
sudo crictl pull 10.100.0.102:32415/my-docker-image/busybox:latest
```

성공 메시지 예시:

```
Image is up to date for sha256:...
```

---

## 💡 5. Kubernetes 파드 테스트

```bash
kubectl run test-busybox --image=10.100.0.102:32415/my-docker-image/busybox --restart=Never -- sleep 3600
kubectl get pods
```

`Running` 상태 확인:

```
NAME           READY   STATUS    RESTARTS   AGE
test-busybox   1/1     Running   0          10s
```

---

## 🔄 다른 노드에 반복

모든 노드에서 **이 과정을 그대로 반복**합니다.

---

## ✨ 요약

✅ 반드시 필요한 것:  
1️⃣ `hosts.toml`에 `server = "http://…"`  
2️⃣ `config.toml`에 `config_path = "/etc/containerd/certs.d"`  
3️⃣ `systemctl restart containerd`  
4️⃣ `crictl pull` 성공 여부 확인

이렇게 하면 Kubernetes가 HTTP 레지스트리에서 이미지를 문제없이 풀합니다.