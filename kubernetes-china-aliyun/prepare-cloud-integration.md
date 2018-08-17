# k8s China - Prepare for cloud integration

To allow kubernetes to automatically manage your networking, storage and other resources, you'll need to create accounts in RAM (访问控制) with the right access.  While you could create an account with full permissions, that isn't as safe as only giving the minimum necessary permissions.

Here, we'll assume you create one account with all the necessary permissions.

### Networking

To allow flannel, the networking plugin we're going to install, to change the VPC router's routing tables, add the `AliyunVPCFullAccess` policy.

### Storage

To use the disk attach functionality, you'll need to add a policy that looks like this:

```json
{
  "Version": "1",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
		"ecs:DescribeDisks",
		"ecs:DetachDisk",
		"ecs:AttachDisk"
      ],
      "Resource": [
        "acs:ecs:*:*:instance/*",
		"acs:ecs:*:*:disk/*"
      ]
    }
  ]
}
```

To allow OSS buckets to be mounted as disks, use this policy:

```json
{
  "Version": "1",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "oss:ListObjects",
        "oss:GetObject",
        "oss:PutObject",
        "oss:DeleteObject",
        "oss:PutObjectAcl",
        "oss:GetObjectAcl"
      ],
      "Resource": [
        "acs:oss:*:*:*",
        "acs:oss:*:*:*/*"
      ]
    }
  ]
}
```

Consider removing DeleteObject if you're only using this for backup and restore.  Instead, set a retention policy on OSS buckets to clean up old files.

### Create secrets

To allow services to use your account credentials, create an .env file (don't check it into git!) and use kubectl to create the secret:

```bash
tee k8s-aliyun.env <<"EOF"
ACCESS_KEY_ID=XXXXYourAccessKeyID
ACCESS_KEY_SECRET=XXXXYourAccessKeySecret
EOF

@kubectl -n kube-system create secret generic k8s-aliyun \
	--from-env-file=k8s-aliyun.env \
	--dry-run -o yaml | kubectl apply -f -
```
