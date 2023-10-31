![RX-M, llc.](http://rx-m.io/rxm-cnc.svg)


# Kubernetes


## Kubernetes Local Setup Lab Solutions

### 10. CHALLENGE: kubectl run

- Run a pod named "`challenge`" based on the image `rxmllc/hostinfo`
  - The hostinfo image reports its hostname and IP address when queried

```
~$ kubectl run challenge --image=docker.io/rxmllc/hostinfo:latest

pod/challenge created

~$
```

Get the IP of the challenge pod:

```
~$ kubectl get pod -o wide

NAME        READY   STATUS    RESTARTS      AGE     IP          NODE               NOMINATED NODE   READINESS GATES
challenge   1/1     Running   0             3s      10.32.0.6   ip-172-31-32-172   <none>           <none>
client      1/1     Running   1 (37s ago)   43s     10.32.0.5   ip-172-31-32-172   <none>           <none>
web         1/1     Running   0             2m32s   10.32.0.4   ip-172-31-32-172   <none>           <none>

~$
```

- Resume your interactive session with the "`client`" pod

```
~$ kubectl attach client -c client -i -t

If you don't see a command prompt, try pressing enter.

/ #
```

- Query the "`challenge`" pod on port `9898` with wget:

```
/ # wget -qO - 10.32.0.6:9898

challenge 10.32.0.6

/ #
```

  - What does the response tell you about pod hostnames?

The pod's name is used as its hostname.

- Delete the `challenge` pod after completing the above steps

```
/ # exit

Session ended, resume using 'kubectl attach client -c client -i -t' command when the pod is running

~$ kubectl delete pod challenge

pod "challenge" deleted

~$
```

_Copyright (c) 2013-2023 RX-M LLC, Cloud Native Consulting, all rights reserved_
