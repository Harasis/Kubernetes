![RX-M, llc.](http://rx-m.io/rxm-cnc.svg)


# Kubernetes


## Pod Architecture, Resource Requirements, Init Containers, and Health Checks

In Kubernetes, pods are the smallest deployable units that can be created, scheduled, and managed. A pod corresponds to
a collocated group of containers running with a shared context. A pod models an application-specific "logical host" in a
containerized environment. It may contain one or more applications which are relatively tightly coupled — in a
pre-container world, they would have executed on the same physical or virtual host.

The context of the pod can be defined as the conjunction of several Linux namespaces:

- **Network** - applications within the pod have access to the same IP and port space
- **IPC** - applications within the pod can use SystemV IPC or POSIX message queues to communicate
- **Hostname and resolver** (parts of the UTS namespace) - applications within a pod have the same hostname and resolver
  configuration but will have separate UTS namespaces

Applications within a pod can also have access to shared volumes, which are defined at the pod level and made available
in each application's file system. Additionally, a pod may define top-level cgroup isolations which form an outer bound
to any individual isolation applied to constituent containers.


### 1. Containers and pods

We have seen how to launch a single container pod, but did not describe it. Here we investigate what are the components
and build upon this initial deployment configuration, to help us understand, what "IS" a pod.

Create a holding directory, change into it, and launch a container based on the` docker.io/nginx:1.21` image in a pod.

```
~$ mkdir -p ~/pods ; cd $_

~/pods$ kubectl run my-nginx --image=docker.io/nginx:1.21 --port=80

pod/my-nginx created

~/pods$
```

The pod name is `my-nginx` and the image we used is `docker.io/nginx:1.21`, an official image pulled from Docker Hub by
the Docker Engine in the background. The `--port` switch tells Kubernetes the service port for our pod which will allow
us to share the service with its users over that port (the program must actually use that port for this flag to work).

List the pods running on the cluster with wide `-o wide` output:

```
~/pods$ kubectl get pods -o wide

NAME       READY   STATUS    RESTARTS   AGE   IP          NODE               NOMINATED NODE   READINESS GATES
my-nginx   1/1     Running   0          6s    10.44.0.1   ip-172-31-35-194   <none>           <none>

~/pods$
```

This shows that our pods are deployed and up to date. It may take a bit to pull the Docker image (READY might be 0/1)).

> nb. NOMINATED NODE - indicates which node a high priority pod may land on (aka preempting another pod on that node)
> nb. READINESS GATES - additional 'conditions' to indicate it is "READY" - view conditions via "describe"

To see the actual containers run by the pod, you need to ask the container runtime that Kubernetes is using directly.
This can be in the form of `docker`-based commands, or through tools like containerd's bundled `ctr` or the CRI-focused
`crictl`. In this lab, we will use `crictl`, which is a generic container runtime client primarily for debugging
containers launched by some orchestration tool (like Kubernetes).

`crictl` gets installed by the `cri-tools` packages (which is pulled as a dependency of `kubeadm`). Chances are, you
already have it installed. But to be sure, install the `cri-tools` package with `apt`:

> N.B. In a multi-node cluster, run these commands on your worker node.

```
~/pods$ sudo apt install cri-tools

Reading package lists... Done
Building dependency tree
Reading state information... Done
cri-tools is already the newest version (1.26.0-00).
cri-tools set to manually installed.
0 upgraded, 0 newly installed, 0 to remove and 3 not upgraded.

~/pods$
```

Since `crictl` is a generic tool, it tries all possible container sockets to see if containers are running. This leads
to warnings that it cannot find certain sockets.

To prevent the warnings, confirm that `/etc/crictl.yaml` is there or create it to tell crictl where to find the
containerd API:

```
~$ sudo nano /etc/crictl.yaml ; sudo cat $_

runtime-endpoint: unix:///run/containerd/containerd.sock

~$
```

> N.B. We use `$_` in commands with `&&` to refer to the file opened by the previous command.

