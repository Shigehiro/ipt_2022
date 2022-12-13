# 1. Chapter07

- [1. Chapter07](#1-chapter07)
- [2. What we learn](#2-what-we-learn)
- [3. ReplicaSets](#3-replicasets)
  - [3.1. Delete Pods managed by ReplicaSets](#31-delete-pods-managed-by-replicasets)
  - [3.2. Creating a ReplicaSet Given That a Matching Pod Already Exists](#32-creating-a-replicaset-given-that-a-matching-pod-already-exists)
  - [3.3. Scaling a ReplicaSet after It Is Created](#33-scaling-a-replicaset-after-it-is-created)
- [4. Deployment](#4-deployment)
  - [4.1. Creating a Simple Deployment with Nginx Containers](#41-creating-a-simple-deployment-with-nginx-containers)
  - [4.2. Rolling Back a Deployment](#42-rolling-back-a-deployment)
- [5. StatefulSets, DaemonSets](#5-statefulsets-daemonsets)
- [6. Jobs](#6-jobs)

# 2. What we learn

Managing Pods by:
- ReplicaSets
- Deployments
- DaemonSets
- StatefulSets
- Jobs

# 3. ReplicaSets

```text
$ kubectl apply -f 01_replicaset-nginx.yaml
```

```text
$ kubectl get po --show-labels -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP           NODE        NOMINATED NODE   READINESS GATES   LABELS
nginx-replicaset-6tw9s   1/1     Running   0          47s   10.42.2.72   lab02-w01   <none>           <none>            environment=production
nginx-replicaset-9pd26   1/1     Running   0          47s   10.42.1.70   lab02-w02   <none>           <none>            environment=production
```

```text
$ kubectl get replicasets.apps nginx-replicaset
NAME               DESIRED   CURRENT   READY   AGE
nginx-replicaset   2         2         2       80s

# short command
$ kubectl get rs nginx-replicaset
```

detailed info.
```text
$ kubectl describe rs nginx-replicaset |grep ^Selector -A20
Selector:     environment=production
Labels:       app=nginx
Annotations:  <none>
Replicas:     2 current / 2 desired
Pods Status:  2 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  environment=production
  Containers:
   nginx-container:
    Image:        nginx
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age    From                   Message
  ----    ------            ----   ----                   -------
  Normal  SuccessfulCreate  3m34s  replicaset-controller  Created pod: nginx-replicaset-9pd26
  Normal  SuccessfulCreate  3m34s  replicaset-controller  Created pod: nginx-replicaset-6tw9s
```

Pods are managed by ReplicaSets.
```text
$ kubectl get po
NAME                     READY   STATUS    RESTARTS   AGE
nginx-replicaset-6tw9s   1/1     Running   0          4m33s
nginx-replicaset-9pd26   1/1     Running   0          4m33s

$ kubectl describe po nginx-replicaset-6tw9s | grep ^Labels -A10
Labels:           environment=production
Annotations:      <none>
Status:           Running
IP:               10.42.2.72
IPs:
  IP:           10.42.2.72
Controlled By:  ReplicaSet/nginx-replicaset ### Here. managed by ReplicaSets
Containers:
  nginx-container:
    Container ID:   containerd://9070e52b031a6e8e8292a942d4ac0612c83c4776f33f37219fd29532fff71377
    Image:          nginx
```

Do not delete Pods, ReplicaSets.

## 3.1. Delete Pods managed by ReplicaSets

confirm two Pods are up and running.
```text
$ kubectl get po
NAME                     READY   STATUS    RESTARTS   AGE
nginx-replicaset-6tw9s   1/1     Running   0          13m
nginx-replicaset-9pd26   1/1     Running   0          13m
```

delete one of Pods. ReplicaSets will create a new Pod.
```text
$ kubectl delete po nginx-replicaset-6tw9s
pod "nginx-replicaset-6tw9s" deleted

$ kubectl get po
NAME                     READY   STATUS              RESTARTS   AGE
nginx-replicaset-9pd26   1/1     Running             0          31m
nginx-replicaset-hwl8m   0/1     ContainerCreating   0          4s  ### new name(newly created pod)
$
$ kubectl get po
NAME                     READY   STATUS    RESTARTS   AGE
nginx-replicaset-9pd26   1/1     Running   0          31m
nginx-replicaset-hwl8m   1/1     Running   0          8s
```

```text
$ kubectl describe rs nginx-replicaset |grep ^Event -A10
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  33m   replicaset-controller  Created pod: nginx-replicaset-9pd26
  Normal  SuccessfulCreate  33m   replicaset-controller  Created pod: nginx-replicaset-6tw9s
  Normal  SuccessfulCreate  114s  replicaset-controller  Created pod: nginx-replicaset-hwl8m
```

delete replicasets.
```text
$ kubectl delete rs nginx-replicaset
replicaset.apps "nginx-replicaset" deleted
$ kubectl get po
No resources found in default namespace.
```

## 3.2. Creating a ReplicaSet Given That a Matching Pod Already Exists

creata a pod (not managed by replicasets)
```text
$ kubectl apply -f 02_pod-matching-replicaset.yaml
pod/pod-matching-replicaset created
```

```
$ kubectl get po -o wide --show-labels
NAME                      READY   STATUS    RESTARTS   AGE   IP           NODE        NOMINATED NODE   READINESS GATES   LABELS
pod-matching-replicaset   1/1     Running   0          37s   10.42.1.71   lab02-w02   <none>           <none>            environment=production
```

```text
$ kubectl apply -f 03_replicaset-nginx.yaml
replicaset.apps/nginx-replicaset created
```

```text
$ kubectl get pod
NAME                      READY   STATUS    RESTARTS   AGE
pod-matching-replicaset   1/1     Running   0          18m # existing pod
nginx-replicaset-tszg2    1/1     Running   0          24s # newly created
```

```text
$ kubectl describe po pod-matching-replicaset |grep -i replica
Name:             pod-matching-replicaset
Controlled By:  ReplicaSet/nginx-replicaset
  Normal  Scheduled  19m   default-scheduler  Successfully assigned default/pod-matching-replicaset to lab02-w02
```

one pod has been creted
```text
$ kubectl describe rs nginx-replicaset |grep ^Event -A10
Events:
  Type    Reason            Age    From                   Message
  ----    ------            ----   ----                   -------
  Normal  SuccessfulCreate  2m19s  replicaset-controller  Created pod: nginx-replicaset-tszg2
```

```text
$ kubectl delete rs nginx-replicaset
replicaset.apps "nginx-replicaset" deleted

$ kubectl get po
No resources found in default namespace.
```

## 3.3. Scaling a ReplicaSet after It Is Created

```text
$ kubectl apply -f 04_replicaset-nginx.yaml
replicaset.apps/nginx-replicaset created
```

scale
```text
$ kubectl get po
NAME                     READY   STATUS    RESTARTS   AGE
nginx-replicaset-bgx8k   1/1     Running   0          33s
nginx-replicaset-lhzrp   1/1     Running   0          33s

$ kubectl scale --replicas 4 rs nginx-replicaset
replicaset.apps/nginx-replicaset scaled

$ kubectl get po
NAME                     READY   STATUS    RESTARTS   AGE
nginx-replicaset-bgx8k   1/1     Running   0          70s
nginx-replicaset-lhzrp   1/1     Running   0          70s
nginx-replicaset-6tgxm   1/1     Running   0          29s
nginx-replicaset-xkzl7   1/1     Running   0          29s

$ kubectl scale --replicas 1 rs nginx-replicaset
replicaset.apps/nginx-replicaset scaled
$ kubectl get po
NAME                     READY   STATUS        RESTARTS   AGE
nginx-replicaset-lhzrp   1/1     Running       0          97s
nginx-replicaset-6tgxm   1/1     Terminating   0          56s
nginx-replicaset-xkzl7   1/1     Terminating   0          56s
nginx-replicaset-bgx8k   1/1     Terminating   0          97s
```

```text
$ kubectl delete rs nginx-replicaset
replicaset.apps "nginx-replicaset" deleted
```

# 4. Deployment

Deployment records a history of revisions to easily roll back to previous states.
ReplicaSets manages Pods.
```text

------ Deploymnet -------
|                       |
|  Replicaset |         |
|  | pod      |         |
|  ------------         |
-------------------------

```

## 4.1. Creating a Simple Deployment with Nginx Containers 


```text
$ kubectl get deployments.apps nginx-deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   0/3     3            0           6s

$ kubectl get deployments.apps nginx-deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           20s
```

Deployments creates ReplicaSets.
ReplicaSets creates Pods.
```text
$ kubectl get deployments.apps
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           3m55s

$ kubectl get rs
NAME                         DESIRED   CURRENT   READY   AGE
nginx-deployment-7b76c4564   3         3         3       3m58s

$ kubectl get pod
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-7b76c4564-2qq79   1/1     Running   0          3m59s
nginx-deployment-7b76c4564-z7c6l   1/1     Running   0          3m59s
nginx-deployment-7b76c4564-mmddb   1/1     Running   0          3m59s
```

```text
$ kubectl delete deployments.apps nginx-deployment
deployment.apps "nginx-deployment" deleted

$ kubectl get rs
No resources found in default namespace.
$ kubectl get po
No resources found in default namespace.
```

## 4.2. Rolling Back a Deployment 

```text
$ kubectl apply -f 06_app-deployment.yaml
deployment.apps/app-deployment created
```

check history.
```text
$ kubectl rollout history deployment app-deployment
deployment.apps/app-deployment
REVISION  CHANGE-CAUSE
1         <none>
```

chagne container name.
```text
$ diff -u 06_app-deployment.yaml 07_app-deployment.yaml
--- 06_app-deployment.yaml      2022-12-13 01:20:24.647437558 +0900
+++ 07_app-deployment.yaml      2022-12-13 01:24:53.616895875 +0900
@@ -17,5 +17,5 @@
                 environment: production
         spec:
             containers:
-            - name: nginx-container
+            - name: nginx
               image: nginx
```

on the worker node, container name is nginx-container.
```text
$ ssh root@192.168.124.20 crictl ps|grep nginx
Warning: Permanently added '192.168.124.20' (ED25519) to the list of known hosts.
47e018fba906a       ac8efec875ce3       7 minutes ago       Running             nginx-container      0                   604e2e49ff439       app-deployment-7b76c4564-h4mh5
```

add `--record`
```
$ kubectl apply -f 07_app-deployment.yaml --record
Flag --record has been deprecated, --record will be removed in the future
deployment.apps/app-deployment configured
```

on the worker node.
```
$ ssh root@192.168.124.20 crictl ps|grep nginx
Warning: Permanently added '192.168.124.20' (ED25519) to the list of known hosts.
6bf7e992add0f       ac8efec875ce3       35 seconds ago      Running             nginx                0                   2e41dc5c6a4df       app-deployment-778d864778-gpnjd
```

check history.
```
~$ kubectl rollout history deployment app-deployment
deployment.apps/app-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl apply --filename=07_app-deployment.yaml --record=true
```

intentionally specify a wrong image name.
```
$ kubectl set image deployment app-deployment nginx=ngnx --record
Flag --record has been deprecated, --record will be removed in the future
deployment.apps/app-deployment image updated
```

```
$ kubectl get pod
NAME                              READY   STATUS         RESTARTS   AGE
app-deployment-778d864778-4gz6t   1/1     Running        0          5m33s
app-deployment-778d864778-gpnjd   1/1     Running        0          5m29s
app-deployment-778d864778-tjp7w   1/1     Running        0          5m24s
app-deployment-5698bddb9-49x8s    0/1     ErrImagePull   0          40s
```

```
$ kubectl rollout history deployment app-deployment
deployment.apps/app-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl apply --filename=07_app-deployment.yaml --record=true
3         kubectl set image deployment app-deployment nginx=ngnx --record=true
```

roll back
```
$ kubectl rollout undo deployment app-deployment --to-revision 2
deployment.apps/app-deployment rolled back
```

```
$ kubectl get po
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-778d864778-4gz6t   1/1     Running   0          8m17s
app-deployment-778d864778-gpnjd   1/1     Running   0          8m13s
app-deployment-778d864778-tjp7w   1/1     Running   0          8m8s
```

```
$ kubectl rollout history deployment app-deployment
deployment.apps/app-deployment
REVISION  CHANGE-CAUSE
1         <none>
3         kubectl set image deployment app-deployment nginx=ngnx --record=true
4         kubectl apply --filename=07_app-deployment.yaml --record=true
```

```
$ kubectl delete deployments.apps app-deployment
```

# 5. StatefulSets, DaemonSets

learn in later chapter.

- Reference
  - [statefulsets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
  - [daemonsets](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)

# 6. Jobs

- Reference
  - [jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)


```text
$ kubectl apply -f 08_one-time-job.yaml
job.batch/one-time-job created

$ kubectl get po
NAME                 READY   STATUS    RESTARTS   AGE
one-time-job-6nlq6   1/1     Running   0          29s

$ kubectl get jobs.batch
NAME           COMPLETIONS   DURATION   AGE
one-time-job   0/1           19s        19s

$ kubectl get po
NAME                 READY   STATUS      RESTARTS   AGE
one-time-job-6nlq6   0/1     Completed   0          50s

$ kubectl get jobs.batch
NAME           COMPLETIONS   DURATION   AGE
one-time-job   1/1           32s        52s
```

```text
$ kubectl logs  pods/one-time-job-6nlq6
Tue Dec 13 03:54:02 UTC 2022
Bye
```

```text
$ kubectl delete jobs.batch one-time-job
job.batch "one-time-job" deleted
```