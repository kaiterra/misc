# k8s China - Networking

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