Now you can use `crictl ps` to list containers and `-l "name=nginx"` to apply a filter to reduce the amount of results.

```
~$ sudo crictl ps --name=nginx

CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID              POD
f47c4cd048f13       0e901e68141fd       2 minutes ago       Running             my-nginx            0                   322a69ab38fb4       my-nginx

~$
```

There's our application container - but that's not the only container launched by the Kubelet for our pod. Note the
`POD ID` column - this refers that yet-to-be-seen container.

In Kubernetes, each Pod instance has an infrastructure container, which is the first container that the Kubelet
instantiates. The infrastructure container uses the image `k8s.gcr.io/pause` and acquires the pod’s IP as well as a pod
wide network and IPC namespace. All of the other containers in the pod then join the infrastructure container’s network
(`--net`) and IPC (`--ipc`) namespace allowing containers in the pod to easily communicate. The initial process
(`/pause`) that runs in the infrastructure container does nothing, its sole purpose is to act as the anchor for the pod
and its shared namespaces.

`crictl` makes the distinction between application containers (the ones you define in your YAML) and the infrastructure
container (the one that Kubelet requests to provide the pod's shared namespaces). To see the infrastructure container,
you need to use `crictl pods` to list pods.

Try to list the pods that `crictl` is aware of:

```
~$ sudo crictl pods --name=nginx

POD ID              CREATED             STATE               NAME                NAMESPACE           ATTEMPT             RUNTIME
322a69ab38fb4       2 minutes ago       Ready               my-nginx            default             0                   (default)

~$
```

You can learn more about the pause container by looking at the source and ultimately what is `pause()`.

- https://github.com/kubernetes/kubernetes/tree/master/build/pause
- https://github.com/kubernetes/kubernetes/blob/master/build/pause/linux/pause.c
- `man 2 pause` or http://man7.org/linux/man-pages/man2/pause.2.html
  - If a man page is missing try `sudo apt install manpages-dev manpages-posix-dev`
  - If you are not sure where a page is `apropos pause`

The Docker listing showed us 2 containers, the pod having an infrastructure container (`pause`) and the container we
asked for (`nginx`).

Kubernetes gives each pod a name and reports on the pod status, the number of times a container in the pod has been
restarted and the pod’s uptime:

```
~$ sudo crictl ps --name=nginx

CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID              POD
f47c4cd048f13       0e901e68141fd       2 minutes ago       Running             my-nginx            0                   322a69ab38fb4       my-nginx

~$
```

Try killing the nginx container using the `crictl stop` subcommand using the nested and filtered (`-f`)
`crictl ps` command to provide the ID of the underlying container based on the nginx image.

```
~$ sudo crictl stop $(sudo crictl ps -q --name nginx)

f47c4cd048f13843fca4e59f6a4fd1208e36180cefa3fa385e0def6e9c0ad522

~$
```

List your nginx containers using the `-a` to include all (running and stopped):

```
~$ sudo crictl ps -a --name nginx

CONTAINER           IMAGE               CREATED              STATE               NAME                ATTEMPT             POD ID              POD
175d115e79736       0e901e68141fd       About a minute ago   Running             my-nginx            1                   322a69ab38fb4       my-nginx
f47c4cd048f13       0e901e68141fd       5 minutes ago        Exited              my-nginx            0                   322a69ab38fb4       my-nginx

~$
```

We can tell by the created time we have a new container. containerd terminates the container specified but Kubernetes has no
knowledge of this action. When the Kubelet process, responsible for the pods assigned to this node, sees the missing
container, it simply reruns a new container from the nginx image.

Kubernetes does not "resurrect" containers that have failed. This is important because the container’s state may be the
reason it failed. Rather, Kubernetes runs a fresh copy of the original image, ensuring the container has a clean new
internal state (cattle not pets!).

> nb. the container ID has also updated, notice the end has a ATTEMPT count (`_1`, different from `_0`)

Having killed the container on the Docker level, check to see how Kubernetes handled the event:

> N.B. In a multi-node cluster, run the following command on your control plane node

```
~/pods$ kubectl get pod

NAME       READY   STATUS    RESTARTS        AGE
my-nginx   1/1     Running   1 (3m55s ago)   7m22s

~/pods$
```

When Kubelet saw that the pod's container had become unavailable, it "restarted" the container (by replacing it) in the
pod, incrementing the number of restarts. The pod itself is still the same, as seen by its AGE _not_ rotating to a
smaller number. In multi-container pods, to figure out which container restarted, you will need to do more investigation
using `events` or `describe`. In theory, the pause container must disappear to make a pod disappear, that is considered
a non-recoverable error in K8s though.

Using the `--template` flag, store the `podIP` in an environment variable and curl the IP address of the pod to ensure
that you can reach the running Nginx web server:

```
~/pods$ NGIP=$(kubectl get pod my-nginx --template={{.status.podIP}}) && echo $NGIP

10.44.0.1

~/pods$ curl -I $NGIP

HTTP/1.1 200 OK
Server: nginx/1.21.6
Date: Mon, 18 Sep 2023 05:15:57 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 25 Jan 2022 15:03:52 GMT
Connection: keep-alive
ETag: "61f01158-267"
Accept-Ranges: bytes

~/pods$
```


### 2. Pod exec

While Kubernetes delegates all of the direct container operations to the container runtime it does pass through some
useful container features (ie. logs, attach, exec).

For example, imagine you need to discover the distro of one of your pods' containers. You can use the `kubectl exec`
subcommand to run arbitrary commands within a pod.

Try listing the running pods and then executing the `cat /etc/os-release` command within one of your pods:

```
~/pods$ kubectl get pods

NAME       READY   STATUS    RESTARTS   AGE
my-nginx   1/1     Running   0          2m15s

~/pods$ kubectl exec my-nginx -- cat /etc/os-release

PRETTY_NAME="Debian GNU/Linux 11 (bullseye)"
NAME="Debian GNU/Linux"
VERSION_ID="11"
VERSION="11 (bullseye)"
VERSION_CODENAME=bullseye
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"

~/pods$
```

Running `cat /etc/os-release` via `kubectl exec` produces the information we needed. The `exec` subcommand chooses the
first container within the pod to execute the command. The command to run on the pod was separated from the rest of the
`kubectl` invocation with `--`.

> nb. the pause container is not considered when using exec

If you would like to execute the command within a specific container in a multi-container pod, you can use the `-c`
switch.

Try it; first find a deployed pod with more than one container:

```
~/pods$ kubectl get pods --all-namespaces

NAMESPACE     NAME                                       READY   STATUS    RESTARTS        AGE
default       my-nginx                                   1/1     Running   1 (4m47s ago)   8m14s
kube-system   coredns-5d78c9869d-dd4q9                   1/1     Running   0               23m
kube-system   coredns-5d78c9869d-jp8s6                   1/1     Running   0               23m
kube-system   etcd-ip-172-31-46-242                      1/1     Running   0               22m
kube-system   kube-apiserver-ip-172-31-46-242            1/1     Running   0               22m
kube-system   kube-controller-manager-ip-172-31-46-242   1/1     Running   0               22m
kube-system   kube-proxy-cz979                           1/1     Running   0               16m
kube-system   kube-proxy-gjnqb                           1/1     Running   0               23m
kube-system   kube-scheduler-ip-172-31-46-242            1/1     Running   0               22m
kube-system   weave-net-c7dlz                            2/2     Running   1 (43m ago)     43m
kube-system   weave-net-zkfm4                            2/2     Running   0               16m

~/pods$
```

The weave-net pod for our cluster's networking has two containers in it.

Try to check the os-release on that pod, _replacing the example pod name with the name of the weave pod from your
cluster_ and making sure to use the `--namespace kube-system` so kubectl knows which namespace to use:

> N.B. Use tab completion to help identify which pod to exec into! Enable it using `source <(kubectl completion bash)`

```
~/pods$ kubectl --namespace kube-system exec weave-net-c7dlz -- cat /etc/os-release  # use one of the weave-net pods

Defaulted container "weave" out of: weave, weave-npc, weave-init (init)
NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.10.5
PRETTY_NAME="Alpine Linux v3.10"
HOME_URL="https://alpinelinux.org/"
BUG_REPORT_URL="https://bugs.alpinelinux.org/"

~/pods$
```

Note the first line of output:

> Defaulted container "weave" out of: weave, weave-npc, weave-init (init)

The weave-net pod has 3 containers and kubectl has let you know that it defaulted to the "weave" container. Pods with
multiple containers can use the label: `kubectl.kubernetes.io/default-container` to have a container preselected for
kubectl commands!

If you wanted to discover the names of the containers without exec-ing into the default you can use the `jsonpath`
outputter:

`kubectl get pod -n kube-system weave-net-c7dlz -o jsonpath='{.spec.containers[*].name}'`

Now, use the `–c` switch to display the `os-release` file from the `weave-npc` container in the weave-net pod,
_once again replace the example pod name with the name of the weave pod from your cluster_:

```
~/pods$ kubectl --namespace kube-system exec weave-net-c7dlz -c weave-npc -- cat /etc/os-release

NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.10.5
PRETTY_NAME="Alpine Linux v3.10"
HOME_URL="https://alpinelinux.org/"
BUG_REPORT_URL="https://bugs.alpinelinux.org/"

~/pods$
```

The output of "`/etc/os-release`" is often useful, but not so much above. Each container is using the same base image.
Instead of "`cat`", lets try "`ps`":

- `kubectl --namespace kube-system exec -c weave weave-net-c7dlz -- ps`
- `kubectl --namespace kube-system exec -c weave-npc weave-net-c7dlz -- ps`

If you compare their output, we see a difference!

Being able to execute commands on pods ad-hoc can be very useful in debugging and normal operation scenarios.

Delete the pod before proceeding:

```
~/pods$ kubectl delete pod my-nginx

pod "my-nginx" deleted

~/pods$
```


### 3. Resource requirements

Next let’s explore resource requirements. Kubernetes configs allow you to specify requested levels of memory and cpu.
You can also assign limits. Requests are used when scheduling the pod to ensure that it is place on a host with enough
resources free. Limits are configured in the kernel cgroups to constrain the runtime use of resources by the pod.

Resource requests and limits are expressed in mils for cpu (`m`, where 1000m of CPU represents 1 CPU core). Memory is
expressed in megabytes (`Mi`) or gigabytes (`Gi`).

Create a new pod config (_limit.yaml_). Start with an imperative command to create a base pod spec:

```
~/pods$ kubectl run limit --image mysql:8 --dry-run=client -o yaml > limit.yaml
```

This prepares a pod spec for the first container. We want to include another container running wordpress, so run another
imperative command to generate a wordpress container spec that you will subsequently add to the `limit.yaml` manifest:

```
~/pods$ kubectl run wp --image wordpress:5.8 --dry-run=client -o yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: wp
  name: wp
spec:
  containers:
  - image: wordpress:5.8
    name: wp
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
```
~/pods$
```

Make the following changes:

- Take the wordpress entry from the `spec.containers` array of the wordpress pod and add it to the `spec.containers`
  array in the `limit.yaml` you generated
- Populate the `resources:` array in the `limit` container that specifies a `500m` cpu and `512Mi` limit and `250m` cpu
  and `256Mi` request
- Populate the `resources:` array in the `wp` container that specifies a `500m` cpu and `128Mi` limit and `250m` cpu and
  `64Mi` request

```
~/pods$ nano limit.yaml ; cat $_
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limit
spec:
  containers:
  - image: mysql:8
    name: limit
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
      requests:
        cpu: 250m
        memory: 256Mi
  - image: wordpress:5.8
    name: wp
    resources:
      limits:
        cpu: 500m
        memory: 128Mi
      requests:
        cpu: 250m
        memory: 64Mi
```
```
~/pods$
```

> nb. Our example above removed null fields, you can leave them if you desire (e.g. `creationTimestamp: null`)

This config will run a pod with the Wordpress image and the MySql image with explicit requested resource levels and
explicit resource constraints.

Before you create the pod verify the resources on your node. First examine the Capacity field for your node.

```
~/pods$ sudo apt update && sudo apt install -y jq

...

~/pods$ kubectl describe $(kubectl get node -o name) | grep -A 8 Capacity
```
```
Capacity:
  cpu:                2
  ephemeral-storage:  50620216Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3968556Ki
  pods:               110
Allocatable:
  cpu:                2
```

If you have multiple nodes, the output of the above command includes results for all of them.

Next examine the Allocated resources section.

```
~/pods$ kubectl describe $(kubectl get node -o name) | grep -A 8 Allocated

Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests   Limits
  --------           --------   ------
  cpu                100m (5%)  0 (0%)
  memory             0 (0%)     0 (0%)
  ephemeral-storage  0 (0%)     0 (0%)
  hugepages-1Gi      0 (0%)     0 (0%)
  hugepages-2Mi      0 (0%)     0 (0%)

~/pods$
```

Will the node be able accept the scheduled pod?

Now create the new pod and verify its construction:

```
~/pods$ kubectl apply -f limit.yaml

pod/limit created

~/pods$ kubectl get pod limit

NAME    READY   STATUS              RESTARTS   AGE
limit   0/2     ContainerCreating   0          11s

~/pods$
```

Use the `kubectl describe pod limit` or `kubectl get events | grep -i limit` command to display the events for
your new pod. You may see the image pulling. This means the Docker daemon is pulling the image in the background.

Once the image has pulled, watch the pod status with the `-w` flag (which you can stop using CTRL C):

```
~/pods$ kubectl get pod -w

NAME    READY   STATUS              RESTARTS   AGE
limit   0/2     ContainerCreating   0          7s
limit   1/2     Error               0          27s
limit   2/2     Running             1 (15s ago)   28s
limit   1/2     Error               1 (18s ago)   31s
limit   1/2     CrashLoopBackOff    1 (16s ago)   44s

^C

~/pods$
```

It errored out (or may be in a CrashLoopBack cycle)! You will have the opportunity to fix this pod during one of the
challenges later in this document.

Regardless of whether the application is actually running, the pod still has resources set aside for it on the host.

Rerun the `describe` command to redisplay your node resource usage:

```
~/pods$ kubectl describe $(kubectl get node -o name) | grep -A 8 Allocated

Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                600m (30%)  1 (50%)
  memory             320Mi (8%)  640Mi (16%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)

~/pods$
```

Can you see the impact of the new pod on the host? How does it correspond with the resource limits and requests you
listed in the pod spec?

Delete the limit pod:

```
~/pods$ kubectl delete -f limit.yaml

pod/limit deleted

~/pods$ kubectl get pod limit
```


### 4. Init containers

Pods with Init Containers run some container(s) to completion first before starting main containers. Init containers are
best used to perform preparatory steps before a main application container in a pod runs. Init Containers have several
benefits:

- Application containers can be built without tools that may increase the filesize of the image or introduce security risks
- A separation of bootstrap operations and logs from the main application container contributes to making the overall app easier to debug
- Mandatory completion of init containers reduces or eliminates the chance that application containers start unprepared
- Environment- or use-case specific configurations can be applied to generic application containers without the need to create or modify a new image

Examine the following Pod manifest (we'll review Volumes, Volume Mounts, and emptyDir in a another lab):

```
~/pods$ nano init-pod.yaml ; cat $_
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-pod
  labels:
    app: init-pod
spec:
  initContainers:
  - name: init-container
    image: rxmllc/alpine-git:0.1
    command: ["bin/sh", "-c"]
    args: ["git clone https://github.com/RX-M/hostinfo.git hostinfo"]
    volumeMounts:
    - name: tempvol
      mountPath: /hostinfo
  containers:
  - name: app-container
    image: busybox
    command: ["bin/sh", "-c"]
    args: ["ls /hostinfo ; head -3 /hostinfo/README.md"]
    volumeMounts:
    - name: tempvol
      mountPath: /hostinfo
  volumes:
  - name: tempvol
    emptyDir: {}
  restartPolicy: Never
```
```
~/pods$
```

- The Pod has two containers: an Init Container named `init-container` and a main container called `app-container`
- The two containers share an `emptyDir` volume. An `emptyDir` volume is an ephemeral volume that is removed when the
  Pod is removed. An emptyDir is initially an empty directory and is shareable between containers of the same Pod.
- The Init Container uses an image with git and clones the RX-M/hostinfo GitHub repo in a shared Volume.
- The main container lists the contents of the shared Volume to confirm the repo was cloned
- The main container then prints the first three lines of README.md from the RX-M/hostinfo repo.

Save this manifest as `init-pod.yaml`, run the Pod and check the logs both containers.

```
~/pods$ kubectl apply -f init-pod.yaml

pod/init-pod created

~/pods$ kubectl logs init-pod -c init-container

Cloning into 'hostinfo'...

~/pods$ kubectl logs init-pod -c app-container

Jenkinsfile
README.md
dockerfile
hi.go
# hostinfo

Simple Go microservice which returns a host info string. Accessible via curl, wget, etc.. Listens on port 9898 by default, if an int is passed on the command line that port is used. Particularly useful when demonstrating k8s service routing mesh operation (deploy several replicas of these, create a service for them then curl the service sequentially to see various hosts engaged).

~/pods$
```


### 5. Health checks

Health checks in Kubernetes are powered by probes defined in a container spec of a pod template. These probes are
usually configured to send a request to a health check endpoint or run some command to see if an application container
is up, ready, or still starting up. There are three different kinds of probes in Kubernetes:

- Liveness - Used by kubelets to know when to restart a container in a pod
- Startup - Used by kubelets to know when a container application has started, preventing liveness and readiness probes
  from running until the startup is complete.
- Readiness - Used by kubelets to know when a container can start accepting network traffic

In this step we will create a pod with a health check. Enter and run the following config (_hc.yaml_):

```
~/pods$ kubectl run nginx-hc --image docker.io/nginx:latest --dry-run=client -o yaml > hc.yaml

~/pods$
```

Make the following edits hc.yaml:
- Add a liveness probe that sends an http get to port 80 at the root path.
  - Give the probe an initial delay of 30 seconds
  - Provide the probe with a 1 second timeout

```
~/pods$ nano hc.yaml ; cat $_
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx-hc
  name: nginx-hc
spec:
  containers:
  - image: docker.io/nginx:latest
    name: nginx-hc
    resources: {}
    livenessProbe:             # Add this
      tcpSocket:               # Add this
        port: 80               # Add this
      initialDelaySeconds: 30  # Add this
      timeoutSeconds: 1        # Add this
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

Now submit the pod to the API:

```
~/pods$ kubectl apply -f hc.yaml

pod/nginx-hc created

~/pods$ kubectl get pod nginx-hc

NAME       READY   STATUS    RESTARTS   AGE
nginx-hc   1/1     Running   0          25s

~/pods$
```

Note that our nginx service listens on port 80 and responds normally to requests for “`/`”, so our health check is
passing.

To trigger the health check repair logic, we need to simulate an error condition. By forcing nginx to report a `404`,
the *httpGet* livenessProbe will fail. We can do this by deleting the nginx configuration file in the nginx container.

Display the events for the first pod in the set:

```
~/pods$ kubectl get events --sort-by=metadata.creationTimestamp | grep nginx-hc

52s         Normal    Scheduled                 pod/nginx-hc                      Successfully assigned default/nginx-hc to ip-172-31-35-194
52s         Normal    Pulling                   pod/nginx-hc                      Pulling image "docker.io/nginx:latest"
50s         Normal    Pulled                    pod/nginx-hc                      Successfully pulled image "docker.io/nginx:latest" in 2.599405716s (2.599417506s including waiting)
50s         Normal    Created                   pod/nginx-hc                      Created container nginx-hc
50s         Normal    Started                   pod/nginx-hc                      Started container nginx-hc

~/pods$
```

The status is good.

Now lets tell the nginx in the first pod to stop serving the root IRI by deleting the nginx default config:

```
~/pods$ kubectl exec -it nginx-hc -- sh -c "rm /etc/nginx/conf.d/default.conf && nginx -s reload"

2022/11/11 19:34:16 [notice] 38#38: signal process started

~/pods$
```

Now redisplay the events for the pod:

```
~/pods$ kubectl get events --sort-by=metadata.creationTimestamp | grep nginx-hc

97s         Normal    Scheduled                 pod/nginx-hc                      Successfully assigned default/nginx-hc to ip-172-31-35-194
97s         Normal    Pulling                   pod/nginx-hc                      Pulling image "docker.io/nginx:latest"
95s         Normal    Started                   pod/nginx-hc                      Started container nginx-hc
95s         Normal    Created                   pod/nginx-hc                      Created container nginx-hc
95s         Normal    Pulled                    pod/nginx-hc                      Successfully pulled image "docker.io/nginx:latest" in 2.599405716s (2.599417506s including waiting)
8s          Warning   Unhealthy                 pod/nginx-hc                      Liveness probe failed: dial tcp 10.44.0.2:80: connect: connection refused

~/pods$
```

What happened?

Events reported by the event stream are not as granular as those provided by the describe, try it:

```
~/pods$ kubectl describe pod nginx-hc | grep -A 15 Events

Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Normal   Scheduled  2m18s                default-scheduler  Successfully assigned default/nginx-hc to ip-172-31-35-194
  Normal   Pulled     2m16s                kubelet            Successfully pulled image "docker.io/nginx:latest" in 2.599405716s (2.599417506s including waiting)
  Normal   Pulling    29s (x2 over 2m18s)  kubelet            Pulling image "docker.io/nginx:latest"
  Warning  Unhealthy  29s (x3 over 49s)    kubelet            Liveness probe failed: dial tcp 10.44.0.2:80: connect: connection refused
  Normal   Killing    29s                  kubelet            Container nginx-hc failed liveness probe, will be restarted
  Normal   Created    28s (x2 over 2m16s)  kubelet            Created container nginx-hc
  Normal   Started    28s (x2 over 2m16s)  kubelet            Started container nginx-hc
  Normal   Pulled     28s                  kubelet            Successfully pulled image "docker.io/nginx:latest" in 200.052505ms (200.077475ms including waiting)

~/pods$
```

As you can see the liveness probe failed at one point, but recovered. The nginx container in the pod was created,
started, found unhealthy, killed, created and started again.


### 6. Clean up

Delete any remaining pod specs you created during this lab by using `kubectl delete -f` on the `~/pods` directory:

```
~/pods$ kubectl get pods

NAME       READY   STATUS      RESTARTS      AGE
init-pod   0/1     Completed   0             54s
nginx-hc   1/1     Running     1 (49s ago)   2m39s

~/pods$ kubectl delete pods init-pod nginx-hc

pod "init-pod" deleted
pod "nginx-hc" deleted

~/pods$
```

You may have already removed some pods, so kubectl will generate errors for any pods that do not exist.


### 6. Challenge: pods and Linux namespaces

Using the pod spec reference as a guide, create a podspec using `nginx:1.21` so that it runs all of the containers in
the pod in the _host network namespace_ (hostNetwork)

The pod and container spec documentation can be found here:

- **Pod Spec Reference** - https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/
- **Container Spec Reference** - https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Container

- Generate a pod spec to serve as your base:
  `kubectl run nginx-ns --image nginx:1.21 --port 80 -o yaml --dry-run=client > nginx-ns.yaml`
- Next modify it to use the `hostNetwork` (use the spec reference if you need help thinking through the edits you
will need to make): `nano nginx-ns.yaml`
- Run the new pod: `kubectl apply -f nginx-ns.yaml`
- Confirm your changes by displaying your pod status in yaml output and use `grep` to view the hostIP and podIP fields:
`kubectl get pod nginx-ns -o yaml | egrep ' (host|pod)IP:'`

If your pod is running in the host network namespace you should see that the pod (_podIP_) and host IP (_hostIP_)
address are identical. Namespace select-ability allows you have the benefits of container deployment while still
empowering infrastructure tools to see and manipulate host based networking features.

- Try curling your pod using the host IP address: `curl -I $(hostname -i)`
- Delete the `nginx-ns` pod: `kubectl delete -f nginx-ns.yaml`

> n.b. You can delete a resource(s) via the name or via the file where `metadata.name` and `kind` fields match


### 7. CHALLENGE: multi-container pod

In this step we’ll experiment with multi-container pods. Keep in mind that by default all containers in a pod share the
same `network` and `ipc` namespaces.

Create a new Pod config which:
  - Has the pod name `hello`
  - runs two `ubuntu:18.04` containers, named `hello1` and `hello2`
  - both executing the command line `tail -f /dev/null`

- Start from a run command and use as many flags as you need to help you create the YAML spec: `kubectl run ....`
- Then edit the new config to meet the challenge requirements: `nano <filename>.yaml`
- Create the pod: `kubectl apply -f <filename>.yaml`

If you got it right the first time the pod will simply run. If not you will get a descriptive error.

- Issue the `kubectl get pods` command.
  -	How many containers are running in your new pod?
  -	How can you tell?

- Use the `kubectl describe pod` subcommand on your new pod.
-	Is there any information about the IP addresses of the containers themselves in this output?

- Shell into the first container to explore its context using the `-c` option with the name of the first container, in
  the example `hello1`: `kubectl exec -it hello -c hello1 -- /bin/bash`
  - Run the `ps -ef` command inside the container.
    -	What processes are running in the container?
    -	What is the container’s host name?
  - Run the `ip a` command inside the container.
    - Run `apt update`
    - Run `apt install iproute2 -y`
    -	What is the MAC address of eth0?
    -	What is the IP address
  - Create a file in the root directory and exit the container:

```
root@hello:/# echo "Hello" > TEST

root@hello:/# exit
```

Kubernetes executed our last command in the first container in the pod. We now want to open a shell into the second
container. To do this we can use the _-c_ switch.

- Exec a shell into the second container using the _-c_ switch and the name of the second container:
  `kubectl exec -it hello -c hello2 -- /bin/bash`
  - Is the TEST file you created previously there?
  - What is the host name in this container?
  - What is the MAC address of eth0?
  - What is the IP address?
  - Which of the following namespaces are shared across the containers in the pod?
    - User
    - Process
    - UTS (hostname & resolver config)
    - Network
    - IPC
    - Mount (filesystem)

Delete the pod: `kubectl delete -f <filename>.yaml`


### 8. CHALLENGE: troubleshooting

- Recreate the limit pod: `kubectl apply -f limit.yaml`
- Try to diagnose any issues using:
  - `kubectl describe pod limit` to find out which container is crashing and
  - `kubectl logs limit` to see what error it is reporting.
  > Hint: some containerized applications need more configuration than others

Delete the limit pod after examining its resource usage: `kubectl delete pod limit`


<br>

Congratulations, you have completed the lab!

<br>

_Copyright (c) 2013-2023 RX-M LLC, Cloud Native Consulting, all rights reserved_
