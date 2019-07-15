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

Finally, if you wish to permit master nodes to run workloads, use:

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

Should you do this? If your master is oversized and cluster resources are scarce, consider it.  The danger is that it's possible for pods scheduled to the master node to overload it and bring it down.  This is bad enough if it happens on a normal node, but worse if it happens on the master node.


### Troubleshooting

If `kubeadm init` fails early on due to an error "port is already in use", that's likely because a previous `kubeadm init` attempt wasn't properly cleaned up. Run `kubeadm reset` and try again.

If you've done that, you can also try re-running kubeadm without first doing a reset:

```bash
kubeadm init --config kubeadm.conf --ignore-preflight-errors=all
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
