# k8s China - Setup Machines

Make sure they're all in the same VPC (专有网络).  The CIDR address block used for the VPC's network (目标网段; often something like 172.17.0.0/16 or so) doesn't seem to be important.

This guide is for Ubuntu 16.04, but any [supported](https://success.docker.com/article/compatibility-matrix) Docker OS should work, though CoreOS is apparently the new hotness when it comes to hosting containerized workloads.
