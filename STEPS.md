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
[...]
-rwxr-xr-x. 1 root root 2.4M Feb 29 19:43 flannel
root@cni-demo-worker:/# ls -lathr /etc/cni/net.d/
[...]
-rw-r--r--. 1 root root 292 Feb 29 19:43 10-flannel.conflist
```

Actually this works out... Now pods won't come up because we don't have bridge CNI!

```
kubectl create -f sample-pod.yml 
kubectl get pods
kubectl describe pod sample-pod
```

Now we can see there's an error in the events from the Kubelet...

```
Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox "668c86dda8924fa560d10d47b48c3a9ca16ee5a23471949aeabedaec2f86a2fb": plugin type="flannel" failed (add): failed to delegate add: failed to find plugin "bridge" in path [/opt/cni/bin]
```

Let's delete that pod:

```
kubectl delete -f sample-pod.yml 
```

So let's install that ourselves...

```
kubectl create -f reference-cni-plugins.yml
```

Let's look at that file and we'll notice:

```yaml
    # We want to schedule this on all nodes.
    tolerations:
    - operator: Exists
    effect: NoSchedule
    # We need this so that it can run without flannel.
    hostNetwork: true
```

Now we can see the binaries on disk...

```bash
$ docker exec -it cni-demo-worker ls /opt/cni/bin -lathr
total 71M
drwxr-xr-x. 1 root root    6 May 25  2023 ..
-rwxr-xr-x. 1 root root 2.4M Feb 29 19:43 flannel
-rwxr-xr-x. 1 root root 3.9M Feb 29 20:57 bandwidth
-rwxr-xr-x. 1 root root 4.4M Feb 29 20:57 bridge
[...]
```

Now we can start our sample pod again...


```

```


Let's check  the kubelet logs...
```
docker exec -it cni-demo-worker /bin/bash
journalctl -u kubelet | grep -i error
```