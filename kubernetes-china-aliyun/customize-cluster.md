# k8s China - Configure kubelet and kubeadm for use with a cluster

## kubeadm.conf

There are a few more kubeadm.conf changes to make, in addition to the ones you've already made.

`podSubnet` is necessary to play nice with the kube-flannel-aliyun.yaml config from the networking setup later on.  This must match.

```bash
sed -i "s/  podSubnet: .*/  podSubnet: \"10.24.0.0\/16\"/g" kubeadm.conf
```

The self-signed x509 certs that kubeadm generates won't work with an Aliyun machine in a VPC. That's because the public IP isn't actually one of the network interfaces available to the machine, so when kubeadm is detecting all the interfaces it needs to add to the cert's alternate names, the public IP won't be one of them.

So, you also need to add any names you'll use to connect to the cluster in the kubeadm.conf file. This includes any public DNS names you'll want to use to connect to your cluster. Add these under the `kind: MasterConfiguration` section:

```yaml 
apiServerCertSANs:
- [[WHATEVER-DNS-NAME]]
- [[AND-MAYBE-THE-PUBLIC-IP-IF-YOU-WANT]]
```

## kubelet configuration

Conveniently, `kubeadm.conf` has some hooks that can change kubelet's configuration.  Under the `kind: MasterConfiguration` section, under `nodeRegistration:`, add this block of yaml:

```yaml
  kubeletExtraArgs:
    # kubelet needs an image from gcr.io. That won't work in China, so we override it
    # here. Run `kubelet --help 2>&1 | grep pod-infra-container-image` to find out
    # what version kubelet uses as the default.
    pod-infra-container-image: registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.1
    # Needed for the flannel (networking) container
    allow-privileged: "true"
    # Yet another place where it's necessary to set the hostname
    hostname-override: [[YOUR-HOSTNAME]]
    # Needed for the storage driver
    enable-controller-attach-detach: "false"
```

**Warning:** --allow-privileged will be removed in Kubernetes v1.12. There's a replacement, but I couldn't figure out what it was after 5 minutes of googling.

Then, do `systemctl daemon-reload && systemctl restart kubelet`. It'll get stuck in a restart crash loop, but that's by design -- it'll start working properly once `kubeadm init` generates kubelet's configuration.


