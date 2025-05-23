cluster1-controlplane ~ ➜  k run test-nslookup --image=busybox:1.28 --rm -it --restart=Never -- nslookup nginx-resolver-service
Server:    172.20.0.10
Address 1: 172.20.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx-resolver-service
Address 1: 172.20.234.191 nginx-resolver-service.ingress-ns.svc.cluster.local
pod "test-nslookup" deleted