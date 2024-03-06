# From CNI Zero to CNI Hero: The Hands on tutorial

## Requirements

* A functional [installation of KIND](https://kind.sigs.k8s.io/docs/user/quick-start/)

## All the steps

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

## Install Flannel

Flannel @ https://github.com/flannel-io/flannel

From the docs:

> Flannel is a simple and easy way to configure a layer 3 network fabric designed for Kubernetes.

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

## Debugging a failed CNI plugin call

We're so close, but so far away... Now pods won't come up:

```
kubectl create -f sample-pod.yml 
kubectl get pods
kubectl describe pod sample-pod
```

Now we can see there's an error in the events...

```
Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox "668c86dda8924fa560d10d47b48c3a9ca16ee5a23471949aeabedaec2f86a2fb": plugin type="flannel" failed (add): failed to delegate add: failed to find plugin "bridge" in path [/opt/cni/bin]
```

*NOTE*: The `bridge` CNI plugin isn't required for all installations. The architecture of Flannel CNI uses other CNI plugins behind the scenes that it "delegates" to. In our case, from a user/administrator perspective, we might not have known this until we execute this code, that is, if we aren't familiar with the code of Flannel.

Let's check the kubelet logs...
```
docker exec -it cni-demo-worker /bin/bash
journalctl -u kubelet | grep -i error
```

Let's delete that pod:

```
kubectl delete -f sample-pod.yml 
```

So let's install the CNI reference plugins ourselves...

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

Now we can see the binaries on the host's disk...

Since we're running KIND (kubernetes in docker), we'll need to "docker exec" to see our "host" (in quotes because it's kind of a "pretend host" if-you-will)

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
kubectl create -f sample-pod.yml
kubectl exec -it sample-pod -- ip a
```

And we can see it has IPs!

Now let's clean up that pod.

```
kubectl delete pod sample-pod
```

## Installing a custom CNI plugin!

We are going to have our own CNI plugin execute for some learning and debug purposes.

While this is intended to be illustrative, writing a CNI plugin with bash isn't sustainable.

If you'd like to use an official debugging plugin, I recommend the [debug CNI plugin from the containernetworking/cni repository on GitHub](https://github.com/s-matyukevich/bash-cni-plugin/blob/master/bash-cni)

Let's "log in to our host" again, with:

```
docker exec -it cni-demo-worker /bin/bash
```

Then we need to create a new "binary" -- it's actually just a bash script. 

We'll do this by creating a bash file on the "host"

```
cat >/opt/cni/bin/dummy <<'EOF'
#!/bin/bash
logit () {
  echo "$1" >> /tmp/cni-inspect.log
}

logit "-------------- CNI call for $CNI_COMMAND on $CNI_CONTAINERID"
logit "CNI_COMMAND: $CNI_COMMAND"
logit "CNI_CONTAINERID: $CNI_CONTAINERID"
logit "CNI_NETNS: $CNI_NETNS"
logit "CNI_IFNAME: $CNI_IFNAME"
logit "CNI_ARGS: $CNI_ARGS"
logit "CNI_PATH: $CNI_PATH"
logit "-- cni config"
stdin=$(cat /dev/stdin)
logit "$stdin"
EOF
```

Make it executable...

```
chmod +x /opt/cni/bin/dummy
```

Now we need a configuration for it...

Let's look at where our CNI configuration is.

```
$ ls /etc/cni/net.d/ -1
10-flannel.conflist
```

Since this is named `10-flannel` what we'll do is make ours ascii-betically first, so we can remove it later and restore the original functionality to the cluster...

```
cat >/etc/cni/net.d/00-dummy.conf <<EOF
{
    "cniVersion": "0.2.0",
    "name": "my_dummy_network",
    "type": "dummy"
}
EOF
```

