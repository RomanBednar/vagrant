# Vagrant configuration

## Installation

```
sudo dnf install qemu-kvm libvirt libguestfs-tools virt-install rsync
sudo systemctl enable --now libvirtd
sudo dnf install vagrant
sudo vagrant plugin install vagrant-libvirt
```

## Troubleshooting

### 1) Sometimes vagrant does not choose provider correctly, to override use:

```
$ export VAGRANT_DEFAULT_PROVIDER=libvirt
```

### 2) Vagrant fails to list networks

When `vagrant status` of `vagrant ssh` throws this error:
```
/usr/share/vagrant/gems/gems/vagrant-libvirt-0.7.0/lib/vagrant-libvirt/driver.rb:158:in `list_all_networks': Call to virConnectListAllNetworks failed: Failed to connect socket to '/var/run/libvirt/virtnetworkd-sock-ro': No such file or directory (Libvirt::RetrieveError)
```

Start virtnetworkd manually:
```
$ sudo systemctl enable --now virtnetworkd
```