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
