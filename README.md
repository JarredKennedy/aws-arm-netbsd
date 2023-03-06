# 64 bit ARM NetBSD Running on AWS T4g Instances
Commands to crosss compile NetBSD for aarch64 target and create minimal OS image to deploy to AWS EC2 arm-based instances (T4g).

## Steps
1. Get NetBSD source
2. Use build.sh to cross-compile toolchain, kernel and sets
3. Use toolchain to compile application
4. Make GPT disk image with EFI and boot partitions
5. Format EFI partition Fat 32, mount partition and copy bootloader
6. Create directory for NetBSD fs root, copy kernel as /netbsd and extract and copy required sets
7. Edit fstab, edit rc and copy application to the fs
8. Make NetBSD fs image using nbmakefs from toolchain and write it to disk image boot partition
9. Test the image using qemu aarch64 emulator
10. Create an EC2 instance, attach an EBS volume, upload the disk image to the instance, write the image to the volume, detatch the volume, create AMI from volume
11. Deploy a T4g instance using the AMI