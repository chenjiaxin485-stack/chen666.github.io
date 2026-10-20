### root密码，更新，nano下载，ssh连接
sudo passwd root
sudo apt update
apt install nano -y
sudo nano /etc/ssh/sshd_config
PermitRootLogin yes
sudo systemctl restart ssh

### 主机名，关闭分区，永久分区，内核配置，时区
sudo hostnamectl set-hostname k8s-master
sudo swapoff -a
sudo sed -i '/\/swap.img/s/^/#/' /etc/fstab

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```
timedatectl set-timezone Asia/Shanghai


### ip地址，ping
nano /etc/netplan/50-cloud-init.yaml
```
network:
  version: 2
  ethernets:
    enp2s0:
      dhcp4: no
      addresses: [192.168.114.145/24]
      gateway4: 192.168.114.2
      nameservers:
        addresses: [8.8.8.8,114.114.114.114]
```
netplan apply


apt update 
apt install -y inetutils-ping