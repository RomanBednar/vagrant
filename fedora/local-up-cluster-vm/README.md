# How to run local Kubernetes cluster using Vagrant

## 1. Configure Vagrant 
(*tested on Fedora 35+ host with kvm*)

### Configure libvirt provisioner on your host
Follow this guide: https://developer.fedoraproject.org/tools/vagrant/vagrant-libvirt.html

### Configure your host to allow NFS file sharing to VM

Follow this guide: https://developer.fedoraproject.org/tools/vagrant/vagrant-nfs.html

## 2. Provision VM with Vagrant (requires sudo for NFS export configuration)
```
$ sudo vagrant up
```
## 3. Go to kubernetes source dir
Vagrant will mount your home directory under /mnt/home so make sure you have 
Kubernetes source code somewhere in home, then cd to this dir.
```
$ vagrant ssh
$ cd /mnt/home/<kube_dir>
```

## 4. Install etcd and add it to $PATH (the script will provide path to etcd)
```
$ ./hack/install-etcd.sh

$ export PATH="<path_to_etcd>:${PATH}"
```

## 5. Clean up any leftovers from previous run
```
$ sudo rm -rf /tmp/kube* /var/run/kubernetes/* /var/lib/kubelet/*
```

## 6. (optional) Unmount devices if you see "Device or resource busy" errors
```
$ sudo umount /var/lib/kubelet/pods/<PATH_TO_BUSY_DEVICE>
```
## 7. (optional) Enable custom FeatureGate
```
$ export FEATURE_GATES="RetroactiveDefaultStorageClass=true"
```

## 8. Start Kubernetes cluster
```
$ ALLOW_PRIVILEGED=true LOG_LEVEL=5 ./hack/local-up-cluster.sh
```


## How to obtain metrics

#### Example - test a metric that is emitted by PV controller which runs in KCM.

1. Find what port is KCM listening on

```
$ ss -tpln  | grep kube-controller
LISTEN 0      4096               *:10257            *:*    users:(("kube-controller",pid=39457,fd=7))
export PORT=10257
```

2. Verify TLS certificate files were created by local-up-cluster.sh at default location

```
ls /var/run/kubernetes/client-admin.crt /var/run/kubernetes/client-admin.key
```

3. Send a get request to metrics endpoint

```
curl -k --cert /var/run/kubernetes/client-admin.crt --key /var/run/kubernetes/client-admin.key https://localhost:$PORT/metrics
```

