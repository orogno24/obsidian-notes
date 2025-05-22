## 🔍 과정 요약

### 1️⃣ **정보 수집** 🕵️

```bash
kubectl describe pod/etcd-controlplane -n kube-system | grep '\-\-'
--advertise-client-urls=https://192.5.120.11:2379
--cert-file=/etc/kubernetes/pki/etcd/server.crt
--client-cert-auth=true
--data-dir=/var/lib/etcd
--experimental-initial-corrupt-check=true
```

- ETCD 파드에서 필요한 인증서 경로와 설정 정보 추출
- 엔드포인트, CA 인증서, 클라이언트 인증서 위치 확인

### 2️⃣ **명령어 구성** 🔧

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db
```

### 3️⃣ **백업 실행** 💾

- etcdctl 클라이언트가 HTTPS로 ETCD 서버(포트 2379)에 연결
- 보안 인증서로 안전한 통신 보장
- ETCD의 전체 데이터를 스냅샷으로 백업 파일 생성

## 🔐 **보안 요소들**

- **CA 인증서**: ETCD 서버가 진짜인지 확인
- **클라이언트 인증서**: etcdctl이 접근 권한이 있는지 증명
- **개인키**: 암호화된 통신으로 데이터 보호

**결과**: `/opt/etcd-backup.db` 파일에 전체 클러스터 상태가 안전하게 백업됨! ✅