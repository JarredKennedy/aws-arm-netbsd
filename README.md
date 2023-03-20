# 64 bit ARM NetBSD Running on AWS T4g Instances
Commands to crosss compile NetBSD for aarch64 target and create minimal OS image to deploy to AWS EC2 arm-based instances (T4g).

_/dev/loop0 is used to demonstrate a generic device. In practice the loop device might be different._

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
rm $file
done
```
2. Use build.sh to cross-compile toolchain, kernel and sets
```
cd usr/src
./build.sh -U -O ../../obj -j$(nproc) -m evbarm -a aarch64 tools
./build.sh -U -u -O ../../obj -j$(nproc) -m evbarm -a aarch64 release
```
3. Use toolchain to compile application  
[See here](program.md)
4. Create directory for NetBSD fs root, copy kernel as /netbsd and extract and copy required sets
```
cd ../../
mkdir netbsd-boot
cp $RELEASEDIR/evbarm/binary/kernel/netbsd-GENERIC64.gz netbsd-boot/netbsd.gz
cp $RELEASEDIR/evbarm/binary/sets/{base,etc}.tgz netbsd-boot/
cd netbsd-boot
gunzip netbsd.gz
for file in *.tgz; do tar -xzf $file; done
rm *.tgz
```
5. Edit fstab, edit rc and copy application to the fs
```
cat <<EOF > etc/fstab
NAME=BOOT   /   ffs rw,log  1 1
EOF

vi etc/rc.conf
rc_configured=YES
dhcpcd=YES
```
6. Make NetBSD fs image using nbmakefs from toolchain and write it to disk image boot partition
```
$TOOLDIR/bin/nbmakefs -t ffs -o bsize=4096 -b 15% netbsd-boot.img netbsd-boot
```
7. Make GPT disk image with EFI and boot partitions
```
imgsize=$(wc -c netbsd-boot.img | grep -o -e '[0-9]*' | tr -d '\n')
# 200MiB for EFI + fs image size for boot + 1MiB padding
size=$(((200*1024*1024)+$imgsize+(1024*1024)))
nchunks=$((($size+4095)/4096))

dd if=/dev/zero of=./netbsd-aarch-uefi.img bs=4096 count=$nchunks

$TOOLDIR/bin/nbgpt ./netbsd-aarch-uefi.img create -A
$TOOLDIR/bin/nbgpt ./netbsd-aarch-uefi.img add -l EFI -s 200m -t efi
$TOOLDIR/bin/nbgpt ./netbsd-aarch-uefi.img add -l BOOT -t ffs

losetup -f ./netbsd-aarch-uefi.img
partprobe /dev/loop0
```
8. Create EFI partition with bootloader and write filesystem image to boot partition.
```
mkfs.fat -F32 /dev/loop0p1
mkdir -p /mnt/netbsd-efi
mount /dev/loop0p1 /mnt/netbsd-efi
mkdir -p /mnt/netbsd-efi/EFI/BOOT
cp $RELEASEDIR/evbarm/installation/misc/bootaa64.efi /mnt/netbsd-efi/EFI/BOOT/bootaa64.efi
dd if=./netbsd-boot.img of=/dev/loop0p2
```
9. Test the image using qemu aarch64 emulator
10. Create an EC2 instance, attach an EBS volume, upload the disk image to the instance, write the image to the volume, detatch the volume, create AMI from volume
11. Deploy a T4g instance using the AMI