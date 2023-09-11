# Test rollout-rollback-rollout

## Configure Vagrant
(*tested on Fedora 35+ host with kvm*)

### Configure libvirt provisioner on your host
Follow this guide: https://developer.fedoraproject.org/tools/vagrant/vagrant-libvirt.html

### Configure your host to allow NFS file sharing to VM

Follow this guide: https://developer.fedoraproject.org/tools/vagrant/vagrant-nfs.htm

## Set initial kubeadm version

We need an initial version of kubeadm and kubelet from which we will upgrade later manually. This is controlled by env vars on your local machine that Vagrantfile will use for installing packages.

### Determine what versions are available

#### (optional) add Kubernetes repo if package is not available on your local machine:

```bash
$ sudo dnf config-manager --add-repo=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
```

#### Check available package versions

```bash
$ yum search kubelet --showduplicates
$ yum search kubeadm --showduplicates
```

### Set env vars with package versions

If versions don't match kubeadm preflight check will fail. When running kubeadm manually this can be overridden by `--ignore-preflight-errors=...`

```bash
$ export KUBELET_PACKAGE=kubelet-1.24.5-0
$ export KUBEADM_PACKAGE=kubeadm-1.24.5-0
```

## Provision VM with Vagrant

```bash
$ git clone https://github.com/RomanBednar/vagrant.git
$ cd fedora/kubeadm-cluster
$ vagrant up && vagrant ssh
```

#### (optional) Troubleshoot potential bugs

