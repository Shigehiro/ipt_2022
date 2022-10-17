# 1. k8s(k3s) service(node port, load balancer)

- [1. k8s(k3s) service(node port, load balancer)](#1-k8sk3s-servicenode-port-load-balancer)
- [2. reference](#2-reference)
- [3. service and pod](#3-service-and-pod)
- [4. service and deployment](#4-service-and-deployment)
- [5. Publishing Services (ServiceTypes)](#5-publishing-services-servicetypes)
  - [5.1. node port](#51-node-port)
  - [5.2. load balancer](#52-load-balancer)

# 2. reference
- https://kubernetes.io/docs/concepts/services-networking/service/

# 3. service and pod

pod_service.yml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app.kubernetes.io/name: proxy
spec:
  containers:
  - name: nginx
    image: nginx:stable
    ports:
      - containerPort: 80
        name: http-web-svc

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app.kubernetes.io/name: proxy
  ports:
  - name: name-of-service-port
    protocol: TCP
    port: 80
    targetPort: http-web-svc
```

```text
[root@lab02-m01 service]# kubectl apply -f pod_service.yml

[root@lab02-m01 service]# kubectl get po -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP          NODE        NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          41s   10.42.2.7   lab02-w01   <none>           <none>

[root@lab02-m01 service]# kubectl get svc
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes      ClusterIP   10.43.0.1      <none>        443/TCP   44d
nginx-service   ClusterIP   10.43.80.100   <none>        80/TCP    44s
[root@lab02-m01 service]#
```

```text
### to pod ip
[root@lab02-m01 service]# curl -I 10.42.2.7
HTTP/1.1 200 OK
Server: nginx/1.22.0
Date: Mon, 17 Oct 2022 15:11:14 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Mon, 23 May 2022 23:59:19 GMT
Connection: keep-alive
ETag: "628c1fd7-267"
Accept-Ranges: bytes

### to service ip
[root@lab02-m01 service]#
[root@lab02-m01 service]# curl -I 10.43.80.100
HTTP/1.1 200 OK
Server: nginx/1.22.0
Date: Mon, 17 Oct 2022 15:11:24 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Mon, 23 May 2022 23:59:19 GMT
Connection: keep-alive
ETag: "628c1fd7-267"
Accept-Ranges: bytes

[root@lab02-m01 service]#
```

# 4. service and deployment

上は pod + serviceの例。podなのでコンテナの数が1個。スケールアウトできない。
deployment + serviceでやる場合はどうなる？

hint:  https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

answer: deployment_service.yml

# 5. Publishing Services (ServiceTypes)

- https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types

## 5.1. node port

- https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport


```text
  196  kubectl delete deployments.apps nginx-deployment
  198  kubectl delete svc nginx-service
```

define node port service.
```
[root@lab02-m01 service]# kubectl apply -f test_node_port_deployment.yml
deployment.apps/nginx-deployment created
service/nginx-service created


[root@lab02-m01 service]# kubectl get deployments.apps
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           17s

[root@lab02-m01 service]# kubectl get po -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP           NODE        NOMINATED NODE   READINESS GATES
nginx-deployment-6595874d85-s2b4p   1/1     Running   0          20s   10.42.2.11   lab02-w01   <none>           <none>
nginx-deployment-6595874d85-j49qc   1/1     Running   0          20s   10.42.1.13   lab02-w02   <none>           <none>
nginx-deployment-6595874d85-xbgxb   1/1     Running   0          20s   10.42.0.20   lab02-m01   <none>           <none>

[root@lab02-m01 service]# kubectl get svc
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes      ClusterIP   10.43.0.1       <none>        443/TCP        44d
nginx-service   NodePort    10.43.169.189   <none>        80:30007/TCP   23s
[root@lab02-m01 service]#
```

check ip.
```text
[root@lab02-m01 service]# ansible -i ./inventory.ini k3s -m setup | grep address | grep 192.168.
            "address": "192.168.124.10",
                "address": "192.168.124.10",
            "address": "192.168.124.21",
                "address": "192.168.124.21",
            "address": "192.168.124.20",
                "address": "192.168.124.20",
```


```
# to master's eth0 ip
[root@lab02-m01 service]# curl 192.168.124.10:30007 -I
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Mon, 17 Oct 2022 15:47:32 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 04 Dec 2018 14:44:49 GMT
Connection: keep-alive
ETag: "5c0692e1-264"
Accept-Ranges: bytes

# to worker's eth0 ip
[root@lab02-m01 service]# ansible -i ./inventory.ini lab02-w01 -m shell -a 'curl 192.168.124.20:30007 -I'
[WARNING]: Platform linux on host lab02-w01 is using the discovered Python interpreter at /usr/bin/python3.6, but future installation of
another Python interpreter could change the meaning of that path. See https://docs.ansible.com/ansible-
core/2.12/reference_appendices/interpreter_discovery.html for more information.
lab02-w01 | CHANGED | rc=0 >>
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Mon, 17 Oct 2022 15:48:21 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 04 Dec 2018 14:44:49 GMT
Connection: keep-alive
ETag: "5c0692e1-264"
Accept-Ranges: bytes  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0   612    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
[root@lab02-m01 service]#
```


```
[root@lab02-m01 service]# kubectl delete -f test_node_port_deployment.yml
deployment.apps "nginx-deployment" deleted
service "nginx-service" deleted
```

## 5.2. load balancer

- k3s builtin load balcner
  - https://docs.k3s.io/networking#service-load-balancer
- the other major LB(on premise k8s) used on on-premises is Metal LB
  - https://metallb.universe.tf/


load balancer info
```text
[root@lab02-m01 service]# kubectl cluster-info dump

<snip>

                "clusterIP": "10.43.24.196",
                "clusterIPs": [
                    "10.43.24.196"
                ],
                "type": "LoadBalancer",
                "sessionAffinity": "None",
                "externalTrafficPolicy": "Cluster",
                "ipFamilies": [
                    "IPv4"
                ],
                "ipFamilyPolicy": "PreferDualStack",
                "allocateLoadBalancerNodePorts": true,
                "internalTrafficPolicy": "Cluster"
            },
            "status": {
                "loadBalancer": {
                    "ingress": [
                        {
                            "ip": "192.168.124.10"
                        },
                        {
                            "ip": "192.168.124.20"
                        },
                        {
                            "ip": "192.168.124.21"
                        }
                    ]

```

```text
[root@lab02-m01 service]# kubectl apply -f load_balancer_deployment.yml                                                                         
```

```text
[root@lab02-m01 service]# kubectl get deployments.apps -o wide
NAME               READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES         SELECTOR
nginx-deployment   3/3     3            3           104s   nginx        nginx:1.14.2   app=nginx

[root@lab02-m01 service]# kubectl get po -o wide
NAME                                READY   STATUS    RESTARTS   AGE    IP           NODE        NOMINATED NODE   READINESS GATES
nginx-deployment-6595874d85-l54ns   1/1     Running   0          105s   10.42.1.14   lab02-w02   <none>           <none>
nginx-deployment-6595874d85-kc26m   1/1     Running   0          105s   10.42.0.21   lab02-m01   <none>           <none>
nginx-deployment-6595874d85-tnhd7   1/1     Running   0          105s   10.42.2.12   lab02-w01   <none>           <none>

[root@lab02-m01 service]# kubectl get svc -o wide
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP                                    PORT(S)          AGE   SELECTOR
kubernetes   ClusterIP      10.43.0.1      <none>                                         443/TCP          45d   <none>
my-service   LoadBalancer   10.43.24.197   192.168.124.10,192.168.124.20,192.168.124.21   8080:31017/TCP   90s   app=nginx
[root@lab02-m01 service]#
```

to load balancer ip
```
[root@lab02-m01 service]# for i in 10 20 21;do curl 192.168.124.$i:8080 -I;done
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Mon, 17 Oct 2022 16:42:15 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 04 Dec 2018 14:44:49 GMT
Connection: keep-alive
ETag: "5c0692e1-264"
Accept-Ranges: bytes

HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Mon, 17 Oct 2022 16:42:15 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 04 Dec 2018 14:44:49 GMT
Connection: keep-alive
ETag: "5c0692e1-264"
Accept-Ranges: bytes

HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Mon, 17 Oct 2022 16:42:15 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 04 Dec 2018 14:44:49 GMT
Connection: keep-alive
ETag: "5c0692e1-264"
Accept-Ranges: bytes

[root@lab02-m01 service]#
```

to cluster ip
```
[root@lab02-m01 service]# curl 10.43.24.197:8080 -I
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Mon, 17 Oct 2022 16:42:55 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 04 Dec 2018 14:44:49 GMT
Connection: keep-alive
ETag: "5c0692e1-264"
Accept-Ranges: bytes

[root@lab02-m01 service]#
```

to pod
```
[root@lab02-m01 service]# kubectl get po -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP           NODE        NOMINATED NODE   READINESS GATES
nginx-deployment-6595874d85-kfxdm   1/1     Running   0          89s   10.42.1.19   lab02-w02   <none>           <none>
nginx-deployment-6595874d85-vp4k9   1/1     Running   0          89s   10.42.0.26   lab02-m01   <none>           <none>
nginx-deployment-6595874d85-rw5lf   1/1     Running   0          89s   10.42.2.17   lab02-w01   <none>           <none>

[root@lab02-m01 service]# for i in 10.42.1.19 10.42.0.26 10.42.2.17;do curl $i -I ;done
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Mon, 17 Oct 2022 16:43:39 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 04 Dec 2018 14:44:49 GMT
Connection: keep-alive
ETag: "5c0692e1-264"
Accept-Ranges: bytes

HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Mon, 17 Oct 2022 16:43:39 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 04 Dec 2018 14:44:49 GMT
Connection: keep-alive
ETag: "5c0692e1-264"
Accept-Ranges: bytes

HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Mon, 17 Oct 2022 16:43:39 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 04 Dec 2018 14:44:49 GMT
Connection: keep-alive
ETag: "5c0692e1-264"
Accept-Ranges: bytes

[root@lab02-m01 service]#
```

```text
[root@lab02-m01 service]# crictl ps |grep svclb
0d6f1baa6d5bd       dbd43b6716a08       5 minutes ago       Running             lb-tcp-8080              0                   2b9a3c4e77a29
 svclb-my-service-8f6ba502-96d42
c4fe5d07c80e6       dbd43b6716a08       2 hours ago         Running             lb-tcp-443               2                   f1056cd74844c
 svclb-traefik-aa1ff2fc-jb28v
6ca51fbb1dd5a       dbd43b6716a08       2 hours ago         Running             lb-tcp-80                2                   f1056cd74844c
 svclb-traefik-aa1ff2fc-jb28v
[root@lab02-m01 service]#
[root@lab02-m01 service]# kubectl get svc
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP                                    PORT(S)          AGE
kubernetes   ClusterIP      10.43.0.1      <none>                                         443/TCP          45d
my-service   LoadBalancer   10.43.24.197   192.168.124.10,192.168.124.20,192.168.124.21   8080:32095/TCP   5m32s
[root@lab02-m01 service]#
```