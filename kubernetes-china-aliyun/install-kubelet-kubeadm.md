# k8s China - Install kubelet and kubeadm

## Add Aliyun's kubernetes package mirror

The [official instructions](https://kubernetes.io/docs/tasks/tools/install-kubeadm) tell you to add the packages.cloud.google.com repo, but that's obviously not going to work. Luckily, Aliyun has a mirror:

```bash
apt update && apt install -y apt-transport-https
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
tee /etc/apt/sources.list.d/kubernetes.list <<EOF
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt update
```

## Install kubelet

Use the version of kubelet required by your Kubernetes version selection.

> Note: It is important to pin this to a specific version; otherwise, when a new version is released, it will silently break your cluster in mysterious ways.

```bash
apt-cache madison kubelet

# Pick one, then:
apt install kubelet=1.10.3-00   # this depends on the k8s version you want to install
```


## Install Kubeadm

You should use at least 1.11.0-00 because `kubeadm config print-default` and the imageRepository mirroring functionality depends on that.

```bash
# Ensure >= 1.11.0-00 is available
apt-cache madison kubeadm

apt install kubeadm
```


## Get initial kubeadm config

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

# This is your chosen Kubernetes version
sed -i "s/kubernetesVersion: .*/kubernetesVersion: v1.10.3/g" kubeadm.conf
```
