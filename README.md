# 基于 helper 节点的 Openshift 4 离线安装

0. 检查系统环境 - kvm 宿主机  

kvm 宿主机安装 rhel 8.2, 至少 16cores(32 threads)/128GB内存/1TB存储空间/网卡; 终端需要 vnc 客户端, 网络直接连通宿主机  
  
至少有以下几个包  
```
rpm -qa |grep virt
libvirt-4.5.0-42.module+el8.2.0+6024+15a2423f.x86_64
libvirt-client-4.5.0-42.module+el8.2.0+6024+15a2423f.x86_64
virt-install-2.2.1-3.el8.noarch
virt-v2v-1.38.4-15.module+el8.2.0+5297+222a20af.x86_64
```

若缺少则安装  
先确认 yum 源  
```
yum repolist
repo id                                 repo name
rhel8                                   rhel8
rhel8-apps                              rhel8-apps
```

然后安装并确认 libvirtd 启动  
```
yum install libvirt libvirt-client virt-install virt-v2v

sytemctl status libvirtd
● libvirtd.service - Virtualization daemon
   Loaded: loaded (/usr/lib/systemd/system/libvirtd.service; enabled; vendor pr>
  Drop-In: /etc/systemd/system/libvirtd.service.d
           └─unlimited-core.conf
   Active: active (running) since Mon 2021-01-25 21:27:26 CST; 1 day 9h ago
     Docs: man:libvirtd(8)
           https://libvirt.org
 Main PID: 20511 (libvirtd)
    Tasks: 29 (limit: 32768)
   Memory: 177.9M
   CGroup: /system.slice/libvirtd.service
           ├─ 2725 /usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/priv->
           ├─ 2788 /usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/opens>
           ├─ 2829 /usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/pub-l>
           └─20511 /usr/sbin/libvirtd --timeout 120

Jan 26 20:56:04 pubsasrv libvirtd[20511]: libvirt version: 6.6.0, package: 7.1.>
Jan 26 20:56:04 pubsasrv libvirtd[20511]: hostname: pubsasrv
```

打开防火墙的 vnc 服务和若个相关端口  
```
firewall-cmd --permanent --add-service=vnc-server
firewall-cmd --permanent --add-port=5900-5910/tcp
firewall-cmd --reload
firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eno1
  sources: 
  services: cockpit dhcpv6-client ntp ssh vnc-server
  ports: 5900-5910/tcp
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```

ova 文件和 libvirt 定义文件分别在 /opt/virt-template 和 /opt/virt-def 目录下, 属于 qemu 用户  
```
chown qemu:qemu -R /opt/virt-template
chown qemu:qemu -R /opt/virt-def
```

创建本地虚拟网络 pub-local  
```
cat /opt/virt-def/vnet-local-def.xml 
<network>
  <name>pub-local</name>
  <bridge name='pub-localbr' stp='on' delay='0'/>
  <ip address='192.168.100.1' netmask='255.255.255.0'>
  </ip>
</network>

virsh net-define /opt/virt-def/vnet-local-def.xml 
Network local defined from /opt/virt-def/vnet-local-def.xml

virsh net-autostart pub-local
Network local marked as autostarted

virsh net-start pub-local
Network local started

virsh net-list
 Name         State    Autostart   Persistent
-----------------------------------------------
 pub-local    active   yes         yes

ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master online state UP group default qlen 1000
    link/ether 6a:eb:6d:ba:35:d1 brd ff:ff:ff:ff:ff:ff
    inet a.b.c.d/24 brd a.b.c.255 scope global noprefixroute online
       valid_lft forever preferred_lft forever
    ...
15: virpub: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 52:54:00:24:07:7d brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.1/24 brd 192.168.100.255 scope global virpub
       valid_lft forever preferred_lft forever
16: virpub-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc fq_codel master virpub state DOWN group default qlen 1000
    link/ether 52:54:00:24:07:7d brd ff:ff:ff:ff:ff:ff
```

创建并确认虚拟池 vms-disk  
```
mkdir -p /opt/vms-disk
chown qemu:qemu /opt/vms-disk
virsh pool-define-as vms-disk --type dir --target /opt/vms-disk
virsh pool-start vms-disk
virsh pool-autostart vms-disk
virsh pool-list
 Name      State    Autostart
-------------------------------
 default   active   yes
 vms-disk  active   yes
```


1. 用模板文件生成虚拟机, 通过 vnc 显示 - kvm 宿主机  
用 virt-v2v 导入 helper.ova 模板文件, 其中:  
"-i ova" 导入 ova 类型的文件  
"-o libvirt" 输出的虚拟机类型  
"-of qcow2" 虚拟机磁盘用 qcow2 文件格式  
"-os vms-disk" 将输出的虚拟机放置在 vms-disk 池中  
"-n pub-local" 配置输出的虚拟机以使用 pub-local 网络  
"-on lshu-helper45" 输出的虚拟机的 dom 名称  
```
virt-v2v -i ova /opt/virt-template/helper.ova -o libvirt -of qcow2 -os vms-disk -n pub-local -on lshu-helper45
[   0.0] Opening the source -i ova /opt/virt-template/helper.ova
virt-v2v: warning: making OVA directory public readable to work around 
libvirt bug https://bugzilla.redhat.com/1045069
[  35.1] Creating an overlay to protect the source from being modified
[  35.2] Initializing the target -o libvirt -os vms-disk
[  35.2] Opening the overlay
[  56.7] Inspecting the overlay
[  65.7] Checking for sufficient free disk space in the guest
[  65.7] Estimating space required on target for each disk
[  65.7] Converting Red Hat Enterprise Linux Server 7.6 (Maipo) to run on KVM
virt-v2v: This guest has virtio drivers installed.
[ 129.5] Mapping filesystem data to avoid copying unused and blank areas
[ 130.0] Closing the overlay
[ 131.1] Checking if the guest needs BIOS or UEFI to boot
[ 131.1] Assigning disks to buses
[ 131.1] Copying disk 1/1 to /opt/vms-disk/lshu-helper45-sda (qcow2)
    (100.00/100%)
[ 327.9] Creating output metadata
Pool vms-disk refreshed

Domain lshu-helper45 defined from /tmp/v2vlibvirt609d62.xml

[ 329.0] Finishing off
```

