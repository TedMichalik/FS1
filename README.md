# FS1 - Debian Member Server
Scripts and configuration files needed to set up an additional Active Directory Domain Controller on Debian in a VirtualBox environment.

Reference links:

* https://wiki.samba.org/index.php/Setting_up_Samba_as_an_Active_Directory_Domain_Controller
* https://wiki.samba.org/index.php/Idmap_config_ad
* https://github.com/christgau/wsdd
* https://wiki.samba.org/index.php/Setting_up_a_Share_Using_Windows_ACLs
* https://www.tecmint.com/manage-samba4-ad-from-windows-via-rsat/

Use these Network settings for all machines in VirtualBox:

* Adapter 1: Enabled
  * Attached to: NAT Network
  * Name: NatNetwork  (10.0.2.0/24 – DHCP & IPv6 disabled)
* Adapter 2: Enabled
  * Attached to: Host-only Adapter
  * Name: VirtualBox Host-Only Ethernet Adapter (192.168.56.0/24 – DHCP & IPv6 disabled)

Download the Debian netinstall image. Boot from it to begin the installation.

* Hostname: FS1.samdom.example.com
* Leave the root password blank.
* Enter the desired user name and password for the admin (sudo) account.
* Make your disk partition selections and write changes to disk.
* Software selection: Only “SSH server” and “standard system utilities”.
* Install the GRUB boot loader on /dev/sda
* Finish the installation and reboot.

Login as the admin user and switch to root.
Install git and download these instructions, scripts and configuration files:
```
apt update
apt dist-upgrade
apt install git
git clone https://github.com/TedMichalik/FS1.git
```
Copy config files to their proper location:
```
FS1/CopyFiles1
```
Add a static IP address for the second adapter.
A second adapter was enabled for SSH logins for configuration and testing in VirtualBox.
Make these changes to the **/etc/network/interfaces** file (Done with CopyFiles1):
```
# The primary network interface
auto enp0s3
iface enp0s3 inet static
        address 10.0.2.6/24
        gateway 10.0.2.1

# The secondary network interface
auto enp0s8
iface enp0s8 inet static
        address 192.168.56.6/24
```
Make these changes for resolving the local host name to the **/etc/hosts** file (Done with CopyFiles1):
```
127.0.0.1 localhost
10.0.2.8 FS1.samdom.example.com FS1
```
Change the default UMASK in the **/etc/login.defs** file (Done with CopyFiles1):
```
UMASK 002
```
Configure NTP (Done with CopyFiles1)

Add this line in the **/etc/systemd/timesyncd.conf** file:
```
NTP=dc1.samdom.example.com dc2.samdom.example.com
```
Reboot the machine to switch to the static IP address.
SSH into the secondary adapter and login as the admin user and switch to root.

Install Samba and packages needed for a member server. Use the FQDN (FS1.samdom.example.com) for the servers in the Kerberos setup.
```
apt install samba krb5-config krb5-user winbind libpam-winbind libnss-winbind
```
Also install some utility programs:
```
apt install smbclient net-tools dnsutils avahi-daemon rsync git
```
Stop all Samba processes:
```
systemctl stop smbd nmbd winbind
```
Copy more config files to their proper location.
```
DC2/CopyFiles2
```
Edit the Samba configuration file:
```
DC2/EditSMB
```
These lines are added by the EditSMB script to the [global] section of **/etc/samba/smb.conf**
```
interfaces = enp0s3
dns forwarder = 8.8.8.8
idmap_ldb:use rfc2307 = yes
winbind nss info = rfc2307
winbind use default domain = yes
winbind offline logon = yes
winbind enum users = yes
winbind enum groups = yes
protocol = SMB3
usershare max shares = 0
```
Use the Samba created Kerberos configuration file for your DC, enable the correct Samba services:
```
cp /var/lib/samba/private/krb5.conf /etc/
systemctl unmask samba-ad-dc
systemctl start samba-ad-dc
systemctl enable samba-ad-dc
```
Join the existing domain, and reboot to make sure everything works:
```
samba-tool domain join samdom.example.com DC -USAMDOM\\administrator
reboot
```
Login as the admin user and switch to root.
Verify the File Server shares provided by the DC:
```
smbclient -L localhost -U%
```
Make these changes for resolving DNS names to the **/etc/resolv.conf** file (Done with CopyFiles2):
```
domain samdom.example.com
search samdom.example.com
nameserver 10.0.2.5
nameserver 10.0.2.6
```
Verify the DNS configuration works correctly:
```
host -t SRV _ldap._tcp.samdom.example.com.
host -t SRV _kerberos._udp.samdom.example.com.
host -t A dc1.samdom.example.com.
host -t A dc2.samdom.example.com.
```
Verify Kerberos:
```
kinit administrator
klist
```
## Install the WSD Daemon which allows Windows Network to detect Linux domain members (Done with CopyFiles2).

