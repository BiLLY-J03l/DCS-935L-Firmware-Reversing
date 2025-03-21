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

- look for harcoded credentials
  - the etc/passwd file is a symlink to mnt/flash/config/passwd, which didn't exist at the time.
  - the etc/passwd_default file only has an admin account with no hardcoded password.
    
  - ![image](https://github.com/user-attachments/assets/a496bf0d-cd91-4e85-8c48-d1f453a48854)

- Check etc/inittab for initialization scripts
  
![image](https://github.com/user-attachments/assets/b8a13bb9-a667-4f4b-81a7-2f33d52ca3a3)

  - interesting finding, etc/rc.d/rcS script is run on booting/starting up of the iot camera.

- cat the etc/rc.d/rcS
  
![image](https://github.com/user-attachments/assets/49334169-b467-4c7f-a4f6-1aab6ae7d9f2)


## User-Mode-Emulation on Raspberry Pi 5

- install qemu-user-static

      apt install qemu-user-static
  
![image](https://github.com/user-attachments/assets/8e9dbe13-fcc9-406e-b127-444c6478b520)

- use qemu-mips-static on bin/busybox binary

      qemu-mips-static bin/busybox
  
![image](https://github.com/user-attachments/assets/0df5801e-90fd-4ad9-8174-4c0a71db9913)

- mount /dev, /sys and /proc directories to their equivalent squashfs-root directory
  
  - ![image](https://github.com/user-attachments/assets/1d60bfd6-0c90-4fc0-9c93-d0f3c3dddee5)
    
  - A bind mount allows you to mount an existing directory or file onto another location in the filesystem. The key point is that it does not copy the contents of the directory or file but instead makes the original directory or file appear at another location in the filesystem.
    
  - This is a common practice when working with Linux systems in a chroot environment or when working with certain containers or live systems where the system's core directories (/dev, /sys, /proc) need to be accessible within a different filesystem.
    
  - Why THOSE directories?
    - Mounting /proc ensures that the environment can access information about the system’s current state, like process IDs, system limits, etc.
      
    - When mounting /dev from the host system into the chroot or the mounted filesystem, it ensures that the chrooted environment or the system within the squashfs has access to devices like disks, USBs, etc.
      
    - Mounting it makes sure that the kernel's internal state and other critical information are available inside the environment.

- Use chroot to change the root directory to the squashfs environment, we're like SSHing into our iot camera.

      chroot . /bin/sh

![image](https://github.com/user-attachments/assets/5dcd5e63-92c0-46b3-b278-b40665f432e7)
![image](https://github.com/user-attachments/assets/d487372b-9195-4f4b-9c76-553fc48f1047)

- Run the /etc/rc.d/rcS script

![image](https://github.com/user-attachments/assets/de863591-d5b4-48d3-a39a-4d3af5d39716)

- check for listening ports that has been opened

      netstat -tln
  - -t: TCP sockets
  - -l: Listening sockets
  - -n: don't resolve names

![image](https://github.com/user-attachments/assets/103d2550-1fcc-4b6f-8962-b8b795c2202e)

- It was supposed to have a web port open for the Admin Panel, but there was a problem in the init scripts
- I traced the problem to the file /etc/rc.d/rcS.d/S90httpd-0
  -It takes the http_port variable from executing '/usr/sbin/userconfig -read HTTP port', which when executed doesn't return any valid port number.
  
  ![image](https://github.com/user-attachments/assets/120cb2ce-e2b8-4fce-b8ba-415261169383)

  -It takes the https_enable variable from executing '/usr/sbin/userconfig -read HTTPS Enable', which when executed doesn't return any valid value as well.

  ![image](https://github.com/user-attachments/assets/c0fa56b0-0d85-4d88-a530-56a8b0b93bfd)

- I patched /etc/rc.d/rcS.d/S90httpd-0 file, and hardcoded the port number to 8088

![image](https://github.com/user-attachments/assets/ac0e5a12-fca1-4a46-8cd2-bffb19d66f3a)

- I chrooted into the environment again and rerun the init script and BINGO, port 8088 is open
  
![image](https://github.com/user-attachments/assets/6528ac6e-d2e9-4c52-adae-7d50622ecbfe)
![image](https://github.com/user-attachments/assets/9f565511-36a2-4ea3-a2bc-5e3249738b95)





  

