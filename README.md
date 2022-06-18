# Setting Up Wordpress examples in K3s

Here is quick start how to Deploy WordpRess example in Ranchers's
K3s (K3s lightweight Kubernetes implementation).


# Setting up Kubernetes

Tested Environment:
- Debian 11 (VM in Proxmox 7)
- 4GB RAM
- 1 CPU
- 16 GB Disk space, 2GB Swap

Add yourself to group `adm` (we will use it later):
```bash
sudo /usr/sbin/usermod -G adm -a $USER
```
Relogin your user so he will be in `adm` group

Install curl (requirement) for K3s:
```bash
sudo apt-get install curl
```

Now we will follow: 
- https://rancher.com/docs/k3s/latest/en/quick-start/

Download install script:
```bash
curl -sfLo k3s_setup.sh https://get.k3s.io
chmod +x k3s_setup.sh 
```
Review `k3s_setup.sh` and run:
```bash
./k3s_setup.sh
```

After setup you modify permissions to K3s configuration:
```bash
sudo chgrp adm /etc/rancher/k3s/k3s.yaml 
sudo chmod g+r /etc/rancher/k3s/k3s.yaml
```

Now you can try few standard Kubernetes commands, for example:
```bash
$ kubectl get nodes

NAME      STATUS   ROLES                  AGE   VERSION
d11-k3s   Ready    control-plane,master   42m   v1.23.6+k3s1

# list all resources you can query with "kubectl get"
$ kubectl api-resources

# see all pods (collocated containers) in all namespaces:
$ kubectl get pods -A

# see CPU and memory usage by Pods:
$ kubectl top pod -A

AMESPACE     NAME                                      CPU(cores)   MEMORY(bytes)
kube-system   coredns-d76bd69b-bnrkh                    3m           21Mi
kube-system   local-path-provisioner-6c79684f77-2rqhq   1m           14Mi
kube-system   metrics-server-7cd5fcb6b7-j27nh           10m          30Mi
kube-system   svclb-traefik-6v2fn                       0m           0Mi
kube-system   traefik-df4ff85d6-7nqhc                   1m           32Mi

# see Node utilization:
$ kubectl top node

NAME      CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
d11-k3s   121m         12%    2815Mi          71%
```

# Deploying WordPress Example

Now we can Deploy (modified) WordPress example from:
- https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/

Setup directory for YAML manifests:
```bash
mkdir -p ~/wp-example
```

Change to that directory and download example manifests from Google:
```bash
cd ~/wp-example
curl -fLO https://raw.githubusercontent.com/kubernetes/\
website/main/content/en/examples/application/wordpress/mysql-deployment.yaml

curl -fLO https://raw.githubusercontent.com/kubernetes/\
website/main/content/en/examples/application/wordpress/wordpress-deployment.yaml
```

Now edit `wordpress-deployment.yaml`
- replace `LoadBalancer` with `NodePort`

Create new file `kustomization.yaml` with this content:
```yaml
# kustomization.yaml
---
namespace: wp1

secretGenerator:
- name: mysql-pass
  literals:
    - password=YOUR_PASSWORD

resources:
  - mysql-deployment.yaml
  - wordpress-deployment.yaml
```

Try dry-run mode to see if YAML files are correct:
```bash
kubectl apply -k ./ --dry-run
```

Now create our custom namespace and run Kustomize:
```bash
kubectl create namespace wp1
kubectl apply -k ./
```

Now poll this command:
```bash
kubectl get pods -n wp1
```
And wait till all Pods are in state `Running`, for example:
```bash
$ kubectl get pods -n wp1

NAME                               READY   STATUS    RESTARTS   AGE
wordpress-mysql-6fc4d469bd-lpxjf   1/1     Running   0          30m
wordpress-69b54b758-4qgd5          1/1     Running   0          30m
```

Now we need to find WP IP address and port:
```bash
$ kubectl get svc wordpress -n wp1

NAME        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
wordpress   NodePort   10.43.213.162   <none>        80:31258/TCP   31m
```

Now you can try:
```bash
curl -fv 10.43.213.162
```
If you see redirect like this:
```
Location: http://10.43.213.162/wp-admin/install.php
```
It works!

Please see:
- https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/
for details on this example

# FAQ

* Where are my data?
* WordPress requires two persistent storages:
  1st for User Uploads
  2nd for MySQL database

Application requests them via PVC - Persistent Volume Claims. In our example
you can see them with:
```bash
$ kubectl get pvc -n wp1

NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
wp-pv-claim      Bound    pvc-58d07332-8482-4786-b946-a6477ed6888f   20Gi       RWO            local-path     37m
mysql-pv-claim   Bound    pvc-83889bdf-48d4-4391-84e2-f396dd397eae   20Gi       RWO            local-path     37m
```

These claims are fulfilled with so called PV - Persistent Volumes - you can see them
with:
```bash
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                STORAGECLASS   REASON   AGE
pvc-58d07332-8482-4786-b946-a6477ed6888f   20Gi       RWO            Delete           Bound    wp1/wp-pv-claim      local-path              37m
pvc-83889bdf-48d4-4391-84e2-f396dd397eae   20Gi       RWO            Delete           Bound    wp1/mysql-pv-claim   local-path              37m
```

If you wan to see real mapped paths you can try this:
```bash
$ sudo apt-get install jq

$ kubectl get pv -o json | \
    jq -r '.items[] | [.spec.claimRef.name,.spec.hostPath.path] | @tsv'

wp-pv-claim	/var/lib/rancher/k3s/storage/pvc-58d07332-8482-4786-b946-a6477ed6888f_wp1_wp-pv-claim
mysql-pv-claim	/var/lib/rancher/k3s/storage/pvc-83889bdf-48d4-4391-84e2-f396dd397eae_wp1_mysql-pv-claim
```
And then try:
```
$ ls -l /var/lib/rancher/k3s/storage/pvc-83889bdf-48d4-4391-84e2-f396dd397eae_wp1_mysql-pv-claim

total 110596
-rw-rw---- 1 systemd-coredump systemd-coredump       56 Jun 18 15:31 auto.cnf
-rw-rw---- 1 systemd-coredump systemd-coredump 12582912 Jun 18 15:32 ibdata1
-rw-rw---- 1 systemd-coredump systemd-coredump 50331648 Jun 18 15:32 ib_logfile0
-rw-rw---- 1 systemd-coredump systemd-coredump 50331648 Jun 18 15:31 ib_logfile1
drwx------ 1 systemd-coredump systemd-coredump     2486 Jun 18 15:32 mysql
drwx------ 1 systemd-coredump systemd-coredump     3296 Jun 18 15:31 performance_schema
drwx------ 1 systemd-coredump systemd-coredump       12 Jun 18 15:32 wordpress
```

# Resources

* https://rancher.com/docs/k3s/latest/en/quick-start/
* https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/
* You can find more on Local Path Provisioner in K3s:
  - https://rancher.com/docs/k3s/latest/en/storage/








