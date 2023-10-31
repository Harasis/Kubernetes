![RX-M, llc.](http://rx-m.io/rxm-cnc.svg)


# Kubernetes


## Volumes

On-disk files in a container are ephemeral, which presents some problems for applications that require data persistence.
When a container in a pod crashes, the _kubelet_ will replace it by rerunning the original **image** within the pod;
thus the ephemeral files from the dead container will be lost on any container restart.

Each container also has its own isolated filesystem, so if multiple containers in a pod need to share files, there's no
way to do so via the normal container filesystem.

Both of the above problems are solved by volumes. Volumes have a lifespan independent of any given container. A volume
in use by a pod will persist across container restarts, avoiding data loss when containers crash. In fact, network
attached volumes can even chase a pod across machines in scenarios where the pod gets evicted from one node and
relaunched on another. Volumes can also be mounted by many containers within a given pod, allowing containers to share
files and/or communication through Unix domain sockets, etc.

Kubernetes volumes can be dynamically provisioned and deleted in conjunction with the lifespan of a pod, but they can
also be created statically and used over and over again. Different applications may require different volume types and
features, fortunately Kubernetes supports pluggable volume drivers and a wide range of settings, allowing users to
configure various storage solutions. Pods can use several different volumes and/or volume types simultaneously if
needed.

Using volumes in Kubernetes involves identifying the volumes at the pod level and then mounting them in the various
containers as needed. A pod specifies volumes to provide using the `spec.volumes` field and containers specify where to
mount pod volumes using the `spec.containers.volumeMounts` field.

A process in a container sees a filesystem view composed of the roof filesystem from the container image and the various
volumes mounted into the root filesystem. Volumes cannot mount onto other volumes or have hard links to other volumes.
Each container in the pod must independently specify where to mount each volume.


### 0. Optional Kubernetes Setup

Before proceeding, make sure to have an active Kubernetes cluster on your VM:

```
~$ wget -qO - https://raw.githubusercontent.com/RX-M/classfiles/master/k8s.sh | sh

...

~$ sudo usermod -aG docker ubuntu

~$ exit

student@laptop:~$ ssh -i key.pem ubuntu@55.55.55.55

...

~$
```


### 1. Using Volumes

Imagine we have an application assembly which involves two containers. One container runs a Redis cache and the other
runs an application that uses the cache. Using a volume to host the Redis data will ensure that if the Redis
container crashes, we can have the `kubelet` start a brand new copy of the Redis image but hand it the pod volume,
preserving the state across crashes.

To simulate this case we’ll start a Deployment with a two container pod. One container will be Redis and the other will
be BusyBox. We’ll mount a shared volume into both containers.

Create a working directory for your project:

```
~$ mkdir ~/vol && cd $_

~/vol$
```

Next create the following Deployment config by generating the manifest:

```
~/vol$ kubectl create deploy sharing-redis --image=redis,busybox --dry-run=client -o yaml > vol.yaml

~/vol$
```

Edit the manifest, making the following changes:

- Add a command to the busybox container that runs `tail -f /dev/null`
- Define an emptyDir volume called `data`
- Mount the `data` emptyDir in the redis container at `/data` and busybox container at `/shared-master-data`
- Remove the empty `status: {}` and `strategy: {}` keys
- Remove any `creationTimestamp: null` keys that exist
- Remove any empty `resource: {}` keys that exist

```
~/vol$ nano vol.yaml && cat $_
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: sharing-redis
  name: sharing-redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sharing-redis
  template:
    metadata:
      labels:
        app: sharing-redis
    spec:
      containers:
      - image: redis
        name: redis
        volumeMounts:
        - mountPath: /data
          name: data
      - image: busybox
        name: busybox
        command:
        - tail
        - -f
        - /dev/null
        volumeMounts:
        - mountPath: /shared-master-data
          name: data
      volumes:
      - name: data
        emptyDir: {}
```
```
~/vol$
```

Here our spec creates an emptyDir volume called _data_ and then mounts it into both containers. Create the Deployment
and then when both containers are running we will _exec_ into them to explore the volume.

First launch the deployment and wait for the pod containers to come up (redis may need to pull from Docker Hub):

```
~/vol$ kubectl apply -f vol.yaml

deployment.apps/sharing-redis created

~/vol$
```

Check the status of your new resources:

```
~/vol$ kubectl get deploy,rs,po

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/sharing-redis   1/1     1            1           4s

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/sharing-redis-6fcd7b656f   1         1         1       4s

NAME                                 READY   STATUS    RESTARTS   AGE
pod/sharing-redis-6fcd7b656f-cxhfh   2/2     Running   0          4s

~/vol$
```

When all of your containers are ready, _exec_ into the "busybox" container of your sharing-redis pod and create a file
in the shared volume:

```
~/vol$ kubectl exec -it -c busybox $(kubectl get pod -l app=sharing-redis -o name) -- /bin/sh

/ # ls -l /shared-master-data/

total 0

/ # echo "hello shared data" > /shared-master-data/hello.txt

/ # ls -l /shared-master-data/

total 4
-rw-r--r--    1 root     root            18 Feb  4 23:46 hello.txt

/ # exit

~/vol$
```

Finally exec into the “redis” container to examine the volume:

```
~/vol$ kubectl exec -it -c redis $(kubectl get pod -l app=sharing-redis -o name) -- /bin/sh

# ls -l /data

total 4
-rw-r--r-- 1 root root 18 Feb  4 23:46 hello.txt

# cat /data/hello.txt

hello shared data

# exit

~/vol$
```

By mounting multiple containers inside a pod to the same shared volumes, you can achieve interoperability between those
containers. Some useful scenarios would be to have a container running a log processor watch an application container's
log directory, or have a container prepare a file or configuration before the primary application container starts.

The deployment defined the volume as an `emptyDir`. This type of volume has no persistence outside of the pod's
lifecycle. If the pod is removed for any reason, the `emptyDir` and its contents are removed as well. A `hostPath`
directory is available to achieve data persistence by ensuring the files are written to the host's disks, but this is
not always an option for some users. Applications that need persistent storage must be configured to use persistent
volumes.


### 2. Persistent Volumes and Persistent Volume Claims

The PersistentVolume subsystem in Kubernetes provides an API for users and administrators that abstracts details of how
storage is provided and how it is consumed. The system uses the `persistentVolume` and `persistentVolumeClaim`
resources. Persistent Volumes (PVs) provide storage with lifecycles independent of pods, capturing the details of a
given storage solution and enabling resources within the cluster to use them free.

A Persistent Volume Claim (PVC) is a request for storage. Kubernetes binds a Persistent Volume to a Persistent Volume
Claim based on the access mode and storage capacity; the Persistent Volume Claim can also state volume name, selectors,
and volume class to match with Persistent Volume(s).

In this step, we will statically provision a persistent volume by creating a Persistent Volume followed by a matching
Persistent Volume Claim. Then we will create a Pod that uses the Persistent Volume Claim as a volume mounted to any path
inside a container.

First create the Persistent Volume (PV) with the manifest below, declaring the `/tmp` path in _.spec.hostPath.path_:

```
~/vol$ nano myvol.yaml && cat $_
```
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: myvol
  labels:
    zone: "us-west"
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp
```
```
~/vol$ kubectl apply -f myvol.yaml

persistentvolume/myvol created

~/vol$
```

Now create a Persistent Volume Claim (PVC) that will match the _myvol_ PV by matching with the label `zone: us-west`:

```yaml
~/vol$ nano mypvc.yaml && cat $_
```
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
  selector:
    matchLabels:
      zone: "us-west"
```
```
~/vol$ kubectl apply -f mypvc.yaml

persistentvolumeclaim/mypvc created

~/vol$
```

Note that the _.spec.selector.matchLabel_ selector will be used to match the PVC to the PV but the _.spec.accessModes_
has to match and the PVC's _.spec.resources.requests.storage_ can not exceed the PV's _.spec.capacity.storage_.

Create a Pod that uses the Persistent Volume Claim as a volume mounted to a path inside a container:

```
~/vol$ kubectl run persistentpod --image=nginx --dry-run=client -o yaml > mypod.yaml

~/vol$
```

Edit the pod manifest with the following changes:
- Declare the `mypvc` PVC as a volume called `mypvc` using `volumes.persistentVolumeclaim`
- Mount the mypvc volume to the container at `/hostmount`
- Remove the `dnsPolicy`, `restartPolicy`, and `status: {}` keys
- Remove any null `resource: {}` and `creationTimestamp: null` keys that exist

```
~/vol$ nano mypod.yaml && cat $_
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: persistentpod
  name: persistentpod
spec:
  containers:
  - image: nginx
    name: persistentpod
    volumeMounts:
    - mountPath: "/hostmount"
      name: persistence
  volumes:
  - name: persistence
    persistentVolumeClaim:
      claimName: mypvc
```
```
~/vol$ kubectl apply -f mypod.yaml

pod/persistentpod created

~/vol$
```

Now let's check the pod. Since we are using a hostPath volume, we need to identify which node our pod is running on in
our cluster so we know which node to look under to confirm the volume mount. Use kubectl get pods with wide output to
view that:

```
~/vol$ kubectl get pods persistentpod -o wide

NAME            READY   STATUS    RESTARTS   AGE   IP          NODE              NOMINATED NODE   READINESS GATES
persistentpod   1/1     Running   0          5s    10.32.0.5   ip-172-31-2-209   <none>           <none>

~/vol$
```

In our test cluster, the pod landed on one of our worker nodes. In single node clusters, your pod will be scheduled on
your master node. In either case, when using persistent volumes backed by a hostPath, make sure to always check the
filesystem of the node in the cluster the pod is running on.

Let's check the files and see what is inside this node's `/tmp` directory.

Exec into the pod and list the contents of `/hostmount`:

```
~/vol$ kubectl exec persistentpod -it -- ls -l /hostmount

total 24
drwx------ 2 root root 4096 Feb  4 23:53 pty977671570
-rw------- 1 root root 1517 Feb  4 23:53 runc-process804971977
drwx------ 3 root root 4096 Feb  4 21:08 snap.lxd
drwx------ 3 root root 4096 Feb  4 21:08 systemd-private-caca90a5a46e4705a1aef65454185944-systemd-logind.service-ZM3lNh
drwx------ 3 root root 4096 Feb  4 21:08 systemd-private-caca90a5a46e4705a1aef65454185944-systemd-resolved.service-CfpP6e
drwx------ 3 root root 4096 Feb  4 21:08 systemd-private-caca90a5a46e4705a1aef65454185944-systemd-timesyncd.service-sRiXSh

~/vol$
```

The Pod successfully used the Persistent Volume via the Persistent Volume Claim. The PV mounted to the /tmp directory of
the node that the scheduler placed it on. The Pod spec used the `mypvc` PVC which matched to the `myvol` PV, which
mounts `/tmp`.

This example used a host path, which is an easy example but not necessarily what you would want to use. Ideally, you
would have the persistent volume be backed by some kind of network storage, like NFS. There are ways you can do this
with host paths by declaring a host path that is backed by an NFS filesystem, though there is another way.

This example represents a static provisioning workflow, where an administrator manually defines PVs to use and binds
them to pods using PVCs. In cloud implementations, persistentVolumeClaims can use `storageClasses`, which provides a way
for administrators to describe the “classes” of storage they offer. These are useful for cloud services that provide
their own storage subsystems, and enables Kubernetes to dynamically provision persistent volumes based on user needs.

Delete all the resources created from files inside the `~/vol/` directory:

```
~/vol$ kubectl delete -f ~/vol/

pod "persistentpod" deleted
persistentvolumeclaim "mypvc" deleted
persistentvolume "myvol" deleted
deployment.apps "sharing-redis" deleted

~/vol$
```


## StatefulSets and Headless Services

StatefulSets are controllers that manage the deployment and scaling of a set of Pods (like deployments) that guarantees
the ordering and uniqueness of these Pods. Each pod in a StatefulSet is started in order and has the pod named followed
by an ordinal number (pod-1, pod-2, etc.).

Applications that need one of the following are better suited to use StatefulSets:

- stable & unique network identifiers
- stable & persistent storage
- ordered & graceful deployment and scaling
- ordered & automated rolling updates

StatefulSets need the following to function correctly within a cluster:

- A Headless Service to be responsible for its Pods' network identity
- Some form of persistent storage for Pods, typically provided by persistent volumes and persistent volume claims


### 3. Working with Headless Services

Kubernetes configures appropriate DNS records for Headless Services to either return _Endpoints_ or _CNAME records_ with
the later having no selectors in the Headless Service's manifest. A Headless Service is configured by setting
_.spec.clusterIP_ to `None`. This type of service does not provide load-balancing with a single service IP, but still
allows the service to work with other service discovery mechanisms.

We will create a Headless Service along with the StatefulSet in the same manifest. The StatefulSet has to have
_.spec.selector.matchLabels_ that matches with the it's own _.spec.template.metadata.labels_.

Create the following manifest in a new "stateful" working directory.

This manifest will consist of two yaml objects: a service and a statefulSet.

```
~/vol$ mkdir ~/state && cd $_

~/state$
```

You can create a statefulSet spec using an imperative command for both a headless service and deployment. We make the
service headless by using the `--clusterip None` flag:

```
~/state$ kubectl create svc clusterip --clusterip None --tcp 80 nginx -o yaml --dry-run=client > webstate.yaml

~/state$ echo "---" >> webstate.yaml

~/state$ kubectl create deploy nginx --image nginx --port 80 -o yaml --dry-run=client >> webstate.yaml

~/state$
```

Make the following changes to the deployment spec:
- Rename the Service _and_ the StatefulSet `myweb`
- Change the deployment `kind` to `StatefulSet`
- Verify that the labels and selectors in the deployment, pod, and container template are `app: nginx`
- Add a `serviceName` key to the StatefulSet spec with the value `myweb` (matching the headless service)
- Remove the `strategy: {}` key
- Ensure the nginx container exposes `containerPort: 80`
- Add a `volumeClaimTemplate` named data that matches `ReadWriteOnce` access mode, requests `100Mi` of storage, and
  the `local` storageClassName
- Mount the data `volumeClaimTemplate` to `/data` in the container spec
- Remove the `status: {}` keys which was generated by the `create` commands we started with
- Clean up any `null` or empty `{}` keys

```
~/state$ nano webstate.yaml && cat $_
```
```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: nginx                           # Set label
  name: myweb                            # Renamed myweb
spec:
  clusterIP: None
  ports:
  - name: "80"
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx                           # Set label selector
  type: ClusterIP
---
apiVersion: apps/v1
kind: StatefulSet                        # Change from Deployment to StatefulSet
metadata:
  creationTimestamp: null
  labels:
    app: nginx                           # Set label
  name: myweb                            # Renamed myweb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx                         # Set label selector
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx                       # Set label
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:                           # Add this key
        - containerPort: 80              # Add this key-value
          name: myweb                    # Add this key-value
        resources: {}
        volumeMounts:
        - mountPath: "/data"
          name: data
  serviceName: myweb                     # Add and match the name of the service
  volumeClaimTemplates:                  # Add this 
  - metadata:                            # Add this
      name: data                         # Add this
    spec:                                # Add this
      accessModes: [ "ReadWriteOnce" ]   # Add this
      storageClassName: local            # Add this
      resources:                         # Add this
        requests:                        # Add this
          storage: 100Mi                 # Add this
```
```
~/state$
```

This manifest has:

- A service named `nginx` that uses a selector of `app: nginx` to configure the DNS to return _Endpoint_ records that
 point to the Pods of the Service. The service is considered "headless" due to the `clusterIP: None`.

- A volumeClaimTemplate in the statefulSet spec. This tells Kubernetes to create a Persistent Volume Claim for each
 Volume Claim Template. In this spec, Each Pod receives a single Persistent Volume with a StorageClass of `local` with
 100 Mi. When a Pod is scheduled, the Volume Claim Template creates a Persistent Volume Claim will bind to any matching
 Persistent Volume or storageClass. The match must include the access mode and at least a storageClassName or a resource
 request.

Launch the Service and StatefulSet in the current configuration and explore the StatefulSet, Pod, PV, and PVC.

```
~/state$ kubectl apply -f webstate.yaml

service/myweb created
statefulset.apps/myweb created

~/state$ kubectl describe sts myweb | grep -A5 Events

Events:
  Type    Reason            Age   From                    Message
  ----    ------            ----  ----                    -------
  Normal  SuccessfulCreate  6s    statefulset-controller  create Claim data-myweb-0 Pod myweb-0 in StatefulSet myweb success
  Normal  SuccessfulCreate  6s    statefulset-controller  create Pod myweb-0 in StatefulSet myweb successful

~/state$ kubectl get pv

No resources found in default namespace.

~/state$ kubectl get pvc

NAME           STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-myweb-0   Pending                                      local          26s

~/state$
```

Looks like the PVC cannot bind to anything, let's check the reason using `kubectl describe`:

```
~/state$ kubectl describe pvc data-myweb-0 | grep -A5 Events

Events:
  Type     Reason              Age               From                         Message
  ----     ------              ----              ----                         -------
  Warning  ProvisioningFailed  4s (x4 over 39s)  persistentvolume-controller  storageclass.storage.k8s.io "local" not found

~/state$
```

We have not created any Persistent Volumes so the newly created PersistentVolumeClaim and the StatefulSet's pods will
stay in a Pending state. The persistentVolumeClaim cannot find any storage classes or persistent volumes that match, so
the pods will stay pending the persistentVolumeClaim can bind to a PV.

Create two Persistent Volumes mounted to a host path, one for the first pod in the StatefulSet and an additional
Persistent Volume in case we decide to scale up the StatefulSet later:

```
~/state$ nano local-pv.yaml && cat $_
```
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv1
  labels:
    type: local
spec:
  storageClassName: local
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/tmp/pv1"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv2
  labels:
    type: local
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  storageClassName: local
  hostPath:
    path: "/tmp/pv2"
```

These persistentVolumes will both bind to folders on the host that a pod that uses them is assigned on, one at
`/tmp/pv1` and the other at `/tmp/pv2`. Both volumes will hold up to 100mb of data and grant one consumer (the
container) read/write capabilities. They are assigned the `local` storageClassName for easier identification.

Create the PVs:

```
~/state$ kubectl apply -f local-pv.yaml

persistentvolume/local-pv1 created
persistentvolume/local-pv2 created

~/state$ kubectl get pv

NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                  STORAGECLASS   REASON   AGE
local-pv1   100Mi      RWO            Retain           Bound       default/data-myweb-0   local                   17s
local-pv2   100Mi      RWO            Retain           Available                          local                   17s

~/state$
```

Despite their `Pending` state, the PVCs continue to watch for new PVs that match their specifications to claim. Since
there are now PVs in our cluster, check if a Persistent Volume Claim bound to one of the PVs:

```
~/state$ kubectl get pvc

NAME           STATUS   VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-myweb-0   Bound    local-pv1   100Mi      RWO            local          2m9s

~/state$
```

Notice that our PVC now has a status of `Bound`. Verify the Pod is running with `kubectl get` and `kubectl describe`:

```
~/state$ kubectl get pods myweb-0

NAME      READY   STATUS    RESTARTS   AGE
myweb-0   1/1     Running   0          2m19s

~/state$ kubectl describe pod myweb-0 | grep -A10 Events

Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  13s (x4 over 78s)  default-scheduler  0/1 nodes are available: 1 pod has unbound immediate PersistentVolumeClaims.
  Normal   Scheduled         4s                 default-scheduler  Successfully assigned default/myweb-0 to ip-172-31-2-209
  Normal   Pulling           3s                 kubelet            Pulling image "nginx"
  Normal   Pulled            3s                 kubelet            Successfully pulled image "nginx" in 107.621867ms
  Normal   Created           3s                 kubelet            Created container nginx
  Normal   Started           3s                 kubelet            Started container nginx

~/state$
```

Once the PVC found a PV to bind to, the scheduler assigned the pod to a node in the cluster and the pod is now running.


### 4. Scaling StatefulSets

StatefulSets can be scaled similarly to Deployments, but in order for them to successfully scale they must have a
persistent volume to bind to. We created two persistent volumes, so we have enough for another replica.

Scale the StatefulSet to 2 replicas with `kubectl scale`:

```
~/state$ kubectl scale sts myweb --replicas=2

statefulset.apps/myweb scaled

~/state$
```

See if the second Persistent Volume is now `Bound`:

```
~/state$ kubectl get pv

NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   REASON   AGE
local-pv1   100Mi      RWO            Retain           Bound    default/data-myweb-0   local                   46s
local-pv2   100Mi      RWO            Retain           Bound    default/data-myweb-1   local                   46s

~/state$ kubectl get pvc

NAME           STATUS   VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-myweb-0   Bound    local-pv1   100Mi      RWO            local          116s
data-myweb-1   Bound    local-pv2   100Mi      RWO            local          20s

~/state$ kubectl describe pvc data-myweb-1

Name:          data-myweb-1
Namespace:     default
StorageClass:  local
Status:        Bound
Volume:        local-pv2
Labels:        app=nginx
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      100Mi
Access Modes:  RWO
VolumeMode:    Filesystem
Used By:       myweb-1
Events:        <none>

~/state$
```

The StatefulSet created a second Persistent Volume Claim and bound to the second Persistent Volume. In the details of
the second PVC from `kubectl describe pvc data-myweb-1` we see that the PVC is mounted by the second Pod, `myweb-1`.

Scale down the StatefulSet and check the Persistent Volumes and Persistent Volume Claims.

```
~/state$ kubectl scale statefulset myweb --replicas=1

statefulset.apps/myweb scaled

~/state$ kubectl get pv

NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   REASON   AGE
local-pv1   100Mi      RWO            Retain           Bound    default/data-myweb-0   local                   114s
local-pv2   100Mi      RWO            Retain           Bound    default/data-myweb-1   local                   114s

~/state$
```

Notice that the second Persistent Volume is still `Bound` with a `RECLAIM POLICY` of `Retain` which is default.
Kubernetes can `Retain` or `Delete` the Persistent Volume after it has been released from a claim.

```
~/state$ kubectl get pvc

NAME           STATUS   VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-myweb-0   Bound    local-pv1   100Mi      RWO            local          3m11s
data-myweb-1   Bound    local-pv2   100Mi      RWO            local          95s

~/state$
```

The Persistent Volumes are only released after deleting the Persistent Volume Claim. This ensures that if the loss of
the pod was temporary, the new StatefulSet replica will retain the data from its previous incarnation. Additional steps
may need to occur in order for the application to recover from such a loss depending on how the application keeps its
state.

Release the persistent Volume by deleting the pvc data-myweb-1:

```
~/state$ kubectl delete pvc data-myweb-1

persistentvolumeclaim "data-myweb-1" deleted

~/state$ kubectl get pv

NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                  STORAGECLASS   REASON   AGE
local-pv1   100Mi      RWO            Retain           Bound      default/data-myweb-0   local                   2m38s
local-pv2   100Mi      RWO            Retain           Released   default/data-myweb-1   local                   2m38s

~/state$
```

The PV that was previously bound to the deleted PVC is placed into the released state. The reclaim policy for a
PersistentVolume tells the cluster what to do with the volume after it has been released of its claim. Currently,
volumes can either be:

- Retained - the PV is not deleted and the data can be manually recovered. The PV must be manually deleted and
  recreated before it can be used
- Deleted - removes both the PersistentVolume object from Kubernetes and the associated storage asset in the
  external infrastructure

As mentioned before, users must consider how an application reacts to finding existing data. Stateful applications like
distributed databases may require additional clean up or rejoining processes to ensure that a returning replica can
operate correctly with its peers. In workflows where persistent volumes are created manually by an administrator,
`Retain` may be the preferred option. In dynamically provisioned workflows, `Delete` may be the preferred (and in some
cases, the default) option to ensure unbound PVs do not drive up unnecessary storage costs.


### 5. CHALLENGE

Now that you've learned how to work with Persistent Volumes, try the following to test your knowledge:

- Create two persistent volumes, one with storage class name "gold" and the other has storage class name "silver".
- Create a persistent volume claim that uses the gold storage class. Verify that the correct persistent volume is used.


### 6. Clean up

Clean up and delete the Headless Service, StatefulSet, Persistent Volume Claims. Check if the Persistent Volumes were
released then delete the Persistent Volumes:

```
~/state$ kubectl delete sts/myweb svc/myweb

statefulset.apps "myweb" deleted
service "myweb" deleted

~/state$ kubectl delete pvc data-myweb-0

persistentvolumeclaim "data-myweb-0" deleted

~/state$ kubectl get pv

NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                  STORAGECLASS   REASON   AGE
local-pv1   100Mi      RWO            Retain           Released   default/data-myweb-0   local                   9m56s
local-pv2   100Mi      RWO            Retain           Released   default/data-myweb-1   local                   9m56s

~/state$ kubectl delete pv local-pv1 local-pv2

persistentvolume "local-pv1" deleted
persistentvolume "local-pv2" deleted

~/state$
```

<br>

Congratulations you have completed the lab!

<br>

_Copyright (c) 2013-2023 RX-M LLC, Cloud Native Consulting, all rights reserved_
