# Limiting deployments to two zones
Limiting applications to specific zones can be done by using affinity rules in the pod spec

For example
```
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: failure-domain.beta.kubernetes.io/zone
          operator: In
          values:
          - eu-west-1a
          - eu-west-1b
```


Applications with affinity rules are scheduled to the required locations
```
root@ip-10-10-10-178:~# kubectl get pods -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP               NODE                                         NOMINATED NODE   READINESS GATES
mynginx-7bd4f7746f-kmpn2   1/1     Running   0          42s   172.23.84.197    ip-10-10-11-4.eu-west-1.compute.internal     <none>           <none>
mynginx-7bd4f7746f-qr4dq   1/1     Running   0          42s   172.23.107.133   ip-10-10-10-134.eu-west-1.compute.internal   <none>           <none>
mynginx-7bd4f7746f-zzhfb   1/1     Running   0          42s   172.23.85.133    ip-10-10-10-229.eu-west-1.compute.internal   <none>           <none>
```
You can quickly see from the node names that zone 3 (eu-west-1c) is not used (subnet 10.10.12.0/24)


Even if scaled really high

```
root@ip-10-10-10-178:~# kubectl scale deployment mynginx --replicas=20
deployment.extensions/mynginx scaled
```

`eu-west-1c` is never touched
```
root@ip-10-10-10-178:~# kubectl get pods -o wide
NAME                       READY   STATUS    RESTARTS   AGE    IP               NODE                                         NOMINATED NODE   READINESS GATES
mynginx-7bd4f7746f-2h785   1/1     Running   0          47s    172.23.85.135    ip-10-10-10-229.eu-west-1.compute.internal   <none>           <none>
mynginx-7bd4f7746f-45pzr   1/1     Running   0          47s    172.23.107.136   ip-10-10-10-134.eu-west-1.compute.internal   <none>           <none>
mynginx-7bd4f7746f-4p9sr   1/1     Running   0          47s    172.23.102.6     ip-10-10-11-12.eu-west-1.compute.internal    <none>           <none>
mynginx-7bd4f7746f-5k9vw   1/1     Running   0          47s    172.23.84.198    ip-10-10-11-4.eu-west-1.compute.internal     <none>           <none>
mynginx-7bd4f7746f-87vm8   1/1     Running   0          47s    172.23.102.8     ip-10-10-11-12.eu-west-1.compute.internal    <none>           <none>
mynginx-7bd4f7746f-cxs7w   1/1     Running   0          47s    172.23.102.5     ip-10-10-11-12.eu-west-1.compute.internal    <none>           <none>
mynginx-7bd4f7746f-flzq2   1/1     Running   0          47s    172.23.84.200    ip-10-10-11-4.eu-west-1.compute.internal     <none>           <none>
mynginx-7bd4f7746f-gj7bh   1/1     Running   0          47s    172.23.102.7     ip-10-10-11-12.eu-west-1.compute.internal    <none>           <none>
mynginx-7bd4f7746f-hgz68   1/1     Running   0          47s    172.23.85.134    ip-10-10-10-229.eu-west-1.compute.internal   <none>           <none>
mynginx-7bd4f7746f-jgtd9   1/1     Running   0          47s    172.23.85.137    ip-10-10-10-229.eu-west-1.compute.internal   <none>           <none>
mynginx-7bd4f7746f-kmpn2   1/1     Running   0          5m4s   172.23.84.197    ip-10-10-11-4.eu-west-1.compute.internal     <none>           <none>
mynginx-7bd4f7746f-ldjw6   1/1     Running   0          47s    172.23.102.10    ip-10-10-11-12.eu-west-1.compute.internal    <none>           <none>
mynginx-7bd4f7746f-m952d   1/1     Running   0          47s    172.23.85.136    ip-10-10-10-229.eu-west-1.compute.internal   <none>           <none>
mynginx-7bd4f7746f-pm985   1/1     Running   0          47s    172.23.102.9     ip-10-10-11-12.eu-west-1.compute.internal    <none>           <none>
mynginx-7bd4f7746f-qr4dq   1/1     Running   0          5m4s   172.23.107.133   ip-10-10-10-134.eu-west-1.compute.internal   <none>           <none>
mynginx-7bd4f7746f-rmw5j   1/1     Running   0          47s    172.23.107.134   ip-10-10-10-134.eu-west-1.compute.internal   <none>           <none>
mynginx-7bd4f7746f-vjbpg   1/1     Running   0          47s    172.23.84.199    ip-10-10-11-4.eu-west-1.compute.internal     <none>           <none>
mynginx-7bd4f7746f-w7zrg   1/1     Running   0          47s    172.23.107.137   ip-10-10-10-134.eu-west-1.compute.internal   <none>           <none>
mynginx-7bd4f7746f-zs866   1/1     Running   0          47s    172.23.107.135   ip-10-10-10-134.eu-west-1.compute.internal   <none>           <none>
mynginx-7bd4f7746f-zzhfb   1/1     Running   0          5m4s   172.23.85.133    ip-10-10-10-229.eu-west-1.compute.internal   <none>           <none>

root@ip-10-10-10-178:~# kubectl get pods -o wide | grep 10-10-12
root@ip-10-10-10-178:~#
```