查看导入的虚拟机  
```
[root@pubsasrv ~]# virsh list --all
 Id   Name                       State
-------------------------------------------
 -    lshu-helper45              shut off

```

配置虚拟机的内存, CPU 和 vnc 显示, 其中:  
"memory=4194304" 4GB 内存  
"currentMemory=4194304" 4GB 内存  
"vcpu=1" 一个 vcpu  
"interface->network='pub-local'" 使用 pub-local 网络  
"graphics->vnc-listen->address='0.0.0.0'" 配置 vnc 显示, 监听地址为 0.0.0.0  
```
virsh edit lshu-helper45
virsh dumpxml lshu-helper45
<domain type='kvm'>
  <name>lshu-helper45</name>
  <uuid>b6119891-79a1-4eef-b1cb-6e9a2a1923ea</uuid>
  <memory unit='KiB'>4194304</memory>
  <currentMemory unit='KiB'>4194304</currentMemory>
  <vcpu placement='static'>1</vcpu>
  <os>
    <type arch='x86_64' machine='pc-i440fx-rhel7.6.0'>hvm</type>
    <boot dev='hd'/>
  </os>
  <features>
    <acpi/>
    <apic/>
  </features>
  <cpu mode='custom' match='exact' check='none'>
    <model fallback='forbid'>qemu64</model>
  </cpu>
  <clock offset='utc'/>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>restart</on_crash>
  <devices>
    <emulator>/usr/libexec/qemu-kvm</emulator>
    <disk type='volume' device='disk'>
      <driver name='qemu' type='qcow2' cache='none'/>
      <source pool='vms-disk' volume='lshu-helper45-sda'/>
      <target dev='vda' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <target dev='sdb' bus='scsi'/>
      <readonly/>
      <address type='drive' controller='0' bus='0' target='0' unit='1'/>
    </disk>
    <controller type='usb' index='0' model='piix3-uhci'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x2'/>
    </controller>
    <controller type='pci' index='0' model='pci-root'/>
    <controller type='scsi' index='0' model='virtio-scsi'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
    </controller>
    <interface type='network'>
      <mac address='52:54:00:42:27:45'/>
      <source network='pub-local'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
    <serial type='pty'>
      <target type='isa-serial' port='0'>
        <model name='isa-serial'/>
      </target>
    </serial>
    <console type='pty'>
      <target type='serial' port='0'/>
    </console>
    <input type='tablet' bus='usb'>
      <address type='usb' bus='0' port='1'/>
    </input>
    <input type='mouse' bus='ps2'/>
    <input type='keyboard' bus='ps2'/>
    <graphics type='vnc' port='5900' autoport='yes' listen='0.0.0.0' keymap='en-us'>
      <listen type='address' address='0.0.0.0'/>
    </graphics>
    <video>
      <model type='qxl' ram='65536' vram='65536' vgamem='16384' heads='1' primary='yes'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
    </video>
    <memballoon model='virtio'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
    </memballoon>
    <rng model='virtio'>
      <backend model='random'>/dev/urandom</backend>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
    </rng>
    <panic model='isa'>
      <address type='isa' iobase='0x505'/>
    </panic>
  </devices>
</domain>
```

启动并查看 vnc 端口  
```
virsh start lshu-helper45
virsh vncdisplay lshu-helper45
:1 
```

用 vnc 访问 a.b.c.d:1 (a.b.c.d 为宿主机 IP 地址), 用 root/rhel7 登录, 配置虚拟机网络, 并更新 hosts 文件中的 ip 地址  
```
nmcli conn
nmcli conn delete ens32
nmcli networking
nmcli conn add con-name eth0 ifname eth0 type ethernet ip4 192.168.100.241/24 gw4 192.168.100.1
ip a
vi /etc/hosts
```

2. 规划 Openshift 4 的节点, 配置 helper 节点的服务 - helper 节点  
用 ssh 客户端登录 lshu-helper45, 确认本地盘符  
```
ssh root@192.168.100.241
lsblk
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sr0            11:0    1 1024M  0 rom  
vda           252:0    0  800G  0 disk 
├─vda1        252:1    0    1G  0 part /boot
└─vda2        252:2    0  799G  0 part 
  ├─rhel-root 253:0    0  767G  0 lvm  /
  ├─rhel-swap 253:1    0    2G  0 lvm  [SWAP]
  └─rhel-home 253:2    0   30G  0 lvm  /home

```

