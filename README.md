# DC1 - Debian AD DC Setup
Scripts and configuration files needed to set up an Active Directory Domain Controller on Debian.

Download the Debian netinstall image. Boot from it to begin the installation.

* Hostname: DC1.samdom.example.com
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
https://github.com/TedMichalik/DC1.git
```

Switch to a static IP address.
A second adapter was enabled for SSH logins for testing in VirtualBox.
Make these changes to the **/etc/network/interfaces** file:
```
# The primary network interface
allow-hotplug enp0s3
iface enp0s3 inet static
        address 10.0.2.5/24
        gateway 10.0.2.1
        # dns-* options are implemented by the resolvconf package, if installed
        dns-nameservers 10.0.2.1
        dns-search samdom.example.com

# The primary network interface
allow-hotplug enp0s8
iface enp0s8 inet static
        address 192.168.56.5/24
```
Make these changes for resolving DNS names to the **/etc/resolv.conf** file:
```
domain samdom.example.com
search samdom.example.com
nameserver 10.0.2.5
nameserver 8.8.8.8
```
Make these changes for resolving the local host name to the **/etc/hosts** file:
```
127.0.0.1 localhost
10.0.2.5 DC1.samdom.example.com DC1
```
Reboot the machine to switch to the static IP address.
Login as the admin user and switch to root.

Change the default UMASK in the **/etc/login.defs** file:
```
UMASK 002
```
Install Samba and packages needed for an AD DC. Use the FQDN for the server in the Kerberos setup.
```
apt install samba attr winbind libpam-winbind libnss-winbind libpam-krb5 krb5-config krb5-user
```
Also install some utility programs:
```
apt install smbclient ldb-tools net-tools dnsutils
```
Stop and disable all Samba processes,  and remove the default smb.conf file:
```
systemctl stop smbd nmbd winbind
systemctl disable smbd nmbd winbind
rm /etc/samba/smb.conf
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
Add these lines to the [global] section of **/etc/samba/smb.conf** (version < 4.6.0).
The winbind lines may not be necessary, but I have not tested that yet.
```
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
Install NTP Time Synchronization
```
apt install ntp
```
Configure NTP by editing these two lines in the **/etc/ntp.conf** file:
```
# Clients from this (example!) subnet have unlimited access, but only if
# cryptographically authenticated.
restrict 10.0.2.0 mask 255.255.255.0 notrust

# If you want to provide time to your local subnet, change the next line.
# (Again, the address is an example only.)
broadcast 10.0.2.255
```
Restart the NTP service and verify it is syncing with other servers
```
systemctl restart ntp.service
ntpq -p
```
Ease AD password restrictions, if desired:
```
samba-tool domain passwordsettings set --complexity=off
samba-tool domain passwordsettings set --min-pwd-length=6
samba-tool domain passwordsettings set --max-pwd-age=0
samba-tool user setexpiry administrator --noexpiry
```
