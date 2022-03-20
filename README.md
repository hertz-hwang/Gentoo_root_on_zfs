[toc]
# 基于根目录为zfs文件系统的Gentoo安装过程
## 0. 配置参数
```
DISK=/dev/disk/by-id/nvme-INTEL_SSDPEKKW512G8_BTHH812205BA512D
MOUNT=/mnt/gentoo
ZPOOL=zroot
STAGE3=[stage3 mirror address]
```
## 1. 磁盘准备
### 1.1 擦除磁盘签名
`wipefs -a ${DISK}`
### 1.2 写入GPT分区表
`sgdisk -Z ${DISK}`
### 1.3.1 使用parted工具来创建分区
```
parted --script -a optimal ${DISK} \
unit mib \
mklabel gpt \
mkpart esp 1 1025 \
mkpart rootfs 1025 100% \
set 1 boot on
```
### 1.3.2 使用sgdisk创建分区
```
sgdisk -n1:1M:+1G	-t1:EF00 ${DISK}
sgdisk -n2:0:0		-t2:BF00 ${DISK}
```
### 1.4 格式化boot分区
`mkfs.vfat -F32 -n EFI ${DISK}-part1`
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
-m none -R ${MOUNT} ${ZPOOL} ${DISK}-part2
```
### 1.6 创建dataset
```
zfs create -o mountpoint=none -o canmount=off ${ZPOOL}/ROOT
zfs create -o mountpoint=none -o canmount=off ${ZPOOL}/DATA
zfs create -o mountpoint=/ ${ZPOOL}/ROOT/default
zfs create -o mountpoint=/home ${ZPOOL}/DATA/home
zfs create -o mountpoint=/var/db/repos ${ZPOOL}/DATA/ebuild
zfs create -o mountpoint=/var/cache/distfiles ${ZPOOL}/DATA/distfiles
zfs create -o mountpoint=/var/cache/ccache ${ZPOOL}/DATA/ccache
zfs create -V 32G -b 8192 -o logbias=throughput -o sync=always -o primarycache=metadata ${ZPOOL}/SWAP
```
### 1.7 配置zpool bootfs
`zpool set bootfs=${ZPOOL}/ROOT/default ${ZPOOL}`
### 1.8 挂载文件系统 (zfs文件系统能够自管理，在创建的时候就已经自己挂载到指定目录了)
```
mkdir -p ${MOUNT}/boot
mount ${DISK}-part1 ${MOUNT}/boot
mkswap /dev/zvol/${ZPOOL}/SWAP
swapon /dev/zvol/${ZPOOL}/SWAP
```
### 1.9 复制zfs缓存
```
mkdir -p ${ZPOOL}/etc/zfs
cp /etc/zfs/zpool.cache ${ZPOOL}/etc/zfs/zpool.cache
```
### 2.0 (可选) 后续如果需要进入chroot环境，执行以下操作
```
zpool import -d ${DISK}-part2 -R ${MOUNT} ${ZPOOL} -N
zfs mount -a
mount ${DISK}-part1 ${MOUNT}/boot/
swapon /dev/zvol/${ZPOOL}/SWAP
mount --types proc /proc ${MOUNT}/proc
mount --rbind /sys ${MOUNT}/sys
mount --rbind /dev ${MOUNT}/dev
mount --bind /run ${MOUNT}/run
test -L /dev/shm && rm /dev/shm && mkdir /dev/shm
mount --types tmpfs --options nosuid,nodev,noexec shm /dev/shm
chmod 1777 /dev/shm /run/shm
chroot ${MOUNT} /bin/bash
source /etc/profile
export PS1="(chroot) $PS1"
```
## 2. 安装基本系统
### 2.1 校验系统时间
`date`
### 2.2 下载stage3到/mnt/gentoo下
```
cd ${MOUNT}
wget ${STAGE3}
tar xvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```
### 2.3 挂载必要的其他文件系统
```
mount --types proc /proc ${MOUNT}/proc
mount --rbind /sys ${MOUNT}/sys
mount --rbind /dev ${MOUNT}/dev
mount --bind /run ${MOUNT}/run
test -L /dev/shm && rm /dev/shm && mkdir /dev/shm
mount --types tmpfs --options nosuid,nodev,noexec shm /dev/shm
chmod 1777 /dev/shm /run/shm
```
### 2.4 复制DNS信息到新系统下
`cp --dereference /etc/resolv.conf ${MOUNT}/etc/`
### 2.5 进入chroot环境
```
chroot ${MOUNT} /bin/bash
source /etc/profile
export PS1="(chroot) $PS1"
```
### 2.6 配置portage参数
```
nano -w /etc/portage/make.conf
```
#### 部分参数
>COMMON_FLAGS="-O2 -march=znver2 -pipe"<Br/>
CHOST="x86_64-pc-linux-gnu"<Br/>
CPU_FLAGS_X86="aes avx avx2 f16c fma3 mmx mmxext pclmul popcnt rdrand sha sse sse2 sse3 sse4_1 sse4_2 sse4a ssse3"<Br/>
MAKEOPTS="-j13"<Br/>
EMERGE_DEFAULT_OPTS="--keep-going --with-bdeps=y --autounmask-write=y --jobs=2 -l"<Br/>
AUTO_CLEAN="yes"<Br/>
GENTOO_MIRRORS="https://mirrors.ustc.edu.cn/gentoo"<Br/>
#FEATURES="${FEATURES} -userpriv -usersandbox -sandbox"<Br/>
GRUB_PLATFORMS="efi-64"<Br/>
ACCEPT_KEYWORDS="amd64"<Br/>
ACCEPT_LICENSE="*"<Br/>
L10N="en-US zh-CN en zh"<Br/>
LINGUAS="en_US zh_CN en zh"<Br/>
VIDEO_CARDS="nvidia"<Br/>
INPUT_DEVICES="libinput"<Br/>
FUCKDE="-gnome -gnome-shell -gnome-online-accounts -gnome-keyring -nautilus -kde"<Br/>
#USE<Br/>
MASKU="-wayland -bindist -jumbo-build -mdev -oss -systemd -qtwebengine -webengine -consolekit -bluetooth -networkmanager -ppp -gnome -gnome-shell -gnome-online-accounts -gnome-keyring -nautilus -kde -plymouth -icu"<Br/>
LIKEU="X xinerama elogind udev blkid efi acpi ccache dbus policykit udisks alsa pulseaudio jack lv2 sf2 grub cjk emoji zsh-completion nvenc"<Br/>
USE="\${MASKU} \${LIKEU}"<Br/>
#Ccache<Br/>
#FEATURES="parallel-fetch ccache"<Br/>
#CCACHE_DIR="/var/cache/ccache"<Br/>

### 2.7 同步镜像站中最新的软件快照
`emerge-webrsync`
### 2.8 配置git方式来同步ebuild数据库
```
emerge -av dev-vcs/git dev-util/ccache app-editors/neovim
eselect vi set nvim
eselect editor set vi
export CCACHE_DIR="/var/cache/ccache"
mkdir -p /etc/portage/repos.conf/
vi /etc/portage/repos.conf/gentoo.conf
```
>[DEFAULT]<Br/>
main-repo = gentoo<Br/>
[gentoo]<Br/>
location = /var/db/repos/gentoo<Br/>
sync-type = git<Br/>
sync-uri = https://mirrors.ustc.edu.cn/gentoo.git<Br/>
sync-depth = 1
auto-sync = yes
```
rm -rf /var/db/repos/gentoo
emerge --sync
```
### 2.9 选择profile
```
eselect profile list
eselect profile set 5
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
echo "options zfs zfs_arc_min=268435456" > /etc/modprobe.d/zfs.conf
echo "options zfs zfs_arc_max=536870912" >> /etc/modprobe.d/zfs.conf