更改 ansible 参数配置  
```
vi /root/ocp4-upi-helpernode/vars.yaml
cat /root/ocp4-upi-helpernode/vars.yaml 
---
disk: vda
helper:
  name: "helper"
  ipaddr: "192.168.100.241"
  networkifacename: "eth0"
dns:
  domain: "example.com"
  clusterid: "ocp4"
  forwarder1: "192.168.100.1"
dhcp:
  router: "192.168.100.1"
  bcast: "192.168.100.255"
  netmask: "255.255.255.0"
  poolstart: "192.168.100.242"
  poolend: "192.168.100.250"
  ipid: "192.168.100.0"
  netmaskid: "255.255.255.0"
bootstrap:
  name: "bootstrap"
  ipaddr: "192.168.100.250"
  macaddr: "00:50:56:20:1f:1c"
masters:
  - name: "master1"
    ipaddr: "192.168.100.242"
    macaddr: "00:50:56:36:57:62"
  - name: "master2"
    ipaddr: "192.168.100.243"
    macaddr: "00:50:56:35:c7:af"
  - name: "master3"
    ipaddr: "192.168.100.244"
    macaddr: "00:50:56:2c:0f:51"
workers:
  - name: "worker1"
    ipaddr: "192.168.100.245"
    macaddr: "00:50:56:24:a7:67"
  - name: "worker2"
    ipaddr: "192.168.100.246"
    macaddr: "00:50:56:31:76:f5"
  - name: "worker3"
    ipaddr: "192.168.100.247"
    macaddr: "00:50:56:3f:ac:02"
```

