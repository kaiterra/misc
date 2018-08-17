# k8s China - Networking

[This pioneering post](https://www.alibabacloud.com/forum/read-830) seems to have everything we need, but the key part -- the flannel yaml file -- is now returning HTTP 403. Kinda makes you feel like [this](https://xkcd.com/979/).

Luckily, the flannel team already did this for us. First, download the [kube-flannel-aliyun.yaml](https://github.com/coreos/flannel/blob/v0.10.0/Documentation/kube-flannel-aliyun.yml) file from the version of flannel you want to run, which is probably the latest one. Note that v0.10.0 uses flannel v0.9.0, because there's no later image available on the Aliyun mirror (the quay.io image took about 10 minutes to pull, but it did eventually work).

**Important:** In `kube-flannel-aliyun.yaml`, verify that net-conf.json's Network setting matches what you specified in kubeadm.conf for podSubnet.

Strangely, the Aliyun flannel config doesn't have a placeholder for account credentials, but we can get clues from [this document](https://coreos.com/flannel/docs/latest/alicloud-vpc-backend.html), which curiously enough talks about how to run flannel without talking at all about kubernetes.

Under the `kube-flannel` container's spec, simply add a reference to the secret we created before:

```yaml
        envFrom:
        - secretRef:
            name: k8s-aliyun
```

Now, apply that file using `kubectl apply -f kube-flannel-aliyun.yaml`. Wait for a while, and you might need to restart kubelet once (`systemctl restart kubelet`) to make sure everything comes up correctly.

Finally, go into your Aliyun control panel and look at the route table details for the VPC (look the VPC's Routing Tables/路由表, and then click Manage/管理 on the default one). If everything worked, you should see an entry for your master node under subnet 10.24.0.0/24.

Further reading: [this doc](https://www.alibabacloud.com/help/doc-detail/64530.htm?spm=a2c63.p38356.b99.82.71a356aa6GlXQ1), which talks about the different subnets involved when using a VPC.

