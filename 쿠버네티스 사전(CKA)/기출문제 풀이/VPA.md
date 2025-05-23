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
   - 
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
