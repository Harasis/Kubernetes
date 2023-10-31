![RX-M, llc.](http://rx-m.io/rxm-cnc.svg)


# Kubernetes


## Network Policy Challenge Solutions


### 4. CHALLENGE: apply the policy to an existing client

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

Describe the network policies to determine the appropriate label to apply to the second client pod.
Then apply the correct label to the second client pod so that it can communicate with the hostinfo pod.

Describe the `default-deny` network policy:

```
~/np$ kubectl describe networkpolicy default-deny

Name:         default-deny
Namespace:    netpollab
Created on:   2022-09-18 17:55:42 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     <none> (Allowing the specific traffic to all pods in this namespace)
  Allowing ingress traffic:
    <none> (Selected pods are isolated for ingress connectivity)
  Not affecting egress traffic
  Policy Types: Ingress

~/np$
```

The `default-deny` network policy does not allow ingress traffic to all Pods.

Describe the `access-hostinfo` network policy:

```
~/np$ kubectl describe networkpolicy access-hostinfo

Name:         access-hostinfo
Namespace:    netpollab
Created on:   2022-09-18 17:56:03 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     app=hi
  Allowing ingress traffic:
    To Port: <any> (traffic allowed to all ports)
    From:
      PodSelector: app=client
  Not affecting egress traffic
  Policy Types: Ingress

~/np$
```

The `access-hostinfo` allows ingress traffic to Pods with the `app=hi` label from Pods with the `app=client` label.

Add the `app=client` to the running client2 pod and test if the client2 pod can reach the hostinfo pod.

```
~/np$ kubectl label pod client2 app=client

pod/client2 labeled

~/np$ kubectl exec client2 -it -- wget -T5 -qO - $HI:9898

hi 10.32.0.6

~/np$
```

Applying the label worked! The client2 pod can now access the hostinfo pod.

_Copyright (c) 2013-2023 RX-M LLC, Cloud Native Consulting, all rights reserved_
