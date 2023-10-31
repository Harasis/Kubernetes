![RX-M, llc.](http://rx-m.io/rxm-cnc.svg)


# Kubernetes


## Pods: Basics

In this lab we will explore the nature of Kubernetes pods and how to work with them.

Like individual application containers, pods are considered to be relatively ephemeral rather than durable entities.
Pods are scheduled to nodes and remain there until termination (according to restart policy) or deletion. When a node
dies, the pods scheduled to that node are deleted. Specific pods are never moved to new nodes; instead, they must be
replaced by running fresh copies of the images on another node.

Kubernetes supports declarative YAML or JSON configuration files. Often times config files are preferable to imperative
commands, since they can be checked into version control and changes to the files can be code reviewed, producing a more
robust, reliable and CI/CD or GitOps friendly system. They can also save a lot of typing if you would like to deploy
complex pods or entire applications.

As a first step in our exploration we will create a simple single container pod.


### 1. A simple pod

To begin our exploration, we’ll create a basic Kubernetes pod from the command line. The easiest way to run a pod is
using the `kubectl run` command. Try creating a simple Apache Web Server pod using the `kubectl run` subcommand as
follows.

```
~$ kubectl run apache --image=httpd:2.2

pod/apache created

~$
```

Now view the pod:

```
~$ kubectl get pod

NAME     READY   STATUS    RESTARTS   AGE
apache   1/1     Running   0          15s

~$
```

What happened here?

The `kubectl run` command takes a name, an image and an API object as parameters and it generates pod template including
the image you specified.

The `run` subcommand syntax is as follows:

```
~$ kubectl run -h | grep COMMAND

  kubectl run NAME --image=image [--env="key=value"] [--port=port] [--dry-run=server|client] [--overrides=inline-json] [--command] -- [COMMAND] [args...] [options]

~$
```

- `--env` switch sets environment variables (just like the `docker run –e` switch)
- `--port` switch exposes ports for service mapping
- `--dry-run` switch (if set to client) allows you to submit the command without executing it to test the parameters
- `--overrides` switch can be used to provide json for fields that don't have flags
- `--command` switch will accept paths to binaries and arguments to pass to those binaries, note the `--` required for the syntax

To get more information about our pod use the `kubectl describe` subcommand:

```
~$ kubectl describe pod apache

Name:             apache
Namespace:        default
Priority:         0
Service Account:  default
Node:             ip-172-31-6-204/172.31.6.204
Start Time:       Mon, 10 Jul 2023 01:38:45 +0000
Labels:           run=apache
Annotations:      <none>
Status:           Running
IP:               10.32.0.4
IPs:
  IP:  10.32.0.4
Containers:
  apache:
    Container ID:   containerd://276601755b2ac2d7a4669246b42abcdb3bfa549a4034d3071abe9b504fe24ebc
    Image:          httpd:2.2
    Image ID:       docker.io/library/httpd@sha256:9784d70c8ea466fabd52b0bc8cde84980324f9612380d22fbad2151df9a430eb
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Mon, 10 Jul 2023 01:38:53 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-pdmfq (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-pdmfq:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  16s   default-scheduler  Successfully assigned default/apache to ip-172-31-6-204
  Normal  Pulling    15s   kubelet            Pulling image "httpd:2.2"
  Normal  Pulled     9s    kubelet            Successfully pulled image "httpd:2.2" in 6.341575262s (6.341583063s including waiting)
  Normal  Created    8s    kubelet            Created container apache
  Normal  Started    8s    kubelet            Started container apache

~$
```

Read through the _Events_ reported for the pod.

You can see that Kubernetes used containerd to pull, create, and start the httpd image requested. You can also
see which part of Kubernetes caused the event. For example the scheduler assigned the pod to the node and then the
Kubelet on the assigned node starts the container.

Kubernetes also injects a standard set of environment variables into the containers of a pod providing the location of
the cluster API service. Use `kubectl exec` to see the pod's environment variables. We'll go into further details about
`kubectl exec` later.

```
~$ kubectl exec apache -- env

PATH=/usr/local/apache2/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=apache
HTTPD_PREFIX=/usr/local/apache2
HTTPD_VERSION=2.2.34
HTTPD_SHA256=e53183d5dfac5740d768b4c9bea193b1099f4b06b57e5f28d7caaf9ea7498160
HTTPD_PATCHES=CVE-2017-9798-patch-2.2.patch 42c610f8a8f8d4d08664db6d9857120c2c252c9b388d56f238718854e6013e46 2.2.x-mod_proxy-without-APR_HAS_THREADS.patch beb66a79a239f7e898311c5ed6a38c070c641ec56706a295b7e5caf3c55a7296
APACHE_DIST_URLS=https://www.apache.org/dyn/closer.cgi?action=download&filename=        https://www-us.apache.org/dist/                                                                         https://www.apache.org/dist/     https://archive.apache.org/dist/
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
HOME=/root

~$
```

The `kubectl get` command also allows you to retrieve pod metadata using the `-o` switch. The `-o` (or `--output`)
switch formats the output of the get command. The output format can be _json_, _yaml_, _wide_, _name_, _template_,
_template-file_, _jsonpath_, or _jsonpath-file_. The golang template specification is also used by Docker (more info
here: http://golang.org/pkg/text/template/).

For example, to retrieve pod data in YAML, try:

```
~$ kubectl get pod apache -o yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2023-07-10T01:38:45Z"
  labels:
    run: apache
  name: apache
  namespace: default
  resourceVersion: "1007"
  uid: 5cab34e1-0d1f-4bb0-8875-fc69e6ade724
spec:
  containers:
  - image: httpd:2.2
    imagePullPolicy: IfNotPresent
    name: apache
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-pdmfq
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: ip-172-31-6-204
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: kube-api-access-pdmfq
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2023-07-10T01:38:45Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2023-07-10T01:38:53Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2023-07-10T01:38:53Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2023-07-10T01:38:45Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: containerd://276601755b2ac2d7a4669246b42abcdb3bfa549a4034d3071abe9b504fe24ebc
    image: docker.io/library/httpd:2.2
    imageID: docker.io/library/httpd@sha256:9784d70c8ea466fabd52b0bc8cde84980324f9612380d22fbad2151df9a430eb
    lastState: {}
    name: apache
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2023-07-10T01:38:53Z"
  hostIP: 172.31.6.204
  phase: Running
  podIP: 10.32.0.4
  podIPs:
  - ip: 10.32.0.4
  qosClass: BestEffort
  startTime: "2023-07-10T01:38:45Z"
```
```
~$
```

We can see the entire pod manifest (spec _and_ status) using the yaml outputter but like `docker container inspect` this
is a lot of information all at once and also like `docker container inspect` it can be parsed for individual properties.

To curl our pod all we need is its IP address. This information is reported dynamically by the kubelet and is part of
the pod's "status", so looking under the `status` section of the excerpts above, you will see that "`podIP`" is one of
the available status keys.

Using the `jsonpath` outputter or `--template` flag, we can extract just the `podIP`, try it:

```
~$ kubectl get pod apache -o jsonpath='{.status.podIP}' && echo

10.32.0.4

~$
```

Using the above command in a nested shell, store the `podIP` in an environment variable and curl the IP address of the
pod to ensure that you can reach the running Apache web server:

```
~$ PIP=$(kubectl get pod apache -o jsonpath='{.status.podIP}') && echo $PIP

10.32.0.4

~$
```

```
~$ curl -I $PIP

HTTP/1.1 200 OK
Date: Mon, 10 Jul 2023 01:39:45 GMT
Server: Apache/2.2.34 (Unix) mod_ssl/2.2.34 OpenSSL/1.0.1t DAV/2
Last-Modified: Sat, 20 Nov 2004 20:16:24 GMT
ETag: "83e86-2c-3e9564c23b600"
Accept-Ranges: bytes
Content-Length: 44
Content-Type: text/html

~$
```

Your Apache web server is reachable from the pod's IP!


### 2. Terminating pods

Because pods house running processes on nodes in the cluster, it is important to allow those processes to gracefully
terminate when they are no longer needed. In Kubernetes, users can request deletion and discover when processes
terminate.

Go ahead and delete the apache pod:

```
~$ kubectl delete pod apache

pod "apache" deleted

~$ kubectl get pod

No resources found in default namespace.

~$
```

You may have noticed a slight delay between your deletion request and when Kubernetes finished the process. When a user
requests deletion of a pod, Kubernetes sends the appropriate termination signal to each container in the pod (either
SIGTERM or the container defined stop signal). Kubernetes then waits for a grace period after which, if the pod has not
shutdown, the pod is forcefully killed with SIGKILL and the pod is then deleted from the API server. If the Kubelet or
the container manager is restarted while waiting for processes to terminate, the termination will be retried with the
full grace period.


### 2.1. CHALLENGE: pod exploration

- Run `kubectl run apache --image httpd` to create another `apache` pod
- Use `kubectl describe` to view the new `apache` pod
  - What image is this new apache pod running?
  - Are there any labels?
- Using the `kubectl delete -h` help page, try to delete the new `apache` pod using any labels


### 3. Pod config files

Let’s try running an nginx container in a pod but this time we’ll create the pod using a YAML configuration file.

```
~$ mkdir ~/pods && cd ~/pods

~/pods$
```

Create a spec for a pod named nginxpod that runs the `nginx:1.11` image on port 80. You can easily do so with `kubectl
run` with the `--dry-run`, and `-o yaml` flags, sending the output to a file:

```
~/pods$ kubectl run nginxpod --image nginx:1.11 --port 80 -o yaml --dry-run=client > nginxpod.yaml
```

Examine the pod spec:

```
~/pods$ cat nginxpod.yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginxpod
  name: nginxpod
spec:
  containers:
  - image: nginx:1.11
    name: nginxpod
    ports:
    - containerPort: 80
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
```
~/pods$
```

> N.B. Creating specs imperatively adds the `dnsPolicy`, `restartPolicy`, `status`, `resources`, and `creationTimestamp`
> keys to the resulting yaml. You can safely remove these keys as Kubernetes will populate default values for them when
> created. You will also see a label `run: podName` in the metadata.

The key `kind` tells Kubernetes we wish to create a pod. The `metadata` section allows us to define a name for the pod
and to apply any other labels we might deem useful.

The `spec` section defines the containers we wish to run in our pod. In our case we will run just a single container
based on the nginx image. The `ports` key allows us to share the ports the pod will be using with the orchestration
layer.

To have Kubernetes create your new pod you can use the `kubectl create` or `kubectl apply` subcommand. The _create_
subcommand will accept a config file via stdin or you can load the config from a file with the `-f` switch (more
common). Whereas _apply_ loads configs from files or whole directories with `-f`. Try the following:

```
~/pods$ kubectl apply -f nginxpod.yaml

pod/nginxpod created

~/pods$
```

Now list the pods on your cluster:

```
~/pods$ kubectl get pods

NAME       READY   STATUS    RESTARTS   AGE
nginxpod   1/1     Running   0          4s

~/pods$
```

Describe your pod:

```
~/pods$ kubectl describe pod nginxpod

Name:             nginxpod
Namespace:        default
Priority:         0
Service Account:  default
Node:             ip-172-31-6-204/172.31.6.204
Start Time:       Mon, 10 Jul 2023 01:40:45 +0000
Labels:           run=nginxpod
Annotations:      <none>
Status:           Running
IP:               10.32.0.4
IPs:
  IP:  10.32.0.4
Containers:
  nginxpod:
    Container ID:   containerd://a6e796b10d4d8f40ae66d709de402ac0316499fcccd918dc9da78578ac854cad
    Image:          nginx:1.11
    Image ID:       docker.io/library/nginx@sha256:e6693c20186f837fc393390135d8a598a96a833917917789d63766cab6c59582
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 10 Jul 2023 01:40:53 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-4zzfz (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-4zzfz:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  13s   default-scheduler  Successfully assigned default/nginxpod to ip-172-31-6-204
  Normal  Pulling    12s   kubelet            Pulling image "nginx:1.11"
  Normal  Pulled     5s    kubelet            Successfully pulled image "nginx:1.11" in 6.745022299s (6.745035923s including waiting)
  Normal  Created    5s    kubelet            Created container nginxpod
  Normal  Started    5s    kubelet            Started container nginxpod

~/pods$
```

Like we did before with _jsonpath_, extract just the status section’s `podIP` value using the `--template` flag and
store that in an environment variable:

```
~/pods$ POD=$(kubectl get pod nginxpod --template={{.status.podIP}}) && echo $POD

10.32.0.4

~/pods$
```

Like the `jsonpath` outputter, the `--template` flag retrieves to the `podIP` key, which is nested under the `.status`
key in the pod spec. The `.key.nestedKey` format is widely used to refer to specific keys in a Kubernetes resource
manifest. For example, if you wanted to retrieve the nginxpod's namespace (from the `kubectl get` command above), you
would enter the template: `.metadata.namespace`.

Now that we have the pod IP we can try curling our nginx server:

```
~/pods$ curl -I $POD

HTTP/1.1 200 OK
Server: nginx/1.11.13
Date: Wed, 08 Dec 2021 18:54:02 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 04 Apr 2017 15:01:57 GMT
Connection: keep-alive
ETag: "58e3b565-264"
Accept-Ranges: bytes

~/pods$
```


### 4. Clean up

Now that we have completed our work with the pod we can delete it declaratively using the manifest:

```
~/pods$ kubectl delete -f nginxpod.yaml

pod "nginxpod" deleted

~/pods$
```

```
~/pods$ kubectl get pod

No resources found in default namespace.

~/pods$
```


### 5. CHALLENGE: a complex pod

Next let’s try creating a pod with a more complex specification.

Create a pod config that describes a pod called `hello` with a:

- container based on an _ubuntu:14.04_ image
- with an environment variable called `MESSAGE` and a value `hello world`
- command that will `echo` that message to stdout: `/bin/sh -c "echo $MESSAGE"`
- make sure that the container is _never restarted_

See if you can design this specification on your own.

The pod and container spec documentation can be found here:

- **Pod Spec Reference** - https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/
- **Container Spec Reference** - https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Container

`kubectl explain` can help here too! For example, if we want to know what the valid fields for a pod are:

```
~/pods$ kubectl explain pod

KIND:     Pod
VERSION:  v1

DESCRIPTION:
     Pod is a collection of containers that can run on a host. This resource is
     created by clients and scheduled onto hosts.

FIELDS:
   apiVersion	<string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources

   kind	<string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds

   metadata	<Object>
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata

   spec	<Object>
     Specification of the desired behavior of the pod. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status

   status	<Object>
     Most recently observed status of the pod. This data may not be up to date.
     Populated by the system. Read-only. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status

~/pods$
```

Since we are interested in the specification or "`pod.spec`", we can use explain to show us the spec fields:

```
~/pods$ kubectl explain pod.spec

KIND:     Pod
VERSION:  v1

RESOURCE: spec <Object>

DESCRIPTION:
     Specification of the desired behavior of the pod. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status

     PodSpec is a description of a pod.

FIELDS:
   activeDeadlineSeconds	<integer>
     Optional duration in seconds the pod may be active on the node relative to
     StartTime before the system will actively try to mark it failed and kill
     associated containers. Value must be a positive integer.

   affinity	<Object>
     If specified, the pod's scheduling constraints


...

~/pods$
```

That's a lot of info! Filter just the objects (without their explanations) with grep:

```
~/pods$ kubectl explain pod.spec | grep "<"

RESOURCE: spec <Object>
  activeDeadlineSeconds	<integer>
  affinity	<Object>
  automountServiceAccountToken	<boolean>
  containers	<[]Object> -required-
  dnsConfig	<Object>
  dnsPolicy	<string>
  enableServiceLinks	<boolean>
  ephemeralContainers	<[]Object>
  hostAliases	<[]Object>
  hostIPC	<boolean>
  hostNetwork	<boolean>
  hostPID	<boolean>
  hostname	<string>
  imagePullSecrets	<[]Object>
  initContainers	<[]Object>
  nodeName	<string>
  nodeSelector	<map[string]string>
  os	<Object>
  overhead	<map[string]string>
  preemptionPolicy	<string>
  priority	<integer>
  priorityClassName	<string>
  readinessGates	<[]Object>
  restartPolicy	<string>
  runtimeClassName	<string>
  schedulerName	<string>
  securityContext	<Object>
  serviceAccount	<string>
  serviceAccountName	<string>
  setHostnameAsFQDN	<boolean>
  shareProcessNamespace	<boolean>
  subdomain	<string>
    "<hostname>.<subdomain>.<pod namespace>.svc.<cluster domain>". If not
  terminationGracePeriodSeconds	<integer>
  tolerations	<[]Object>
  topologySpreadConstraints	<[]Object>

~/pods$
```

`kubectl explain` can navigate through any/all objects/fields, providing explanations, no matter how granular! For
example:

```
~/pods$ kubectl explain pod.spec.containers.ports.containerPort

KIND:     Pod
VERSION:  v1

FIELD:    containerPort <integer>

DESCRIPTION:
     Number of port to expose on the pod's IP address. This must be a valid port
     number, 0 < x < 65536.

~/pods$
```

When you have the configuration complete, create your pod:

```
~/pods$ kubectl apply -f complex-pod.yaml # this file only exists after completing the challenge

pod/hello created

~/pods$
```

List your pods:

```
~/pods$ kubectl get pod

NAME    READY   STATUS      RESTARTS   AGE
hello   0/1     Completed   0          12s

~/pods$
```

We can verify that the container did what it was supposed to do by checking the log output of the pod using the
`kubectl logs` subcommand:

```
~/pods$ kubectl logs hello

hello world

~/pods$
```


### 6. Clean up

Now that we have given our new cluster a good test we can clean up by deleting the pods we have created. The
`kubectl delete` subcommand allows you to delete objects you have created in the cluster.

Delete any pods you created:

```
~/pods$ kubectl get pod

NAME    READY   STATUS      RESTARTS   AGE
hello   0/1     Completed   0          60s

~/pods$
```

```
~/pods$ kubectl delete pod hello

pod "hello" deleted

~/pods$
```

Double-check:

```
~/pods$ kubectl get pod

No resources found in default namespace.

~/pods$
```

You Kubernetes cluster should now be cleaned up:

```
~/pods$ kubectl get services,deployments,replicasets,pods

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   39m

~/pods$
```

Be sure to leave the `service/kubernetes` service!

<br>

Congratulations, you have completed the lab!

<br>

_Copyright (c) 2013-2023 RX-M LLC, Cloud Native Consulting, all rights reserved_
