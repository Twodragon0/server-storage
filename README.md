# server-storage (iSCSI)

### iSCSI install and update:
```sh
sudo apt -y install open-iscsi
sudo apt-get update && sudo apt-get upgrade -y
```
### Configure iSCSI Initiator
 ```sh
sudo nano /etc/iscsi/initiatorname.iscsi
```
change to the same IQN you set on the iSCSI target server:  
InitiatorName=iqn.2020-05.kr.re.kist.imrc:55  

서버의 iqn확인:
```sh
sudo nano /etc/iscsi/iscsid.conf
```
- line 48: uncomment: node.leading_login = Yes  
- line 56: uncomment: node.session.auth.authmethod = CHAP  
- line 60,61: To set a CHAP username and password for initiator:  
node.session.auth.username = iqn.2020-05.kr.re.kist.imrc:55  
node.session.auth.password = 1111!@  

username = 스토리지 initiator 계정  
password = 임의 설정 (이 후 스토리지에서 host initiator 구성 시 CHAP 인증 password와 동일 하게 설정)  

### discover target
```sh
iscsiadm -m discovery -t sendtargets -p 161.*.*.*
```
- 161.*.*.*:3260,6 iqn.1988-11.com.dell:01.array.bc305bf0890  
- 161.*.*.*:3260,5 iqn.1988-11.com.dell:01.array.bc305bf0890  
- 161.*.*.*은 스토리지 IP
```sh
systemctl restart iscsid open-iscsi
```
iscsid & open-iscsi 둘다 동작중이어야 합니다. 
확인 방법:
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

상단에 설명 한 것처럼 스토리지에서 host initiator 구성 시 CHAP 인증 정보와 서버의 CHAP 정보가 일치해야 함.  
여기까지 진행 하면 스토리지에서 서버 iqn이 자동 등록되며, 스토리지에서 서버 생성 및 볼륨 할당을 진행함  

### confirm the established session
```sh
iscsiadm -m session -o show
```
 tcp: [1] 161.*.*.*:3260,1 iqn.2020-05.kr.re.kist.imrc:55 (non-flash)  
스토리지에서 연결된 정보 확인 – 일반적으로 2개임 (듀얼컨트롤러)  
 
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
fdisk –l
```
Brand	Model	Version  
- 새로운 파티션 안보임.  
- 로그 아웃 후 재 로그인시 추가된 스토리지 볼륨이 정상적으로 확인됨  

```sh
root@www:~# iscsiadm -m node --logout
root@www:~# df 
```

추가 된 스토리지 볼륨은 로그 아웃 후 로그인 해야 정상 적으로 확인 됨.  

```sh
mkfs.ext4 /dev/mapper/3600c0ff0003cc84c8529755c01000000
mount /dev/mapper/3600c0ff0003cc84c8529755c01000000-part1 /tmp
df -Th
```
/dev/mapper/3600c0ff0003cc84c8529755c01000000-part1 ext4      9.1T   80M  8.6T   1% /tmp  

After setting iSCSI devide, configure on Initiator to use it like follows.  

링크 참고:  
https://www.server-world.info/en/note?os=Ubuntu_18.04&p=iscsi&f=3  

### Error solution

1. Press button E for GRUB menu
2. Select Advanced options for Ubuntu in GRUB menu
3. Select recovery mode 
4. Select and Enter Root mode
5. Disk write Authenrization allow 
6. Modify or making #, and then reboot
```sh
mount -o remount,rw /
sudo nano /etc/fstab
sudo reboot
```
