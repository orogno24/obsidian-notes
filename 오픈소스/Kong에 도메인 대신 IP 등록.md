
ubuntu@eris-hm-dev-master-node01:~$ k get svc -o yaml -n eris-fe eris-fe-ugis
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"argocd.argoproj.io/instance":"prd-eris-fe-ugis"},"name":"eris-fe-ugis","namespace":"eris-fe"},"spec":{"ports":[{"port":8080,"protocol":"TCP","targetPort":8080}],"selector":{"app":"eris-fe-ugis"},"type":"ClusterIP"}}
  creationTimestamp: "2025-07-21T07:54:07Z"
  labels:
    argocd.argoproj.io/instance: prd-eris-fe-ugis
  name: eris-fe-ugis
  namespace: eris-fe
  resourceVersion: "54826025"
  uid: 0cd539b1-2eff-47dd-a320-96984daa6f98
spec:
  clusterIP: 10.233.21.207
  clusterIPs:
  - 10.233.21.207
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: eris-fe-ugis
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
ubuntu@eris-hm-dev-master-node01:~$ k get svc -o yaml -n kong eris-fe-ugis-kong
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"argocd.argoproj.io/instance":"prd-eris-fe-ugis"},"name":"eris-fe-ugis-kong","namespace":"kong"},"spec":{"ports":[{"nodePort":30005,"port":80,"protocol":"TCP","targetPort":8000}],"selector":{"app":"kong-kong"},"type":"NodePort"}}
  creationTimestamp: "2025-07-22T00:30:18Z"
  labels:
    argocd.argoproj.io/instance: prd-eris-fe-ugis
  name: eris-fe-ugis-kong
  namespace: kong
  resourceVersion: "55226483"
  uid: 15a8f253-0425-4780-99d6-c27f216201f8
spec:
  clusterIP: 10.233.7.85
  clusterIPs:
  - 10.233.7.85
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - nodePort: 30005
    port: 80
    protocol: TCP
    targetPort: 8000
  selector:
    app: kong-kong
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}


Konga에 Service 등록
Name: eris-fe-ugis
Host: eris-fe-uags.eris-fe.svc.cluster.local
Port: 8080
Path: /

Konga에 Routes 등록
Hosts: Konga에 서비스 등록
Path: /