Clone git repository and edit file:
```
git clone https://github.com/christgau/wsdd
cd wsdd
```
Use this for service file etc/systemd/wsdd.service
```
After=multi-user.target
Wants=multi-user.target
ExecStart=/usr/bin/wsdd -d SAMDOM -4 -s
User=daemon
Group=daemon
```
Copy the files to the correct locations, enable the service, and start it:
```
cp src/wsdd.py /usr/bin/wsdd
cp etc/systemd/wsdd.service /etc/systemd/system
systemctl daemon-reload
systemctl enable wsdd.service
systemctl start wsdd.service
```
## Configure AD Accounts Authentication

Add winbind value for passwd and group lines in the /etc/nsswitch.conf configuration file (Done with CopyFiles2):
```
passwd: files systemd winbind
group:  files systemd winbind
```
Manually run this to automatically create home directories for each domain account at the first login (NOT done CopyFiles2):
```
pam-auth-update
```
Give sudo access to members of “domain admins” (Done with CopyFiles2):
```
echo "%SAMDOM\\domain\ admins ALL=(ALL) ALL" > /etc/sudoers.d/SAMDOM
chmod 0440 /etc/sudoers.d/SAMDOM
```
## Configure the DHCP Service (Done with CopyFiles2):

Just use IPv4 on the NatNetwork with these edits to the /etc/default/isc-dhcp-server configuration file:
```
# On what interfaces should the DHCP server (dhcpd) serve DHCP requests?
# Separate multiple interfaces with spaces, e.g. "eth0 eth1".
INTERFACESv4="enp0s3"
INTERFACESv6=""
```
Edit the /etc/dhcp/dhcpd.conf configuration file:
```
# dhcpd.conf
#
# Sample configuration file for ISC dhcpd
#
#
# option definitions common to all supported networks...
option domain-name "samdom.example.com";
option domain-name-servers 10.0.2.6,10.0.2.5;
#
default-lease-time 86400;
max-lease-time 604800;
#
# The ddns-updates-style parameter controls whether or not the server will
# attempt to do a DNS update when a lease is confirmed. We default to the
# behavior of the version 2 packages ('none', since DHCP v2 didn't
# have support for DDNS.)
ddns-update-style none;
#
# If this DHCP server is the official DHCP server for the local
# network, the authoritative directive should be uncommented.
authoritative;
#
# Use this to send dhcp log messages to a different log file (you also
# have to hack syslog.conf to complete the redirection).
#log-facility local7;
#
# No service will be given on this subnet, but declaring it helps the
# DHCP server to understand the network topology.
#
subnet 192.168.56.0 netmask 255.255.255.0 {
}
#
# This is a very basic subnet declaration.
#
subnet 10.0.2.0 netmask 255.255.255.0 {
range 10.0.2.101 10.0.2.150;
option routers 10.0.2.1;
}
```
Restart the service:
```
systemctl restart isc-dhcp-server.service
```

## Sysvol Replication & Active Directory Management
Sysvol replication is currently not supported on Samba. To use a Sysvol Replication workaround, all domain controllers (DC) must use the same ID mappings for built-in users and groups.
From root on the primary DC, copy the public key:
```
scp .ssh/id_rsa.pub pi@c3po:
```
Then from root on a secondary DC, create SSH keys and authorize the primary DC public key:
```
ssh-keygen -t rsa
cat /home/pi/id_rsa.pub > .ssh/authorized_keys
```
Back on the primary DC, verify you can ssh to the secondary. 
To use a Sysvol Replication workaround, all domain controllers (DC) must use the same ID mappings for built-in users and groups.
By default, a Samba DC stores the user & group IDs in 'xidNumber' attributes in 'idmap.ldb'. Because of the way 'idmap.ldb' works, you cannot guarantee that each DC will use the same ID for a given user or group. To ensure that you do use the same IDs, you must:
Create a hot-backup of the /var/lib/samba/private/idmap.ldb file on the existing DC:
```
tdbbackup -s .bak /var/lib/samba/private/idmap.ldb
```
 
This creates a backup file /var/lib/samba/private/idmap.ldb.bak.
Move the backup file to the /var/lib/samba/private/ folder on the new joined DC and remove the .bak suffix to replace the existing file.
Run net cache flush on the new DC.
You will now need to sync Sysvol to the new DC.
Reset the Sysvol folder's file system access control lists (ACL) on the new DC:
```
samba-tool ntacl sysvolreset
```
 
Add the following cron job:
```
nano  /etc/cron.hourly/sysvol
#!/bin/bash

#Sync sysvol with c3po
rsync -az --delete /var/lib/samba/sysvol c3po:/var/lib/samba
```

## Test the AD DC

Verify the domain users are shown by both commands:
```
wbinfo -u
getent passwd
```
Verify the domain groups are shown by both commands:
```
wbinfo -g
getent group
```
Verify the domain ownership on a test file:
```
touch /tmp/testfile
chown ted:"Domain Admins" /tmp/testfile
ls -l /tmp/testfile
```
