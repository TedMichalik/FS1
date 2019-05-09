# DC1
Debian AD DC Setup

Scripts and configuration files needed to set up an Active Directory Domain Controller on Debian.

Download the Debian netinstall image. Boot from it to begin the installation.

*Hostname: DC1.samdom.example.com
*Leave the root password blank.
*Enter the desired user name and password for the admin (sudo) account.
*Make your disk partition selections and write changes to disk.
*Software selection: Only “SSH server” and “standard system utilities”.
*Install the GRUB boot loader on /dev/sda
*Finish the installation and reboot.

Login as the admin user and switch to root. Change to use static IP address. A second adapter was
enabled for SSH logins for testing in VirtualBox. Make these changes to /etc/network/interfaces


