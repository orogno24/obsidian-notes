**문제 1** `ssh cka9412`에서 해결:

`cka9412`의 kubeconfig 파일 `/opt/course/1/kubeconfig`에서 다음 정보를 추출하세요:

1. 모든 kubeconfig 컨텍스트 이름을 `/opt/course/1/contexts`에 한 줄에 하나씩 작성
2. 현재 컨텍스트 이름을 `/opt/course/1/current-context`에 작성
3. 사용자 `account-0027`의 클라이언트 인증서를 base64 디코딩하여 `/opt/course/1/cert`에 작성

**문제 2** `ssh cka7968`에서 해결:

Helm을 사용하여 _네임스페이스_ `minio`에 MinIO Operator를 설치하고 _Tenant_ CRD를 구성 및 생성하세요:

1. _네임스페이스_ `minio` 생성
2. 새 _네임스페이스_에 Helm 차트 `minio/operator` 설치. Helm 릴리스 이름은 `minio-operator`
3. `/opt/course/2/minio-tenant.yaml`의 `Tenant` 리소스를 업데이트하여 `features` 하위에 `enableSFTP: true` 포함
4. `/opt/course/2/minio-tenant.yaml`에서 `Tenant` 리소스 생성

**문제 3** `ssh cka3962`에서 해결:

_네임스페이스_ `project-h800`에 `o3db-*` 이름의 _파드_ 2개가 있습니다. 프로젝트 H800 관리팀에서 리소스 절약을 위해 이를 1개 복제본으로 축소하도록 요청했습니다.

**문제 4** `ssh cka2556`에서 해결:

_네임스페이스_ `project-c13`의 모든 사용 가능한 _파드_를 확인하고, 노드의 리소스(cpu 또는 memory)가 부족할 때 먼저 종료될 가능성이 높은 파드들의 이름을 찾으세요.

_파드_ 이름들을 `/opt/course/4/pods-terminated-first.txt`에 작성하세요.

**문제 5** `ssh cka5774`에서 해결:

이전에 `api-gateway` 애플리케이션이 외부 오토스케일러를 사용했는데, 이제 _HorizontalPodAutoscaler_ (_HPA_)로 교체해야 합니다. 애플리케이션은 _네임스페이스_ `api-gateway-staging`과 `api-gateway-prod`에 다음과 같이 배포되었습니다:

```
kubectl kustomize /opt/course/5/api-gateway/staging | kubectl apply -f -
kubectl kustomize /opt/course/5/api-gateway/prod | kubectl apply -f -
```

`/opt/course/5/api-gateway`의 Kustomize 구성을 사용하여 다음을 수행하세요:

1. _ConfigMap_ `horizontal-scaling-config` 완전히 제거
2. _배포_ `api-gateway`에 대해 최소 `2`, 최대 `4` 복제본으로 `api-gateway`라는 _HPA_ 추가. 평균 CPU 사용률 `50%`에서 스케일링
3. prod에서 _HPA_는 최대 `6` 복제본이어야 함
4. 변경사항을 staging과 prod에 적용하여 클러스터에 반영

**문제 6** `ssh cka7968`에서 해결:

`safari-pv`라는 새로운 _PersistentVolume_을 생성하세요. 용량 _2Gi_, accessMode _ReadWriteOnce_, hostPath `/Volumes/Data`, storageClassName 정의 없음이어야 합니다.

다음으로 _네임스페이스_ `project-t230`에 `safari-pvc`라는 새로운 _PersistentVolumeClaim_을 생성하세요. _2Gi_ 스토리지 요청, accessMode _ReadWriteOnce_, storageClassName 정의 없음이어야 합니다. _PVC_는 _PV_에 올바르게 바인딩되어야 합니다.

마지막으로 _네임스페이스_ `project-t230`에 해당 볼륨을 `/tmp/safari-data`에 마운트하는 새로운 _배포_ `safari`를 생성하세요. 해당 _배포_의 _파드_들은 `httpd:2-alpine` 이미지여야 합니다.

**문제 7** `ssh cka5774`에서 해결:

