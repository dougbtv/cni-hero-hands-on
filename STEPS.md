## Create a cluster

```bash
kind create cluster --name cni-demo --config kind-config.yaml
```

Then we can get nodes...


```bash
$ kubectl get nodes
NAME                     STATUS     ROLES           AGE    VERSION
cni-demo-control-plane   NotReady   control-plane   103s   v1.27.3
cni-demo-worker          NotReady   <none>          84s    v1.27.3
```

We notice that they're not ready. So, let's take a look inside...

```bash
docker ps
docker exec -it cni-demo-worker /bin/bash
cd /etc/cni/net.d/
ls -lathr
# nothing here!
```

So! Now we've got to install a plugin.

```
kubectl create -f kube-flannel.yml
```

Now we can see that the nodes are ready:

```
[fedora@knidevel-master cni-hero-hands-on]$ kubectl get nodes
NAME                     STATUS   ROLES           AGE     VERSION
cni-demo-control-plane   Ready    control-plane   7m32s   v1.27.3
cni-demo-worker          Ready    <none>          7m13s   v1.27.3
```

And we can see there are CNI configurations...


```
[fedora@knidevel-master cni-hero-hands-on]$ docker exec -it cni-demo-worker /bin/bash
root@cni-demo-worker:/# ls -lathr /opt/cni/bin/
total 18M
-rwxr-xr-x. 1 root root 3.4M May 25  2023 host-local
-rwxr-xr-x. 1 root root 3.4M May 25  2023 loopback
-rwxr-xr-x. 1 root root 4.2M May 25  2023 ptp
-rwxr-xr-x. 1 root root 3.8M May 25  2023 portmap
drwxr-xr-x. 1 root root    6 May 25  2023 ..
drwxr-xr-x. 1 root root   14 Feb 29 19:43 .
-rwxr-xr-x. 1 root root 2.4M Feb 29 19:43 flannel
root@cni-demo-worker:/# ls -lathr /etc/cni/net.d/
total 4.0K
drwxr-xr-x. 1 root root  10 Jun 15  2023 ..
-rw-r--r--. 1 root root 292 Feb 29 19:43 10-flannel.conflist
drwx------. 1 root root  38 Feb 29 19:43 .
```