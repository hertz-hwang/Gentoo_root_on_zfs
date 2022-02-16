[toc]
# 基于根目录为zfs文件系统的Gentoo安装过程
## 1. 磁盘准备
### 1.1 写入GPT分区表
`sgdisk -Z /dev/sdX`
### 1.2 擦除磁盘签名
`wipefs -a /dev/sdX`
### 1.3 使用parted工具来创建分区
```
parted --script -a optimal /dev/sdX \
unit mib \
mklabel gpt \
mkpart esp 1 1025 \
mkpart rootfs 1025 100% \
set 1 boot on
```
### 1.4 格式化boot分区
`mkfs.vfat -F32 -n EFI /dev/sdX1`
### 1.5 创建zpool池
```
zpool create -f \
-o ashift=12 \
-o cachefile=/etc/zfs/zpool.cache \
-O normalization=formD \
-O compression=lz4 \
-O acltype=posixacl \
-O relatime=on \
-O atime=off \
-O xattr=sa \
-m none -R /mnt/gentoo zroot /dev/disk/by-id/ata-...-part2
```
### 1.6 创建vdev以及dataset
```
zfs create -o mountpoint=none -o canmount=off zroot/ROOT
zfs create -o mountpoint=none -o canmount=off zroot/DATA
zfs create -o mountpoint=/ zroot/ROOT/default
zfs create -o mountpoint=/home zroot/DATA/home
zfs create -o mountpoint=/var/db/repos zroot/DATA/ebuild
zfs create -o mountpoint=/var/cache/distfiles zroot/DATA/distfiles
zfs create -o mountpoint=/var/cache/ccache zroot/DATA/ccache
zfs create -V 32G -b 8192 -o logbias=throughput -o sync=always -o primarycache=metadata zroot/SWAP
```
### 1.7 配置zpool bootfs
`zpool set bootfs=zroot/ROOT/default zroot`
### 1.8 挂载文件系统 (zfs文件系统能够自管理，在创建的时候就已经自己挂载到指定目录了)
```
mkdir -p /mnt/gentoo/boot
mount /dev/sda1 /mnt/gentoo/boot
mkswap /dev/zvol/zroot/SWAP
swapon /dev/zvol/zroot/SWAP
```
### 1.9 复制zfs缓存
```
mkdir -p /mnt/gentoo/etc/zfs
cp /etc/zfs/zpool.cache /mnt/gentoo/etc/zfs/zpool.cache
```
### 2.0 (可选) 后续如果需要进入chroot环境，执行以下操作
```
zpool import -d /dev/sdX2 -R /mnt/gentoo zroot -N
zfs mount -a
mount /dev/sda1 /mnt/gentoo/boot/
swapon /dev/zvol/zroot/SWAP
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
test -L /dev/shm && rm /dev/shm && mkdir /dev/shm
mount --types tmpfs --options nosuid,nodev,noexec shm /dev/shm
chmod 1777 /dev/shm
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) $PS1"
```
## 2. 安装基本系统
### 2.1 校验系统时间
`date`
### 2.2 下载stage3到/mnt/gentoo下
```
cd /mnt/gentoo
wget https://mirrors.ustc.edu.cn/gentoo/releases/amd64/autobuilds/current-stage3-amd64-desktop-openrc/stage3-amd64-desktop-openrc-[...].tar.xz
tar xvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```
### 2.3 挂载必要的其他文件系统
```
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
test -L /dev/shm && rm /dev/shm && mkdir /dev/shm
mount --types tmpfs --options nosuid,nodev,noexec shm /dev/shm
chmod 1777 /dev/shm
```
### 2.4 复制DNS信息到新系统下
`cp --dereference /etc/resolv.conf /mnt/gentoo/etc/`
### 2.5 进入chroot环境
```
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) $PS1"
```
### 2.6 配置portage参数
```
nano -w /etc/portage/make.conf
```
#### 部分参数
>COMMON\_FLAGS="-O3 -march=znver2 -pipe"<Br/>
CHOST="x86\_64-pc-linux-gnu"<Br/>
CPU\_FLAGS\_X86="aes avx avx2 f16c fma3 mmx mmxext pclmul popcnt rdrand sha sse sse2 sse3 sse4\_1 sse4\_2 sse4a ssse3"<Br/>
MAKEOPTS="-j13"<Br/>
EMERGE\_DEFAULT\_OPTS="--keep-going --with-bdeps=y --autounmask-write=y --jobs=2 -l"<Br/>
AUTO\_CLEAN="yes"<Br/>
GENTOO\_MIRRORS="https://mirrors.ustc.edu.cn/gentoo"<Br/>
#FEATURES="\${FEATURES} -userpriv -usersandbox -sandbox"<Br/>
GRUB\_PLATFORMS="efi-64"<Br/>
ACCEPT\_KEYWORDS="amd64"<Br/>
ACCEPT\_LICENSE="*"<Br/>
L10N="en-US zh-CN en zh"<Br/>
LINGUAS="en_US zh_CN en zh"<Br/>
VIDEO_CARDS="nvidia"<Br/>
INPUT_DEVICES="libinput"<Br/>
FUCKDE="-gnome -gnome-shell -gnome-online-accounts -gnome-keyring -nautilus -kde"<Br/>
FUCKSV="-bindist -jumbo-build -mdev elogind -oss -plymouth -systemd"<Br/>
SOFTWARE="-icu udev blkid efi acpi ccache dbus policykit udisks"<Br/>
AUDIO="alsa jack pulseaudio lv2"<Br/>
NET="network -networkmanager -ipv6 -dhcpcd -ppp -qtwebengine -webengine"<Br/>
VIDEO="X -wayland nvidia xinerama"<Br/>
ELSE="-bluetooth -cups cjk emoji"<Br/>
USE="\${FUCKDE} \${FUCKSV} \${SOFTWARE} \${AUDIO} \${NET} \${VIDEO} \${ELSE}"<Br/>
\# Ccache<Br/>
\# FEATURES="parallel-fetch ccache"<Br/>
\# CCACHE_DIR="/var/cache/ccache"<Br/>
### 2.7 同步镜像站中最新的软件快照
`emerge-webrsync`
### 2.8 配置git方式来同步ebuild数据库
```
emerge -av dev-vcs/git
mkdir -p /etc/portage/repos.conf/
nano -w /etc/portage/repos.conf/gentoo.conf
```
>[DEFAULT]<Br/>
main-repo = gentoo<Br/>
<Br/>
[gentoo]<Br/>
location = /var/db/repos/gentoo<Br/>
sync-type = git<Br/>
sync-uri = https://mirrors.ustc.edu.cn/gentoo.git<Br/>
auto-sync = yes
```
rm -rf /var/db/repos/gentoo
emerge --sync
```
### 2.9 选择profile
```
eselect profile list
eselect profile set Y
```
### 2.10 更新world集合
`emerge -avuDN @world`
## 3. 配置内核
```
echo "sys-kernel/gentoo-kernel-bin -initramfs" > /etc/portage/package.use/gentoo-kernel-bin
echo "sys-fs/zfs dist-kernel" > /etc/portage/package.use/zfs
echo "sys-fs/zfs-kmod dist-kernel" >> /etc/portage/package.use/zfs
echo "sys-boot/grub libzfs" > /etc/portage/package.use/grub
echo "app-text/ghostscript-gpl -l10n_zh-CN" > /etc/portage/package.use/ghostscript-gpl
echo "sys-fs/zfs ~amd64" > /etc/portage/package.accept_keywords/zfs
echo "=sys-fs/zfs-9999 **" >> /etc/portage/package.accept_keywords/zfs
echo "sys-fs/zfs-kmod ~amd64" >> /etc/portage/package.accept_keywords/zfs
echo "=sys-fs/zfs-kmod-9999 **" >> /etc/portage/package.accept_keywords/zfs
echo "options zfs zfs_arc_max=2147483648" > /etc/modprobe.d/zfs.conf

emerge -av sys-kernel/linux-firmware sys-kernel/genkernel sys-kernel/gentoo-kernel-bin zfs zfs-kmod app-editors/neovim
eselect vi set nvim
rm -rf /etc/hostid && zgenhostid
genkernel initramfs --zfs --compress-initramfs --kernel-config=/usr/src/linux/.config --makeopts=-j`nproc`
```
## 4. 配置系统
### 4.1 本地化配置
```
echo "Asia/Shanghai" > /etc/timezone
emerge --config sys-libs/timezone-data
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
echo "zh_CN.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
eselect locale set en_US.UTF-8
env-update && source /etc/profile && export PS1="(chroot) $PS1"
```
### 4.2 配置fstab (这里使用archlinux的一个工具genfstab来自动生成fstab)
```
非chroot环境下执行: genfstab -U -p /mnt/gentoo >> /mnt/gentoo/etc/fstab
因为zfs能够自行管理dataset，不需要在fstab中指定挂载选项，之后需要删除zfs文件系统的相关段落
```
### 4.3 配置主机名
`vi /etc/conf.d/hostname`
### 4.4 配置网络
`vi /etc/conf.d/net`
>config\_enp34s0="null"<Br/>
<Br/>
bridge\_br0="enp34s0 tap1 tap2 tap3"<Br/>
rc\_net\_br0\_need="net.enp34s0 net.tap1 net.tap2 net.tap3"<Br/>
<Br/>
config\_br0="172.16.26.126/16 fd00::32/64"<Br/>
routes\_br0="default via 172.16.26.2<Br/>
default via fd00::1"<Br/>
<Br/>
bridge\_forward\_delay\_br0=0<Br/>
bridge\_hello\_time\_br0=1000<Br/>
bridge\_stp\_state\_br0=0<Br/>
### 4.5 配置hosts
`vi /etc/hosts`
>127.0.0.1 localhost<Br/>
::1 localhost<Br/>
127.0.1.1 gentoo.localdomain gentoo
### 4.6 配置硬件时钟
`vi /etc/conf.d/hwclock`
>clock="local"
### 4.7 配置密码规则
`vi /etc/security/passwdqc.conf`
### 4.8 (可选) 配置音频规则
`vi /etc/security/limits.conf`
>@audio - rtprio 95<Br/>
@audio - memlock unlimited
## 5. 安装工具
```
emerge -av doas cronie eix zsh lsd gentoolkit dosfstools parted neofetch ntfs3g bpytop
```
## 6. 启用服务
```
rc-update add zfs-import boot
rc-update add zfs-mount boot
rc-update add zfs-share default
rc-update add zfs-zed default
rc-update add elogind boot
```
## 7. 配置引导程序
```
emerge --ask --verbose sys-boot/grub
grub-install --target=x86_64-efi --efi-directory=/boot
vi /etc/default/grub
grub-mkconfig -o /boot/grub/grub.cfg
```
## 8. 新建用户
```
useradd -mG users,wheel,portage,usb,input,audio,video,sys,adm,tty,disk,lp,mem,news,console,cdrom,sshd,kvm,render,lpadmin,cron,crontab -s /bin/zsh jaus
passwd jaus
```
## 9. 使用zfs snapshot创建新系统快照
```
zfs snapshot zroot/ROOT/default@install
```
## 10. 重启
```
exit
umount /mnt/gentoo/boot
umount -Rl /mnt/gentoo/{dev,proc,sys,}
swapoff /dev/zvol/zroot/SWAP
zfs umount -a
zpool export -f zroot
```
