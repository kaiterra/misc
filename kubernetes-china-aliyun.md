# Kubernetes in China (Aliyun)

Thanks to Aliyun mirrors, in spring/summer 2018 this got a lot easier.

This guide uses kubeadm to set up a Kubernetes cluster on Aliyun. Details:

- Ubuntu 16.04
- Single master (non-HA). Should be straightforward to adapt to HA though.
- Integration with VPC networking

Not covered:

- Backup and Upgrading (maybe coming soon?)
- Tool-assisted machine provisioning (you'll need to do it manually)
- Aliyun storage or load balancer integration. It's unclear how difficult this is to set up. Instead, [local persistent volumes](https://kubernetes.io/blog/2018/04/13/local-persistent-volumes-beta/), which are in beta in k8s 1.10, seem like a viable alternative.


## 1. Set up your machines

Make sure they're all in the same VPC (专有网络).  The CIDR address block used for the VPC's network (目标网段; should be 172.17.0.0/16 or so) doesn't seem to be important.

This guide is for Ubuntu 16.04, but any [supported](https://success.docker.com/article/compatibility-matrix) Docker OS should work, though CoreOS is apparently the new hotness when it comes to hosting containerized workloads.



## 2. Install docker

> The only requirement for Kubernetes is a supported container runtime. For example, rkt will also work. But that's beyond the scope of this document.

17.03 is the [latest supported version](https://kubernetes.io/docs/tasks/tools/install-kubeadm/#installing-docker) with kubeadm right now, but as of 2018-07-12 Aliyun's own K8s service was using 17.06.

Docker Community Edition releases technically only have 4 months of support, so choose wisely.

Here's a summary of the [official installation instructions](https://docker.github.io/engine/installation/linux/ubuntu/#install-using-the-repository):

```bash
# TODO: Call sudo apt-key fingerprint.  Get this from the official instructions;
# this is the kind of thing for which you shouldn't trust a third-party site.
 
# Add the stable repository
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
 
# Find out what versions there are
apt-cache madison docker-ce

# Pick a version
apt-get install docker-ce=17.03.2~ce-0~ubuntu-xenial
```

**Important:** Configure log rotation so logs don't fill up your hard disk:

```bash
tee /etc/docker/daemon.json <<"EOF"
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5"
  }
}
EOF

systemctl daemon-reload && systemctl restart docker
```



## 3. Pick a Kubernetes version

You'll need to manually install kubelet (the main Kubernetes runtime component that runs on each machine), and that basically has to match the API server version (what you normally think of as the "Kubernetes Version" you're running).

Constraints on version choices include:

- kubelet can be at most 1 minor version behind the version of kubeadm you're using. So if you're using kubeadm v1.11.2, you can use kubelet >= v1.10.0, but not v1.9.6 for example.
- The Aliyun `google-containers` mirror may not have every single Kubernetes version. Mirrored versions are listed [here](https://dev.aliyun.com/detail.html?spm=5176.1972343.2.28.ItEn5B&repoId=91673). Note that there are `google-containers` images and `google_containers` images (with an underscore). Use the underscore one. (They're both provided by Alibaba, but the underscore one has many more versions, because... reasons?)
- Consider choosing one or two versions behind the latest, so you can practice upgrading before deploying a production workload.



## 4. Install and run kubelet

The [official instructions](https://kubernetes.io/docs/tasks/tools/install-kubeadm) tell you to add the packages.cloud.google.com repo, but that's obviously not going to work. Luckily, Aliyun has a mirror:

```bash
apt update && apt install -y apt-transport-https
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt update
```

So, pick one based on your Kubernetes version selection:

```bash
apt-cache madison kubelet

# Pick one, then:
apt install kubelet=1.10.3-00   # this depends on the k8s version you want to install
```



### Customizing kubelet

Add a line like this to the .conf file under `/etc/systemd/system/kubelet.service.d` in the kubeadm .conf file:

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



## 5. Install Kubeadm


You should use at least 1.11.0-00 because `kubeadm config print-default` and the imageRepository mirroring functionality depends on that.

```bash
# Ensure >= 1.11.0-00 is available
apt-cache madison kubeadm

apt install kubeadm
```



## 6. Bootstrap your cluster

Again, gcr.io is _persona non grata_ in China. So, first tell kubeadm that when it's starting up the Kuberentes components, it should get images from Aliyun's mirror. This and other settings are below:

```bash
# It's best if you run this command on the machine you're setting up the cluster on.
# If you keep getting TLS timeouts, then run it somewhere else, and change:
#  - the two places where your machine's hostname appears
#  - advertiseAddress to your machine's non-public IP
#
# And yes, you need to start from a full config file.  Just specifying the
# settings you want to override doesn't seem to work.
kubeadm config print-default > kubeadm.conf

# Use the gcr.io mirror
sed -i "s/imageRepository: .*/imageRepository: registry.cn-hangzhou.aliyuncs.com\/google_containers/g" kubeadm.conf

# And then since you're using kubeadm.conf, you can't mix that with command-line
# switches, so we throw the other setting changes into kubeadm.conf too.
#
# podSubnet is necessary to play nice with the kube-flannel-aliyun.yaml config
# from the networking setup below.  This must match.
sed -i "s/  podSubnet: .*/  podSubnet: \"10.24.0.0\/16\"/g" kubeadm.conf

# This is your chosen Kubernetes version
sed -i "s/kubernetesVersion: .*/kubernetesVersion: v1.10.3/g" kubeadm.conf
```



Then, start the cluster.

```bash
kubeadm init --config kubeadm.conf
```

If everything went well, you should see something like this near the end:

```
Your Kubernetes master has initialized successfully!
```



As shown in the output, the kubectl config file you need to control the cluster is already on the system. Do this to get it:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```



And now you should be able to run `kubectl cluster-info`.

Finally, if you wish to permit master nodes to run workloads (this is a security vs. resource consumption tradeoff), use:

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```



### Troubleshooting

If `kubeadm init` fails early on due to an error "port is already in use", that's likely because a previous `kubeadm init` attempt wasn't properly cleaned up. Run `kubeadm reset` and try again.

To completely reset the cluster and start over, do this:

```bash
kubeadm reset
rm -rf /etc/systemd/system/kubelet.service.d
apt remove kubelet
```

In general, if things don't seem to be working, use `kubectl get pod --all-namespaces` and see if any pods aren't showing `READY 1/1`. Use a combination of `kubectl describe pod -n kube-system <pod>`, looking at logs, and google to try to diagnose and solve the problem.





## 7. Networking

[This pioneering post](https://www.alibabacloud.com/forum/read-830) seems to have everything we need, but the key part -- the flannel yaml file -- is now returning HTTP 403. Kinda makes you feel like [this](https://xkcd.com/979/).

Luckily, the flannel team already did this for us. First, download the [kube-flannel-aliyun.yaml](https://github.com/coreos/flannel/blob/v0.10.0/Documentation/kube-flannel.yml) file from the version of flannel you want to run, which is probably the latest one. Note that as of 2017-07-12 it's using flannel v0.9.0, because there's no later image available on the Aliyun mirror (the quay.io image mostly works, but in one test I did it took about 10 minutes to pull).

**Important:** In `kube-flannel-aliyun.yaml`, verify that net-conf.json's Network setting matches what you specified in kubeadm.conf for podSubnet.

Next, follow parts of [this document](https://coreos.com/flannel/docs/latest/alicloud-vpc-backend.html), which curiously enough talks about how to run flannel without talking at all about kubernetes. The key parts are that you need to create user accounts that flannel can use (ID and secret), and then flannel needs those Aliyun credentials to change the VPC router's routing tables.

Strangely, the Aliyun flannel config doesn't have a placeholder for account credentials, but the document above tells us where they should go. Add the following under `spec.template.spec.containers` (if there's an existing `env:` section, use it):

```
        env:
        - name: ACCESS_KEY_ID
          value: XXXXYourAccessKey
        - name: ACCESS_KEY_SECRET
          value: XXXXAccessKeySecret
```

**WARNING: This is insecure.** You should never store secrets in plain text; this is only for demonstration purposes.

Next, copy the YAML file to your instance and run `kubectl apply -f kube-flannel-aliyun.yaml`. Wait for a while, and you might need to restart kubelet once (`systemctl restart kubelet`) to make sure everything comes up correctly.

Finally, go into your Aliyun control panel and look at the route table details for the VPC (look the VPC's Routing Tables/路由表, and then click Manage/管理 on the default one). If everything worked, you should see an entry for your master node under subnet 10.24.0.0/24.

Further reading: [this doc](https://www.alibabacloud.com/help/doc-detail/64530.htm?spm=a2c63.p38356.b99.82.71a356aa6GlXQ1), which talks about the different subnets involved when using a VPC.



## 8. Bringing up additional nodes

The instructions here are pretty straightforward:

1. Install Docker
2. Add the aliyun Kubernetes repository and `apt install kubelet==1.10.3-00` or whatever version you wanted.
3. Copy your 10-kubeadm.conf over from the master, and restart kubelet.
4. `apt install kubeadm`
5. On the master, run `kubeadm token create --print-join-command`. Copy the command over to the new node and run it.
6. Wait a bit.

After a while, `kubectl get node` should show two nodes in the `Ready` state.



## 9. Backup and Upgrades

\# TODO

There are two gaping holes in this guide:

- This is a single-master Kubernetes cluster. That's not _terrible_... a master going down just means that new workloads can't be scheduled, not that existing ones break. This might be the right tradeoff for you. However, if you lose the master's disk, etcd configuration goes with it, and the cluster must be rebuilt from scratch. It must be periodically backed up.
- Upgrading isn't addressed at all. This is something that kops does extremely well. Doing it manually on a cluster of modest size shouldn't be terrible -- drain a node, upgrade its components, re-add it to the cluster -- but there's a lot of potential for things to go wrong.
