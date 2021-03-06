
Lenovo ThinkSystem SD530 (Stark) Rack General Test Process

==
1. Lab topology introduction
2. switch confiugrations
3. xcat configurations
4. confluent configuration
5. node discovery
6. os installations
7. diskless os deployment
8. linpack test
9. power cycling
10. data collections
11. FAQ
---

###  1. Lab topology introduction

- in LSTC ,I have 2 Stark chassis(7x20) .  
- each chassis have 2 node （7X21）,so total 4 nodes in my lab .
- SMM in chassis 1 is named smm01 
- SMM in chassis 2 is named smm02 
- node is identify from node01 to node04 as picture as descript as below 
- node01 10G port 1 is connect to the switch1 port 49
- node02 10G port 1 is connect to the switch1 port 50
- node03 10G port 1 is connect to the switch1 port 51
- node04 10G port 1 is connect to the switch1 port 52
- smm01 is connect to switch1 port 35
- smm02 is connect to switch1 port 36
- 
- node01 ib0 port is connect to myib1 port 1
- node02 ib0 port is connect to myib1 port 2
- node03 ib0 port is connect to myib1 port 3
- node04 ib0 port is connect to myib1 port 4

#### Note:
##### It is very important to Test Engineer to take care of the topology in customer order ,you need to pay more attention to the LeROM File
##### especially on the Diagram tables and P2P tables for each rack 
##### the xcat configuration should be base on above topology (customer P2P/diagram) to configure
---
### switch configurations 

```
version "8.3.5"
switch-type "Lenovo RackSwitch G8052"
iscli-new
!
!

snmp-server read-community "RO"
!
!
!
no system dhcp
no system default-ip
hostname "switch1"
!
!
...

!
interface ip 1
        ip address 172.30.50.1 255.255.0.0
        enable
        exit
!
!

```
#### for the switch configurations ,the key part is the IP address , SNMP community strings and Hostname settings 
#### for customer order ,if customer have special settings ,then you need to configure as customer required
#### like the IP , hostname ,SNMP community strings ,VLAN desing and so on 

---
###  3. xcat configurations

