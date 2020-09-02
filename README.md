# server-storage (iSCSI)

### iSCSI install and update:
```sh
sudo apt-get update && sudo apt-get upgrade -y
sudo apt -y install open-iscsi
```
### Configure iSCSI Initiator
 ```sh
sudo nano /etc/iscsi/initiatorname.iscsi
```
change to the same IQN you set on the iSCSI target server:  
InitiatorName=iqn.2020-05.kr.re.kist.imrc:55  

Check the server's iqn:
```sh
sudo nano /etc/iscsi/iscsid.conf
```
- line 48: uncomment: node.leading_login = Yes  
- line 56: uncomment: node.session.auth.authmethod = CHAP  
- line 60,61: To set a CHAP username and password for initiator:  
node.session.auth.username = iqn.2020-05.kr.re.kist.imrc:55  
node.session.auth.password = 1111!@  

username = Storage initiator account
password = Arbitrary setting (then, the same as the CHAP authentication password when configuring the host initiator in storage)

### discover target
```sh
iscsiadm -m discovery -t sendtargets -p 161.*.*.*
```
- 161.*.*.*:3260,6 iqn.1988-11.com.dell:01.array.bc305bf0890  
- 161.*.*.*:3260,5 iqn.1988-11.com.dell:01.array.bc305bf0890  
- 161.*.*.* is Storage IP
```sh
systemctl restart iscsid open-iscsi
```
Both iscsid & open-iscsi must be running.

checking way:
```sh
systemctl status iscsid
systemctl status open-iscsi
```

confirm status after discovery:
```sh
iscsiadm -m node -o show
```
BEGIN RECORD 2.0-874:
node.name = iqn.1988-11.com.dell:01.array.bc305bf0890  
node.tpgt = 6  
node.startup = automatic  
node.leading_login = No  
node.leading_login = No  
node.session.auth.authmethod = CHAP  
node.session.auth.username = iqn.2019-01.kr.re.kist.imrc:05  
.....  
.....  
node.conn[0].iscsi.IFMarker = No  
node.conn[0].iscsi.OFMarker = No  

END RECORD  

### login to the target
```sh
iscsiadm -m node --login
```
Logging in to [iface: default, target: iqn.1988-11.com.dell:01.array.bc305bf0890, portal: 161.*.*.*,3260] (multiple)  
Logging in to [iface: default, target: iqn.1988-11.com.dell:01.array.bc305bf0890, portal: 161.*.*.*,3260] (multiple)  
Login to [iface: default, target: iqn.1988-11.com.dell:01.array.bc305bf0890, portal: 161.*.*.*,3260] successful.  
Login to [iface: default, target: iqn.1988-11.com.dell:01.array.bc305bf0890, portal: 161.*.*.*,3260] successful.  

As described above, when configuring a host initiator in storage, the CHAP authentication information and the server's CHAP information must match. Proceeding to this point, server iqn is automatically registered in storage, and server creation and volume allocation are performed in storage.  

### confirm the established session
```sh
iscsiadm -m session -o show
```
 tcp: [1] 161.*.*.*:3260,1 iqn.2020-05.kr.re.kist.imrc:55 (non-flash)  
Checking connected information in storage – usually 2 (dual controller)

### confirm the partitions
```sh
sudo cat /proc/partitions
```
major minor  #blocks  name

252        0   31457280 sda

252        1   31455232 sda1

253        0   30441472 dm-0

253        1     999424 dm-1

   8        0   10485760 sdb

```sh
fdisk -l
```
Brand	Model	Version  
- If you do not see the new partition, the added storage volume is checked normally when you logout and login again.

```sh
iscsiadm -m node --logout
df -Th
```

## Multipath && Auto mount
Since the storage is configured with dual controllers, multipath operation is essential because the volume is assigned to the server in two Path:
```sh
sudo apt-get install multipath-tools multipath-tools-boot -y
sudo nano /etc/multipath.conf
defaults {
        user_friendly_names yes
}
```
Systemctl reboot for multipath:
```sh
systemctl stop multipath-tools.service
multipath -F
systemctl start multipath-tools.service
mkdir /kist && cd /kist
fdisk -l
```
Create a Linux partition larger than 2TB:
```sh
parted /dev/mapper/mpatha
(parted) unit TB
(parted) mkpart primary 0 0
(parted) rm
Partition number? 1
(parted) mklabel gpt
(parted) mkpart primary 0.00TB 10.00TB
(parted) print
(parted) quit
mkfs.ext4 /dev/mapper/mpatha-part1
mount /dev/mapper/mpatha-part1 /kist
df -Th # disk check
blkid  # UUID check
```
Copy /dev/mapper/mpatha-part1 UUID for auto mount:  
```sh
sudo nano /etc/fstab
UUID=dde9*       /kist   ext4    defaults,auto,_netdev   0       0
```

*Link Note:
https://www.server-world.info/en/note?os=Ubuntu_18.04&p=iscsi&f=3  
https://ubuntu.com/server/docs/device-mapper-multipathing-introduction  


### Error solution

1. Press button E for GRUB menu after reboot
2. Select Advanced options for Ubuntu in GRUB menu
3. Select recovery mode 
4. Select and Enter Root mode
5. Disk write Authenrization allow 
6. Modify or making #, and then reboot
```sh
sudo mount -o remount,rw /
sudo nano /etc/fstab
sudo reboot
```
### GRUB recovery
We need to ubuntu18.04 CD/USB. and select 'Try to ubuntu without install'. it has to do network setting.  
Install boot-repair:
```sh
sudo add-apt-repository ppa:yannubuntu/boot-repair
sudo apt-get update -y
sudo apt-get install boot-repair -y
```
click Recommended Repair
