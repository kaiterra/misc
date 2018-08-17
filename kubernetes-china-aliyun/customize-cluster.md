# k8s China - Configure kubelet and kubeadm for use with a cluster

## kubeadm.conf

There are a few more kubeadm.conf changes to make, in addition to the ones you've already made.

`podSubnet` is necessary to play nice with the kube-flannel-aliyun.yaml config from the networking setup later on.  This must match.

```bash
sed -i "s/  podSubnet: .*/  podSubnet: \"10.24.0.0\/16\"/g" kubeadm.conf
```


## kubelet configuration

When you installed kubeadm, it created the `/etc/systemd/system/kubelet.service.d` directory so it can configure kubelet.  Find the .conf file there, and add a line like this to it:

```
Environment="KUBELET_CUSTOM_ARGS=--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.1 --cgroup-driver=cgroupfs --allow-privileged=true"
```

And then add `$KUBELET_CUSTOM_ARGS` to the end of the `ExecStart` command.

> If for some reason this file doesn't exist, get the appropriate one from [github](https://github.com/kubernetes/kubernetes/tree/b930d7f9153f1c74a8927de3f061a448c2a5a98c/build/debs). The one for kubelet <1.11 is much fatter and has settings like `$KUBELET_NETWORK_ARGS`.

Here's what's going on here:

- kubelet needs an image from gcr.io. That won't work in China, so we override it with `--pod-infra-container-image`. Run `kubelet --help 2>&1 | grep pod-infra-container-image` to find out what version kubelet uses as the default.
- kubelet needs to be told what cgroup driver to use. Do `docker info | grep -i cgroup` to double check that it's `cgroupfs`.
- `--allow-privileged=true` is needed for the flannel (networking) container.

**Warning:** --allow-privileged will be removed in Kubernetes v1.12. There's a replacement, but I couldn't figure out what it was after 5 minutes of googling.

Then, do `systemctl daemon-reload && systemctl restart kubelet`. It'll get stuck in a restart crash loop, but that's by design -- it'll start working properly once `kubeadm init` generates kubelet's configuration.

> Given [this Github issue](https://github.com/kubernetes/kubeadm/issues/28), it seems like such manual configuration of kubelet shouldn't be necessary anymore, but perhaps that issue is referring to something else.


