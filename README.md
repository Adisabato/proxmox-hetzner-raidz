## Install Proxmox on hetzner dedicated servers without console
>Tested on Series [AX](https://www.hetzner.com/dedicated-rootserver/matrix-ax) , [EX](https://www.hetzner.com/dedicated-rootserver/matrix-ex) , [SX](https://www.hetzner.com/dedicated-rootserver/matrix-sx)

<img src="https://github.com/ariadata/proxmox-hetzner/raw/main/files/icons/proxmox.png" alt="Proxmox" height="48" /> <img src="https://github.com/ariadata/proxmox-hetzner/raw/main/files/icons/hetzner.png" alt="Hetzner" height="38" /> 

![](https://img.shields.io/github/stars/ariadata/proxmox-hetzner.svg)
![](https://img.shields.io/github/watchers/ariadata/proxmox-hetzner.svg)
![](https://img.shields.io/github/forks/ariadata/proxmox-hetzner.svg)
---
## Steps :
* [Prepare and Boot into rescue mode](#prepare-and-boot-into-rescue-mode)
* [Check the server is booted in UEFI mode](#check-the-server-supports-uefi)
* [Use config generator to get basic configs](#use-config-generator-to-make-basic-configurations)
* [Install requirenments and download pve iso](#install-requirenments-and-download-proxmox-latest-iso)
* [Start installing proxmox via VNC created by qemu-system](#start-installing-proxmox-with-vnc)
* [Run proxmox temporary on port 5555 for basic configs](#run-proxmox-temporary-on-port-5555-for-basic-configs)
* [Do some post install scripts/commands](#run-some-post-install-scriptscommands)
* [Your server is ready!](#your-server-is-ready-to-use)
* [Useful links](#useful-links)

## Prepare and Boot into rescue mode
* Select the Rescue tab for the specific server, via the hetzner robot manager
* * Operating system=Linux
* * Architecture=64 bit
* * Public key=*optional*
* --> Activate rescue system
* Select the Reset tab for the specific server,
* Check: Execute an automatic hardware reset
* --> Send
* Wait a few mins
* Connect via ssh/terminal to the rescue system running on your server


## Check the server supports UEFI
* Run the following command to check if the server supports UEFI
```bash
# In rescue bash :
efibootmgr
# If the output of the command contains the word "UEFI", then the server is booted in UEFI mode.
```

## Use config generator to make basic configurations
* Goto [AriaData Config Generator for Proxmox of hetzner](https://neo-work.ariadata.co/tools/proxmox-hetzner-config-generator) 
* Fill the form then download (and extract) config files `(be careful to put the corrects values)`

## Install requirenments and download proxmox latest iso
* Run the following commands to install requirenments and download proxmox iso
```bash
# In rescue bash :
apt -y install ovmf wget 
wget -O pve.iso http://download.proxmox.com/iso/proxmox-ve_7.4-1.iso
# you can check the latest one on http://download.proxmox.com/iso/
```

## Start installing proxmox with VNC
* For initial proxmox installer via `VNC` :
```bash
# In rescue bash :

#### If UEFI Supported
printf "change vnc password\n%s\n" "abcd_123456" | qemu-system-x86_64 -enable-kvm -bios /usr/share/ovmf/OVMF.fd -cpu host -smp 4 -m 4096 -boot d -cdrom ./pve.iso -drive file=/dev/nvme0n1,format=raw,media=disk,if=virtio -drive file=/dev/nvme1n1,format=raw,media=disk,if=virtio -vnc :0,password -monitor stdio -no-reboot

#### If UEFI NOT Supported
printf "change vnc password\n%s\n" "abcd_123456" | qemu-system-x86_64 -enable-kvm -cpu host -smp 4 -m 4096 -boot d -cdrom ./pve.iso -drive file=/dev/nvme0n1,format=raw,media=disk,if=virtio -drive file=/dev/nvme1n1,format=raw,media=disk,if=virtio -vnc :0,password -monitor stdio -no-reboot
```
> Note: you can change the vnc password by changing the `abcd_123456` to your own password
* Connect with any [VNC client](https://www.google.com/search?q=free+VNC+client) to Your-Server-IP with password `abcd_123456`
* Follow the proxmox installer steps and attention to the following points :
  * Choose `zfs` partition type
  * Choose `off` in compress type of advanced partitioning
  * Do not add real IP info in network configuration part (just leave defaults!)
  * Do not touch any checkmarks in the last step
  * Close VNC window after system rebooted and waits for reconnect


## Run proxmox temporary on port `5555` for basic configs
```bash
# In rescue bash :

#### If UEFI Supported
qemu-system-x86_64 -enable-kvm -bios /usr/share/ovmf/OVMF.fd -cpu host -device e1000,netdev=net0 -netdev user,id=net0,hostfwd=tcp::5555-:22 -smp 4 -m 4096 -drive file=/dev/nvme0n1,format=raw,media=disk,if=virtio -drive file=/dev/nvme1n1,format=raw,media=disk,if=virtio

#### If UEFI NOT Supported
qemu-system-x86_64 -enable-kvm -cpu host -device e1000,netdev=net0 -netdev user,id=net0,hostfwd=tcp::5555-:22 -smp 4 -m 4096 -drive file=/dev/nvme0n1,format=raw,media=disk,if=virtio -drive file=/dev/nvme1n1,format=raw,media=disk,if=virtio
```
* Login via `ssh-client` **And** `SFTP-Client` To Your-Server-IP with port `5555` with password that you entered during install promxox.
* Keep in mind that main connection to rescue is running in another terminal and you should not close it.
* Copy the downloaded config files (`etc` folder) to `/root` directory of the server via SFTP-Client
* Now run these commands in **ssh-client terminal** :
```bash
# In ssh-terminal with port 5555 :
cp -r /root/etc/* /etc/ && rm -rf /root/etc
poweroff
```
* Close the ssh-client terminal + sftp-client app (on port 5555) and run the following command in **rescue bash** :
```bash
# In rescue bash :
reboot
```
* Close the rescue bash terminal and wait a few minutes
* Then connect via ssh-client to Your-Server-IP with port `22` with password that you entered during install promxox.

## Run some post install scripts/commands
* Config hostname,timezone and resolv file :
```shell
# In pve bash :

# Change Timezone
timedatectl set-timezone Europe/Istanbul

# Change DNS servers
printf "nameserver 1.1.1.1\nnameserver 2606:4700:4700::1111\n" > /etc/resolv.conf

systemctl disable --now rpcbind rpcbind.socket

sed -i 's/^\([^#].*\)/# \1/g' /etc/apt/sources.list.d/pve-enterprise.list

echo "deb [arch=amd64] http://download.proxmox.com/debian/pve bullseye pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription-repo.list

sed -i "s|ftp.*.debian.org|ftp.debian.org|g" /etc/apt/sources.list

apt update && apt -y upgrade && apt -y autoremove

pveupgrade

pveam update

apt install -y curl libguestfs-tools unzip iptables-persistent net-tools

sed -Ezi.bak "s/(Ext.Msg.show\(\{\s+title: gettext\('No valid sub)/void\(\{ \/\/\1/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js && systemctl restart pveproxy.service

echo "nf_conntrack" >> /etc/modules
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.d/99-proxmox.conf
echo "net.ipv6.conf.all.forwarding=1" >> /etc/sysctl.d/99-proxmox.conf
echo "net.netfilter.nf_conntrack_max=1048576" >> /etc/sysctl.d/99-proxmox.conf
echo "net.netfilter.nf_conntrack_tcp_timeout_established=28800" >> /etc/sysctl.d/99-proxmox.conf
```

* Limit ZFS Memory Usage According to [This Link](https://pve.proxmox.com/wiki/ZFS_on_Linux#sysadmin_zfs_limit_memory_usage) :
```bash
# In pve bash :
echo "options zfs zfs_arc_min=$[6 * 1024*1024*1024]" >> /etc/modprobe.d/99-zfs.conf
echo "options zfs zfs_arc_max=$[12 * 1024*1024*1024]" >> /etc/modprobe.d/99-zfs.conf
update-initramfs -u
```

* Update system , ssh port and root password , add lxc templates ,then `reboot` your system!
```shell
bash <(curl -Ls https://gist.github.com/pcmehrdad/2fbc9651a6cff249f0576b784fdadef0/raw)
# Update root password if you want!
passwd

# Reboot The System
reboot
```

## Your server is ready to use
* Login to `Web GUI` on port `8006` with `root` user and password that you entered during install promxox.
    > https://Your-Server-IP:8006

* Your could use `notes.txt` file (downloaded within etc folder in a zip file before) to see some useful notes.



## Useful links
[ReadMe-v1.md](https://github.com/ariadata/proxmox-hetzner/blob/main/README-v1.md)

Other Useful Links :
```
https://tteck.github.io/Proxmox/
https://github.com/extremeshok/xshok-proxmox
https://github.com/extremeshok/xshok-proxmox/tree/master/hetzner
https://88plug.com/linux/what-to-do-after-you-install-proxmox/
https://gist.github.com/gushmazuko/9208438b7be6ac4e6476529385047bbb
https://github.com/johnknott/proxmox-hetzner-autoconfigure
https://github.com/CasCas2/proxmox-hetzner
https://github.com/west17m/hetzner-proxmox
https://github.com/SOlangsam/hetzner-proxmox-nat
https://github.com/HoleInTheSeat/ProxmoxStater
https://github.com/rloyaute/proxmox-iptables-hetzner
https://computingforgeeks.com/how-to-install-and-configure-firewalld-on-debian/
```