emerge -av sys-kernel/linux-firmware sys-kernel/genkernel sys-kernel/gentoo-kernel-bin zfs zfs-kmod
rm -rf /etc/hostid && zgenhostid
vi /etc/genkernel.conf
genkernel initramfs --compress-initramfs --kernel-config=/usr/src/linux/.config --makeopts=-j`nproc`
```
## 4. 配置系统
### 4.1 本地化配置
```
echo "Asia/Shanghai" > /etc/timezone
rm -rf /etc/localtime
emerge --config sys-libs/timezone-data
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
echo "zh_CN.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
eselect locale set en_US.UTF-8
env-update && source /etc/profile && export PS1="(chroot) $PS1"
```
### 4.2 配置fstab (这里使用archlinux的一个工具genfstab来自动生成fstab)
```
git clone https://github.com/26hz/install-tools.git
cd install-tools
./genfstab -U -p / >> /etc/fstab
因为zfs能够自行管理dataset，不需要在fstab中指定挂载选项，之后需要删除zfs文件系统的相关段落
```
### 4.3 配置主机名
`vi /etc/conf.d/hostname`
### 4.4 配置网络
`vi /etc/conf.d/net`
>config_enp34s0="null"<Br/>
bridge_br0="enp34s0 tap1 tap2 tap3"<Br/>
rc_net_br0_need="net.enp34s0 net.tap1 net.tap2 net.tap3"<Br/>
config_br0="172.16.26.126/16 fd00::32/64"<Br/>
routes_br0="default via 172.16.26.2<Br/>
default via fd00::1"<Br/>
bridge_forward_delay_br0=0<Br/>
bridge_hello_time_br0=1000<Br/>
bridge_stp_state_br0=0<Br/>
config_tap1="null"<Br/>
tuntap_tap1="tap"<Br/>
iproute2_tap1="user root"<Br/>
config_tap2="null"<Br/>
tuntap_tap2="tap"<Br/>
iproute2_tap2="user root"<Br/>
config_tap3="null"<Br/>
tuntap_tap3="tap"<Br/>
iproute2_tap3="user root"<Br/>
````
ln -s /etc/init.d/net.lo /etc/init.d/net.br0
ln -s /etc/init.d/net.lo /etc/init.d/net.enp34s0
ln -s /etc/init.d/net.lo /etc/init.d/net.tap1
ln -s /etc/init.d/net.lo /etc/init.d/net.tap2
ln -s /etc/init.d/net.lo /etc/init.d/net.tap3
rc-update add net.br0 default
rc-update add net.enp34s0 default
rc-update add net.tap1 default
rc-update add net.tap2 default
rc-update add net.tap3 default
````
### 4.5 配置hosts
`vi /etc/hosts`
>127.0.0.1 localhost<Br/>
::1 localhost<Br/>
127.0.1.1 gentoo.localdomain gentoo
### 4.6 配置硬件时钟
`vi /etc/conf.d/hwclock`
>clock="local"
### 4.7 配置密码规则
`vi /etc/security/passwdqc.conf`<Br/>
`passwd`
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
rc-update add dbus default
```
## 7. 配置引导程序
```
emerge --ask --verbose sys-boot/grub
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Gentoo
vi /etc/default/grub
grub-mkconfig -o /boot/grub/grub.cfg
```
>例如:<Br/> 
`GRUB_CMDLINE_LINUX="dozfs=cache quiet amd_iommu=on iommu=pt loglevel=5 nowatchdog"`<Br/>
`GRUB_CMDLINE_LINUX_DEFAULT="spectre_v1=off spectre_v2=off spec_store_bypass_disable=off pti=off"`
## 8. 新建用户
```
useradd -mG users,wheel,portage,usb,input,audio,video,sys,adm,tty,disk,lp,mem,news,console,cdrom,sshd,kvm,render,lpadmin,cron,crontab -s /bin/zsh jaus
passwd jaus
echo "permit keepenv nopass :wheel" > /etc/doas.conf
```
## 9. 使用zfs snapshot创建新系统快照
```
zfs snapshot zroot/ROOT/default@install
```
## 10. 重启
```
exit
umount /mnt/gentoo/boot
umount -Rl /mnt/gentoo/{dev,proc,sys,run,}
swapoff /dev/zvol/zroot/SWAP
zfs umount -a
zpool export -f zroot
reboot
```
