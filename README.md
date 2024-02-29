# FS1 - Debian File Server in AD
Scripts and configuration files needed to set up an Active Directory File Server on Debian in a VirtualBox environment.

Reference links:

* https://wiki.samba.org/index.php/Setting_up_Samba_as_an_Active_Directory_Domain_Controller
* https://wiki.samba.org/index.php/Idmap_config_ad
* https://wiki.samba.org/index.php/Setting_up_a_Share_Using_Windows_ACLs

Create a machine in VirtualBox:

* Name: FS1
* Type: Linux
* Version: Debian (64-bit)
* CPUs: 1
* RAM: 1024 MB
* Video Memory: 64 MB
* Virtual HD: 8.00 GB
* HD Type: VDI, dynamically allocated

Use these Network settings for all machines in VirtualBox:

* Adapter 1: Enabled
  * Attached to: NAT Network
  * Name: NatNetwork  (10.0.2.0/24 – DHCP & IPv6 disabled)
* Adapter 2: Enabled
  * Attached to: Host-only Adapter
  * Name: vboxnet0 (192.168.56.1/24 – DHCP disabled)

Download the Debian netinstall image. Boot from it to begin the installation.

* Hostname: FS1
* Domain name: samdom.example.com
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
apt full-upgrade
apt install git
git clone https://github.com/TedMichalik/FS1.git
```
Copy config files to their proper location:
```
FS1/CopyFiles
```
Add a static IP address for the first adapter.
Modify file **/etc/network/interfaces** with this content (Done with CopyFiles):
```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug enp0s3
iface enp0s3 inet static
 address 10.0.2.6/24
	gateway 10.0.2.1
```
Add a static IP address for the second adapter.
A second adapter was enabled for SSH logins for configuration and testing in VirtualBox.
Create file **/etc/network/interfaces.d/VirtualBox** with this content (Done with CopyFiles):
```
# This file describes the VirtualBox network interface

# VirtualBox network interface
auto enp0s8
iface enp0s8 inet static
        address 192.168.56.6/24
```
Change the default UMASK in the **/etc/login.defs** file (Done with CopyFiles1):
```
UMASK 002
```
Sync time with the AD DC by adding this line to the /etc/systemd/timesyncd.conf file:
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
apt install smbclient net-tools dnsutils avahi-daemon rsync
```
Stop all Samba processes:
```
systemctl stop smbd nmbd winbind
```
Copy more config files to their proper location.
```
FS1/CopyFiles2
```
Make these changes for resolving DNS names to the **/etc/resolv.conf** file (Done with CopyFiles2):
```
domain samdom.example.com
search samdom.example.com
nameserver 10.0.2.5
nameserver 10.0.2.6
```
Edit the Samba configuration file **/etc/samba/smb.conf** (Done with CopyFiles2):
```
[global]
workgroup = SAMDOM
realm = SAMDOM.EXAMPLE.COM
security = ADS
local master = no
idmap config * : backend = tdb
idmap config *:range = 3000-9999
idmap config SAMDOM : backend = ad
idmap config SAMDOM : range = 10000-999999
idmap config SAMDOM : unix_nss_info = yes
winbind use default domain = yes
winbind offline logon = yes
winbind enum users = yes
winbind enum groups = yes
vfs objects = acl_xattr
map acl inherit = Yes
store dos attributes = Yes
protocol = SMB3
usershare max shares = 0

[homes]

comment = Home Directories
browseable = no
read only = no
create mask = 0644
directory mask = 2755

[Public]

comment = Shared Files
path = /opt/Public
read only = no
guest ok = yes
create mask = 0664
directory mask = 2775
```
Use this Kerberos configuration file for **/etc/krb5.conf** (Done with CopyFiles2):
```
[libdefaults]
        default_realm = SAMDOM.EXAMPLE.COM
        dns_lookup_realm = false
        dns_lookup_kdc = true

[realms]
SAMDOM.EXAMPLE.COM = {
        default_domain = samdom.example.com
}

[domain_realm]
        DC1 = SAMDOM.EXAMPLE.COM
```
Test Kerberos authentication against an AD administrative account and list the ticket by issuing the commands:
```
kinit administrator
klist
```
Join the domain, and restart Samba:
```
net ads join -k
systemctl start smbd nmbd winbind

```
Verify the File Server shares:
```
smbclient -L localhost -U%
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
echo "%domain\ admins ALL=(ALL) ALL" > /etc/sudoers.d/SAMDOM
chmod 0440 /etc/sudoers.d/SAMDOM
```
Create the Public folder (Done with CopyFiles2):
```
mkdir /opt/Public
chgrp "Domain Users" /opt/Public
chmod 2775 /opt/Public
```
## Test the Domain connection

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
touch /opt/Public/testfile
chown ted:"Domain Admins" /opt/Public/testfile
ls -l /opt/Public/
```
