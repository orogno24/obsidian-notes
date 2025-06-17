
1. 먼저 kubernetes 공식 사이트에서 HorizontalPodAutoscaler Walkthrough 접속
2. hpa yaml파일 복사

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

api버전에 .k8s.io/ 추가, 버전은 v2에서 v1로, kind 변경
```
autoscaling/v2 -> autoscaling.k8s.io/v1
```

scaleTargetRef은 targetRef로 변경
```
scaleTargetRef: -> targetRef:
```

updatePolicy, resourcePolicy(필요 시) 추가
```
updatePolicy:
  updateMode: "Auto"
```





```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
 name: api-server-vpa
spec:
# 🎯 어떤 워크로드를 감시할지 지정
 targetRef:
   apiVersion: apps/v1
   kind: Deployment
   name: api-server

 # 🔄 VPA가 어떻게 동작할지 결정
 updatePolicy:
   updateMode: "Auto"            # "Auto": 자동으로 Pod 재시작하며 리소스 조정
                                 # "Off": 권장사항만 제공, 실제 적용 안함
                                 # "Initial": 새 Pod 생성시에만 적용
 
 # 📊 리소스 정책 (선택사항 - 없어도 VPA 작동함)
 resourcePolicy:
   containerPolicies:            # 컨테이너별 정책 설정
   - containerName: '*'
     # 🔺 최대 허용 리소스 (상한선)
     maxAllowed:
       cpu: "500m"
       memory: "1Gi"
     
     # 🔻 최소 허용 리소스 (하한선)  
     minAllowed:
       cpu: "100m"
       memory: "256Mi"
     
     # 💡 VPA는 이 범위 내에서만 리소스를 조정함
     # 예: 사용량이 적으면 100m으로, 많으면 500m으로 조정  
```