运行 ansible 剧本完成 helper 节点服务的配置  
```
cd /root/ocp4-upi-helpernode
ansible-playbook -e @vars.yaml tasks/main.yml
 
PLAY [all] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [localhost]

TASK [Write out dhcp file] *****************************************************
changed: [localhost]

TASK [Write out named file] ****************************************************
changed: [localhost]

TASK [Installing DNS Serialnumber generator] ***********************************
ok: [localhost]

TASK [Set zone serial number] **************************************************
changed: [localhost]

TASK [Setting serial number as a fact] *****************************************
ok: [localhost]

TASK [Write out "example.com" zone file] ***************************************
changed: [localhost]

TASK [Write out reverse zone file] *********************************************
changed: [localhost]

TASK [Write out haproxy config file] *******************************************
changed: [localhost]

TASK [Copy httpd conf file] ****************************************************
ok: [localhost]

TASK [Create apache directories for installing] ********************************
ok: [localhost] => (item=/var/www/html/install)
ok: [localhost] => (item=/var/www/html/ignition)

TASK [Delete OCP4 files, if requested, to download again] **********************
skipping: [localhost] => (item=/usr/local/src/openshift-client-linux.tar.gz) 
skipping: [localhost] => (item=/usr/local/src/openshift-install-linux.tar.gz) 
skipping: [localhost] => (item=/var/www/html/install/bios.raw.gz) 
skipping: [localhost] => (item=/var/lib/tftpboot/rhcos/initramfs.img) 
skipping: [localhost] => (item=/var/lib/tftpboot/rhcos/kernel) 

TASK [Downloading OCP4 installer Bios] *****************************************
ok: [localhost]

TASK [Open up firewall ports] **************************************************
ok: [localhost] => (item=67/udp)
ok: [localhost] => (item=53/tcp)
ok: [localhost] => (item=53/udp)
ok: [localhost] => (item=443/tcp)
ok: [localhost] => (item=80/tcp)
ok: [localhost] => (item=8080/tcp)
ok: [localhost] => (item=6443/tcp)
ok: [localhost] => (item=6443/udp)
ok: [localhost] => (item=22623/tcp)
ok: [localhost] => (item=22623/udp)
ok: [localhost] => (item=9000/tcp)
ok: [localhost] => (item=69/udp)
ok: [localhost] => (item=111/tcp)
ok: [localhost] => (item=2049/tcp)
ok: [localhost] => (item=20048/tcp)
ok: [localhost] => (item=50825/tcp)
ok: [localhost] => (item=53248/tcp)

TASK [Best effort SELinux repair - DNS] ****************************************
changed: [localhost]

TASK [Best effort SELinux repair - Apache] *************************************
changed: [localhost]

TASK [Create NFS export directory] *********************************************
ok: [localhost]

TASK [Copy NFS export conf file] ***********************************************
ok: [localhost]

TASK [Create TFTP config] ******************************************************
ok: [localhost]

TASK [Create TFTP RHCOS dir] ***************************************************
ok: [localhost]

TASK [SEBool allow haproxy connect any port] ***********************************
ok: [localhost]

TASK [Copy over files needed for TFTP] *****************************************
changed: [localhost]

TASK [Downloading OCP4 installer initramfs] ************************************
ok: [localhost]

TASK [Downloading OCP4 installer kernel] ***************************************
ok: [localhost]

TASK [Set the default tftp file] ***********************************************
changed: [localhost]

TASK [Set the bootstrap specific tftp file] ************************************
changed: [localhost]

TASK [Set the master specific tftp files] **************************************
changed: [localhost] => (item={u'macaddr': u'00:50:56:36:57:62', u'ipaddr': u'192.168.100.242', u'name': u'master1'})
changed: [localhost] => (item={u'macaddr': u'00:50:56:35:c7:af', u'ipaddr': u'192.168.100.243', u'name': u'master2'})
changed: [localhost] => (item={u'macaddr': u'00:50:56:2c:0f:51', u'ipaddr': u'192.168.100.244', u'name': u'master3'})

TASK [Set the worker specific tftp files] **************************************
changed: [localhost] => (item={u'macaddr': u'00:50:56:24:a7:67', u'ipaddr': u'192.168.100.245', u'name': u'worker1'})
changed: [localhost] => (item={u'macaddr': u'00:50:56:31:76:f5', u'ipaddr': u'192.168.100.246', u'name': u'worker2'})
changed: [localhost] => (item={u'macaddr': u'00:50:56:3f:ac:02', u'ipaddr': u'192.168.100.247', u'name': u'worker3'})

TASK [Installing TFTP Systemd helper] ******************************************
ok: [localhost]

TASK [Installing TFTP Systemd unit file] ***************************************
ok: [localhost]

TASK [Systemd daemon reload] ***************************************************
ok: [localhost]

TASK [Starting services] *******************************************************
ok: [localhost] => (item=named)
ok: [localhost] => (item=haproxy)
ok: [localhost] => (item=httpd)
ok: [localhost] => (item=rpcbind)
ok: [localhost] => (item=nfs-server)
ok: [localhost] => (item=nfs-lock)
ok: [localhost] => (item=nfs-idmap)

TASK [Starting DHCP/PXE services] **********************************************
changed: [localhost] => (item=dhcpd)
changed: [localhost] => (item=tftp)
changed: [localhost] => (item=helper-tftp)

TASK [Unmasking Services] ******************************************************
ok: [localhost] => (item=tftp)

TASK [Set the local resolv.conf file] ******************************************
changed: [localhost]

TASK [Copy info script over] ***************************************************
changed: [localhost]

TASK [Copying over nfs-provisioner rbac] ***************************************
ok: [localhost]

TASK [Copying over nfs-provisioner deployment] *********************************
changed: [localhost]

TASK [Copying over nfs-provisioner storageclass] *******************************
ok: [localhost]

TASK [Copying over nfs-provisioner setup script] *******************************
ok: [localhost]

TASK [Copying over a sample PVC file for NFS] **********************************
ok: [localhost]

TASK [Downloading OCP4 client] *************************************************
ok: [localhost]

TASK [Downloading OCP4 Installer] **********************************************
ok: [localhost]

TASK [Unarchiving OCP4 client] *************************************************
changed: [localhost]

TASK [Unarchiving OCP4 Installer] **********************************************
changed: [localhost]

TASK [Removing files that are not needed] **************************************
changed: [localhost]

TASK [Installing filetranspiler] ***********************************************
ok: [localhost]

TASK [Get network device system name] ******************************************
changed: [localhost]

TASK [Setting network device system name as a fact] ****************************
ok: [localhost]

TASK [Setting ipv4.ignore-auto-dns on network interface "eth0" to yes] *********
changed: [localhost]

TASK [Setting DNS server ip on network interface "eth0" to 127.0.0.1] **********
changed: [localhost]

TASK [Setting DNS search path on network interface "eth0" to "ocp4.example.com"] ***
changed: [localhost]

TASK [Restarting NetworkManager] ***********************************************
changed: [localhost] => (item=NetworkManager)

TASK [Information about this install] ******************************************
ok: [localhost] => {
    "msg": [
        "Please run /usr/local/bin/helpernodecheck for information"
    ]
}

RUNNING HANDLER [restart bind] *************************************************
changed: [localhost]

RUNNING HANDLER [restart haproxy] **********************************************
changed: [localhost]

RUNNING HANDLER [restart dhcpd] ************************************************
changed: [localhost]

RUNNING HANDLER [restart tftp] *************************************************
changed: [localhost]

PLAY RECAP *********************************************************************
localhost                  : ok=57   changed=29   unreachable=0    failed=0   


helpernodecheck
Usage:
helpernodecheck {dns-masters|dns-workers|dns-etcd|install-info|haproxy|services|nfs-info}


helpernodecheck dns-masters
======================
DNS Config for Masters
======================

; Create entries for the master hosts
master1		IN	A	192.168.100.242
master2		IN	A	192.168.100.243
master3		IN	A	192.168.100.244
;

======================
DNS Lookup for Masters
======================

master1.ocp4.example.com
-------------------------------------------------
IP: 192.168.100.242
Reverse: master1.ocp4.example.com.

master2.ocp4.example.com
-------------------------------------------------
IP: 192.168.100.243
Reverse: master2.ocp4.example.com.

master3.ocp4.example.com
-------------------------------------------------
IP: 192.168.100.244
Reverse: master3.ocp4.example.com.


helpernodecheck dns-workers
======================
DNS Config for Workers
======================

; Create entries for the worker hosts
worker1		IN	A	192.168.100.245
worker2		IN	A	192.168.100.246
worker3		IN	A	192.168.100.247
;

======================
DNS Lookup for Workers
======================

worker1.ocp4.example.com
-------------------------------------------------
IP: 192.168.100.245
Reverse: worker1.ocp4.example.com.

worker2.ocp4.example.com
-------------------------------------------------
IP: 192.168.100.246
Reverse: worker2.ocp4.example.com.

worker3.ocp4.example.com
-------------------------------------------------
IP: 192.168.100.247
Reverse: worker3.ocp4.example.com.


helpernodecheck dns-etcd
===================
DNS Config for ETCD
===================

; The ETCd cluster lives on the masters...so point these to the IP of the masters
etcd-0	IN	A	192.168.100.242
etcd-1	IN	A	192.168.100.243
etcd-2	IN	A	192.168.100.244
;

===================
DNS lookup for ETCD
===================
192.168.100.242
192.168.100.243
192.168.100.244

===================
SRV config for ETCD
===================

; The SRV records are IMPORTANT....make sure you get these right...note the trailing dot at the end...
_etcd-server-ssl._tcp	IN	SRV	0 10 2380 etcd-0.ocp4.example.com.
_etcd-server-ssl._tcp	IN	SRV	0 10 2380 etcd-1.ocp4.example.com.
_etcd-server-ssl._tcp	IN	SRV	0 10 2380 etcd-2.ocp4.example.com.
;

===================
SRV lookup for ETCD
===================
0 10 2380 etcd-0.ocp4.example.com.
0 10 2380 etcd-1.ocp4.example.com.
0 10 2380 etcd-2.ocp4.example.com.


helpernodecheck install-info

This server should also be used as the install node. Apache is running on http://192.168.100.241:8080 You can put your openshift-install artifacts (bios images and ignition files) in /var/www/html

Quickstart Notes:
	mkdir ~/install
	cd ~/install
	vi install-config.yaml
	openshift-install create ignition-configs
	cp *.ign /var/www/html/ignition/
	restorecon -vR /var/www/html/

(See https://docs.openshift.com/container-platform/4.2/installing/installing_bare_metal/installing-bare-metal.html for more details)


helpernodecheck services
Status of services:
===================
Status of dhcpd svc 		->    Active: active (running) since Tue 2021-01-26 15:41:49 CST; 9min ago
Status of named svc 		->    Active: active (running) since Tue 2021-01-26 15:41:48 CST; 9min ago
Status of haproxy svc 		->    Active: active (running) since Tue 2021-01-26 15:41:48 CST; 9min ago
Status of httpd svc 		->    Active: active (running) since Mon 2021-01-25 22:06:58 CST; 17h ago
Status of tftp svc 		->    Active: active (running) since Tue 2021-01-26 15:41:49 CST; 9min ago


helpernodecheck haproxy

HAProxy stats are on http://192.168.100.241:9000 and you should use it to monitor the install when you start.


helpernodecheck nfs-info

An NFS server has been installed and the entire /export directory has been shared out. To set up the nfs-auto-provisioner; you just need to run the following command after "openshift-install wait-for bootstrap-complete --log-level debug" has finished...

	helpernodecheck nfs-setup

Thats it! Right now, this is an "opinionated" setup (there is no "how do I set this up for..."). For now, this is what you get.

Once it's setup, create a PVC for the registry (an example of one has been provided)

	oc create -f /usr/local/src/registry-pvc.yaml -n openshift-image-registry

Check that with "oc get pv" and "oc get pvc -n openshift-image-registry". Then set the registry to use this NFS volume.

	oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"pvc":{ "claim": "registry-pvc"}}}}'


Check the status by watching "oc get pods -n openshift-image-registry"
```

