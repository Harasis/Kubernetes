![RX-M, llc.](http://rx-m.io/rxm-cnc.svg)


# Kubernetes


## Volumes Challenge Solutions

Now that you've learned how to work with Persistent Volumes, try the following to test your knowledge:

- Create two persistent volumes, one with storage class name is gold and the other has storage class name is silver.
- Create a persistent volume claim that uses the gold storage class. Verify that the correct persistent volume is used.

```
~/state$ nano pvchallenge.yaml && cat $_
```
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pvgold
spec:
  storageClassName: gold
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/tmp/pvgold"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pvsilver
spec:
  storageClassName: silver
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/tmp/pvsilver"
```
```
~/state$ kubectl apply -f pvchallenge.yaml

persistentvolume/pvgold created
persistentvolume/pvsilver created

~/state$ kubectl get pv

NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                  STORAGECLASS   REASON   AGE
local-pv1   100Mi      RWO            Retain           Bound       default/data-myweb-0   local                   7m50s
local-pv2   100Mi      RWO            Retain           Released    default/data-myweb-1   local                   7m50s
pvgold      100Mi      RWO            Retain           Available                          gold                    4s
pvsilver    100Mi      RWO            Retain           Available                          silver                  4s

~/state$
```

Create the persistent volume claim that claims the gold storage class persistent volume.

```
~/state$ nano pvcchallenge.yaml && cat $_
```
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvcchallenge
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 100Mi
  storageClassName: gold
```
```
~/state$ kubectl apply -f pvcchallenge.yaml

persistentvolumeclaim/pvcchallenge created

~/state$ kubectl get pvc

NAME           STATUS   VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-myweb-0   Bound    local-pv1   100Mi      RWO            local          7m51s
pvcchallenge   Bound    pvgold      100Mi      RWO            gold           3s

~/state$
```

Verify the correct persistent volume is bound to the persistent volume claim:

```
~/state$ kubectl get pv

NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                  STORAGECLASS   REASON   AGE
local-pv1   100Mi      RWO            Retain           Bound       default/data-myweb-0   local                   8m26s
local-pv2   100Mi      RWO            Retain           Released    default/data-myweb-1   local                   8m26s
pvgold      100Mi      RWO            Retain           Bound       default/pvcchallenge   gold                    40s
pvsilver    100Mi      RWO            Retain           Available                          silver                  40s

~/state$

```

The persistent volume claim bounded the correct persistent volume using the storage class name.

Delete the persistent volumes and persistent volume claims:

```
~/state$ kubectl delete pvc pvcchallenge

persistentvolumeclaim "pvcchallenge" deleted

~/state$ kubectl delete pv pvgold pvsilver

persistentvolume "pvgold" deleted
persistentvolume "pvsilver" deleted

~/state$
```

_Copyright (c) 2013-2023 RX-M LLC, Cloud Native Consulting, all rights reserved_
