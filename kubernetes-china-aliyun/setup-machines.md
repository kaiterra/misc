# k8s China - Setup Machines

### Trimming down DNS options

Flannel, which we'll be using for networking, runs in a container that uses Alpine Linux as its base image (the [Dockerfile](https://github.com/coreos/flannel/blob/master/Dockerfile.amd64) doesn't specify which version, but you can find it by checking `/etc/alpine-release` -- as of August 2018, flannel v0.10.0-amd64 was using Alpine 3.7).

Alpine is [notorious](https://www.google.com/search?q=alpine+resolv.conf+options) for tricky DNS behavior, where the presence of certain settings in `/etc/resolv.conf` simply causes all DNS resolutions to fail, without any indication as to why.  Some options break DNS altogether, and some only break it for golang binaries that were cross-compiled to run on Alpine Linux.

> If you don't perform these steps, the symptom will be that the flannel container will fail to start, and logs will show DNS resolution failures.

As a workaround, you'll need to make sure the options inside `/etc/resolv.conf` are compatible with Alpine and golang.  `resolv.conf` is a generated file, so instead modify `/etc/resolvconf/resolv.conf.d/tail`, and if it contains an `options` line, remove all of them except `rotate`:

```
# disable options that are incompatible with alpine linux
#options timeout:2 attempts:3 rotate single-request-reopen
options rotate
```

### Opening up the security group

By default, VPCs only allow traffic from other nodes on the interface that falls within the VPC's subnet (the one starting with `172.16.0.0/20`).  However, flannel will be creating VPC routes and forwarding traffic in the pod's subnet, which by default is `10.244.0.0/16`.  So, this trafic will be dropped by the cloud-provided firewall.

Assuming you have a security group (安全组) that covers all the nodes in your cluster, add an exception to allow all traffic from any node with this address.  In chinese, the options will look like this:

授权策略 | 协议类型 | 端口范围 | 授权类型 | 授权对象
--------|---------|---------|---------|-------
允许    |    全部  | -1/-1   | 地址段访问 | 10.244.0.0/16


### Node host names

The storage driver used in this guide requires the node name to be of the form `regionId.instanceId`, for example: `cn-hangzhou.i-bp12gei4ljuzilgwzahc`.  While this is inconvenient, if you know about it while you're still setting up your cluster, not much extra work is required.

> Normally, kubeadm uses the machine's host name (likely the output of something like `$(hostname)`) as the node name. There are ways to override this, but they increase complexity.  The downside is that hostnames with dots in them are generally [a terrible idea](https://serverfault.com/questions/229331/can-i-have-dots-in-a-hostname) unless it's the machine's addressable FQDN. And in this case, the storage driver requires the region to come FIRST, which is the opposite of how DNS works. Future updates to this guide could detail how to change the hostname back to something sane after adding the node to the cluster.

Aliyun automatically sets the machine's hostname to something like its instance ID (`i-[a-z0-9]+`), replacing the dash after the 'i' with the character 'Z', and sometimes adding a final 'Z'.  This doesn't make any sense so we'll change it.

Changes made with `hostnamectl` don't always stick after a reboot, so to be safe we'll edit `/etc/hostname` directly:

```bash
echo 'cn-hangzhou.i-bp12gei4ljuzilgwzahc' | tee /etc/hostname

# Add the above string in the obvious place to /etc/hosts
nano /etc/hosts

# Ensure your settings aren't getting overwritten on boot
reboot now

# When the machine comes back up, verify that these match:
hostname
hostname -f
```

Once this is done, the storage driver will be able to work properly.