确认 docker-distribution 启动  
```
systemctl status docker-distribution
systemctl status docker-distribution
● docker-distribution.service - v2 Registry server for Docker
   Loaded: loaded (/usr/lib/systemd/system/docker-distribution.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2021-01-25 22:06:57 CST; 17h ago
 Main PID: 3157 (registry)
   CGroup: /system.slice/docker-distribution.service
           └─3157 /usr/bin/registry serve /etc/docker-distribution/registry/c...

Jan 25 22:45:58 helper.ocp4.example.com registry[3157]: time="2021-01-25T22:4...
Jan 25 22:45:58 helper.ocp4.example.com registry[3157]: time="2021-01-25T22:4...
Jan 25 22:45:58 helper.ocp4.example.com registry[3157]: time="2021-01-25T22:4...
Jan 25 22:45:58 helper.ocp4.example.com registry[3157]: time="2021-01-25T22:4...
Jan 25 22:45:58 helper.ocp4.example.com registry[3157]: time="2021-01-25T22:4...
Jan 25 22:45:58 helper.ocp4.example.com registry[3157]: time="2021-01-25T22:4...
Jan 25 22:45:58 helper.ocp4.example.com registry[3157]: time="2021-01-25T22:4...
Jan 25 22:45:58 helper.ocp4.example.com registry[3157]: time="2021-01-25T22:4...
Jan 26 15:58:43 helper.ocp4.example.com registry[3157]: time="2021-01-26T15:5...
Jan 26 15:58:43 helper.ocp4.example.com registry[3157]: 192.168.100.241 - - [...
Hint: Some lines were ellipsized, use -l to show in full.


curl -k https://registry.ocp4.example.com:5443/v2/_catalog
{"repositories":["external_storage/nfs-client-provisioner","olm/certified-operators","olm/community-operators","olm/redhat-operators","openjdk/openjdk-11-rhel7","openshift-release-dev/ocp-release","redhat-openjdk-18/openjdk18-openshift","rhscl/mysql-57-rhel7","rhscl/mysql-80-rhel7"]}
```



3. 生成 Openshift 4 的配置文件 - helper 节点  
创建安装目录  
```
rm -rf /root/ocp4 && mkdir -p /root/ocp4 && cd /root/ocp4
```

