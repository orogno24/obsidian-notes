
현재 배포된 eris-fe-uags

eris-fe                pod/eris-fe-uags-5845f4bfc4-799hm                                     3/3     Running            0               17h
eris-fe                pod/eris-fe-uags-5845f4bfc4-szk86                                     3/3     Running            0               17h
eris-fe                service/eris-fe-uags                                         ClusterIP      10.233.3.125    <none>                                                                       8080/TCP                                                             17h
kong                   service/eris-fe-uags-kong                                    NodePort       10.233.51.247   <none>                                                                       80:30006/TCP                                                         56m
eris-fe                deployment.apps/eris-fe-uags                                             2/2     2            2           25h

eris-fe-uags 서비스

apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"argocd.argoproj.io/instance":"prd-eris-fe-uags"},"name":"eris-fe-uags","namespace":"eris-fe"},"spec":{"ports":[{"port":8080,"protocol":"TCP","targetPort":8080}],"selector":{"app":"eris-fe-uags"},"type":"ClusterIP"}}
  creationTimestamp: "2025-07-21T07:55:41Z"
  labels:
    argocd.argoproj.io/instance: prd-eris-fe-uags
  name: eris-fe-uags
  namespace: eris-fe
  resourceVersion: "54826665"
  uid: 7d795036-5979-409d-8b43-ea636962e4a3
spec:
  clusterIP: 10.233.3.125
  clusterIPs:
  - 10.233.3.125
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: eris-fe-uags
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}


eris-fe-uags-kong 서비스

apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"argocd.argoproj.io/instance":"prd-eris-fe-uags"},"name":"eris-fe-uags-kong","namespace":"kong"},"spec":{"ports":[{"nodePort":30006,"port":80,"protocol":"TCP","targetPort":8000}],"selector":{"app":"kong-kong"},"type":"NodePort"}}
  creationTimestamp: "2025-07-22T00:28:48Z"
  labels:
    argocd.argoproj.io/instance: prd-eris-fe-uags
  name: eris-fe-uags-kong
  namespace: kong
  resourceVersion: "55225798"
  uid: 5d307662-14cc-4ff8-9ba3-14da8dbea5b6
spec:
  clusterIP: 10.233.51.247
  clusterIPs:
  - 10.233.51.247
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - nodePort: 30006
    port: 80
    protocol: TCP
    targetPort: 8000
  selector:
    app: kong-kong
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
