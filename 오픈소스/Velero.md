Velero는 Kubernetes 클러스터의 데이터를 백업하고 복구하는 오픈소스 도구

**백업 목록:**

1. **쿠버네티스 리소스(메타데이터)**
    - 예: 네임스페이스, 디플로이먼트, 서비스, PVC 정의 등 YAML 리소스들
    - 즉, 클러스터의 "구성"

2. **볼륨의 실제 데이터(스냅샷 또는 파일백업)**
    - 예: Persistent Volume에 저장된 애플리케이션 데이터

---
## 🧩 **Velero에 MinIO는 왜 필요한가?**

Velero가 백업 데이터를 어딘가에 보관해야 하는데, 이걸 _오브젝트 스토리지_에 저장함

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

1. MinIO에서 백업 데이터를 가져온다.
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

---

## 🔑 **핵심 포인트**

- Velero는 _리소스 정의 + 볼륨 데이터_ 백업
- MinIO는 S3 호환 저장소 역할
- 복원 시 클러스터를 동일 상태로 되돌린다