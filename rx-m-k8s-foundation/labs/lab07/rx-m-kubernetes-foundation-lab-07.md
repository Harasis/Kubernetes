![RX-M, llc.](http://rx-m.io/rxm-cnc.svg)


# Kubernetes


## Network Policy

A network policy is a specification of how groups of pods are allowed to communicate with each other and other network
endpoints. NetworkPolicy resources use labels to select pods and define rules which specify what traffic is allowed to
and from the selected pods. Network policies are defined by Kubernetes but implemented by CNI network plugins and/or
other networking agents, so you must use a CNI networking solution or agent which supports NetworkPolicy for policies
to work.

By default, pods are non-isolated; they accept traffic from any source. Pods become isolated by having a NetworkPolicy
that selects them. Adding a NetworkPolicy to a namespace selecting a particular pod, causes that pod to become
"isolated", rejecting any connections that are not explicitly allowed by a NetworkPolicy. Other pods in the namespace
that are not selected by any NetworkPolicy will continue to accept all traffic.

In this lab you will try controlling pod connectivity with Kubernetes Network Policy.


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


### 1. Create a namespace for lab work

Namespaces enable multiple teams to work in relative isolation in a multi-tenant environment, but also provide scope for
Kubernetes objects. We will create a namespace for our lab work so that we can remove objects that would potentially
impact subsequent labs (like NetworkPolicy objects!).

Create a new namespace called netpollab:

```
~$ cd ~

~$ kubectl create namespace netpollab

namespace/netpollab created

~$
```

Now set your kubectl context to use our new namespace by default:

```
~$ kubectl config set-context --current --namespace netpollab

Context "kubernetes-admin@kubernetes" modified.

~$
```

Until we change our config, all subsequent command will be run in the "netpollab" namespace.


### 1. Run a service and access it from a client

To begin let's run a pod and try connecting to it from another pod. The rxmllc/hostinfo image listens on port 9898 and
responds to requests with the pod hostname. Run a pod with a hostinfo pod having the label `app=hi`:

```
~$ kubectl run hi --image rxmllc/hostinfo --port 9898 -l app=hi

pod/hi created

~$
```

Extract the "hi" pod's IP and store it in an environment variable:

```
~$ HI=$(kubectl get po hi --template={{.status.podIP}}) && echo $HI

10.32.0.6

~$
```

Now run a client busybox pod with the label `app=client` and the `HI` environment variable:

```
~$ kubectl run client --image busybox -l app=client --env HI=$HI --command -- tail -f /dev/null

pod/client created

~$ kubectl get po

NAME     READY   STATUS    RESTARTS   AGE
client   1/1     Running   0          5s
hi       1/1     Running   0          2m9s

~$
```

Try accessing the hostinfo pod from within the busybox pod:

```
~$ kubectl exec -it client -- wget -T5 -qO - $HI:9898

hi 10.32.0.6

~$
```

So with no network policy we can access one pod from another freely.


### 2. Create a blocking network policy

For our first network policy we'll create a policy that denies all inbound connections to pods in the default namespace.
Create the following policy using an example from the Kubernetes documentation found here:

- https://kubernetes.io/docs/concepts/services-networking/network-policies/#default-deny-all-ingress-traffic

```
~$ mkdir ~/np && cd $_

~/np$ nano np.yaml && cat $_
```
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: netpollab
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```
```
~/np$ kubectl apply -f np.yaml

networkpolicy.networking.k8s.io/default-deny created

~/np$ kubectl get networkpolicy

NAME           POD-SELECTOR   AGE
default-deny   <none>         3s

~/np$
```

This policy selects all pods (`{}`) and has no ingress policies (`Ingress`) for them. By creating any network policy
however, we automatically isolate all pods.

Retry connecting to the hostinfo pod with a 5 second timeout (`-T 5`):

```
~/np$ kubectl exec -it client -- wget -T5 -qO - $HI:9898

wget: download timed out
command terminated with exit code 1

~/np$
```

The presence of a network policy shuts down our ability to reach the other pod!


### 3. Create a permissive network policy

To enable our busybox pod to access our hostinfo pod we will need to create a network policy that selects the hostinfo
pods and allows ingress from the busybox pod. The hostinfo pod has the label `app=hi` so we can use that to select the
target pod and our busybox pod has the label "app=client" so we'll use that to identify the ingress pods allowed.

Create the new policy, basing it off of the default allow example on the Kubernetes documentation found here:

- https://kubernetes.io/docs/concepts/services-networking/network-policies/#default-allow-all-ingress-traffic

```
~/np$ nano np-hi.yaml && cat $_
```
```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: access-hostinfo
  namespace: netpollab
spec:
  podSelector:
    matchLabels:
      app: hi
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: client
```
```
~/np$
```

This network policy does the following:

- `spec.podSelector.matchLabels.app=hi` selects all pods in the current namespace with the label `app=hi`
- `ingress.from.podSelector.matchLabels.app=client` allows ingress from pods in the same namespace labeled `app=client`

The `ingress.from` array enables you to define multiple rules. Any traffic sent into this namespace is evaluated against
these Ingress rules in the order specified, and if the traffic matches one of the Ingress rules it is let through.

Apply the Ingress rule to your namespace:

```
~/np$ kubectl apply -f np-hi.yaml

networkpolicy.networking.k8s.io/access-hostinfo created

~/np$ kubectl get networkpolicy

NAME              POD-SELECTOR   AGE
access-hostinfo   app=hi         3s
default-deny      <none>         24s

~/np$
```

Now retry access the hostinfo pod from the busybox pod:

```
~/np$ kubectl exec -it client -- wget -T5 -qO - $HI:9898

hi 10.32.0.6

~/np$
```

It works! With network policies in place, pods must have explicit permission to communicate with each other. The only
way to restore free access between pods is to create a permissive network policy like above, or remove network policies
from your cluster all together.

For more information on networkpolicy take a look at the resource reference:

- https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/#networkpolicy-v1-networking-k8s-io


### 4. CHALLENGE: apply the policy to an existing client

As we saw in the previous step, network policies use labels to select pods and apply traffic rules to the pods.

With the `default-deny` and `access-hostinfo` network policies in place, run the following imperative command to create
a second busybox pod and test if the pod can reach the hostinfo pod:

```
~/np$ kubectl run client2 --image busybox --env HI=$HI --command -- tail -f /dev/null

pod/client2 created

~/np$ kubectl exec client2 -it -- wget -T5 -qO - $HI:9898

wget: download timed out
command terminated with exit code 1

~/np$
```

Describe the network policies to determine the appropriate label to apply to the second busybox pod.
Then apply the correct label to the second busybox pod so that it can communicate with the hostinfo pod.

See the solution document for guidance.


### 5. Egress Traffic and ipBlock

As you have seen so far, network policy is only affecting labeled pods and namespaces within the cluster. In some cases,
you may want to affect traffic incoming from or outgoing to other places in the network (e.g. things that exist outside
of the cluster and thus have no labels). Your option for doing so is using the `ipBlock` exception under either the
`ingress` or `egress` spec.

In this step, you will see how to control traffic to entities outside of the cluster and take the opportunity to control
outgoing (egress) traffic from a pod using a network policy.

First, use `kubectl explain` to get an overview of the `ipBlock` spec:

```
~/np$ kubectl explain networkpolicy.spec.egress.to.ipBlock

KIND:     NetworkPolicy
VERSION:  networking.k8s.io/v1

RESOURCE: ipBlock <Object>

DESCRIPTION:
     IPBlock defines policy on a particular IPBlock. If this field is set then
     neither of the other fields can be.

     IPBlock describes a particular CIDR (Ex. "192.168.1.1/24","2001:db9::/64")
     that is allowed to the pods matched by a NetworkPolicySpec's podSelector.
     The except entry describes CIDRs that should not be included within this
     rule.

FIELDS:
   cidr	<string> -required-
     CIDR is a string representing the IP Block Valid examples are
     "192.168.1.1/24" or "2001:db9::/64"

   except	<[]string>
     Except is a slice of CIDRs that should not be included within an IP Block
     Valid examples are "192.168.1.1/24" or "2001:db9::/64" Except values will
     be rejected if they are outside the CIDR range

~/np$
```

When you set an `ipBlock` exception, all traffic from the given `cidr` block will be allowed. If you want to restrict
traffic within a given `cidr`, then the `except` field will allow you to list subsets within the given `cidr` block that
will continue to have traffic stopped.

Let's see what the `client` pod can do in terms of submitting traffic to outside sources. In this case, the outside entity will be the AWS metadata endpoint found at `http://169.254.169.254/latest/meta-data`

```
~/np$ kubectl exec -it client -- wget -T5 -qO - http://169.254.169.254/latest/meta-data

ami-id
ami-launch-index
ami-manifest-path

...

~/np$
```

Right now, traffic outgoing from the pod is not being blocked. You can confirm this by describing the `default-deny` and seeing if `egress` is mentioned at all:

```
~/np$ kubectl describe netpol default-deny | grep egress
```

Say there is a requirement to block this particular endpoint. That is something you can achieve using the following
network policy spec:

```
~/np$ nano restrict-node-metadata.yaml && cat $_
```
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-node-metadata
  namespace: netpollab
spec:
  podSelector:
    matchLabels:
      app: client               # Selects the client pod in the netpollab namespace
  policyTypes:
  - Egress                      # This netpol will block only outgoing traffic, incoming traffic not affected
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0         # Allows the pod to connect everywhere else
        except:                 # Any ip blocks listed under here will have traffic stopped
        - 169.254.169.254/32    # Specific IP addresses can be listed with /32
    - namespaceSelector: {}     # Allows the pod to continue communications with other pods
```

This network policy ensures that the `client` pod can connect to all other endpoints except `169.254.169.254`, including
other pods in the cluster.

Aside from being nested under a `to` key, the `egress` rule syntax is identical to the `ingress` syntax. You can use the
same `podSelector` or `namespaceSelector` style of rules to control traffic within the cluster.

Apply the network policy:

```
~/np$ kubectl apply -f restrict-node-metadata.yaml

networkpolicy.networking.k8s.io/restrict-node-metadata created

~/np$
```

And try connecting again:

```
~/np$ kubectl exec -it client -- wget -T5 -qO - http://169.254.169.254/latest/meta-data

wget: download timed out
command terminated with exit code 1

~/np$
```

Traffic to the metadata endpoint is now restricted!

Try accessing another website like `example.com`:

```
~$ kubectl exec -it client -- wget -T5 -S http://example.com

Connecting to example.com (93.184.216.34:80)
  HTTP/1.1 200 OK
  Age: 353957
  Cache-Control: max-age=604800
  Content-Type: text/html; charset=UTF-8
  Date: Sun, 18 Sep 2022 18:17:05 GMT
  Etag: "3147526947+ident"
  Expires: Sun, 25 Sep 2022 18:17:05 GMT
  Last-Modified: Thu, 17 Oct 2019 07:18:26 GMT
  Server: ECS (oxr/8319)
  Vary: Accept-Encoding
  X-Cache: HIT
  Content-Length: 1256
  Connection: close

saving to 'index.html'
index.html           100% |*******************************************************************************************************|  1256  0:00:00 ETA
'index.html' saved

~$
```

And ensure the pod-to-pod communications for the pod are still valid:

```
~$ kubectl exec client2 -it -- wget -T5 -qO - $HI:9898

hi 10.32.0.6

~$
```

You have successfully controlled traffic to an entity outside the Kubernetes cluster using a network policy!


### 6. Clean up

When you are finished exploring, tear down all of your components by deleting the netpollab namespace (`ns`):

```
~/np$ kubectl delete ns netpollab

namespace "netpollab" deleted

~/np$
```

Reconfigure kubectl to use the default namespace:

```
~/np$ kubectl config set-context --current --namespace default

Context "kubernetes-admin@kubernetes" modified.

~/np$ kubectl get all

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   8h

~/np$ cd ~

~$
```

<br>

Congratulations you have completed the lab!

<br>

_Copyright (c) 2013-2023 RX-M LLC, Cloud Native Consulting, all rights reserved_
