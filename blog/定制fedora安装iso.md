
一、创建ks文件，缩进的是文件内容：
repo.ks：
```
repo --name=f18 --baseurl=http://186.100.8.151/repo/fedora/linux/releases/18/Everything/x86_64/os/
repo --name=f18-updates --baseurl=http://186.100.8.151/repo/fedora/linux/updates/18/x86_64/
```
install.ks
```
# platform=x86, AMD64, or Intel EM64T
# version=DEVEL
# Install OS instead of upgrade
install
# Firewall configuration
firewall --disabled
# Use CDROM installation media
cdrom
# Network information
network  --bootproto=dhcp --device=eth0
# Root password
rootpw --iscrypted $1$50Gx/SMe$C3FiP9W1NnAf0Sx3Fr.BI.
# System authorization information
auth  --useshadow  --passalgo=sha256
# Use text mode install
text
# System keyboard
keyboard us
# System language
lang en_US
# SELinux configuration
selinux --disabled
# Do not configure the X Window System
skipx
# Installation logging level
logging --level=info
# Reboot after installation
reboot
# System timezone
timezone  Asia/Shanghai
# System bootloader configuration
bootloader --location=mbr
# Clear the Master Boot Record
zerombr
# Partition clearing information
clearpart --all --initlabel 
# Disk partitioning information
part swap --fstype="swap" --recommended
part / --asprimary --fstype="ext4" --grow --size=1

%packages --excludedocs
@standard
@core
@hardware-support
kernel
grub2
%end
```
notinstall.ks
```
%packages
-coolkey
-dos2unix
-dump
-finger
-fprintd-pam
-hunspell
-irda-utils
-jwhois
-lftp
-mlocate
-nano
-nc
-nss_ldap
-numactl
-pcmciautils
-pm-utils
-rdate
-rdist
-rsh
-rsync
-sendmail
-sos
-stunnel
-system-config-firewall-tui
-system-config-network-tui
-talk
-time
-tree
-words
-ypbind
-foomatic*
-ghostscript*
-ivtv-firmware
-ql2100-firmware
-ql2200-firmware
-ql23xx-firmware
-ql2400-firmware
-ql2500-firmware

# These are listed somewhere other than hardware support!
-irda-utils
-fprintd*

# dictionaries are big
-aspell-*
-hunspell-*
-man-pages*
-words
-sendmail
%end
```
iso.ks
```
%include repo.ks
%include install.ks
%include notinstall.ks
```
二、创建命令
```
yum install pungi fedora-kickstarts
```
直接做好iso：
```
pungi --name=liteV --ver=0.1 --nosource --nodebuginfo --norelnotes --all-stages -c iso.ks --all-stages
```
 
自动安装的iso：
```
pungi --name=liteV --ver=0.1 --nosource --nodebuginfo --norelnotes --all-stages -c iso.ks -GCB
```
更改isolinux.cfg，添加：
```
ks=cdrom:sr0:/isolinux/install.ks
```
```
pungi --name=liteV --ver=0.1 --nosource --nodebuginfo --norelnotes --all-stages -c iso.ks -I
```
 

参考：

[Kickstart](http://fedoraproject.org/wiki/Anaconda/Kickstart)
