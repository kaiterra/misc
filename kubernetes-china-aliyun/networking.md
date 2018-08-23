# k8s China - Networking

[This pioneering post](https://www.alibabacloud.com/forum/read-830) seems to have everything we need, but the key part -- the flannel yaml file -- is now returning HTTP 403. Kinda makes you feel like [this](https://xkcd.com/979/).

Luckily, the flannel team already did this for us, but [kube-flannel-aliyun.yml](https://github.com/coreos/flannel/blob/v0.10.0/Documentation/kube-flannel-aliyun.yml) runs flannel v0.9.0 which [isn't compatible](https://github.com/coreos/flannel/blob/master/Documentation/troubleshooting.md) with Docker versions 1.13 and later.  You need at least flannel 0.9.1.

Instead, start from [kube-flannel.yml](https://github.com/coreos/flannel/blob/v0.10.0/Documentation/kube-flannel.yml) and make these changes:

```bash
sed -i -e 's/        "Type": "vxlan"/        "Type": "ali-vpc"/g' kube-flannel.yml

sed -i -e "s/        image: .*/        image: registry.cn-hangzhou.aliyuncs.com\/kubernetes_containers\/flannel:v0.10.0-amd64/g" kube-flannel.yml
```

You may also remove the non-amd64 configs if you want, but they won't hurt anything.

Strangely, even the Aliyun flannel config doesn't have a placeholder for account credentials, but we can get clues from [this document](https://coreos.com/flannel/docs/latest/alicloud-vpc-backend.html), which curiously enough talks about how to run flannel without talking at all about kubernetes.

Under the `kube-flannel` container's spec, simply add a reference to the secret we created before:

```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
# ... ...
spec:
  template:
    # ... ... 
    spec:
      # ... ... 
      initContainers:
        # ... ...
      containers:
      - name: kube-flannel
        # ... ...
        # ADD THIS
        envFrom:
        - secretRef:
            name: k8s-aliyun
```

Now, apply that file using `kubectl apply -f kube-flannel.yml`. Wait for a while, and you might need to restart kubelet once (`systemctl restart kubelet`) to make sure everything comes up correctly.

Finally, go into your Aliyun control panel and look at the route table details for the VPC (look the VPC's Routing Tables/路由表, and then click Manage/管理 on the default one). If everything worked, you should see an entry for your master node under subnet 10.244.0.0/24.

Further reading: [this doc](https://www.alibabacloud.com/help/doc-detail/64530.htm?spm=a2c63.p38356.b99.82.71a356aa6GlXQ1), which talks about the different subnets involved when using a VPC.