metrics-server가 클러스터에 설치되었습니다. `kubectl`을 사용하는 두 개의 bash 스크립트를 작성하세요:

1. 스크립트 `/opt/course/7/node.sh`는 _노드_의 리소스 사용량을 보여줘야 함
2. 스크립트 `/opt/course/7/pod.sh`는 _파드_와 해당 컨테이너의 리소스 사용량을 보여줘야 함

**문제 8** `ssh cka3962`에서 해결:

동료가 노드 `cka3962-node1`이 오래된 Kubernetes 버전을 실행하고 있으며 아직 클러스터의 일부가 아니라고 알려왔습니다.

1. 노드의 Kubernetes를 컨트롤플레인의 정확한 버전으로 업데이트
2. kubeadm을 사용하여 노드를 클러스터에 추가

**문제 9** `ssh cka9412`에서 해결:

_네임스페이스_ `project-swan`에 _ServiceAccount_ `secret-reader`가 있습니다. 이 _ServiceAccount_를 사용하는 `nginx:1-alpine` 이미지의 `api-contact`라는 _파드_를 생성하세요.

_파드_에 exec하여 `curl`을 사용해 Kubernetes API에서 모든 _시크릿_을 수동으로 쿼리하세요.

결과를 `/opt/course/9/result.json` 파일에 작성하세요.

**문제 10** `ssh cka3962`에서 해결:

_네임스페이스_ `project-hamster`에 새로운 _ServiceAccount_ `processor`를 생성하세요. `processor`라는 _Role_과 _RoleBinding_도 생성하세요. 이들은 새 _SA_가 해당 _네임스페이스_에서 _시크릿_과 _ConfigMap_만 생성할 수 있도록 허용해야 합니다.

**문제 11** `ssh cka2556`에서 해결:

다음에 대해 _네임스페이스_ `project-tiger`를 사용하세요. `httpd:2-alpine` 이미지와 레이블 `id=ds-important`, `uuid=18426a0b-5f59-4e10-923f-c0e078e82462`로 `ds-important`라는 _DaemonSet_을 생성하세요. 생성되는 _파드_들은 10 millicore cpu와 10 mebibyte 메모리를 요청해야 합니다. 해당 _DaemonSet_의 _파드_들은 컨트롤플레인을 포함한 모든 노드에서 실행되어야 합니다.

**문제 12** `ssh cka2556`에서 해결:

_네임스페이스_ `project-tiger`에서 다음을 구현하세요:

- `3`개 복제본으로 `deploy-important`라는 _배포_ 생성
- _배포_와 해당 _파드_들은 레이블 `id=very-important`를 가져야 함
- `nginx:1-alpine` 이미지의 `container1`이라는 첫 번째 컨테이너
- `google/pause` 이미지의 `container2`라는 두 번째 컨테이너
- 해당 _배포_의 **하나**의 _파드_만이 **하나**의 워커 노드에서 실행되어야 함. 이를 위해 `topologyKey: kubernetes.io/hostname` 사용

**문제 13** `ssh cka7968`에서 해결:

프로젝트 r500 팀이 Ingress(networking.k8s.io)를 Gateway Api(gateway.networking.k8s.io) 솔루션으로 교체하고자 합니다. 기존 Ingress는 `/opt/course/13/ingress.yaml`에서 확인할 수 있습니다.

_네임스페이스_ `project-r500`과 이미 존재하는 _Gateway_에 대해 다음을 수행하세요:

1. 기존 Ingress의 라우트를 복제하는 `traffic-director`라는 새로운 _HTTPRoute_ 생성
2. User-Agent가 정확히 `mobile`이면 mobile로, 그렇지 않으면 desktop으로 리디렉션하는 경로 `/auto`로 새 _HTTPRoute_ 확장

**문제 14** `ssh cka9412`에서 해결:

클러스터 인증서에 대한 몇 가지 작업을 수행하세요:

