# k8s China - Join an existing cluster

On the master, get the join token:

```bash
kubeadm token create --print-join-command
```

> Note: this command requires ~/.kube/config to be a valid kubectl configuration that points to the current cluster.  If it's not (perhaps from a previous build of the cluster), you'll need to copy it from `/etc/kubernetes/admin.conf`.

Obviously, copy the command over to the new node and run it.  If successful, you'll see some text liks this:

```
This node has joined the cluster:
* Certificate signing request was sent to master and a response
  was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
```

However, the custom kubelet settings you gave the master weren't inherited.  You'll need to copy those over manually.

Locate /var/lib/kubelet/kubeadm-flags.env, and copy that file to the same location on the new worker (it consists of a single line that sets `KUBELET_KUBEADM_ARGS`).  Then:

```bash
systemctl daemon-reload && systemctl restart kubelet
```

After a while, `kubectl get node` should show two nodes in the `Ready` state.
