# k8s China - Install Docker

17.03 is the [latest supported version](https://kubernetes.io/docs/setup/independent/install-kubeadm/#installing-docker) with kubeadm right now, but as of 2018-07-12 Aliyun's own K8s service was using 17.06.

Docker Community Edition releases technically only have 4 months of support, and at some point they'll age out of the Ubuntu repos entirely, so choose wisely.

Here's a summary of the [official installation instructions](https://docker.github.io/engine/installation/linux/ubuntu/#install-using-the-repository):

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -

# TODO: Call sudo apt-key fingerprint to verify the key you just added.  Compare
# it with the ones from the official instructions, since this is the kind of
# thing for which you shouldn't trust a third-party guide like this one.
 
# Add the stable repository
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
apt update

# Find out what versions there are
apt-cache madison docker-ce

# Pick a version
apt install docker-ce=17.03.2~ce-0~ubuntu-xenial
```

**Important:** Configure log rotation so logs don't fill up your hard disk, and enable the Docker China mirror:

```bash
tee /etc/docker/daemon.json <<"EOF"
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5"
  },
  "registry-mirrors": ["https://registry.docker-cn.com"]
}
EOF

systemctl daemon-reload && systemctl restart docker
```
