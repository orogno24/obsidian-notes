## 📋 **목차**

1. [[#1️⃣ 경로 문제|🔍 Kubelet 바이너리 경로 문제]]
2. [[#2️⃣ 설정 오타 문제|📝 config.yaml 오타 문제]]
3. [[#3️⃣ 인증서 만료 문제|🔐 인증서 만료 문제]]
4. [[#4️⃣ 포트 충돌 문제|🚪 포트 중복 사용 문제]]
5. [[#5️⃣ 권한 문제|👤 파일 권한/소유권 문제]]

---
## 1️⃣ **🔍 Kubelet 바이너리 경로 문제**

### 🚨 **증상**

```bash
$ kubectl get nodes
NAME     STATUS     ROLES    AGE   VERSION
node01   NotReady   <none>   5m    v1.28.0

$ sudo systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled)
   Active: failed (Result: exit-code) since Mon 2024-01-15 10:30:22 UTC; 2min ago
  Process: 12345 ExecStart=/usr/local/bin/kubelet $KUBELET_ARGS (code=exited, status=2)
Main PID: 12345 (code=exited, status=2)

$ journalctl -u kubelet
Jan 15 10:30:22 node01 systemd[1]: kubelet.service: Main process exited, code=exited, status=2/INVALIDARGUMENT
Jan 15 10:30:22 node01 systemd[1]: kubelet.service: Failed with result 'exit-code'.
Jan 15 10:30:22 node01 systemd[1]: kubelet.service: Service hold-off time over, scheduling restart.
Jan 15 10:30:22 node01 systemd[1]: kubelet.service: Scheduled restart job
Jan 15 10:30:22 node01 systemd[1]: Started kubelet: The Kubernetes Node Agent.
Jan 15 10:30:22 node01 kubelet[12346]: /usr/local/bin/kubelet: no such file or directory
```

### 🔍 **원인 분석**

```bash
# 실제 kubelet 위치 확인
$ which kubelet
/usr/bin/kubelet

$ whereis kubelet  
kubelet: /usr/bin/kubelet

# 서비스 파일에서 잘못된 경로 확인
$ cat /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# 문제 라인 ↓
ExecStart=/usr/local/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_EXTRA_ARGS
```

### ✅ **해결 방법**

```bash
# 1. 올바른 경로로 수정
$ sudo vim /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf

# 수정 내용:
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_EXTRA_ARGS

# 2. 데몬 리로드 및 재시작
$ sudo systemctl daemon-reload
$ sudo systemctl restart kubelet

# 3. 확인
$ sudo systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled)
   Active: active (running) since Mon 2024-01-15 10:35:22 UTC; 10s ago

$ kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
node01   Ready    <none>   10m   v1.28.0
```

---

## 2️⃣ **📝 config.yaml 오타 문제**

### 🚨 **증상**

```bash
$ kubectl get nodes
NAME     STATUS     ROLES    AGE   VERSION
node01   NotReady   <none>   3m    v1.28.0

$ journalctl -u kubelet -f
Jan 15 10:45:33 node01 kubelet[15234]: E0115 10:45:33.123456   15234 run.go:74] "command failed" err="failed to load kubelet config file /var/lib/kubelet/config.yaml, error failed to read kubelet config file \"/var/lib/kubelet/config.yaml\", error: yaml: line 25: found character that cannot start any token"
Jan 15 10:45:33 node01 kubelet[15234]: F0115 10:45:33.123789   15234 server.go:266] failed to run Kubelet: failed to load kubelet config file /var/lib/kubelet/config.yaml
```

### 🔍 **원인 분석**

```bash
$ sudo cat /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
cgroupDriver: systemd
cgroupsPerQOS: true
enforceNodeAllocatable:
- pods
eventRecordQPS: 0
maxOpenFiles: 1000000
maxPods: 110
registryPullQPS: 10
registryBurst: 20
enableControllerAttachDetach: true
failSwapOn: false
authentication:
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
  webhook:
    enabled: true
authorization:
  mode: Webhook
clusterDNS:
  - 10.96.0.10
# 문제 라인 ↓ (잘못된 들여쓰기)
clusterDomain cluster.local
serverTLSBootstrap: true
```

### ✅ **해결 방법**

```bash
# 1. YAML 문법 오류 수정
$ sudo vim /var/lib/kubelet/config.yaml

# 수정 내용 (25번째 줄):
clusterDomain: cluster.local    # 콜론(:) 추가

# 2. YAML 문법 검증
$ python3 -c "import yaml; yaml.safe_load(open('/var/lib/kubelet/config.yaml'))"
# 오류 없으면 아무것도 출력되지 않음

# 3. kubelet 재시작
$ sudo systemctl restart kubelet

# 4. 확인
$ journalctl -u kubelet --no-pager | tail -5
Jan 15 10:50:22 node01 kubelet[16789]: I0115 10:50:22.456789 16789 server.go:853] "Started kubelet"
Jan 15 10:50:22 node01 kubelet[16789]: I0115 10:50:22.456950 16789 server.go:127] "Starting to listen" address="0.0.0.0" port=10250

$ kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
node01   Ready    <none>   8m    v1.28.0
```

---

## 3️⃣ **🔐 인증서 만료 문제**

### 🚨 **증상**

```bash
$ kubectl get nodes
NAME     STATUS     ROLES    AGE   VERSION
node01   NotReady   <none>   15m   v1.28.0

$ journalctl -u kubelet -f
Jan 15 11:00:15 node01 kubelet[18234]: E0115 11:00:15.123456 18234 certificate_manager.go:385] Failed while requesting a signed certificate from the master: cannot create certificate signing request: certificatesigningrequests.certificates.k8s.io is forbidden: User "system:node:node01" cannot create resource "certificatesigningrequests" in API group "certificates.k8s.io" at the cluster scope: client certificate expired
Jan 15 11:00:15 node01 kubelet[18234]: E0115 11:00:15.123789 18234 auth.go:65] Unable to authenticate the request due to an error: x509: certificate has expired or is not yet valid
```

### 🔍 **원인 분석**

```bash
# 인증서 만료 날짜 확인
$ sudo openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -text -noout | grep -A2 Validity
        Validity
            Not Before: Jan 15 09:00:00 2024 GMT
            Not After : Jan 15 10:30:00 2024 GMT    # ← 이미 만료됨!

# 현재 시간 확인
$ date
Mon Jan 15 11:05:22 UTC 2024    # ← 만료 시간 지남
```

### ✅ **해결 방법**

```bash
# 1. 만료된 인증서 삭제
$ sudo rm -f /var/lib/kubelet/pki/kubelet-client-current.pem
$ sudo rm -f /var/lib/kubelet/pki/kubelet-client-*.pem

# 2. kubelet 재시작 (새 인증서 자동 생성)
$ sudo systemctl restart kubelet

# 3. 새 인증서 생성 확인
$ sudo ls -la /var/lib/kubelet/pki/
total 16
drwxr-xr-x 2 root root 4096 Jan 15 11:10 .
drwxr-xr-x 6 root root 4096 Jan 15 10:15 ..
-rw------- 1 root root 1220 Jan 15 11:10 kubelet-client-2024-01-15-11-10-30.pem
lrwxrwxrwx 1 root root   59 Jan 15 11:10 kubelet-client-current.pem -> /var/lib/kubelet/pki/kubelet-client-2024-01-15-11-10-30.pem

# 4. 새 인증서 유효 기간 확인
$ sudo openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -text -noout | grep -A2 Validity
        Validity
            Not Before: Jan 15 11:10:00 2024 GMT
            Not After : Jan 16 11:10:00 2024 GMT    # ← 1일 연장됨

# 5. 확인
$ kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
node01   Ready    <none>   20m   v1.28.0
```

---

## 4️⃣ **🚪 포트 중복 사용 문제**

### 🚨 **증상**

```bash
$ kubectl get nodes
NAME     STATUS     ROLES    AGE   VERSION
node01   NotReady   <none>   2m    v1.28.0

$ journalctl -u kubelet -f
Jan 15 11:15:33 node01 kubelet[20123]: F0115 11:15:33.123456 20123 server.go:266] failed to run Kubelet: listen tcp 0.0.0.0:10250: bind: address already in use
Jan 15 11:15:33 node01 systemd[1]: kubelet.service: Main process exited, code=exited, status=1/FAILURE
```

### 🔍 **원인 분석**

```bash
# 10250 포트 사용 중인 프로세스 확인
$ sudo netstat -tlnp | grep :10250
tcp6       0      0 :::10250               :::*                    LISTEN      19876/kubelet

$ sudo ps aux | grep 19876
root     19876  0.5  2.1 1789456 87312 ?       Sl   11:10   0:15 /usr/bin/kubelet --config=/var/lib/kubelet/config.yaml

# 중복 kubelet 프로세스 확인
$ sudo ps aux | grep kubelet
root     19876  0.5  2.1 1789456 87312 ?       Sl   11:10   0:15 /usr/bin/kubelet --config=/var/lib/kubelet/config.yaml
root     20123  0.0  0.1  12345  4567 ?        Ss   11:15   0:00 /usr/bin/kubelet --config=/var/lib/kubelet/config.yaml
```

### ✅ **해결 방법**

```bash
# 1. 기존 kubelet 프로세스 강제 종료
$ sudo pkill -f kubelet

# 2. 포트 해제 확인
$ sudo netstat -tlnp | grep :10250
# 아무것도 출력되지 않아야 함

# 3. systemd로 정상 재시작
$ sudo systemctl restart kubelet

# 4. 프로세스 정상 확인
$ sudo ps aux | grep kubelet
root     21234  1.2  2.1 1789456 87312 ?       Sl   11:20   0:05 /usr/bin/kubelet --config=/var/lib/kubelet/config.yaml

$ sudo netstat -tlnp | grep :10250
tcp6       0      0 :::10250               :::*                    LISTEN      21234/kubelet

# 5. 확인
$ kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
node01   Ready    <none>   7m    v1.28.0
```

---

## 5️⃣ **👤 파일 권한/소유권 문제**

### 🚨 **증상**

```bash
$ kubectl get nodes
NAME     STATUS     ROLES    AGE   VERSION
node01   NotReady   <none>   5m    v1.28.0

$ journalctl -u kubelet -f
Jan 15 11:25:15 node01 kubelet[22345]: F0115 11:25:15.123456 22345 server.go:266] failed to run Kubelet: failed to load kubelet config file /var/lib/kubelet/config.yaml, error failed to read kubelet config file "/var/lib/kubelet/config.yaml", error: open /var/lib/kubelet/config.yaml: permission denied
```

### 🔍 **원인 분석**

```bash
# 파일 권한 확인
$ sudo ls -la /var/lib/kubelet/config.yaml
---------- 1 student student 1234 Jan 15 11:20 /var/lib/kubelet/config.yaml

# kubelet 프로세스 실행 사용자 확인
$ sudo ps aux | grep kubelet
root     22345  0.0  0.1  12345  4567 ?        Ss   11:25   0:00 /usr/bin/kubelet

# 문제: kubelet은 root로 실행되지만 파일 권한이 000 (아무도 읽을 수 없음)
```

### ✅ **해결 방법**

```bash
# 1. 파일 소유권을 root로 변경
$ sudo chown root:root /var/lib/kubelet/config.yaml

# 2. 적절한 권한 설정 (읽기 권한 추가)
$ sudo chmod 644 /var/lib/kubelet/config.yaml

# 3. 권한 변경 확인
$ sudo ls -la /var/lib/kubelet/config.yaml
-rw-r--r-- 1 root root 1234 Jan 15 11:20 /var/lib/kubelet/config.yaml

# 4. 다른 중요 파일들도 확인
$ sudo ls -la /var/lib/kubelet/pki/
total 16
drwxr-xr-x 2 root root 4096 Jan 15 11:10 .
drwxr-xr-x 6 root root 4096 Jan 15 10:15 ..
-rw------- 1 root root 1220 Jan 15 11:10 kubelet-client-current.pem  # ← 600 권한 정상

# 5. kubelet 재시작
$ sudo systemctl restart kubelet

# 6. 확인
$ journalctl -u kubelet --no-pager | tail -3
Jan 15 11:30:22 node01 kubelet[23456]: I0115 11:30:22.456789 23456 server.go:853] "Started kubelet"

$ kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
node01   Ready    <none>   10m   v1.28.0
```

---

## 🎯 **트러블슈팅 체크리스트**

### 📋 **단계별 진단 순서**

1. **[ ]** `kubectl get nodes` - 노드 상태 확인
2. **[ ]** `sudo systemctl status kubelet` - 서비스 상태 확인
3. **[ ]** `journalctl -u kubelet -f` - 실시간 로그 확인
4. **[ ]** `ps aux | grep kubelet` - 프로세스 상태 확인
5. **[ ]** `which kubelet` / `whereis kubelet` - 바이너리 위치 확인

### 🔧 **문제별 해결 패턴**

|에러 메시지 키워드|문제 유형|우선 확인사항|
|---|---|---|
|`no such file or directory`|경로 문제|`which kubelet`, 서비스 파일 경로|
|`yaml: line X:`|설정 오타|`config.yaml` 문법 검증|
|`certificate expired`|인증서 만료|인증서 날짜, `/var/lib/kubelet/pki/`|
|`address already in use`|포트 충돌|`netstat -tlnp`, 중복 프로세스|
|`permission denied`|권한 문제|파일 소유권, 권한 설정|

### 💡 **CKA 시험 꿀팁**

- **로그 먼저!** `journalctl -u kubelet` 에러 메시지가 힌트
- **경로 확인:** `which kubelet`로 실제 위치 파악
- **권한 체크:** `ls -la`로 파일 권한 확인
- **프로세스 정리:** 문제 해결 안 되면 `pkill kubelet` 후 재시작
- **설정 백업:** 수정 전 `cp config.yaml config.yaml.bak`

---

## 🔗 **관련 노트**

- [[🏠 Kubernetes 파일구조 - 집비유#안방 큐블릿 아저씨 방|Kubelet 설정 파일 위치]]
- [[CKA 시험 전략]] ← 시험 시간 단축 팁
- [[Kubernetes 인증서 관리]] ← 인증서 심화

---

**🏷️ 태그:** #CKA #Kubelet #트러블슈팅 #실전예시 #에러해결