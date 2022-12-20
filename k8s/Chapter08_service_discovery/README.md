# 1. Service Disocery

- [1. Service Disocery](#1-service-disocery)
- [2. K8s Service object](#2-k8s-service-object)
- [3. NodePort service](#3-nodeport-service)
- [4. ClusterIP service](#4-clusterip-service)
- [5. Creating a ClusterIP Service with a Custom IP](#5-creating-a-clusterip-service-with-a-custom-ip)
- [6. LoadBalancer service](#6-loadbalancer-service)

# 2. K8s Service object

- NodePort
- ClusterIP
- LoadBalancer
- ExtrnalName

# 3. NodePort service

```text
$ kubectl apply -f 01_nginx-deployment.yaml
```

```test
$ kubectl get po -o wide
NAME                               READY   STATUS    RESTARTS   AGE   IP           NODE        NOMINATED NODE   READINESS GATES
nginx-deployment-7b76c4564-5pxhn   1/1     Running   0          42s   10.42.2.99   lab02-w01   <none>           <none>
nginx-deployment-7b76c4564-7f6ds   1/1     Running   0          42s   10.42.1.96   lab02-w02   <none>           <none>
nginx-deployment-7b76c4564-fkjx8   1/1     Running   0          42s   10.42.0.91   lab02-m01   <none>           <none>
```

check labels
```text
$ kubectl get deployments.apps --show-labels -o wide
NAME               READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS        IMAGES   SELECTOR                           LABELS
nginx-deployment   3/3     3            3           2m34s   nginx-container   nginx    app=nginx,environment=production   app=nginx
```

```text
$ kubectl apply -f 02_nginx-service-nodeport.yaml
```

```text
service ---------> deployment ---> Pod
        selector
```

```text
node port(32023) -- service port(80) -- pod's port(80)
```

```text
$ kubectl get service
NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes               ClusterIP   10.43.0.1      <none>        443/TCP        108d
nginx-service-nodeport   NodePort    10.43.184.30   <none>        80:32023/TCP   31s
```

```text
$ kubectl describe service nginx-service-nodeport
Name:                     nginx-service-nodeport
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=nginx,environment=production
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.43.184.30
IPs:                      10.43.184.30
Port:                     <unset>  80/TCP ### here 
TargetPort:               80/TCP ### here
NodePort:                 <unset>  32023/TCP ### here
Endpoints:                10.42.0.91:80,10.42.1.96:80,10.42.2.99:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

on the master
```text
[root@lab02-m01 ~]# ip -4 a s |grep inet |grep -v -E 'flannel|cni|127.0' | awk '{print $2}'
192.168.124.10/24

[root@lab02-m01 ~]# curl 192.168.124.10:32023 -I
HTTP/1.1 200 OK
Server: nginx/1.23.3
Date: Tue, 20 Dec 2022 03:18:54 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 13 Dec 2022 15:53:53 GMT
Connection: keep-alive
ETag: "6398a011-267"
Accept-Ranges: bytes
```

on one of worker nodes
```text
[root@lab02-w01 ~]# curl 192.168.124.20:32023 -I
HTTP/1.1 200 OK
Server: nginx/1.23.3
Date: Tue, 20 Dec 2022 03:19:50 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 13 Dec 2022 15:53:53 GMT
Connection: keep-alive
ETag: "6398a011-267"
Accept-Ranges: bytes
```

open an web browser, access to http://<k8's node ip>:32023

```text
$ kubectl delete deployments.apps nginx-deployment

$ kubectl delete service nginx-service-nodeport
```

# 4. ClusterIP service

accessible **from inside the cluster**

```text
$ kubectl apply -f 03_nginx-deployment.yaml
```

```text
$ kubectl get po --show-labels
NAME                               READY   STATUS    RESTARTS   AGE   LABELS
nginx-deployment-7b76c4564-dc78d   1/1     Running   0          57s   app=nginx,environment=production,pod-template-hash=7b76c4564
nginx-deployment-7b76c4564-g2jb6   1/1     Running   0          57s   app=nginx,environment=production,pod-template-hash=7b76c4564
nginx-deployment-7b76c4564-5wjfb   1/1     Running   0          57s   app=nginx,environment=production,pod-template-hash=7b76c4564
```

```text
$ kubectl apply -f 04_nginx-service-clusterip.yaml
```

```
$ kubectl get service nginx-service-clusterip
NAME                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
nginx-service-clusterip   ClusterIP   10.43.90.238   <none>        80/TCP    36s
```

```text
$ kubectl describe service nginx-service-clusterip
Name:              nginx-service-clusterip
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=nginx,environment=production
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.43.90.238
IPs:               10.43.90.238
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.42.0.92:80,10.42.1.97:80,10.42.2.100:80
Session Affinity:  None
Events:            <none>
```

- which IPs k8s will pick up for clusterIP?
- when deploying k8s cluster, specify IP ranges for clusterIP.
```text
$ kubectl cluster-info dump |grep clusterIP -A1 | grep [0-1].*
                "clusterIP": "10.43.0.10",
                    "10.43.0.10"
                "clusterIP": "10.43.175.133",
                    "10.43.175.133"
                "clusterIP": "10.43.24.196",
                    "10.43.24.196"
                "clusterIP": "10.43.0.1",
                    "10.43.0.1"
```

on the one of k8's nodes
```text
[root@lab02-w01 ~]# curl 10.43.90.238 -I
HTTP/1.1 200 OK
Server: nginx/1.23.3
Date: Tue, 20 Dec 2022 03:33:18 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 13 Dec 2022 15:53:53 GMT
Connection: keep-alive
ETag: "6398a011-267"
Accept-Ranges: bytes
```

clusterIPs are accessible from inside the cluster.

```text
$ kubectl delete deployments.apps nginx-deployment

$ kubectl delete service nginx-service-clusterip
```

# 5. Creating a ClusterIP Service with a Custom IP

```text
$ kubectl apply -f 03_nginx-deployment.yaml
```

```text
$ grep -i 'clusterip:' 05_nginx-service-custom-clusterip.yaml
  clusterIP: 10.43.100.100

$ kubectl apply -f 05_nginx-service-custom-clusterip.yaml

$ kubectl get service nginx-service-clusterip
NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-service-clusterip   ClusterIP   10.43.100.100   <none>        80/TCP    19s
```

```text
[root@lab02-w01 ~]# curl -I 10.43.100.100
HTTP/1.1 200 OK
Server: nginx/1.23.3
Date: Tue, 20 Dec 2022 03:42:55 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 13 Dec 2022 15:53:53 GMT
Connection: keep-alive
ETag: "6398a011-267"
Accept-Ranges: bytes
```

```text
$ kubectl delete deployments.apps nginx-deployment
$ kubectl delete service nginx-service-clusterip
```

# 6. LoadBalancer service