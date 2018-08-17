# k8s China - Install Docker

> The only requirement for Kubernetes is a supported container runtime. For example, rkt will also work. But that's beyond the scope of this document.

17.03 is the [latest supported version](https://kubernetes.io/docs/tasks/tools/install-kubeadm/#installing-docker) with kubeadm right now, but as of 2018-07-12 Aliyun's own K8s service was using 17.06.

Docker Community Edition releases technically only have 4 months of support, and at some point they'll age out of the Ubuntu repos entirely, so choose wisely.

Here's a summary of the [official installation instructions](https://docker.github.io/engine/installation/linux/ubuntu/#install-using-the-repository):

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -

# TODO: Call sudo apt-key fingerprint.  Get this from the official instructions;
# this is the kind of thing for which you shouldn't trust a third-party site.
 
# Add the stable repository
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
apt update

# Find out what versions there are
apt-cache madison docker-ce

# Pick a version
apt install docker-ce=18.03.1~ce-0~ubuntu-xenial
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

sudo systemctl daemon-reload && sudo systemctl restart docker
```
