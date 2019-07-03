# Stateful application behaviour in multizone setups

When deploying stateful applications across availability zones where storage is bound
to the specific avialability zone, the preferred approach is to use dynamic provisioning
using a topology aware provisioner, and ensuring that `volumeBindingMode: WaitForFirstConsumer`
is set in the storageclass.
This will ensure that the workload is spread across the availability zones, and rescheduled pods
are scheduled to the same zone as the storage. In the case of failure of a zone workloads are not rescheduled outside the original zone.


Application is created in each AZ
```
web-0                      1/1     Running   0          128m    172.23.125.197   ip-10-10-12-162.eu-west-1.compute.internal   <none>           <none>
web-1                      1/1     Running   0          127m    172.23.107.138   ip-10-10-10-134.eu-west-1.compute.internal   <none>           <none>
web-2                      1/1     Running   0          127m    172.23.84.201    ip-10-10-11-4.eu-west-1.compute.internal     <none>           <none>
```


PVCs are created
```
root@ip-10-10-10-178:~/examples/staging/https-nginx# kubectl get pvc --show-labels
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE    LABELS
www-web-0   Bound    pvc-f689ca22-9ce6-11e9-b402-06a942a7f68c   4Gi        RWO            slow           125m   app=nginx
www-web-1   Bound    pvc-0764276e-9ce7-11e9-b402-06a942a7f68c   4Gi        RWO            slow           125m   app=nginx
www-web-2   Bound    pvc-17f7293c-9ce7-11e9-b402-06a942a7f68c   4Gi        RWO            slow           125m   app=nginx
```

PVs are created in each AZ
```
pvc-0764276e-9ce7-11e9-b402-06a942a7f68c   4Gi        RWO            Delete           Bound    default/www-web-1                           slow                                126m   failure-domain.beta.kubernetes.io/region=eu-west-1,failure-domain.beta.kubernetes.io/zone=eu-west-1a
pvc-17f7293c-9ce7-11e9-b402-06a942a7f68c   4Gi        RWO            Delete           Bound    default/www-web-2                           slow                                125m   failure-domain.beta.kubernetes.io/region=eu-west-1,failure-domain.beta.kubernetes.io/zone=eu-west-1b
pvc-f689ca22-9ce6-11e9-b402-06a942a7f68c   4Gi        RWO            Delete           Bound    default/www-web-0                           slow                                126m   failure-domain.beta.kubernetes.io/region=eu-west-1,failure-domain.beta.kubernetes.io/zone=eu-west-1c
```


Draining one node in a AZ moves the workload
```
root@ip-10-10-10-178:~/examples/staging/https-nginx# kubectl drain ip-10-10-12-162.eu-west-1.compute.internal --ignore-daemonsets
node/ip-10-10-12-162.eu-west-1.compute.internal already cordoned
WARNING: Ignoring DaemonSet-managed pods: audit-logging-fluentd-ds-55w4v, calico-node-drggv, image-manager-init-certs-d2s5r, k8s-proxy-r55gx, logging-elk-filebeat-ds-dbj4s, metering-reader-z75fm, monitoring-prometheus-nodeexporter-jjdhz, nvidia-device-plugin-tdhr8
pod/web-0 evicted
node/ip-10-10-12-162.eu-west-1.compute.internal evicted
```

```
root@ip-10-10-10-178:~/examples/staging/https-nginx# kubectl get pods -o wide
NAME                       READY   STATUS    RESTARTS   AGE     IP               NODE                                         NOMINATED NODE   READINESS GATES
web-0                      1/1     Running   0          51s     172.23.34.5      ip-10-10-12-202.eu-west-1.compute.internal   <none>           <none>
web-1                      1/1     Running   0          129m    172.23.107.138   ip-10-10-10-134.eu-west-1.compute.internal   <none>           <none>
web-2                      1/1     Running   0          129m    172.23.84.201    ip-10-10-11-4.eu-west-1.compute.internal     <none>           <none>
```

PVs are still in the same AZ
```
gion=eu-west-1,failure-domain.beta.kubernetes.io/zone=eu-west-1a
pvc-17f7293c-9ce7-11e9-b402-06a942a7f68c   4Gi        RWO            Delete           Bound    default/www-web-2                           slow                                130m   failure-domain.beta.kubernetes.io/region=eu-west-1,failure-domain.beta.kubernetes.io/zone=eu-west-1b
pvc-f689ca22-9ce6-11e9-b402-06a942a7f68c   4Gi        RWO            Delete           Bound    default/www-web-0                           slow                                131m   failure-domain.beta.kubernetes.io/region=eu-west-1,failure-domain.beta.kubernetes.io/zone=eu-west-1c
```


Draining last worker node in AZ
```
root@ip-10-10-10-178:~# kubectl drain --ignore-daemonsets ip-10-10-12-202.eu-west-1.compute.internal
node/ip-10-10-12-202.eu-west-1.compute.internal cordoned
WARNING: Ignoring DaemonSet-managed pods: audit-logging-fluentd-ds-lhqtw, calico-node-nt64l, image-manager-init-certs-gsc8d, k8s-proxy-9vvfj, logging-elk-filebeat-ds-qf296, metering-reader-nsh6r, monitoring-prometheus-nodeexporter-j9dt5, nvidia-device-plugin-j5tbf
pod/web-0 evicted
node/ip-10-10-12-202.eu-west-1.compute.internal evicted
```

The POD in AZ which is now unavailable remains pending
```
web-0                      0/1     Pending   0          56s   <none>           <none>                                       <none>           <none>
web-1                      1/1     Running   0          15h   172.23.107.138   ip-10-10-10-134.eu-west-1.compute.internal   <none>           <none>
web-2                      1/1     Running   0          15h   172.23.84.201    ip-10-10-11-4.eu-west-1.compute.internal     <none>           <none>
```

The POD can not be scheduled to a different AZ because the volume can not migrate
```
root@ip-10-10-10-178:~# kubectl describe web-0
error: the server doesn't have a resource type "web-0"
root@ip-10-10-10-178:~# kubectl describe pod web-0
Name:               web-0
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               <none>
Labels:             app=nginx
                    controller-revision-hash=web-67bb74dc
                    statefulset.kubernetes.io/pod-name=web-0
Annotations:        kubernetes.io/psp: ibm-privileged-psp
Status:             Pending
IP:
Controlled By:      StatefulSet/web
Containers:
  nginx:
    Image:        nginx
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:
      /usr/share/nginx/html from www (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-8tc7z (ro)
Conditions:
  Type           Status
  PodScheduled   False
Volumes:
  www:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  www-web-0
    ReadOnly:   false
  default-token-8tc7z:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-8tc7z
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason            Age               From               Message
  ----     ------            ----              ----               -------
  Warning  FailedScheduling  37s (x3 over 2m)  default-scheduler  0/14 nodes are available: 2 node(s) were unschedulable, 4 node(s) had volume node affinity conflict, 8 node(s) had taints that the pod didn't tolerate.
```
