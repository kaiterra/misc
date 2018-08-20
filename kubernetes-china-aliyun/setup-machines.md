# k8s China - Setup Machines

The storage driver used in this guide requires the node name to be of the form `regionId.instanceId`; for example: `cn-hangzhou.i-bp12gei4ljuzilgwzahc`.  While this is inconvenient, if you know about it while you're still setting up your cluster, not much extra work is required.

Aliyun automatically sets the machine's hostname to something like its instance ID (`i-[a-z0-9]+`), replacing the dash after the 'i' with the character 'Z', and sometimes adding a final 'Z'.  This doesn't make any sense so we'll fix it.

Changes made with `hostnamectl` don't always stick after a reboot, so to be safe we'll edit `/etc/hostname` directly:

```bash
echo 'cn-hangzhou.i-bp12gei4ljuzilgwzahc' | tee /etc/hostname

# Add the above string in the obvious place to /etc/hosts
nano /etc/hosts

# Ensure your settings aren't getting overwritten on boot
reboot now

# When the machine comes back up, verify that these match:
hostname
hostname -f
```

Once this is done, the storage driver will be able to work properly.
