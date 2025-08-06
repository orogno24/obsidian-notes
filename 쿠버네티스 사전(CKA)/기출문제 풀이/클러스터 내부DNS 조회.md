## 🎯 문제 요약

**nginx Pod와 Service를 만들고, 클러스터 내부에서 DNS가 잘 작동하는지 테스트하기**

## 📋 해결 과정

### 1단계: nginx Pod 생성

```bash
kubectl run nginx-resolver --image=nginx
```

### 2단계: Service 생성 (Pod 노출)

```bash
kubectl expose pod nginx-resolver --name=nginx-resolver-service --port=80
```

### 3단계: 클러스터 내부에서 DNS 테스트

#### Service DNS 조회 테스트

```bash
kubectl run test-nslookup --image=busybox:1.28 --rm -it --restart=Never -- nslookup nginx-resolver-service
```

#### Pod DNS 조회 테스트

```bash
kubectl run test-nslookup --image=busybox:1.28 --rm -it --restart=Never -- nslookup 172-17-1-18.default.pod
```

### 4단계: 결과 파일에 저장

```bash
# Service 조회 결과 저장
kubectl run test-nslookup --image=busybox:1.28 --rm -it --restart=Never -- nslookup nginx-resolver-service > /root/CKA/nginx.svc

# Pod 조회 결과 저장  
kubectl run test-nslookup --image=busybox:1.28 --rm -it --restart=Never -- nslookup 172-17-1-18.default.pod > /root/CKA/nginx.pod
```

## 🤔 `172-17-1-18.default.pod`가 뭔가요?

### Pod DNS 이름 생성 규칙

```
Pod IP를 대시로 바꾼 것.네임스페이스.pod.cluster.local
```

### 실제 예시:

```bash
# Pod IP 확인
kubectl get pod nginx-resolver -o wide
# IP: 172.17.1.18

# Pod DNS 이름 만들기
172.17.1.18 → 172-17-1-18 (점을 대시로 변경)
네임스페이스: default
최종 DNS 이름: 172-17-1-18.default.pod.cluster.local
```

### 왜 이런 이름을 사용할까?

- **Service 이름**: 사람이 기억하기 쉬움 (`nginx-resolver-service`)
- **Pod IP 기반 이름**: 특정 Pod에 직접 접근할 때 사용

## 🔍 DNS 조회 결과 해석

### Service DNS 조회 결과:

```
Name:      nginx-resolver-service
Address 1: 172.20.234.191 nginx-resolver-service.default.svc.cluster.local
```

**의미**: "nginx-resolver-service라는 이름으로 172.20.234.191에 접근 가능"

### Pod DNS 조회 결과:

```
Name:      172-17-1-18.default.pod
Address 1: 172.17.1.18
```

**의미**: "Pod IP(172.17.1.18)로 직접 접근 가능"

## 💡 핵심 이해

### 두 가지 접근 방법:

1. **Service 이름으로 접근** (권장)
    ```bash
    curl http://nginx-resolver-service
    ```
    - 로드밸런싱 됨
    - Pod가 바뀌어도 이름은 동일

2. **Pod IP로 직접 접근**
    ```bash
    curl http://172.17.1.18
    # 또는 DNS 이름으로
    curl http://172-17-1-18.default.pod.cluster.local
    ```
    - 특정 Pod에만 접근
    - Pod가 재시작되면 IP 바뀜

## 🎯 문제의 목적

**확인하고 싶은 것:**

- ✅ 클러스터 내부 DNS가 정상 작동하는가?
- ✅ Service 이름으로 접근이 가능한가?
- ✅ Pod IP로도 접근이 가능한가?

**결론**: 클러스터 내부에서 nginx에 두 가지 방법(Service 이름, Pod IP)으로 모두 접근할 수 있음을 DNS 조회로 증명