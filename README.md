# DCS-935L-Firmware-Hacking
A simple walkthrough to analysing, emulating and hacking DCS-935L firmware

The exact firmware version is DCS-935L FW 1.06.02

- I'll start up the Pi 5 and download the firmware from the official D-Link website https://support.dlink.com.au/Download/download.aspx?product=DCS-935L

![image](https://github.com/user-attachments/assets/aa7827ba-2246-4ea1-9376-1fa706af3034)

- Run strings on the .bin file to check for interesting entries that migh help us understand the structure of the firmware

![image](https://github.com/user-attachments/assets/59688f78-8197-4592-8a9c-0c568359bb20)

- Here, we see a LZMA which is a compression algorithm for linux files.

- We can use Binwalk, which is a tool for searching binary images for embedded files and executable code, to gather more information about the .bin file

![image](https://github.com/user-attachments/assets/ac9ee776-bbae-4669-8893-7a7952d8b88c)

  - LZMA compressed data
  - Squashfs, which is a compressed read-only file system for Linux.

- Now we carve out the squashfs using DD

      dd if=DCS-935L_A1_FW_1.06.02_20150717_r3108.bin skip=1431586 bs=1 of=firmware.sqsh

    - if --> read from FILE instead of stdin
    - skip --> skip N ibs-sized input blocks
    - bs --> read and write up to BYTES bytes at a time
    - of --> write to FILE instead of stdout
![image](https://github.com/user-attachments/assets/48c55b8e-be23-4a8b-8a25-5e73e0dccdee)

- uncompress the .sqsh file using unsquashfs
![image](https://github.com/user-attachments/assets/c6cc1f26-778a-4d8e-a29e-53fb72db7fd6)

- navigate to squashfs-root directory and by listing the files, you will just see that it's a linux filesystem
![image](https://github.com/user-attachments/assets/a33b1267-332c-41b7-96f8-07a827da2c97)

- list the bin/ directory, you will see that all the binaries are symbolic links to busybox
![image](https://github.com/user-attachments/assets/150a5332-a619-4843-b61c-5ebca6990ee2)

**- What is BusyBox?**
  - BusyBox is a software suite that provides several Unix utilities in a single executable file. It runs in a variety of POSIX environments such as Linux, Android, and FreeBSD, although many of the tools it provides are designed to work with interfaces provided by the Linux kernel. It was specifically created for embedded operating systems with very limited resources.

- Check what the binary arch is

      file bin/busybox
![image](https://github.com/user-attachments/assets/9cf9727f-4a4b-4743-9f7c-f8fbafc8c88d)
  
  - the binary file is of MIPS architecture
 
## Emulation on Raspberry Pi 5

- install qemu-user-static

      apt install qemu-user-static
![image](https://github.com/user-attachments/assets/8e9dbe13-fcc9-406e-b127-444c6478b520)