See [Known issues](https://github.com/RomanBednar/vagrant/tree/master/fedora/kubeadm-cluster#known-issues)

## (example) Perform pre-upgrade tests

Set default storage class:
```bash
$ kc patch sc/csi-hostpath-sc -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
storageclass.storage.k8s.io/csi-hostpath-sc patched
```

PVC does not get updated and remains pending:
```bash
$ kc get pvc
NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
csi-pvc   Pending                              
```


## Upgrade cluster

Check available 1.25 versions:
```bash
$ yum search kubeadm --showduplicates --quiet | grep 1.25
kubeadm-1.25.0-0.x86_64 : Command-line utility for administering a Kubernetes cluster.
kubeadm-1.25.1-0.x86_64 : Command-line utility for administering a Kubernetes cluster.
kubeadm-1.25.2-0.x86_64 : Command-line utility for administering a Kubernetes cluster.
```

Install/update kubeadm:
```bash
$ sudo yum install -y kubeadm-1.25.2-0
```

Check our config file that enables FeatureGate (examples should be already mounted):
```bash
$ cat /mnt/clusterconf-examples/featuregate.yaml 
## Example kubeadm configuration for enabling a feature gate.
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
apiServer:
  extraArgs:
    feature-gates: RetroactiveDefaultStorageClass=true
controllerManager:
  extraArgs:
    cluster-cidr: 10.244.0.0/16
    feature-gates: RetroactiveDefaultStorageClass=true
```

Perform kubeadm upgrade:
```bash
$ sudo kubeadm upgrade plan --config /mnt/clusterconf-examples/featuregate.yaml
$ sudo kubeadm upgrade apply --config /mnt/clusterconf-examples/featuregate.yaml v1.25.2
```

Perform kubelet upgrade:
```bash
$ sudo yum install -y kubelet-1.25.2-0
$ sudo systemctl daemon-reload 
$ sudo systemctl restart kubelet
```


## (example) Perform post-upgrade tests

Verify that PVC got SC assigned right after upgrade and PV was provisioned and bound:
```bash
$ kc get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
csi-pvc   Bound    pvc-06a964ca-f997-4780-8627-b5c3bf5a87d8   1Gi        RWO            csi-hostpath-sc   87m

```

## Downgrade cluster

```bash
$ yum history | grep -E "kubeadm|kubelet"
    10 | install -y kubelet-1.25.2-0                                                                                                                                                                                                                                       | 2022-10-12 11:06 | Upgrade        |    1   
     8 | install -y kubeadm-1.25.2-0                                                                                                                                                                                                                                       | 2022-10-12 09:45 | Upgrade        |    1   
     7 | install -y kubelet-1.24.5-0 kubeadm-1.24.5-0 kubectl

$ sudo yum -y history undo 8 && sudo yum -y history undo 10
```

## (example) Perform post-rollback tests

Remove default SC:
```bash
$ kc patch sc/csi-hostpath-sc -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
storageclass.storage.k8s.io/csi-hostpath-sc patched
```

Create new PVC without SC:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-pvc-2
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```bash
$ kc get pvc
NAME        STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
csi-pvc     Bound     pvc-06a964ca-f997-4780-8627-b5c3bf5a87d8   1Gi        RWO            csi-hostpath-sc   96m
csi-pvc-2   Pending     
```

Add default SC again:
```bash
$ kc patch sc/csi-hostpath-sc -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
storageclass.storage.k8s.io/csi-hostpath-sc patched
```

Verify that the new PVC did not get updated with SC this time:
```bash
$ kc get pvc/csi-pvc-2
NAME        STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
csi-pvc-2   Pending    
```

## Upgrade cluster again

Install/update kubeadm:
```bash
$ sudo yum install -y kubeadm-1.25.2-0
```

Perform kubeadm upgrade:
```bash
$ sudo kubeadm upgrade plan --config /mnt/clusterconf-examples/featuregate.yaml
$ sudo kubeadm upgrade apply --config /mnt/clusterconf-examples/featuregate.yaml v1.25.2
```

Perform kubelet upgrade:
```bash
$ sudo yum install -y kubelet-1.25.2-0
$ sudo systemctl daemon-reload 
$ sudo systemctl restart kubelet
```

## (example) Perform post-upgrade tests again

Verify that PVC got SC assigned right after upgrade and PV was provisioned and bound:
```bash
$ kc get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
csi-pvc   Bound    pvc-06a964ca-f997-4780-8627-b5c3bf5a87d8   1Gi        RWO            csi-hostpath-sc   117m

```

---

# Known issues

## Flannel pods failing to start

Versions affected:
```bash
[vagrant@fedora ~]$ kc version
WARNING: This version information is deprecated and will be replaced with the output from kubectl version --short.  Use --output=yaml|json to get the full version.
Client Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.2", GitCommit:"5835544ca568b757a8ecae5c153f317e5736700e", GitTreeState:"clean", BuildDate:"2022-09-21T14:33:49Z", GoVersion:"go1.19.1", Compiler:"gc", Platform:"linux/amd64"}
Kustomize Version: v4.5.7
Server Version: version.Info{Major:"1", Minor:"24", GitVersion:"v1.24.6", GitCommit:"b39bf148cd654599a52e867485c02c4f9d28b312", GitTreeState:"clean", BuildDate:"2022-09-21T13:12:04Z", GoVersion:"go1.18.6", Compiler:"gc", Platform:"linux/amd64"}
[vagrant@fedora ~]$ uname -ar
Linux fedora 5.14.10-300.fc35.x86_64 #1 SMP Thu Oct 7 20:48:44 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
```

`$ kc -n kube-flannel logs pod/kube-flannel-ds-jrbpg`

Contains error:
`Error registering network: failed to acquire lease: node "fedora" pod cidr not assigned`

Solution:
```bash
$ sudo cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep -i cluster-cidr
$ kubectl patch node <node> -p '{"spec":{"podCIDR":"<your-cidr>"}}'
```

Source: https://stackoverflow.com/a/58618952

However, if that happens CNI network has already used wrong IP address and pod scheduling will fail (for example csi-hostpath driver)

`$ kc describe pod/csi-hostpathplugin-0`

Contains error:
`Failed to create pod sandbox: rpc error: code = Unknown desc = failed to create pod network sandbox k8s_csi-hostpathplugin-0_default_dae0b06f-47f5-43d1-8852-ac743cc46497_0(6552d7fb0b200c0f0b0255a4e5cc992d3d132acfa67cc4f280f1843a3dc0c400): error adding pod default_csi-hostpathplugin-0 to CNI network "cbr0": failed to delegate add: failed to set bridge addr: "cni0" already has an IP address different from 10.244.0.1/16`

Solution
```bash
$ sudo ip link set cni0 down && sudo ip link set flannel.1 down 
$ sudo ip link delete cni0 && sudo ip link delete flannel.1
$ sudo systemctl restart crio && sudo systemctl restart kubelet
$ kc delete pod/csi-hostpathplugin-0
```

Source: https://stackoverflow.com/a/61544151

## Flannel failing to create pod network

`Warning  FailedCreatePodSandBox  5s    kubelet            Failed to create pod sandbox: rpc error: code = Unknown desc = failed to create pod network sandbox k8s_csi-hostpathplugin-0_default_e5347532-bb23-4aff-b179-9a6cffc4c8be_0(d7598d6197c3b5903bed879991f92bd1c923c99ebe900a5bf72f100e27a1aaae): error adding pod default_csi-hostpathplugin-0 to CNI network "cbr0": loadFlannelSubnetEnv failed: open /run/flannel/subnet.env: no such file or directory`

Fix it by manually adding the file:

`/run/flannel/subnet.env`

```
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```

Source: https://github.com/kubernetes/kubernetes/issues/70202#issuecomment-481173403