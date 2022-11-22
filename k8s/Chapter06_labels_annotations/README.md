# 1. Labels and Annotations

- [1. Labels and Annotations](#1-labels-and-annotations)
- [2. k8s official reference](#2-k8s-official-reference)
- [3. Labels](#3-labels)
  - [3.1. 02\_pod-without-initial-labels.yaml ( add labels to the running pod )](#31-02_pod-without-initial-labelsyaml--add-labels-to-the-running-pod-)
  - [3.2. 03\_pod-with-some-labels.yaml ( modify labels )](#32-03_pod-with-some-labelsyaml--modify-labels-)
  - [3.3. 04\_\*.yaml ( label selector )](#33-04_yaml--label-selector-)
  - [3.4. 05 ( label selector advanced )](#34-05--label-selector-advanced-)
- [4. Annotations](#4-annotations)

# 2. k8s official reference

[Labels and Selectors](#https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)

# 3. Labels

## 3.1. 02_pod-without-initial-labels.yaml ( add labels to the running pod )

```
$ kubectl apply -f 02_pod-without-initial-labels.yaml 
pod/pod-no-label created
```

no labels
```
$ kubectl describe po pod-no-label |grep Labels
Labels:           <none>
```

add labels to the running pod.
```
$ kubectl label pod pod-no-label app=nginx
pod/pod-no-label labeled

$ kubectl describe po pod-no-label |grep ^Labels
Labels:           app=nginx
```

add multiple labels
```
$ kubectl label pod pod-no-label foo=bar foo2=bar2
pod/pod-no-label labeled

$ kubectl describe po pod-no-label |grep ^Label -A2
Labels:           app=nginx
                  foo=bar
                  foo2=bar2
```

```
$ kubectl delete po pod-no-label 
pod "pod-no-label" deleted
```

## 3.2. 03_pod-with-some-labels.yaml ( modify labels )

```
$ kubectl create -f 03_pod-with-some-labels.yaml 
pod/pod-with-labels created

$ kubectl describe po pod-with-labels |grep ^Labels
Labels:           app=nginx
```

- modify labels
```
$ kubectl label --overwrite pod pod-with-labels app=nginx-app
pod/pod-with-labels labeled

$ kubectl describe po pod-with-labels |grep ^Labels
Labels:           app=nginx-app
```

- delete labels

```
$ kubectl label pod pod-with-labels app-

pod/pod-with-labels unlabeled

$ kubectl describe po pod-with-labels |grep ^Labels
Labels:           <none>
```

## 3.3. 04_*.yaml ( label selector )

```
$ ls 04_*
04_01_pod-frontend-production.yaml  04_02_pod-backend-production.yaml  04_03_pod-frontend-staging.yaml

$ ls 04_* | xargs -I{} kubectl apply -f {}
pod/frontend-pod created
pod/backend-pod created
pod/frontend-stating created
```

```
$ kubectl get po --show-labels 
NAME               READY   STATUS    RESTARTS   AGE    LABELS
frontend-pod       1/1     Running   0          8m2s   environment=production,role=frontend
backend-pod        1/1     Running   0          8m2s   environment=production,role=backend
frontend-stating   1/1     Running   0          8m1s   environment=staging,role=frontend
```

list pods by label selector
```
$ kubectl get po -l environment=production
NAME           READY   STATUS    RESTARTS   AGE
frontend-pod   1/1     Running   0          3m17s
backend-pod    1/1     Running   0          3m17s

$ kubectl get po -l role=frontend,environment=production
NAME           READY   STATUS    RESTARTS   AGE
frontend-pod   1/1     Running   0          4m

$ kubectl get po -l environment!=production
NAME               READY   STATUS    RESTARTS   AGE
frontend-stating   1/1     Running   0          4m17s
```

*please do not delete pods*

## 3.4. 05 ( label selector advanced )

```
$ kubectl get po --show-labels 
NAME               READY   STATUS    RESTARTS   AGE    LABELS
frontend-pod       1/1     Running   0          8m2s   environment=production,role=frontend
backend-pod        1/1     Running   0          8m2s   environment=production,role=backend
frontend-stating   1/1     Running   0          8m1s   environment=staging,role=frontend
```

```
$ kubectl get po -l 'role in (frontend,backend), environment in (production,staging)'
NAME               READY   STATUS    RESTARTS   AGE
frontend-pod       1/1     Running   0          22m
backend-pod        1/1     Running   0          22m
frontend-stating   1/1     Running   0          22m
```

```
$ kubectl get po -l 'environment notin (staging,foobar)' --show-labels 
NAME           READY   STATUS    RESTARTS   AGE   LABELS
frontend-pod   1/1     Running   0          24m   environment=production,role=frontend
backend-pod    1/1     Running   0          24m   environment=production,role=backend
```

# 4. Annotations

```text
$ kubectl apply -f 05_pod-with-annotations.yaml
pod/pod-with-annotations created
```

```text
$ kubectl get po
NAME                   READY   STATUS    RESTARTS   AGE
pod-with-annotations   1/1     Running   0          2m34s
```

```text
$ kubectl describe po pod-with-annotations |grep ^Anno -A5
Annotations:      JIRA-issue: https://your-jira-link.com/issue/ABC-1234
                  commit-SHA: d6s9shb82365yg4ygd782889us28377gf6
                  owner: https://internal-link.to.website/username
                  timestamp: 123456789
Status:           Running
IP:               10.42.1.64
```

add annotations to the running pod.
```text
$ kubectl annotate pod pod-with-annotations key01=value01
pod/pod-with-annotations annotated

$ kubectl describe po pod-with-annotations |grep ^Anno -A5
Annotations:      JIRA-issue: https://your-jira-link.com/issue/ABC-1234
                  commit-SHA: d6s9shb82365yg4ygd782889us28377gf6
                  key01: value01
                  owner: https://internal-link.to.website/username
                  timestamp: 123456789
Status:           Running
```


