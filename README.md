# 64 bit ARM NetBSD Running on AWS T4g Instances
Commands to crosss compile NetBSD for aarch64 target and create minimal OS image to deploy to AWS EC2 arm-based instances (T4g).

## Steps
1. Get NetBSD source
```
ftp -i -p ftp.NetBSD.org
# user = anonymous
# pass = <empty>
ftp> cd pub/NetBSD/NetBSD-9.3/source/sets/
ftp> mget gnusrc.tgz sharesrc.tgz src.tgz syssrc.gz
ftp> quit

for file in *.tgz
do
tar -xzf $file
done
```
2. Use build.sh to cross-compile toolchain, kernel and sets
```
cd usr/src
build.sh -U -O ../../obj -j$(nproc) -m evbarm -a aarch64 tools
build.sh -U -u -O ../../obj -j$(nproc) -m evbarm -a aarch64 release
```
3. Use toolchain to compile application
4. Make GPT disk image with EFI and boot partitions
```
dd if=/dev/zero of=./netbsd-aarch-uefi.img bs=1G count=3
losetup -f ./netbsd-aarch-uefi.img
$TOOLDIR/bin/nbgpt /dev/loop0 create -A
$TOOLDIR/bin/nbgpt /dev/loop0 add -l EFI -s 200m -t efi
$TOOLDIR/bin/nbgpt /dev/loop0 add -l BOOT -t ffs
partprobe /dev/loop0
```
5. Format EFI partition Fat 32, mount partition and copy bootloader
```
mkfs.fat -F32 /dev/loop0p1
mkdir -p /mnt/netbsd-efi
mount /dev/loop0p1 /mnt/netbsd-efi
mkdir -p /mnt/netbsd-efi/EFI/BOOT
cp $RELEASEDIR/evbarm/installation/misc/bootaa64.efi /mnt/netbsd-efi/EFI/BOOT/bootaa64.efi
```
6. Create directory for NetBSD fs root, copy kernel as /netbsd and extract and copy required sets
```
mkdir netbsd-boot
cp $RELEASEDIR/evbarm/binary/kernel/netbsd-GENERIC64.gz netbsd-boot/netbsd.gz
cp $RELEASEDIR/evbarm/binary/sets/{base,etc}.tgz netbsd-boot/
gunzip netbsd.gz
tar -xzpf netbsd-boot/*.tgz
```
7. Edit fstab, edit rc and copy application to the fs
8. Make NetBSD fs image using nbmakefs from toolchain and write it to disk image boot partition
```
$TOOLDIR/bin/nbmakefs -t ffs -o bsize=4096 -b 4000 netbsd-boot.img netbsd-boot
dd if=./netbsd-boot.img of=/dev/loop0p2
```
9. Test the image using qemu aarch64 emulator
10. Create an EC2 instance, attach an EBS volume, upload the disk image to the instance, write the image to the volume, detatch the volume, create AMI from volume
11. Deploy a T4g instance using the AMI