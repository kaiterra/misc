# k8s China - Storage

This guide uses the [AliyunContainerService flexVolume driver](https://github.com/AliyunContainerService/flexvolume).  I decided to use it because it's "official" and it supports mounting of OSS buckets to the filesystem.

There are other options out there:

- [kubeup/kube-aliyun](https://github.com/kubeup/kube-aliyun) has support for other cloud integration points like ELB and dynamic disk provisioning.
- [pragkent/aliyun-disk](https://github.com/pragkent/aliyun-disk) isn't nearly as widely used.

`AliyunContainerService/flexvolume` has a couple drawbacks:

- It doesn't support dynamic provisioning.  That is, you must manually create a volume in ECS, and then manually create a resource for it in k8s.
- It requires your nodes' hostnames to follow a specific format.  See [k8s China - Setup Machines](setup-machines.md) for details.
- It uses old-style kubelet-controlled attach and detach (this is the `enable-controller-attach-detach: "false"` setting in kubelet). This means that disks attached to nodes that go down, can't be detached until the node comes back up.

### Installing the plugin

Assuming you've already created a secret called `k8s-aliyun` with your Aliyun credentials, this is rather straightfoward:

```yaml
apiVersion: apps/v1 # for versions before 1.8.0 use extensions/v1beta1
kind: DaemonSet
metadata:
  name: flexvolume
  namespace: kube-system
  labels:
    k8s-volume: flexvolume
spec:
  selector:
    matchLabels:
      name: acs-flexvolume
  template:
    metadata:
      labels:
        name: acs-flexvolume
    spec:
      hostPID: true
      hostNetwork: true
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: acs-flexvolume
        image: registry-vpc.cn-beijing.aliyuncs.com/acs/flexvolume:v1.10.4-5f4bbef
        securityContext:
          privileged: true
        env:
        - name: ACS_DISK
          value: "true"
        - name: ACS_NAS
          value: "false"
        - name: ACS_OSS
          value: "true"
        - name: SLB_ENDPOINT
          value: ""
          # This is the only value that works here.  If you try to use the default (empty string),
          # you get InvalidVersion.  If you try to use the VPC URL, you get InvalidVersion.
          # ecs-cn-beijing.aliyuncs.com doesn't exist.
        - name: ECS_ENDPOINT
          value: "https://ecs-cn-hangzhou.aliyuncs.com"
        envFrom:
        - secretRef:
            name: k8s-aliyun
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        # flexVolume plugins must be installed to the host OS of every node in the
        # cluster. This plugin uses the common trick of being deployed as a DaemonSet
        # so that it's automatically installed whenever a new node joins the cluster.
        # This requires that /etc and /usr be mounted into the container.
        - name: usrdir
          mountPath: /host/usr/
        - name: etcdir
          mountPath: /host/etc/
        - name: logdir
          mountPath: /var/log/alicloud/
      volumes:
      - name: usrdir
        hostPath:
          path: /usr/
      - name: etcdir
        hostPath:
          path: /etc/
      - name: logdir
        hostPath:
          path: /var/log/alicloud/
```


### Using the plugin

Storage classes, labels, etc. aren't strictly necessary.  You can even skip creating PersistentVolumes entirely and just use the Volume field of a StatefulSet (or even Deployment).  However, that won't get you very far because it doesn't allow you to create StatefulSets with more than one replica -- there's no way to address all the individual disks involved.

So, to allow a database like PostgreSQL to run as a StatefulSet, you'll need to declare a StorageClass and a PersistentVolume, as below.  We're abusing k8s here in a couple ways:

- To account for the fact that a PersistentVolume may be resized, and to avoid having to duplicate its size in mulitple places, my convention is to set all storage to 1Gi, and use StorageClassName to select which PersistentVolumes to use with a particular service.
- StorageClasses are generally for dynamic provisioning, so we're abusing them here.  The provisioner, no-provisioning, is simply a dummy label to remind k8s that it shouldn't expect to be able to automatically provision volumes unless it knows about a plugin called "no-provisioning", which obviously doesn't exist.

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: postgres
  labels:
    kubernetes.io/cluster-service: "true"
reclaimPolicy: Retain
provisioner: no-provisioning  # Any string here works, as long as it doesn't start with kubernetes.io/

---

kind: PersistentVolume
apiVersion: v1
metadata:
  # For alicloud/disk, this MUST match the volumeId below, or the disk can't be detached
  name: "d-xxxxxxxxxxxxx3434f"
spec:
  storageClassName: postgres
  capacity:
    # This is unused; selection is by storageClassName
    storage: 1Gi
  persistentVolumeReclaimPolicy: Retain
  accessModes:
  - ReadWriteOnce
  - ReadOnlyMany
  flexVolume:
    driver: "alicloud/disk"
    fsType: "ext4"
    options:
      volumeId: "d-xxxxxxxxxxxxx3434f"

```

Then, in the StatefulSet, you'll create a ClaimTemplate as you would in any other cloud provider that supports dynamic provisioning:

```yaml
  volumeClaimTemplates:
  - metadata:
      name: disk
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: postgres
      resources:
        requests:
          storage: 1Gi
```

## Disk Resizing

Auto-resizing is an alpha feature in 1.11, but flexVolumes aren't supported (not that our flexVolume driver would suport it anyway), so we're out of luck there.  Luckily, however, manual resizing isn't hard.

Basically, the steps are:

1. Resize the volume
2. Restart the service
3. Resize the file system

#### Resize the volume

Normally this is done in the Aliyun console. The resize will complete, but the OS won't recognize it until the disk is detached and re-attached.

#### Restart the service

For StatefulSets, it's enough to schedule some sort of rolling update -- once all the containers in the pod have terminated, the disk will get detached; when the pod is restarted, the disk will be re-attached and the new size will appear.

Throughout all this, the StatefulSet's PVCs will remain bound to their PersistentVolumes.

#### Resize the file system

Once the container is up and running, locate the block device from which the filesystem was mounted, and run:

```bash
resize2fs /dev/vdb
```

or whatever block device the volume is mounted from.  This performs an online resize of the file system.