1. openssl 또는 cfssl을 사용하여 kube-apiserver 서버 인증서의 유효 기간 확인. 만료 날짜를 `/opt/course/14/expiration`에 작성. `kubeadm` 명령을 실행하여 만료 날짜를 나열하고 두 방법이 동일한 날짜를 보여주는지 확인
2. kube-apiserver 인증서를 갱신하는 `kubeadm` 명령을 `/opt/course/14/kubeadm-renew-certs.sh`에 작성

**문제 15** `ssh cka7968`에서 해결:

침입자가 하나의 해킹된 백엔드 _파드_에서 전체 클러스터에 액세스할 수 있었던 보안 사고가 있었습니다.

이를 방지하기 위해 _네임스페이스_ `project-snake`에 `np-backend`라는 _NetworkPolicy_를 생성하세요. `backend-*` _파드_들이 다음에만 연결할 수 있도록 허용해야 합니다:

- 포트 `1111`에서 `db1-*` _파드_들에 연결
- 포트 `2222`에서 `db2-*` _파드_들에 연결

정책에서 `app` _파드_ 레이블을 사용하세요.

**문제 16** `ssh cka5774`에서 해결:

클러스터의 CoreDNS 구성을 업데이트해야 합니다:

1. 기존 구성 Yaml의 백업을 만들어 `/opt/course/16/coredns_backup.yaml`에 저장. 백업에서 빠르게 복구할 수 있어야 함
2. `SERVICE.NAMESPACE.custom-domain`에 대한 DNS 해상도가 `SERVICE.NAMESPACE.cluster.local`과 정확히 동일하게 작동하고 추가로 작동하도록 클러스터의 CoreDNS 구성 업데이트

**문제 17** `ssh cka2556`에서 해결:

_네임스페이스_ `project-tiger`에서 `httpd:2-alpine` 이미지의 `tigers-reunite`라는 _파드_를 레이블 `pod=container`, `container=pod`로 생성하세요. _파드_가 예약된 노드를 찾아내세요. 해당 노드에 ssh로 접속하여 해당 _파드_에 속하는 containerd 컨테이너를 찾으세요.

`crictl` 명령을 사용하여:

1. 컨테이너의 ID와 `info.runtimeType`을 `/opt/course/17/pod-container.txt`에 작성
2. 컨테이너의 로그를 `/opt/course/17/pod-container.log`에 작성

**미리보기 문제 1 | ETCD 정보**

`ssh cka9412`에서 해결:

클러스터 관리자가 `cka9412`에서 실행 중인 etcd에 대한 다음 정보를 찾아달라고 요청했습니다:

- 서버 개인 키 위치
- 서버 인증서 만료 날짜
- 클라이언트 인증서 인증이 활성화되어 있는지

이 정보들을 `/opt/course/p1/etcd-info.txt`에 작성하세요.

**미리보기 문제 2 | Kube-Proxy iptables**

`ssh cka2556`에서 해결:

kube-proxy가 올바르게 실행되고 있는지 확인하도록 요청받았습니다. _네임스페이스_ `project-hamster`에서 다음을 수행하세요:

1. `nginx:1-alpine` 이미지로 _파드_ `p2-pod` 생성
2. 포트 `3000->80`에서 클러스터 내부적으로 _파드_를 노출하는 _서비스_ `p2-service` 생성
3. 생성된 _서비스_ `p2-service`에 속하는 노드 `cka2556`의 iptables 규칙을 `/opt/course/p2/iptables.txt` 파일에 작성
4. _서비스_를 삭제하고 iptables 규칙이 다시 사라졌는지 확인

**미리보기 문제 3 | Service CIDR 변경**

`ssh cka9412`에서 해결:

1. `httpd:2-alpine` 이미지를 사용하여 _네임스페이스_ `default`에 `check-ip`라는 _파드_ 생성
2. 포트 `80`에서 ClusterIP _서비스_ `check-ip-service`로 노출. 해당 _서비스_의 IP 기억/출력
3. 클러스터의 Service CIDR을 `11.96.0.0/12`로 변경
4. 동일한 _파드_를 가리키는 `check-ip-service2`라는 두 번째 _서비스_ 생성

> ℹ️ 두 번째 _서비스_는 새 CIDR 범위에서 IP 주소를 가져야 합니다