拷贝并更改 Openshift 安装配置文件, 其中:   
"compute->worker->replicas=0" 注意 worker 节点副本数必须为 0  
"pullSecret" 由于 docker-distribution 没有设置登录密码, 空字符即可  
"sshKey" 填写 /root/.ssh/id_rsa.pub 里的内容  
"additionalTrustBundle" 填写 /etc/pki/ca-trust/source/anchors/ocp4.example.com.crt 里的内容, 注意要有前面2个空格以符合语法  
```
cp /root/install-config.yaml /root/ocp4
cat install-config.yaml
apiVersion: v1
baseDomain: example.com
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ocp4
networking:
  clusterNetworks:
  - cidr: 10.254.0.0/16
    hostPrefix: 24
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: '{"auths":{"registry.ocp4.example.com:5443":{"auth":"YTph"}}}'
sshKey: 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCrqO+DxAsxKmY1pkJTv8HuQ6593SNcjdE0b1SlOGe2YXZtyXqdCFpR/WUVwgMvt6IQ/K51UR9QQ/OMh7WNACVVzquwlze+cHtsCC9T0tpQkxDoll0aBim96+99TLEpcIElOfFP0SQttrtMeDTQkq/h5+/U5TTJPy3aZmA9TPP+dcijvjRMrVk3xfK/R4G8v7+mbpdYMAw12W6ZHwM9j0g1mhZXBEkuXRUSF6DI2F3+7QLxN1/a2b+JWcJXigVbSl/p0V8MpT1yiYmlapc6SRW0Uhk6mZXr5SOHlDTrCb8MQnXXd4FGry/nfV0gH3eYIC4kTwVi9kROzIJGxxH0VqJ1 root@helper.ocp4.example.com'
additionalTrustBundle: |
  # registry.ocp4.example.com
  -----BEGIN CERTIFICATE-----
  MIIDnzCCAoegAwIBAgIJAJCnBipXI9CqMA0GCSqGSIb3DQEBCwUAMGYxCzAJBgNV
  BAYTAkNOMQswCQYDVQQIDAJHRDELMAkGA1UEBwwCR1oxEDAOBgNVBAoMB1JlZCBI
  YXQxDjAMBgNVBAsMBVNhbGVzMRswGQYDVQQDDBIqLm9jcDQuZXhhbXBsZS5jb20w
  HhcNMjAwMTA3MTU1MjUyWhcNMzAwMTA0MTU1MjUyWjBmMQswCQYDVQQGEwJDTjEL
  MAkGA1UECAwCR0QxCzAJBgNVBAcMAkdaMRAwDgYDVQQKDAdSZWQgSGF0MQ4wDAYD
  VQQLDAVTYWxlczEbMBkGA1UEAwwSKi5vY3A0LmV4YW1wbGUuY29tMIIBIjANBgkq
  hkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAy4YC34C+1SHusZZ66JHbVYtfTeguIEI4
  +hnMuoWVleGnXyJCOoNEgwaNoGffceLMBTLGaENi6s16gveRZtBkbpwEvkgKjxzx
  niMxS552Yh54juGhVNeZtW6eIuKoZHKIBJeAIQfXmzIjnNVpk3H6IeslMOsWLeF8
  7DT3poISlAt2qZftpoXX6/WvMGo/4TnWr5XabXS9TvuErr7PhP7phCBkIYKGw4O7
  yiffD1uWv+qhL10uXQI7Hi3Ld9+nfK2QY0DfDHlcV0U14BcOR4BDetxaXhXbIaU8
  IbM4LvhFJ5uJ2Qd3QYva0oz7pk5i+73Qdy9OcNHws+UgqAaM8Pen8wIDAQABo1Aw
  TjAdBgNVHQ4EFgQUEv7GFjmoJxkpzREhZw7MNkQzPw8wHwYDVR0jBBgwFoAUEv7G
  FjmoJxkpzREhZw7MNkQzPw8wDAYDVR0TBAUwAwEB/zANBgkqhkiG9w0BAQsFAAOC
  AQEAb03hi30KIQRvlBN3LOKr99fRuUaGnTAt24bk0y77nBIHeiOxWPFnt+WLD4qf
  /8Jgo6FLSzibDhO8RbMw4Zj8jpqdzTqEUKKtmYEzjNUMVvPMwohWGArUI535NRrp
  3htjmB9BjC8HmS8xVt6rEL3q6AzGEAfG/b75VLj++bZXN9lOZMhiBBUmqum6Ln7w
  YLfJv5gxJ+ziCMUVr0jcJCalznRCCNVx8OCnk8UU96uhNoiB5FQFWagsFGeyVhvm
  TYIAkdP8Nt3pu/UZR0MLKRtYBpc5D3RPdkvaN7sRaXNmZJDpZePpO0u9zMArkITz
  CackTzrvKLmpqOkPzzzRE6KcEg==
  -----END CERTIFICATE-----
imageContentSources:
- mirrors:
  - registry.ocp4.example.com:5443/openshift-release-dev/ocp-release
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - registry.ocp4.example.com:5443/openshift-release-dev/ocp-release
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```


根据安装配置文件创建点火配置文件并拷贝到 httpd 的服务目录中  
```
openshift-install create ignition-configs
INFO Consuming Install Config from target directory 
WARNING Making control-plane schedulable by setting MastersSchedulable to true for Scheduler cluster settings 


cp -r /root/ocp4/*.ign /var/www/html/ignition/
chmod 644 /var/www/html/ignition/*
curl http://registry.ocp4.example.com:8080/ignition/bootstrap.ign
curl http://registry.ocp4.example.com:8080/ignition/master.ign
curl http://registry.ocp4.example.com:8080/ignition/worker.ign
```

4. 安装 Openshift 4 的 Masters 节点 - kvm 宿主机  
用 virt-install 启动 Masters 和 bootstrap 虚拟机, 指定 mac 地址, 用 pxe 启动, 用 vnc 显示  
```
virt-install --name lshu-ocp4m1 --memory 16384 --vcpus=4 --network network=pub-local,mac.address=00:50:56:36:57:62 --pxe  --graphics vnc,listen=0.0.0.0 --noautoconsole --boot hd,network,menu=on --disk path=/opt/lshu-master1.qcow2,size=30,format=qcow2 --os-variant=rhel8.0

virt-install --name lshu-ocp4m2 --memory 16384 --vcpus=4 --network network=pub-local,mac.address=00:50:56:35:c7:af --pxe  --graphics vnc,listen=0.0.0.0 --noautoconsole --boot hd,network,menu=on --disk path=/opt/lshu-master2.qcow2,size=30,format=qcow2 --os-variant=rhel8.0

virt-install --name lshu-ocp4m3 --memory 16384 --vcpus=4 --network network=pub-local,mac.address=00:50:56:2c:0f:51 --pxe  --graphics vnc,listen=0.0.0.0 --noautoconsole --boot hd,network,menu=on --disk path=/opt/lshu-master3.qcow2,size=30,format=qcow2 --os-variant=rhel8.0

virt-install --name lshu-ocp4bootstrap --memory 16384 --vcpus=4 --network network=pub-local,mac.address=00:50:56:20:1f:1c --pxe  --graphics vnc,listen=0.0.0.0 --noautoconsole --boot hd,network,menu=on --disk path=/opt/lshu-bootstrap.qcow2,size=30,format=qcow2 --os-variant=rhel8.0
```

