# k8s China - Join an existing cluster

First, copy your `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf` file over from the master node.  Update the `--hostname-override` to match the node's hostname.

On the master, get the join token:

```bash
kubeadm token create --print-join-command
```

Obviously, copy the command over to the new node and run it.

After a while, `kubectl get node` should show two nodes in the `Ready` state.
