# k8s China - Pick a Kubernetes version

You'll need to manually install kubelet (the main Kubernetes runtime component that runs on each machine), and that basically has to match the API server version (what you normally think of as the "Kubernetes Version" you're running).

Constraints on version choices include:

- Don't pick a version that has any known security vulnerabilities, such as [CVE-018-1002105](https://github.com/kubernetes/kubernetes/issues/71411). This rules out anything < 1.10.11, for instance.
- kubelet can be at most 1 minor version behind the version of kubeadm you're using. So if you're using kubeadm v1.11.2, you can use kubelet >= v1.10.0, but not v1.9.6 for example.
- kubelet **must not be** ahead of the kubernetes version you're using.  If it is, kubeadm will fail to install.  Accoding to kubeadm: "This is not a supported version skew and may lead to a malfunctional cluster."
- The Aliyun `google-containers` mirror may not have every single Kubernetes version. Mirrored versions are listed [here](https://dev.aliyun.com/detail.html?spm=5176.1972343.2.28.ItEn5B&repoId=91673). Note that there are `google-containers` images and `google_containers` images (with an underscore). Use the underscore one. (They're both provided by Alibaba, but the underscore one has many more versions, because... reasons?)
- Consider choosing one or two versions behind the latest, so you can practice upgrading before deploying a production workload.
