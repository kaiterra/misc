# k8s China - Run `kubeadm init`

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


#### kubelet configuration

Ensure the file `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf` exists and contains a line like this:

```
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
```

If not, get the appropriate one from [github](https://github.com/kubernetes/kubernetes/tree/b930d7f9153f1c74a8927de3f061a448c2a5a98c/build/debs), then:

```bash
systemctl daemon-reload && systemctl restart kubelet
```