启动后会自动安装 coreos 系统到硬盘, 完成安装后自动关机.  
用 virsh 启动各个完成的虚拟机...  
```
virsh start lshu-ocp4m1
virsh start lshu-ocp4m2
virsh start lshu-ocp4m3
virsh start lshu-ocp4bootstrap
```

查看以上虚拟机对应的 vnc 端口, 用 vnc a.b.d.c:? 查看各个节点情况  
```
virsh vncdisplay lshu-ocp4m1
:3

virsh vncdisplay lshu-ocp4m2
:2

virsh vncdisplay lshu-ocp4m3
:4

virsh vncdisplay lshu-ocp4bootstrap
:5
```

5. 用 openshift-install 命令查看安装情况 - helper 节点  
```
openshift-install wait-for bootstrap-complete --log-level debug
DEBUG OpenShift Installer 4.5.15                   
DEBUG Built from commit 9893a482f310ee72089872f1a4caea3dbec34f28 
INFO Waiting up to 20m0s for the Kubernetes API at https://api.ocp4.example.com:6443... 
INFO API v1.18.3+2fbd7c7 up                       
INFO Waiting up to 40m0s for bootstrapping to complete... 
DEBUG Bootstrap status: complete                   
INFO It is now safe to remove the bootstrap resources 
DEBUG Time elapsed per stage:                      
DEBUG Bootstrap Complete: 12m46s                   
INFO Time elapsed: 12m46s                         
```

6. 查看 bootstrap 节点日志 - bootstrap 节点  
从 helper 节点登录到 bootstrap 节点, 查看日志. bootstrap 是将 etcd, control plane 等容器安装到 master 节点上的过程, 到 bootkube.service complete 即以完成, 可以停止该节点了. Openshift 4 之后的安装过程通过 operators 自动完成.  
```
ssh core@bootstrap

journalctl -b -f -u release-image.service -u bootkube.service
-- Logs begin at Tue 2021-01-26 13:05:18 UTC. --
Jan 26 13:21:12 bootstrap.ocp4.example.com bootkube.sh[2506]: Skipped "secret-service-network-serving-signer.yaml" secrets.v1./service-network-serving-signer -n openshift-kube-apiserver-operator as it already exists
Jan 26 13:21:21 bootstrap.ocp4.example.com bootkube.sh[2506]: Skipped "user-ca-bundle-config.yaml" configmaps.v1./user-ca-bundle -n openshift-config as it already exists
Jan 26 13:21:22 bootstrap.ocp4.example.com bootkube.sh[2506]: [#1] failed to create some manifests:
Jan 26 13:21:22 bootstrap.ocp4.example.com bootkube.sh[2506]: "00_etcd-endpoints-cm.yaml": failed to create configmaps.v1./etcd-endpoints -n openshift-etcd: etcdserver: leader changed
Jan 26 13:21:25 bootstrap.ocp4.example.com bootkube.sh[2506]: Skipped "00_etcd-endpoints-cm.yaml" configmaps.v1./etcd-endpoints -n openshift-etcd as it already exists
Jan 26 13:21:27 bootstrap.ocp4.example.com bootkube.sh[2506]: Sending bootstrap-finished event.Tearing down temporary bootstrap control plane...
Jan 26 13:21:31 bootstrap.ocp4.example.com bootkube.sh[2506]: Waiting for CEO to finish...
Jan 26 13:21:35 bootstrap.ocp4.example.com bootkube.sh[2506]: I0126 13:21:35.370980       1 waitforceo.go:64] Cluster etcd operator bootstrapped successfully
Jan 26 13:21:35 bootstrap.ocp4.example.com bootkube.sh[2506]: I0126 13:21:35.372769       1 waitforceo.go:58] cluster-etcd-operator bootstrap etcd
Jan 26 13:21:35 bootstrap.ocp4.example.com bootkube.sh[2506]: bootkube.service complete
```

7. 查看 Openshift 4 集群的状态 - helper 节点  
```
oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version             False       True          35m     Working towards 4.5.15: 86% complete, waiting on marketplace

oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.5.15    True        False         False      5m21s
cloud-credential                           4.5.15    True        False         False      35m
cluster-autoscaler                         4.5.15    True        False         False      8m9s
config-operator                            4.5.15    True        False         False      6m
console                                    4.5.15    True        False         False      7m41s
csi-snapshot-controller                    4.5.15    True        False         False      9m29s
dns                                        4.5.15    True        False         False      26m
etcd                                       4.5.15    True        False         False      28m
image-registry                             4.5.15    True        False         False      20m
ingress                                    4.5.15    True        False         False      12m
insights                                   4.5.15    True        False         False      20m
kube-apiserver                             4.5.15    True        False         False      25m
kube-controller-manager                    4.5.15    True        False         False      26m
kube-scheduler                             4.5.15    True        False         False      25m
kube-storage-version-migrator              4.5.15    True        False         False      17m
machine-api                                4.5.15    True        False         False      19m
machine-approver                           4.5.15    True        False         False      24m
machine-config                             4.5.15    True        False         False      7m51s
marketplace                                                                    False      
monitoring                                 4.5.15    True        False         False      6m5s
network                                    4.5.15    True        False         False      30m
node-tuning                                4.5.15    True        False         False      30m
openshift-apiserver                        4.5.15    True        False         False      8m22s
openshift-controller-manager               4.5.15    True        False         False      15m
openshift-samples                          4.5.15    True        False         False      6m3s
operator-lifecycle-manager                 4.5.15    True        False         False      28m
operator-lifecycle-manager-catalog         4.5.15    True        False         False      28m
operator-lifecycle-manager-packageserver   4.5.15    True        False         False      6m50s
service-ca                                 4.5.15    True        False         False      30m
storage                                    4.5.15    True        False         False      19m
```


