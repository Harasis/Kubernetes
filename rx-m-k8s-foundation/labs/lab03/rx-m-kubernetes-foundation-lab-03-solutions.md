![RX-M, llc.](http://rx-m.io/rxm-cnc.svg)


# Kubernetes


## Pod Architecture, Resource Requirements, Init Containers, and Health Checks Challenge Solutions


### 6. CHALLENGE: pods and Linux namespaces

Using the pod spec reference as a guide, create a podspec using `nginx:1.21` so that it runs all
of the containers in the pod in the _host network namespace_ (hostNetwork).

```
~/pods$ kubectl run nginx-ns --image nginx:1.21 --port 80 -o yaml --dry-run=client > nginx-ns.yaml

~/pods$
```

Next modify it to use the `hostNetwork` (use the spec reference if you need help thinking through the edits you
will need to make):

Pod nginx-ns.yaml:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-ns
spec:
  hostNetwork: true    # add this
  containers:
  - name: nginx-ns
    image: nginx:1.21
    ports:
    - containerPort: 80
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

- Run the new pod: `kubectl apply -f nginx-ns.yaml`

```
~/pods$ kubectl apply -f nginx-ns.yaml 

pod/nginx-ns created

~/pods$ 
```

- Confirm your changes by displaying your pod status in yaml output and use `grep` to view the hostIP and podIP fields:
`kubectl get pod nginx-ns -o yaml | egrep ' (host|pod)IP:'`

```
~/pods$ kubectl get pod nginx-ns -o yaml | egrep ' (host|pod)IP:'

  hostIP: 172.31.6.204
  podIP: 172.31.6.204

~/pods$ 
```

If your pod is running in the host network namespace you should see that the pod (_podIP_) and host IP (_hostIP_)
address are identical. Namespace select-ability allows you have the benefits of container deployment while still
empowering infrastructure tools to see and manipulate host based networking features.

- Try curling your pod using the host IP address: `curl -I <Either the hostIP or podIP values>`

```
~/pods$ curl -I $(hostname -i)

HTTP/1.1 200 OK
Server: nginx/1.21.6
Date: Mon, 10 Jul 2023 01:51:24 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 25 Jan 2022 15:03:52 GMT
Connection: keep-alive
ETag: "61f01158-267"
Accept-Ranges: bytes

~/pods$ 
```

- Delete the `nginx-ns` pod: `kubectl delete -f nginx-ns.yaml`

```
~/pods$ kubectl delete -f nginx-ns.yaml

pod "nginx-ns" deleted

~/pods$ 
```

### 7. CHALLENGE: multi-container pod

Create a new Pod config which:

- Has the pod name `hello`
- runs two `ubuntu:18.04` containers, named hello1 and hello2
- both executing the command line `tail -f /dev/null`

```
~/pods$ kubectl run hello \
--dry-run=client -o yaml \
--image ubuntu:18.04 \
--command -- /usr/bin/tail -f /dev/null > hello.yaml
```

Edit `hello.yaml`, copying the first container to a second container (don't forget to rename the second container!):

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: hello
  name: hello
spec:
  containers:
  - command:
    - /usr/bin/tail
    - -f
    - /dev/null
    image: ubuntu:18.04
    name: hello1            ### Rename this container
    resources: {}
  - command:                ### Add this 2nd container to the spec
    - /usr/bin/tail
    - -f
    - /dev/null
    image: ubuntu:18.04
    name: hello2
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

- Create the pod: `kubectl apply -f <filename>.yaml`

```
~/pods$ kubectl apply -f hello.yaml

pod/nginx-ns created

~/pods$
```

- Issue the `kubectl get pods` command.

```
~/pods$ kubectl get pods

NAME    READY   STATUS    RESTARTS   AGE
hello   2/2     Running   0          19s

~/pods$ 
```

  -	How many containers are running in your new pod? `2`
  -	How can you tell? `Ready 2/2`

- Use the `kubectl describe pod` subcommand on your new pod.

```
~/pods$ kubectl describe pod hello

Name:             hello
Namespace:        default
Priority:         0
Service Account:  default
Node:             ip-172-31-6-204/172.31.6.204
Start Time:       Mon, 10 Jul 2023 01:53:03 +0000
Labels:           run=hello
Annotations:      <none>
Status:           Running
IP:               10.32.0.4

...
```

-	Is there any information about the IP addresses of the containers themselves in this output? `No, only the pod.`

- Shell into the first container to explore its context using the `-c` option with the name of the first container, in
  the example `hello1`: `kubectl exec -it hello -c hello1 -- /bin/bash`

  ```
  ~/pods$ kubectl exec -it hello -c hello1 -- /bin/bash
  
  root@hello:/#
  ```

  - Run the `ps -ef` command inside the container.

    ```
    root@hello:/# ps -ef
    UID          PID    PPID  C STIME TTY          TIME CMD

    root           1       0  0 18:00 ?        00:00:00 /usr/bin/tail -f /dev/null
    root           7       0  0 18:01 pts/0    00:00:00 /bin/bash
    root          17       7  0 18:01 pts/0    00:00:00 ps -ef

    root@hello:/# 
    ```

    -	What processes are running in the container? The tail, our new shell, and `ps`
    -	What is the container’s host name? `hello`
  - Run the `ip a` command inside the container.
    - Run `apt update`
    - Run `apt install iproute2 -y`

    ```
    root@hello:/# apt update

    ...

    root@hello:/# apt install iproute2 -y

    ...

    root@hello:/# ip a

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
          valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
          valid_lft forever preferred_lft forever
    37: eth0@if38: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1376 qdisc noqueue state UP group default
        link/ether 6e:ed:a1:cb:a4:5e brd ff:ff:ff:ff:ff:ff link-netnsid 0
        inet 10.32.0.4/12 brd 10.47.255.255 scope global eth0
          valid_lft forever preferred_lft forever
        inet6 fe80::6ced:a1ff:fecb:a45e/64 scope link
          valid_lft forever preferred_lft forever

    root@hello:/#
    ```

    -	What is the MAC address of eth0? In the example, `6e:ed:a1:cb:a4:5e`
    -	What is the IP address? In the example, `10.32.0.4`, the pod IP
  - Create a file in the root directory and exit the container:

```
root@hello:/# echo "Hello" > TEST

root@hello:/# exit
```

Kubernetes executed our last command in the first container in the pod. We now want to open a shell into the second
container. To do this we can use the _-c_ switch. 

- Exec a shell into the second container using the _-c_ switch and the name of the second container:
  `kubectl exec -it hello -c hello2 -- /bin/bash`
  - Is the TEST file you created previously there? No

  ```
  root@hello:/# find TEST

  find: 'TEST': No such file or directory

  root@hello:/# 
  ```

  - What is the host name in this container? `hello`
  - What is the MAC address of eth0?

  ```
  root@hello:/# apt update && apt install iproute2 -y && ip a

  ...

  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
      inet6 ::1/128 scope host
        valid_lft forever preferred_lft forever
  37: eth0@if38: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1376 qdisc noqueue state UP group default
      link/ether 6e:ed:a1:cb:a4:5e brd ff:ff:ff:ff:ff:ff link-netnsid 0
      inet 10.32.0.4/12 brd 10.47.255.255 scope global eth0
        valid_lft forever preferred_lft forever
      inet6 fe80::6ced:a1ff:fecb:a45e/64 scope li

  root@hello:/# 
  ```

  The mac address is same as the other container

  - What is the IP address? Same as the other container
  - Which of the following namespaces are shared across the containers in the pod?
    - UTS (hostname & resolver config)
    - Network
    - IPC

Delete the pod: `kubectl delete -f <filename>.yaml`

```
~/pods$ kubectl delete -f hello.yaml 

pod "hello" deleted

~/pods$
```


### 8. CHALLENGE: troubleshooting

- Recreate the limit pod: `kubectl apply -f limit.yaml`

```
~/pods$ kubectl apply -f limit.yaml

pod/limit created

~/pods$ 
```

Try to diagnose any issues using `kubectl describe pod limit` to find out which container is crashing and `kubectl logs
limit` to see what error it is reporting. (Hint: some containerized applications need more configuration than others).

Use `kubectl describe pod limit` and look at the events to see which container is crashing:

```
...

Events:
  Type     Reason     Age              From               Message
  ----     ------     ----             ----               -------
  Normal   Scheduled  6s               default-scheduler  Successfully assigned default/limit to ip-172-31-6-204
  Normal   Pulled     6s               kubelet            Container image "wordpress:5.8" already present on machine
  Normal   Created    5s               kubelet            Created container wp
  Normal   Started    5s               kubelet            Started container wp
  Normal   Pulled     4s (x2 over 6s)  kubelet            Container image "mysql:8" already present on machine
  Normal   Created    4s (x2 over 6s)  kubelet            Created container limit
  Normal   Started    4s (x2 over 6s)  kubelet            Started container limit
  Warning  BackOff    2s               kubelet            Back-off restarting failed container limit in pod limit_default(255fea2b-3ddd-4cc8-b4cb-eb805ac536e2)
```

The `limit` container (mysql) is being pulled, created, and started over and over `(x3 over 61s)`.

Use `kubectl logs` on the pod and Kubernetes will tell you the container names!

```
~/pods$ kubectl logs limit

Defaulted container "limit" out of: limit, wp
2023-07-10 01:56:11+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 8.0.33-1.el8 started.
2023-07-10 01:56:12+00:00 [Note] [Entrypoint]: Switching to dedicated user 'mysql'
2023-07-10 01:56:12+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 8.0.33-1.el8 started.
2023-07-10 01:56:13+00:00 [ERROR] [Entrypoint]: Database is uninitialized and password option is not specified
    You need to specify one of the following as an environment variable:
    - MYSQL_ROOT_PASSWORD
    - MYSQL_ALLOW_EMPTY_PASSWORD
    - MYSQL_RANDOM_ROOT_PASSWORD

~/pods$
```

Kubernetes 1.24 and above will default to showing logs from the container that is "first" (lower line number) in the
`containers:` list. In previous versions, you had to specify which container to retrieve logs from.

Use `kubectl logs` on the frontend pod's `limit` and `wp` containers to see what's wrong:

```
~/pods$ kubectl logs limit -c limit

2023-07-10 01:56:27+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 8.0.33-1.el8 started.
2023-07-10 01:56:28+00:00 [Note] [Entrypoint]: Switching to dedicated user 'mysql'
2023-07-10 01:56:28+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 8.0.33-1.el8 started.
2023-07-10 01:56:29+00:00 [ERROR] [Entrypoint]: Database is uninitialized and password option is not specified
    You need to specify one of the following as an environment variable:
    - MYSQL_ROOT_PASSWORD
    - MYSQL_ALLOW_EMPTY_PASSWORD
    - MYSQL_RANDOM_ROOT_PASSWORD

~/pods$ kubectl logs limit -c wp

WordPress not found in /var/www/html - copying now...
Complete! WordPress has been successfully copied to /var/www/html
AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 10.32.0.4. Set the 'ServerName' directive globally to suppress this message
AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 10.32.0.4. Set the 'ServerName' directive globally to suppress this message
[Mon Jul 10 01:56:11.025133 2023] [mpm_prefork:notice] [pid 1] AH00163: Apache/2.4.51 (Debian) PHP/7.4.27 configured -- resuming normal operations
[Mon Jul 10 01:56:11.025404 2023] [core:notice] [pid 1] AH00094: Command line: 'apache2 -D FOREGROUND'

~/pods$
```

Wordpress is okay, but the limit container requires additional environment variables.

Add one of the suggested environment variables to the spec, like `MYSQL_ALLOW_EMPTY_PASSWORD`:

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
    env:                                    ### Add this line
    - name: MYSQL_ALLOW_EMPTY_PASSWORD      ### Add this line
      value: "true"                         ### Add this line
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

Not all fields in pods can be updated, so in order to make this change you need to delete the pod and recreate it from
the yaml file:

```
~/pods$ kubectl delete -f limit.yaml && kubectl apply -f $_

pod "limit" deleted
pod/limit created

~/pods$ kubectl get pods limit

NAME    READY   STATUS    RESTARTS   AGE
limit   2/2     Running   0          4s

~/pods$
```

Delete the limit pod after examining its resource usage:

```
~/pods$ kubectl describe $(kubectl get node -o name) | grep -A 8 Allocated

Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests     Limits
  --------           --------     ------
  cpu                1450m (72%)  1 (50%)
  memory             560Mi (14%)  980Mi (26%)
  ephemeral-storage  0 (0%)       0 (0%)
  hugepages-1Gi      0 (0%)       0 (0%)
  hugepages-2Mi      0 (0%)       0 (0%)

~/pods$ kubectl delete pod limit

pod "limit" deleted

~/pods$
```

_Copyright (c) 2013-2023 RX-M LLC, Cloud Native Consulting, all rights reserved_
