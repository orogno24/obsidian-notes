
# ✅ MinIO Operator 기반 MinIO 클러스터 Distributed 모드  구축 전체 정리

---

## Helm으로 MinIO Operator 설치

### 🔹 Helm Chart values.yaml 사전 확인 및 수정

```bash
# namespace 생성
kubectl create namespace minio-operator

# MinIO Operator Helm 차트 설치 (최신 버전 확인)
helm repo add minio https://operator.min.io/
helm repo update

# MinIO 밸류만 추출
helm show values minio-operator/operator > minio-operator-values.yaml
```

👉 여기서 quay.io/minio/operator-sidecar:v7.0.1 (사이드카) 이미지 수정

### 🔹 Helm Chart 설치

```bash
helm install minio-operator minio-operator/operator \
  -n minio-operator \
  --create-namespace \
  -f minio-operator-values.yaml
```
### 🔹 설치 확인

```bash
k get po -n minio-operator

NAME                              READY   STATUS    RESTARTS   AGE
minio-operator-7c4bc4dc87-4t8sb   1/1     Running   0          98m
minio-operator-7c4bc4dc87-s8j2p   1/1     Running   0          98m
```

### 🔹 문제

만약 Helm 설치 중 `already exists` 에러가 발생한다면:
- 기존 CRD(`tenants.minio.min.io` 등)가 남아 있었음
- Helm이 생성하지 않은 리소스를 Helm이 관리하려 해서 충돌

### 🔹 해결 방법

```bash
# 기존 리소스 제거
kubectl delete crd tenants.minio.min.io
kubectl delete crd policybindings.sts.min.io
kubectl delete clusterrole minio-operator-role
kubectl delete clusterrolebinding minio-operator-binding
```

---

## MinIO Tenant용 Secret 생성

MinIO Operator v2는 **Secret 이름 규칙**을 기반으로 자동 사용자 등록 수행

```bash
kubectl create secret generic minio-tenant-env-configuration \ --from-literal=config.env="export MINIO_ROOT_USER=minio export MINIO_ROOT_PASSWORD=minio123" \ --namespace minio-operator
```

---

## Tenant YAML 정의 및 생성

```yaml
apiVersion: minio.min.io/v2
kind: Tenant
metadata:
  name: minio-tenant
  namespace: minio-operator
spec:
  # MinIO 서버 이미지 설정 (독립망에서 교체 필요)
  image: minio/minio:RELEASE.2024-12-18T13-15-44Z
  
  # - 사전에 생성한 Secret을 참조
  configuration:
    name: minio-tenant-env-configuration
  
  # MinIO 서버 풀 구성 (Distributed 모드 설정)
  pools:
    - servers: 4              # MinIO Pod 개수 (4개 이상으로 해야 Distributed 모드 적용)
      volumesPerServer: 4     # 각 서버당 할당할 볼륨 개수
      name: pool-0            # 풀 이름 (여러 풀 운영 시 구분용)
      
      # 각 볼륨에 대한 PVC(PersistentVolumeClaim) 템플릿
      volumeClaimTemplate:
        metadata:
          name: data          # PVC 이름 prefix (실제로는 data-0, data-1 등으로 생성됨)
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 10Gi   # 각 볼륨의 용량 (필요에 따라 조정)
          # 환경에 맞는 StorageClass로 변경 필요
          storageClassName: "okestro-nfs-retain"
  
  # - 각 Pod 내부에서 데이터가 저장될 디렉토리 경로
  # - 기본값: /export (변경 불필요)
  mountPath: /export
  
  # 외부 서비스 노출 설정
  exposeServices:
    minio: true       # MinIO API 서비스 노출 (S3 호환 API, 포트 9000)
    console: true     # MinIO Console 웹 UI 노출 (관리 인터페이스, 포트 9001)
```

```
kubectl apply -f minio-tenant.yaml
```

---

## Tenant 및 리소스 상태 확인

```bash
kubectl get tenant -n minio-operator
kubectl get pods -n minio-operator
kubectl get pvc -n minio-operator
```

---

## MinIO CLI `mc`로 접속 확인

```bash
ps aux | grep minio

sudo systemctl status minio

mc alias set minio http://localhost:30015(minio address) minioadmin minioadmin

mc ls minio
```

---
## Ingress 설정 (선택사항)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minio-ingress
  namespace: minio-tenant
spec:
  rules:
    - host: minio.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: minio
                port:
                  number: 80
```



