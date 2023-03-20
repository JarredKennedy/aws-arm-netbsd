# Cross-compiling a Program for NetBSD aarch64

- Using aarch64--netbsd-gcc to compile eventually calls the linker and it wants to link against `crt0.o`, `crtbegin.o`, `crtend.o`.
- These files are in usr/lib from the _comp_ set.
- aarch64--netbsd-gcc calls the linker specifying those files. It specifies them at the beginning of the parameter list, before any user-supplied -L. Because of this (I think), it is not able to find these files and the linker fails. Passing `-Wl,-rpath=...` or `-Wl,-R...` did not resolve the problem Using `-v` to see the linker script confirmed the options were being passed to `collect2`.
- The only way I could get it to link is to change into the usr/lib directory and compile from there.
- When using the shared standard lib, the linker options include -lgcc_s which links against libgcc_s.so. This library is not in the _comp_ set, it is in the _base_ set.
- With all of those issues addressed, a program can be compiled. When transferring the executable to a aarch64 NetBSD instance, it cannot be executed: _Cannot execute ELF binary filename_
- Checking other binaries on the instance, shows they are also 64bit ELF executables:
```
file myprogram
ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /usr/libexec/ld.elf_so, not stripped

file /bin/echo
ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /libexec/ld.elf_so, for NetBSD 9.3, not stripped
```
- Binaries included on NetBSD are somehow "for NetBSD" and the binary compiled with aarch64--netbsd-gcc is not.
- Running objdump on the compiled program and NetBSD binaries shows the NetBSD binaries include sections `.note.netbsd.ident` and `.note.netbsd.pax` and the compiled program does not.
- Running objdump on usr/lib/sysident.o shows it includes these sections. Linking against sysident.o makes the program executable on the NetBSD instance.