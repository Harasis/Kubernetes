![RX-M, llc.](http://rx-m.io/rxm-cnc.svg)


# Kubernetes


## Pods: Basics Challenge Solutions


### 2.1. CHALLENGE: pod exploration

- Run `kubectl run apache --image httpd` to create another `apache` pod

```
~$ kubectl run apache --image httpd

pod/apache created

~$
```

- Use `kubectl describe` to view the new `apache` pod
  - What image is this new apache pod running? How is it different from the initial `apache` pod?

```
~$ kubectl describe pod apache | grep Image

    Image:          httpd
    Image ID:       docker.io/library/httpd@sha256:8059bdd0058510c03ae4c808de8c4fd2c1f3c1b6d9ea75487f1e5caa5ececa02

~$
```

  - Are there any labels?

```
~$ kubectl describe pod apache | grep Labels

Labels:       run=apache

~$
```

The `run` label is attached to every pod created using `kubectl run` with the pod's name as the value.

- Using the `kubectl delete -h` help page, try to delete the new `apache` pod using any labels you find

```
~$ kubectl delete -h | grep label

Delete resources by file names, stdin, resources and names, or by resources and label selector.
 JSON and YAML formats are accepted. Only one type of argument may be specified: file names, resources and names, or resources and label selector.
  # Delete pods and services with label name=myLabel
  -l, --selector='': Selector (label query) to filter on, not including uninitialized ones.
  kubectl delete ([-f FILENAME] | [-k DIRECTORY] | TYPE [(NAME | -l label | --all)]) [options]

~$ kubectl delete pod -l run=apache

pod "apache" deleted

~$
```


### 5. CHALLENGE: a complex pod

Next let’s try creating a pod with a more complex specification.

Create a pod config that describes a pod called `hello` with a:

- container based on an _ubuntu:14.04_ image
- with an environment variable called `MESSAGE` and a value `hello world`
- command that will `echo` that message to stdout: `/bin/sh -c "echo $MESSAGE"`
- make sure that the container is _never restarted_

This challenge can be completed imperatively with `kubectl run` flags:

```
~$ kubectl run hello \
--image ubuntu:14.04 \
--env MESSAGE="hello world" \
--restart Never \
--command -- /bin/sh -c "echo \$MESSAGE"
```

`complex-pod.yaml` declaratively may also look like:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello
spec:
  restartPolicy: Never
  containers:
  - name: hellocont
    image: "ubuntu:14.04"
    env:
    - name: MESSAGE
      value: "hello world"
    command: ["/bin/sh","-c"]
    args: ["/bin/echo \"${MESSAGE}\""]
```

Separating the `command` and `args` arrays is a best practice for readability; however there is no flag for `args` in
the `kubectl run` command so the entire shell and echo is stated with the `--command` flag.

Check for success by running `kubectl logs hello` after applying.


_Copyright (c) 2013-2023 RX-M LLC, Cloud Native Consulting, all rights reserved_