- in general , when you finish the xCAT installation with the special Best Recipt version ,you need to configure the xCAT Tables to make the xCAT work for you 
- the latest version BR is here [[17E BR](https://support.lenovo.com/us/zh/solutions/ht505773)]
- restort the default settings for the xCAT tables 
```
cd /opt/xcat/share/xcat/templates/e1350/
for a in *csv; do tabrestore $a; echo $a; done


```
- configure the site tables (you should configure the xCAT tables base on your management node configurations )
```
chtab key=master site.value=mgt1
chtab key=domain site.value=cluster
chtab key=dhcpinterfaces site.value=ens8
chtab key=nameservers site.value=172.20.0.1

```

- configure the networks tables so we can provide the DHCP for the compute node (here the compute node is node01-node04 )
- in most case ,we will provide the:
- compute network (default is 172.20.0.0/16)
- XCC/IMM network(default is 172.29.0.0/16)
- other peripheral device like the PDU,Switch,etc (default is 172.30.0.0/16)
```
 chtab net=172.20.0.0 networks.dynamicrange=172.20.255.1-172.20.255.254
````
### here is my final networks tables
```
[root@mgt1 ~]# tabdump networks
#netname,net,mask,mgtifname,gateway,dhcpserver,tftpserver,nameservers,ntpservers,logservers,dynamicrange,staticrange,staticrangeincrement,nodehostname,ddnsdomain,vlanid,domain,mtu,comments,disable
"data","172.20.0.0","255.255.0.0","ens8","<xcatmaster>",,"<xcatmaster>",,,,"172.20.255.1-172.20.255.254",,,,,,,"1500",,
"imm_xcc","172.29.0.0","255.255.0.0","ens8","<xcatmaster>",,"<xcatmaster>",,,,,,,,,,,"1500",,
"sw_net","172.30.0.0","255.255.0.0","ens8","<xcatmaster>",,"<xcatmaster>",,,,,,,,,,,"1500",,
[root@mgt1 ~]#

```


###  node hardware management table : nodehm
###   nodehm tables is use to specify the Serial on Lan parameters ,man nodehm will provide more informations here
```
 chtab node=compute nodehm.serialport=0 nodehm.serialspeed=115200 nodehm.serialflow=hard

```

### nodetype tables specify node type informations ,it will provide the node architech,os installation profile and os image 
```

chtab node=compute nodetype.os=rhels7.4 nodetype.arch=x86_64 nodetype.Profile=compute

```
### noderes tables specify how OS is pre

```
chtab node=compute noderes.netboot=xnba noderes.nfsserver=172.20.0.1 noderes.installnic=mac noderes.primarynic=mac
```
### mp tables specify the node is manage by which SMM, and how to identify the node by the location informations

```
[root@mgt1 ~]# tabdump mp
#node,mpa,id,nodetype,comments,disable
"blade","|\D+(\d+).*$|amm(($1-1)/14+1)|","|\D+(\d+).*$|(($1-1)%14+1)|",,,
"x220",,,"blade",,
"node01","smm01","1",,,
"node02","smm01","2",,,
"node03","smm02","1",,,
"node04","smm02","2",,,
"x240",,,"blade",,
"x440",,,"blade",,

```

### ipmi table specify the node managemnet relationship 
```
[root@mgt1 ~]# tabdump ipmi
#node,bmc,bmcport,taggedvlan,bmcid,username,password,comments,disable
"ipmi","|(.*)|($1)-xcc|",,,,,,,
"smm","|\D+(\d+).*$|172.30.101.($1+130)|",,,,"USERID","PASSW0RD",,
```
#### note :
#### SMM is managed with specify IP address ,and make sure SMM USERNAME and PASSWORD IS CORRECT 

### add node in the xCAT tables
```
nodeadd node01-node04   groups=21perswitch,ipmi,compute,all,lab,CPOMxxxxx,MFGxxxxx,4nodeperrack
nodeadd node[01-04]-xcc groups=xcc,lab-xcc
nodeadd smm[01-02] groups=smm,lab-smm
nodeadd switch1 groups=mysw
chdef switch1 ip=172.30.50.1
nodeadd myib1 groups=myib
chdef myib1 ip=172.30.10.1
```
### checking the definision of each compute node /XCC /SMM 
```
[root@mgt1 ~]# lsdef node01
Object name: node01
    arch=x86_64
    bmc=node01-xcc
    chain=runcmd=bmcsetup,shell
    groups=21perswitch,ipmi,compute,all,lab,CPOMxxxxx,MFGxxxxx,4nodeperrack
    installnic=mac
    ip=172.20.101.1
    mgt=ipmi
    mpa=smm01
    netboot=xnba
    nfsserver=172.20.0.1
    ondiscover=nodediscover
    os=rhels7.4
    postbootscripts=otherpkgs
    postscripts=syslog,remoteshell,syncfiles
    primarynic=mac
    profile=compute
    serialflow=hard
    serialport=0
    serialspeed=115200
    slotid=1
    switch=switch1
    switchport=49
```
#### NOTE:
####  make sure the bmc attribute is correctly map ,or you need to check the ipmi tables 
####  make sure mpa is point to the correct SMM ,or you need to check with mp tables
####  slotid is the location of node in the  chassis  ,and you need to check with the mp tables
####  check the switch/switchport attribute to make sure match the LeROM file
```
[root@mgt1 ~]# lsdef node01-xcc
Object name: node01-xcc
    groups=xcc,lab-xcc
    ip=172.29.101.1
    postbootscripts=otherpkgs
    postscripts=syslog,remoteshell,syncfiles
```
#### NOTE:
#### check the IP address ,this is the IP that we will flash to the node's XCC
#### 

```
[root@mgt1 ~]# lsdef smm01
Object name: smm01
    bmc=172.30.101.131
    bmcpassword=PASSW0RD
    bmcusername=USERID
    groups=smm,lab-smm
    ip=172.30.101.131
    mgt=ipmi
    postbootscripts=otherpkgs
    postscripts=syslog,remoteshell,syncfiles
    switch=switch1
    switchport=35
```
#### NOTE:
####  check the bmc attribute ,it should be the SMM IP address 
####  bmcusername and bmcpassword should be match in the ipmi tables
####  IP should be same as bmc value
####  switch/switchport attribute should be same in the P2P tables in LeROM file



### if all above is ok ,then we can start to  build the /etc/hosts tables 
#### create the default /etc/hosts as following (or you should change base on the customer network design )
```
[root@mgt1 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.20.0.1      mgt mgt1        mgt1.cluster mgt.cluster
```
### create the new /etc/hosts table with following command 

```
cat /etc/hosts
makehosts compute
makehosts xcc
makehosts smm
makehosts mysw
makehosts myib

```
#### the result of our /etc/hosts as following 

```
[root@mgt1 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.20.0.1      mgt mgt1        mgt1.cluster mgt.cluster
172.20.101.1 node01 node01.cluster
172.20.101.2 node02 node02.cluster
172.20.101.3 node03 node03.cluster
172.20.101.4 node04 node04.cluster
172.29.101.1 node01-xcc node01-xcc.cluster
172.29.101.2 node02-xcc node02-xcc.cluster
172.29.101.3 node03-xcc node03-xcc.cluster
172.29.101.4 node04-xcc node04-xcc.cluster
172.30.101.131 smm01 smm01.cluster
172.30.101.132 smm02 smm02.cluster
172.30.50.1 switch1 switch1.cluster
172.30.10.1 myib1 myib1.cluster

```
#### build DNS database and provide dhcp service 


```
[root@mgt1 ~]# makedns -n
Handling smm02 in /etc/hosts.
Handling node01-xcc in /etc/hosts.
Handling mgt in /etc/hosts.
Handling myib1 in /etc/hosts.
Handling localhost in /etc/hosts.
Handling node03-xcc in /etc/hosts.
Handling node02-xcc in /etc/hosts.
Handling node02 in /etc/hosts.
Handling node03 in /etc/hosts.
Handling switch1 in /etc/hosts.
Handling localhost in /etc/hosts.
Handling node04 in /etc/hosts.
Handling smm01 in /etc/hosts.
Handling node04-xcc in /etc/hosts.
Handling node01 in /etc/hosts.
Getting reverse zones, this may take several minutes for a large cluster.
Completed getting reverse zones.
Updating zones.
Completed updating zones.
Restarting named
Restarting named complete
Updating DNS records, this may take several minutes for a large cluster.
Completed updating DNS records.
DNS setup is completed
[root@mgt1 ~]# makedhcp -n
Renamed existing dhcp configuration file to  /etc/dhcp/dhcpd.conf.xcatbak

Warning: No dynamic range specified for 172.16.0.0. If hardware discovery is being used, a dynamic range is required.
Warning: No dynamic range specified for 172.29.0.0. If hardware discovery is being used, a dynamic range is required.
Warning: No dynamic range specified for 172.30.0.0. If hardware discovery is being used, a dynamic range is required.
Warning: No dynamic range specified for 192.168.122.0. If hardware discovery is being used, a dynamic range is required.
[root@mgt1 ~]# service  dhcpd restart
Redirecting to /bin/systemctl restart dhcpd.service
[root@mgt1 ~]#

```
===
switch and switches table configurations

```
[root@mgt1 ~]# tabdump switches
#switch,snmpversion,username,password,privacy,auth,linkports,sshusername,sshpassword,protocol,switchtype,comments,disable
"switch1",,,"RO",,,,,,,,,
[root@mgt1 ~]# tabdump switch
#node,switch,port,vlan,interface,comments,disable
"compute","|\D+(\d+).*$|switch(($1-1)/40+1)|","|\D+(\d+).*$|(($1-1)%40+49)|",,,,
"smm","|\D+(\d+).*$|switch(($1-1)/40+1)|","|\D+(\d+).*$|(($1-1)%40+35)|",,,,
"41perswitch","|\D+(\d+).*$|switch(($1-1)/41+1)|","|\D+(\d+).*$|(($1-1)%41+1)|",,,,
"42perswitch","|\D+(\d+).*$|switch(($1-1)/42+1)|","|\D+(\d+).*$|(($1-1)%42+1)|",,,,
"20perswitch","|\D+(\d+).*$|switch(($1-1)/20+1)|","|\D+(\d+).*$|(($1-1)%20+1)|",,,,
"node01",,"49",,,,
"node02",,"50",,,,
"node03",,"51",,,,
"node04",,"52",,,,

```


---
### Confluent service configurations 

```
service  confluent start
systemctl enable confluent.service
confetty set /nodegroups/everything/attributes/current discovery.policy=open
confetty create /nodegroups/switch
confetty create /nodes/switch1 groups=switch
confetty set /nodegroups/switch/attributes/current secret.hardwaremanagementpassword="RO"
makeconfluentcfg all
makeconfluentcfg smm
nodeattrib compute console.method=ipmi
```
### check the node attribute with confluent build-in command nodeattrib

```
[root@mgt1 ~]# nodeattrib node01
node01: console.logging: full
node01: console.method: ipmi
node01: discovery.policy: open
node01: enclosure.bay: 1
node01: enclosure.manager: smm01
node01: groups: 21perswitch,ipmi,compute,all,lab,CPOMxxxxx,MFGxxxxx,4nodeperrack,everything
node01: hardwaremanagement.manager: node01-xcc
node01: id.model: 7X21CTO1WW
node01: id.serial: J3002DTH
node01: id.uuid: 17412ce8-c435-11e7-908f-0a94ef4ef7f9
node01: net.switch: switch1
node01: net.switchport: 49
node01: pubkeys.tls_hardwaremanager: sha512$21ea32e546b98767b4113d5822db57085d7a92fe66f466f178974437be69c5a201fc6314b71943631c743794f9b3ed8a321ca64787a9685dc15a19b77bbfc219
node01: secret.hardwaremanagementpassword: ********
node01: secret.hardwaremanagementuser: ********

```
#### Note
#### nodeattribu should have similar output like lsdef command 
#### console.method=ipmi is use for the capture the console output with ipmi channel
#### enclosure.bay is the location of the node in chassis .like the slotid in lsdef output
#### check net.switch and net switchport for link connections 

### tips for check with switch snmp v1 community string 

```
[root@mgt1 ~]#  snmpwalk -v 1 -c RO  switch1 .1.3.6.1.2.1.4.20.1.1
IP-MIB::ipAdEntAddr.172.30.50.1 = IpAddress: 172.30.50.1

```
#### check switch settings if you have problem in snmpwalk run 

### all above is ok ,power on the chassis (not power on the node ) ,and wait for serval minutes ,the node will be discovery by the confluent service 
```
[root@mgt1 ~]# cat /var/log/confluent/events
Jan 08 03:20:13 {"info": "Detected unknown XCC with hwaddr 08:94:ef:4e:f7:f7 at address fe80::a94:efff:fe4e:f7f7%ens8"}
Jan 08 03:20:13 {"info": "Detected unknown XCC with hwaddr 08:94:ef:4e:f8:17 at address fe80::a94:efff:fe4e:f817%ens8"}
Jan 08 03:20:13 {"info": "Detected unknown XCC with hwaddr 08:94:ef:4e:0a:81 at address fe80::a94:efff:fe4e:a81%ens8"}
Jan 08 03:20:13 {"info": "Detected unknown XCC with hwaddr 08:94:ef:4e:11:0d at address fe80::a94:efff:fe4e:110d%ens8"}
Jan 08 03:23:09 {"error": "Timeout or bad SNMPv1 community string trying to reach switch 'switch1'"}
Jan 08 03:24:46 {"error": "Timeout or bad SNMPv1 community string trying to reach switch 'switch1'"}
Jan 08 03:35:17 {"error": "Timeout or bad SNMPv1 community string trying to reach switch 'switch1'"}
Jan 08 17:01:59 {"info": "Discovered node01 (XCC)"}
Jan 08 17:02:00 {"info": "Discovered node04 (XCC)"}
Jan 08 17:02:02 {"info": "Discovered node03 (XCC)"}
Jan 08 17:02:04 {"info": "Discovered node02 (XCC)"}
Jan 08 17:03:43 {"info": "Discovered smm01 (SMM)"}
Jan 08 17:03:59 {"info": "Discovered smm02 (SMM)"}

```

#### in customer order ,you should be able to let confluent discovery all the node /SNM in this phase 
###  make the node Location LED on with rbeacon command to make sure the SMM is correct connect for


```
[root@mgt1 ~]# for node in `nodelist compute`; do rbeacon $node on; sleep 0.5; done
node01: on
node02: on
node03: on
node04: on

```
####  Note :
####   in above command ,you should see the node Location LED will be turn on in turn ,if not ,there should be some cable connection issue there ,check the SMM P2P connection to make sure issue fixed
#### again ,this is rbeacon command ,so it will check on the ipmi channel only , since all XCC go by SMM port ,so it only check the SMM cable P2P only 

---
 
 
 ###  start to power on the SD530 node  for node discover 
 
```
[root@mgt1 ~]# rpower compute on
node01: on
node03: on
node02: on
node04: on

```
### at this time of point ,you can check the node output with nodeconsole (rcons)


```
[root@mgt1 ~]# nodeconsole node01
ELILO v3.16 for EFI/x86_64
.
Loading kernel /tftpboot/xcat/genesis.kernel.x86_64...  done
Loading file /tftpboot/xcat/genesis.fs.x86_64.gz...done





```
after node discovery again by the switch base method ,try to test the cable P2P by turn on the node Location LED with 10G port channel
==

```
[root@mgt1 ~]#rpower all reset 
```
rpower all reset will make all node diskless load with the genisis image (will provide the ipmitool /nodediscovery / bmcsetup ...)
```

[root@mgt1 ~]# nodelist compute
node01
node02
node03
node04

[root@mgt1 ~]# for a in `nodelist compute`; do ssh $a ipmitool chassis  identify 15; echo $a; sleep 0.5; done


```
Note:
    "ssh nodexx ipmitool chassis  identify 15" command will go with the 10G network channel(different when compare to the rbeacon/SMM channel) ,so it will related with the 10G network ,you need to make sure the LED is turn on by sequence to make sure all cable is correct connect

---
###  node OS preload 

```
[root@mgt1 ~]# nodeset all osimage=rhels7.4-x86_64-install-compute
node01: install rhels7.4-x86_64-compute
node02: install rhels7.4-x86_64-compute
node03: install rhels7.4-x86_64-compute
node04: install rhels7.4-x86_64-compute
[root@mgt1 ~]# rsetboot all net -u
node03: Network
node04: Network
node02: Network
node01: Network
[root@mgt1 ~]# rpower all reset
node04: reset
node02: reset
node03: reset
node01: reset
```
####  NOTE:
####   When the Node is preloading ,you can use the nodeconsole to monitor the preload procedure ,here is an example 
```
[root@mgt1 ~]# nodeconsole node01
Installing pyOpenSSL (213/390)
Installing rhnlib (214/390)
Installing python-schedutils (215/390)
Installing openssl (216/390)
Installing bind-libs-lite (217/390)
Installing redhat-logos (218/390)
Installing alsa-lib (219/390)
Installing logrotate (220/390)
Installing binutils (221/390)
Installing nss-pem (222/390)
Installing nss (223/390)
Installing nss-sysinit (224/390)
Installing NetworkManager-libnm (225/390)
Installing nss-tools (226/390)
Installing fipscheck-lib (227/390)
Installing fipscheck (228/390)
Installing libssh2 (229/390)
Installing libcurl (230/390)
Installing curl (231/390)
Installing rpm-libs (232/390)
Installing rpm (233/390)
Installing openldap (234/390)
Installing libuser (235/390)

```
####  after OS preload done ,the node will automatically reboot and boot from HDD 
####  for some node with Hardware RAID support system  ,before the OS preload ,you need to setup the RAID configuration base on customer requirement or base on the Lenovo  RACK default RAID settings instructions 
####  for some new Hardware RAID Controller ,the OS without the build-in driver to support ,you need to provide the driver disk to xCAT before you deploy the OS preloading

### OS boot up and check


```
[root@mgt1 ~]# nodeconsole node01
[  OK  ] Started Load CPU microcode update.
[  OK  ] Started System Logging Service.
[  OK  ] Started GSSAPI Proxy Daemon.
[  OK  ] Reached target NFS client services.
[  OK  ] Reached target Remote File Systems (Pre).
[  OK  ] Reached target Remote File Systems.
         Starting Permit User Sessions...
[  OK  ] Started Permit User Sessions.
[  OK  ] Started Command Scheduler.
         Starting Command Scheduler...
         Starting Terminate Plymouth Boot Screen...
         Starting Wait for Plymouth Boot Screen to Quit...
[  OK  ] Started NTP client/server.
[  OK  ] Started OpenSSH Server Key Generation.
[   11.907870] power_meter ACPI000D:00: Found ACPI power meter.
[   11.914391] wmi: Mapper loaded
[   12.854305] power_meter ACPI000D:00: Found ACPI power meter.
[   12.860661] power_meter ACPI000D:00: Ignoring unsafe software power cap!
[   16.264712] i40e 0000:31:00.0 enp49s0f0: NIC Link is Up, 10 Gbps Full Duplex, Flow Control: None

Red Hat Enterprise Linux Server 7.4 (Maipo)
Kernel 3.10.0-693.el7.x86_64 on an x86_64


node01 login: root
Password:
[root@node01 ~]#

```
#### the default system user and password is root/cluster , and you can find the settings in xCAT passwd table 

### 

---


diskless os deployment 
==
diskless OS image  should be build by Global TE ,but local TE can build themselves for better work with site test environment ,in most case ,the diskles os image will provide the linpack function for group linpack test

```

nodeset noderange  osimage=myedr74
rsetboot noderange net -u
rpower noderange reset
nodeconsole nodename
  
```

---
linpack test  ==> check the linpack test proccess
---

power cycling   ==> check with general Rack test 
--- 
data collections  ==>check with general Rack test
---



### all Server related FW can be get by nodefirmware commmand  
#### all FW ship to customer should be match to the BR, any exceptions should be get WWTE approved 

```
[root@mgt1 ~]# nodefirmware smm01
smm01: BMC Version: 1.1


[root@mgt1 ~]# nodefirmware node03
node03: XCC: 1.5 (TEI308O 2017-08-15T11:21:28)
node03: XCC Backup: 1.05 (TEI308O 2017-08-15T00:00:00)
node03: XCC Trusted Image: TEI308O
node03: UEFI: 1.01 (TEE116E 2017-08-11T00:00:00)
node03: LXPM: 1.02 (PDL106Y 2017-09-05T00:00:00)
node03: LXPM Windows Driver Bundle: 1.01 (PDL304T 2017-07-31T00:00:00)
node03: LXPM Linux Driver Bundle: 1.01 (PDL204O 2017-07-29T00:00:00)
node03: FPGA: 3.8.0
node03: Intel X722 LOM Combined Option ROM Image: 1.1638.0
node03: Intel X722 LOM Etrack ID: 80000ADE
node03: Mellanox ConnectX-4 1x100GbE / EDR IB QSFP28 VPI Adapter Software Bundle: 12.20.1030 (2017-08-06T00:00:00)
[root@mgt1 ~]#
```

FAQ  

1: reset the smm to factory default 
```
ipmitool -I lanplus -U USERID -P PASSW0RD -H x.x.x.x raw 0x32 0xAD 
```
2: reset the XCC to factory default 
```
  ipmitool -I lanplus  -H r137c15s04-imm -U USERID -P PASSW0RD raw  0x2E 0xCC 0x5E 0x2B 0x00 0x0A 0x01 0xFF 0x00 0x00 0x00
```

3: rediscovery the node 
```
 nodeattrib noderange  pubkeys.tls_hardwaremanager=
 ```

4： re-config the confluent service 
```
service confluent stop
mv /etc/confluent/cfg   /etc/conflunet/new_name
service confluet start

```
then config the ohter confluent attribute again

```

confetty set /nodegroups/everything/attributes/current discovery.policy=open
confetty create /nodegroups/switch
confetty create /nodes/switch1 groups=switch
confetty set /nodegroups/switch/attributes/current secret.hardwaremanagementpassword="RO"
makeconfluentcfg all
makeconfluentcfg smm


```




























