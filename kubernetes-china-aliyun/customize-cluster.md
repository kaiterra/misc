# k8s China - kubeadm.conf cluster configuration

To configure a k8s master for an Aliyun cluster, use these steps.

### Certificate domain names

The self-signed x509 certs that kubeadm generates won't work with an Aliyun machine in a VPC. That's because the public IP isn't actually one of the network interfaces available to the machine, so when kubeadm is detecting all the interfaces it needs to add to the cert's alternate names, the public IP won't be one of them.

So, you also need to add any names you'll use to connect to the cluster in the kubeadm.conf file. This includes any public DNS names you'll want to use to connect to your cluster. Add these under the `kind: MasterConfiguration` section:

```yaml 
apiServerCertSANs:
- [[WHATEVER-DNS-NAME]]
- [[AND-MAYBE-THE-PUBLIC-IP-IF-YOU-WANT]]
```

Conveniently, `kubeadm.conf` has some hooks that can change kubelet's configuration.  Under the `kind: MasterConfiguration` section, under `nodeRegistration:`, add this block of yaml:

### kubelet config

```yaml
  kubeletExtraArgs:
    # kubelet needs an image from gcr.io. That won't work in China, so we override it
    # here. Run `kubelet --help 2>&1 | grep pod-infra-container-image` to find out
    # what version kubelet uses as the default.
    pod-infra-container-image: registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.1
    # Needed for the flannel (networking) container.
    # **WARNING**: this will be removed in Kubernetes v1.12.  There's a replacement, but I couldn't
    # figure out what it was after 5 minutes of googling.
    allow-privileged: "true"
    # Needed for the storage driver
    enable-controller-attach-detach: "false"
```
