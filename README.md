# FS1 - Debian File Server in AD
Scripts and configuration files needed to set up an Active Directory File Server on Debian in a VirtualBox environment.

Reference links:

* https://wiki.samba.org/index.php/Setting_up_Samba_as_a_Domain_Member
* https://wiki.samba.org/index.php/Idmap_config_ad
* https://wiki.samba.org/index.php/Setting_up_a_Share_Using_Windows_ACLs

Create a machine in VirtualBox:

* Name: FS1
* Type: Linux
* Version: Debian (64-bit)
* CPUs: 1
* RAM: 1024 MB
* Virtual HD: 8.00 GB
* HD Type: VDI, dynamically allocated

Use these Network settings for all machines in VirtualBox:

* Adapter 1: Enabled
  * Attached to: NAT Network
  * Name: NatNetwork  (10.0.2.0/24 – DHCP & IPv6 disabled)
* Adapter 2: Enabled
  * Attached to: Host-only Adapter
  * Name: vboxnet0 (192.168.56.0/24 – DHCP disabled)

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
apt install git
git clone https://github.com/TedMichalik/FS1.git
```
## Install software and copy config files to their proper location:
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
 address 10.0.2.8/24
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
        address 192.168.56.8/24
```
Change the default UMASK in the **/etc/login.defs** file (Done with CopyFiles):
```
UMASK 002
```
Add the umask option to **/etc/pam.d/common-session** file (Done with CopyFiles):
```
session optional pam_umask.so
```
Sync time with the AD DC by adding this line to the /etc/systemd/timesyncd.conf file (Done with CopyFiles):
```
NTP=dc1.samdom.example.com
```
If not using the CopyFiles script,reboot the machine to switch to the static IP addressand then
SSH into the secondary adapter and login as the admin user and switch to root.

Install Samba and packages needed for a member server (Done with CopyFiles):
```
apt install -y samba winbind libpam-winbind libnss-winbind libpam-krb5 krb5-config krb5-user
```
Also install some utility programs (Done with CopyFiles):
```
apt install -y smbclient net-tools wsdd
```
Stop all Samba processes (Done with CopyFiles):
```
systemctl stop smbd nmbd winbind
```
Edit the Samba configuration file **/etc/samba/smb.conf** (Done with CopyFiles):
```
[global]
	realm = SAMDOM.EXAMPLE.COM
	security = ADS
	server role = member server
	winbind enum groups = Yes
	winbind enum users = Yes
	winbind offline logon = Yes
	winbind use default domain = Yes
	workgroup = SAMDOM
	idmap config * : backend = tdb
	idmap config * : range = 3000-9999
	idmap config samdom : backend = ad
	idmap config samdom : range = 10000-999999
	idmap config samdom : unix_nss_info = yes
	map acl inherit = Yes
	vfs objects = acl_xattr
	server min protocol = NT1
	client min protocol = NT1

[homes]
	browseable = No
	comment = Home Directories
	create mask = 0644
	directory mask = 02755
	read only = No

[Public]
	comment = Shared Files
	create mask = 0664
	directory mask = 02775
	guest ok = Yes
	path = /opt/Public
	read only = No
```
Use this Kerberos configuration file for **/etc/krb5.conf** (Done with CopyFiles):
```
[libdefaults]
    default_realm = SAMDOM.EXAMPLE.COM
    dns_lookup_realm = false
    dns_lookup_kdc = true
```
Give sudo access to members of “domain admins” (Done with CopyFiles):
```
echo "%domain\ admins ALL=(ALL) ALL" > /etc/sudoers.d/SAMDOM
chmod 0440 /etc/sudoers.d/SAMDOM
```
Test Kerberos authentication against an AD administrative account and list the ticket by issuing the commands:
```
kinit administrator
klist
```
Join the domain, and restart Samba:
```
samba-tool domain join samdom.example.com MEMBER -U administrator
systemctl start smbd nmbd winbind
```
Verify the File Server shares:
```
smbclient -L localhost -U%
```
Enable "Create home directory on login":
```
pam-auth-update
```
Create the Public folder:
```
mkdir /opt/Public
chgrp "Domain Users" /opt/Public
chmod 2775 /opt/Public
```
Reboot to make sure everything works:
```
reboot
```
## Test the Member Server
Login as the admin user. Verify the DNS configuration works correctly:
```
host -t SRV _ldap._tcp.samdom.example.com.
host -t SRV _kerberos._udp.samdom.example.com.
host -t A dc1.samdom.example.com.
```
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
Logout the admin user. You should now be able to login a domain user.

Verify that a home directory was created and that umask is 002:
```
pwd
umask
```
Verify the domain ownership on a test file:
```
touch /opt/Public/testfile
ls -l /opt/Public/
```
