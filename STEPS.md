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



