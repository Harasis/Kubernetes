![RX-M, llc.](http://rx-m.io/rxm-cnc.svg)


# Kubernetes


## Deployments & ReplicaSets Challenge Solutions

### 9.1. CHALLENGE: labels

Recreate the website deployment, the cache-pod, and dev-pod resources from earlier in the lab:

```
~$ cd ~/dep

~/dep kubectl apply -f ~/dep/mydep.yaml

deployment.apps/website created

~/dep kubectl run cache-pod -l app=cache --image redis

pod/cache-pod created

~/dep kubectl run dev-pod -l targetenv=dev --image httpd:2.2

pod/dev-pod created

~/dep
```

- Run a pod named `labelpod` with the label `targetenv=prod` and a container from `nginx:1.7.9`

```
~/dep$ kubectl run labelpod -l targetenv=prod --image nginx:1.7.9

pod/labelpod created

~/dep$
```

- Enter a command to display all of the pods with either the “demo” or “prod” value for targetenv

```
~/dep kubectl get po -l "targetenv in (prod,demo)" --show-labels

NAME                       READY   STATUS    RESTARTS   AGE   LABELS
labelpod                   1/1     Running   0          3s    targetenv=prod
website-74fcfd87c8-5557k   1/1     Running   0          63s   app=website,pod-template-hash=74fcfd87c8,targetenv=demo
website-74fcfd87c8-fszw4   1/1     Running   0          63s   app=website,pod-template-hash=74fcfd87c8,targetenv=demo
website-74fcfd87c8-x4wsp   1/1     Running   0          63s   app=website,pod-template-hash=74fcfd87c8,targetenv=demo

~/dep
```

- Find all pods other than those with the “demo” or “prod” value for targetenv

```
~/dep kubectl get po -l "targetenv notin (prod,demo)" --show-labels

NAME        READY   STATUS    RESTARTS   AGE   LABELS
cache-pod   1/1     Running   0          70s   app=cache
dev-pod     1/1     Running   0          68s   targetenv=dev

~/dep
```

- Enter a command to display all of the pods with either the “demo” or “prod” value for targetenv and the app key
  set to website

```
~/dep kubectl get po -l "targetenv in (prod,demo), app=website" --show-labels

NAME                       READY   STATUS    RESTARTS   AGE   LABELS
website-74fcfd87c8-5557k   1/1     Running   0          80s   app=website,pod-template-hash=74fcfd87c8,targetenv=demo
website-74fcfd87c8-fszw4   1/1     Running   0          80s   app=website,pod-template-hash=74fcfd87c8,targetenv=demo
website-74fcfd87c8-x4wsp   1/1     Running   0          80s   app=website,pod-template-hash=74fcfd87c8,targetenv=demo

~/dep
```

- Delete the pods you created for this challenge at the end

```
~/dep kubectl delete deploy/website pod/cache-pod pod/dev-pod pod/labelpod

deployment.apps "website" deleted
pod "cache-pod" deleted
pod "dev-pod" deleted
pod "labelpod" deleted

~/dep
```

### 9.2. Challenge: Deployments

- Create a deployment named `webserver-special`
  - Ensure the deployment is labeled with `app=commerce`, `vendor=student-tech`, `component=webserver`
  - The deployment should maintain 3 pods that have the labels `component=webserver` and `server=nginx` present
  - The containers of pods in the deployment should be created from the `nginx:1.20.1` image

```
~/dep$ nano websvr-spec.yaml && cat $_
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: commerce               # Add this label
    vendor: student-tech        # Add this label
    component: webserver        # Add this label
  name: webserver-special
spec:
  replicas: 3
  selector:
    matchLabels:
      component: webserver      # Must be present in template label
      server: nginx             # Must be present in template label
  template:
    metadata:
      labels:
        component: webserver    # Must be present if included in matchLabels
        server: nginx           # Must be present if included in matchLabels
        app: commerce           # Optional: not all pod labels need to be in matchLabels
    spec:
      containers:
      - image: nginx:1.20.1
        name: nginx
```
```
~/dep$ kubectl apply -f websvr-spec.yaml

deployment.apps/webserver-special created

~/dep$
```

- After creating the deployment, modify it so its pods:
  - run from the `nginx:1.23.2` image
  - Have the environment variable `PROXY=main`
  - Have 5 copies instead of 3

```
~/dep$ nano websvr-spec.yaml && cat $_
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: commerce
    vendor: student-tech
    component: webserver
  name: webserver-special
spec:
  replicas: 5                 # Change this
  selector:
    matchLabels:
      component: webserver
      server: nginx
  template:
    metadata:
      labels:
        component: webserver
        server: nginx
        app: commerce
    spec:
      containers:
      - image: nginx:1.23.2   # Change this
        name: nginx
        env:                  # Add this
        - name: PROXY         # Add this
          value: main         # Add this
```
```
~/dep$ kubectl apply -f websvr-spec.yaml

deployment.apps/webserver-special configured
~/dep$
```

- What is different about the pods now in `kubectl get`?

```
~/dep$ kubectl get pods,rs,deployment

NAME                                   READY   STATUS    RESTARTS   AGE
pod/webserver-special-889655df-489t4   1/1     Running   0          30s
pod/webserver-special-889655df-6pxqw   1/1     Running   0          26s
pod/webserver-special-889655df-8tfxs   1/1     Running   0          27s
pod/webserver-special-889655df-bg2vl   1/1     Running   0          30s
pod/webserver-special-889655df-jdtjg   1/1     Running   0          30s

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/webserver-special-58b5ff774c   0         0         0       2m48s
replicaset.apps/webserver-special-889655df     5         5         5       30s

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/webserver-special   5/5     5            5           2m48s

~/dep$
```

The replicaSet hash is different and there are 5

- Delete the deployment when finished

```
~/dep$ kubectl delete -f websvr-spec.yaml

deployment.apps "webserver-special" deleted

~/dep$
```

_Copyright (c) 2013-2023 RX-M LLC, Cloud Native Consulting, all rights reserved_