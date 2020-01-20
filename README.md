# DC1 - Debian AD DC Setup
Scripts and configuration files needed to set up an Active Directory Domain Controller on Debian.

Reference links:

* https://wiki.samba.org/index.php/Setting_up_Samba_as_an_Active_Directory_Domain_Controller
* https://wiki.samba.org/index.php/Idmap_config_ad
* https://github.com/christgau/wsdd
* https://wiki.samba.org/index.php/Setting_up_a_Share_Using_Windows_ACLs

Download the Debian netinstall image. Boot from it to begin the installation.

* Hostname: DC1.samdom.example.com
* Leave the root password blank.
* Manually set the enp0s3 network interface:
** address 10.0.2.5/24
** gateway 10.0.2.1
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
git clone https://github.com/TedMichalik/DC1.git
```
Copy config files to their proper location:
```
DC1/CopyFiles1
```
Add a static IP address for the second adapter.
A second adapter was enabled for SSH logins for testing in VirtualBox.
Make these changes to the **/etc/network/interfaces** file (Done with CopyFiles1):
```
# The primary network interface
allow-hotplug enp0s3
iface enp0s3 inet static
        address 10.0.2.5/24
        gateway 10.0.2.1

# The secondary network interface
allow-hotplug enp0s8
iface enp0s8 inet static
        address 192.168.56.5/24
```
Make these changes for resolving the local host name to the **/etc/hosts** file (Done with CopyFiles1):
```
127.0.0.1 localhost
10.0.2.5 DC1.samdom.example.com DC1
```
Reboot the machine to switch to the static IP address.
Login as the admin user and switch to root.

Change the default UMASK in the **/etc/login.defs** file (Done with CopyFiles1):
```
UMASK 002
```
Install Samba and packages needed for an AD DC. Use the FQDN (DC1.samdom.example.com) for the servers in the Kerberos setup.
```
apt install samba attr winbind libpam-winbind libnss-winbind libpam-krb5 krb5-config krb5-user
```
Also install some utility programs:
```
apt install smbclient ldb-tools net-tools dnsutils ntp
```
Stop and disable all Samba processes,  and remove the default smb.conf file:
```
systemctl stop smbd nmbd winbind
systemctl disable smbd nmbd winbind
mv /etc/samba/smb.conf /etc/samba/smb.conf.orig
```
Provision the Samba AD, giving your desired password for the Administrator:
```
samba-tool domain provision --use-rfc2307 --interactive
Realm=SAMDOM.EXAMPLE.COM
Domain=SAMDOM
Server Role=dc
DNS backend=SAMBA_INTERNAL
DNS forwarder IP address=8.8.8.8
```
Edit the Samba configuration file:
```
DC1/EditSMB
```
These lines are added by the EditSMB script to the [global] section of **/etc/samba/smb.conf** (version < 4.6.0).
```
template shell = /bin/bash
template homedir = /home/%U
winbind nss info = rfc2307
winbind use default domain = yes
winbind offline logon = yes
winbind enum users = yes
winbind enum groups = yes
protocol = SMB3
usershare max shares = 0
```
Use the Samba created Kerberos configuration file for your DC, enable the correct Samba services, and reboot to make sure everything works:
```
cp /var/lib/samba/private/krb5.conf /etc/
systemctl unmask samba-ad-dc
systemctl start samba-ad-dc
systemctl enable samba-ad-dc
reboot
```
Login as the admin user and switch to root.
Verify the File Server shares provided by the DC:
```
smbclient -L localhost -U%
```
Copy more config files to their proper location:
```
DC1/CopyFiles2
```
Make these changes for resolving DNS names to the **/etc/resolv.conf** file (Done with CopyFiles2):
```
domain samdom.example.com
search samdom.example.com
nameserver 10.0.2.5
nameserver 8.8.8.8
```
Verify the DNS configuration works correctly:
```
host -t SRV _ldap._tcp.samdom.example.com.
host -t SRV _kerberos._udp.samdom.example.com.
host -t A dc1.samdom.example.com.
```
Verify Kerberos:
```
kinit administrator
klist
```
Configure NTP by editing these two lines in the **/etc/ntp.conf** file (Done with CopyFiles2):
```
# Clients from this (example!) subnet have unlimited access, but only if
# cryptographically authenticated.
restrict 10.0.2.0 mask 255.255.255.0 notrust

# If you want to provide time to your local subnet, change the next line.
# (Again, the address is an example only.)
broadcast 10.0.2.255
```
Restart the NTP service and verify it is syncing with other servers (Done with CopyFiles2):
```
systemctl restart ntp.service
```
Verify the NTP service is syncing with other servers
```
ntpq -p
```
Ease AD password restrictions for testing, if desired:
```
samba-tool domain passwordsettings set --complexity=off
samba-tool domain passwordsettings set --min-pwd-length=6
samba-tool domain passwordsettings set --max-pwd-age=0
samba-tool user setexpiry administrator --noexpiry
```
Install the WSD Daemon which allows Windows Network to detect Linux domain members (Done with CopyFiles2).

Clone git repository and edit file:
```
git clone https://github.com/christgau/wsdd
cd wsdd
```
Edit service file nano etc/systemd/wsdd.service
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
Configure AD Accounts Authentication

Add winbind value for passwd and group lines in the /etc/nsswitch.conf configuration file (Done with CopyFiles2):
```
passwd: files systemd winbind
group:  files systemd winbind
```
Enable  entries required for winbind service to automatically create home directories for each domain account at the first login:
```
pam-auth-update
```
Give sudo access to members of “domain admins” (Done with CopyFiles2):
```
echo "%SAMDOM\\domain\ admins ALL=(ALL) ALL" > /etc/sudoers.d/SAMDOM
chmod 0440 /etc/sudoers.d/SAMDOM
```