8. 安装 Openshift 4 的 Workers 节点 - kvm 宿主机  
用 virt-install 启动 Workers 虚拟机, 指定 mac 地址, 用 pxe 启动, 用 vnc 显示  
```
virt-install --name lshu-ocp4w1 --memory 16384 --vcpus=4 --network network=pub-local,mac.address=00:50:56:24:a7:67 --pxe  --graphics vnc,listen=0.0.0.0 --noautoconsole --boot hd,network,menu=on --disk path=/opt/lshu-worker1.qcow2,size=30,format=qcow2 --os-variant=rhel8.0

virt-install --name lshu-ocp4w2 --memory 16384 --vcpus=4 --network network=pub-local,mac.address=00:50:56:31:76:f5 --pxe  --graphics vnc,listen=0.0.0.0 --noautoconsole --boot hd,network,menu=on --disk path=/opt/lshu-worker2.qcow2,size=30,format=qcow2 --os-variant=rhel8.0

virt-install --name lshu-ocp4w3 --memory 16384 --vcpus=4 --network network=pub-local,mac.address=00:50:56:3f:ac:02 --pxe  --graphics vnc,listen=0.0.0.0 --noautoconsole --boot hd,network,menu=on --disk path=/opt/lshu-worker3.qcow2,size=30,format=qcow2 --os-variant=rhel8.0
```

启动后会自动安装 coreos 系统到硬盘, 完成安装后自动关机.  
用 virsh 启动各个完成的虚拟机...  
```
virsh start lshu-ocp4w1
virsh start lshu-ocp4w2
virsh start lshu-ocp4w3
```

用 oc 命令批准 Pending 状态的 Workers 节点证书, 使之加入集群  
```
oc get csr
NAME        AGE     SIGNERNAME                                    REQUESTOR                                                                   CONDITION
csr-8pmx7   5m57s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-c5mj6   70m     kubernetes.io/kubelet-serving                 system:node:master2.ocp4.example.com                                        Approved,Issued
csr-d2fwk   71m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-hl6px   70m     kubernetes.io/kubelet-serving                 system:node:master1.ocp4.example.com                                        Approved,Issued
csr-kx57p   6m13s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-lr67r   71m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-ltv65   6m      kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-p4mpt   70m     kubernetes.io/kubelet-serving                 system:node:master3.ocp4.example.com                                        Approved,Issued
csr-qvzc8   71m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued

oc adm certificate approve csr-8pmx7
certificatesigningrequest.certificates.k8s.io/csr-8pmx7 approved

oc adm certificate approve csr-kx57p
certificatesigningrequest.certificates.k8s.io/csr-kx57p approved

oc adm certificate approve csr-ltv65
certificatesigningrequest.certificates.k8s.io/csr-ltv65 approved
```

9. 完成 Openshift 4 集群的安装 - helper 节点  
```
openshift-install wait-for install-complete
INFO Waiting up to 30m0s for the cluster at https://api.ocp4.example.com:6443 to initialize... 
INFO Waiting up to 10m0s for the openshift-console route to be created... 
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/root/ocp4/auth/kubeconfig' 
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.ocp4.example.com 
INFO Login to the console with user: "kubeadmin", and password: "oe8Ao-938DZ-h7bSd-dVdeE" 
INFO Time elapsed: 22s                            

oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.5.15    True        False         104s    Cluster version is 4.5.15

oc get clusteroperators
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.5.15    True        False         False      61m
cloud-credential                           4.5.15    True        False         False      92m
cluster-autoscaler                         4.5.15    True        False         False      64m
config-operator                            4.5.15    True        False         False      62m
console                                    4.5.15    True        False         False      64m
csi-snapshot-controller                    4.5.15    True        False         False      23m
dns                                        4.5.15    True        False         False      82m
etcd                                       4.5.15    True        False         False      85m
image-registry                             4.5.15    True        False         False      76m
ingress                                    4.5.15    True        False         False      69m
insights                                   4.5.15    True        False         False      76m
kube-apiserver                             4.5.15    True        False         False      82m
kube-controller-manager                    4.5.15    True        False         False      82m
kube-scheduler                             4.5.15    True        False         False      82m
kube-storage-version-migrator              4.5.15    True        False         False      12m
machine-api                                4.5.15    True        False         False      75m
machine-approver                           4.5.15    True        False         False      81m
machine-config                             4.5.15    True        False         False      30m
marketplace                                4.5.15    True        False         False      2m33s
monitoring                                 4.5.15    True        False         False      8m35s
network                                    4.5.15    True        False         False      87m
node-tuning                                4.5.15    True        False         False      86m
openshift-apiserver                        4.5.15    True        False         False      10m
openshift-controller-manager               4.5.15    True        False         False      71m
openshift-samples                          4.5.15    True        False         False      62m
operator-lifecycle-manager                 4.5.15    True        False         False      84m
operator-lifecycle-manager-catalog         4.5.15    True        False         False      84m
operator-lifecycle-manager-packageserver   4.5.15    True        False         False      9m29s
service-ca                                 4.5.15    True        False         False      86m
storage                                    4.5.15    True        False         False      75m
```
