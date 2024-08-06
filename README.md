# RPI4 OS configuration
This configuration enable to create normal and administrative operating system on one device.

### Description

1. **Preparation of an "administrative" Linux system**: This system was designed as a rescue system operating in the initramfs file system and included tools necessary for managing the SD card and loading Linux kernel images.

2. **Loading the "administrative" system**: The device was prepared for user system boot. Memory on the card was divided into three partitions with the following formats: two ext4 partitions and one VFAT partition. Finally, the user system was loaded, including the kernel image and file system.

3. **Bootloader Preparation**: A bootloader was created to determine which system (administrative or user) should be loaded. A WHITE LED was used to indicate that buttons would be checked shortly. After one second, the state of the buttons was read. If no button was pressed, the "user" system was to be loaded. After checking the buttons, the WHITE LED was turned off. If the "user" system was selected, a GREEN LED was lit; if the "administrative" system was chosen, a RED LED was lit.

4. **Preparation of the "user" Linux system**: This system operated with an ext4 file system on the second partition and included a WWW server in the Tornado environment, controlled via a web interface. The server was to provide files from the third partition on the SD card, display a list of these files, and allow file selection for download. The server was also to permit uploading new files to this partition after user authentication.

### Project Restoration Procedure from Attached Archive

The attached archive contained two folders: `admin` and `user`. Each of these folders had to be unpacked using Buildroot. The `admin` folder included files necessary to restore the administrative system project with the appropriate bootloader. The `.config` file was located in the main Buildroot directory. The system image was then built using the `make` command. The second folder contained files needed to restore the user system project. The `.config` file was located in the main Buildroot directory. The `overlay` folder was to be placed one level below the `buildroot-xxx` directory. In the `/buildroot-xxx/board/raspberrypi4-64/` directory, there was a file named `genimage-raspberrypi4-64.cfg`. The system image was built using the `make` command.

### Administrative System and Bootloader Description

A configuration file was prepared by saving the default configuration. Then, a bootloader was written, which allowed for system selection according to the exercise requirements. The `dd` tool was included in the system by default. After building the system image, the `boot.txt` file was processed using the `mkimage` tool into the `boot.scr` file, which was loaded by the device by default. For the laboratory, the bootloader was executed by the internal RPi4 bootloader.

### Modifications to Buildroot Configuration, Kernel, and Other Packages

**Default Buildroot Configuration:**

1. Set the target system architecture by running the command `make raspberrypi4_64_defconfig` in the Buildroot directory.
2. Set the TTY port to `console` in the section: `System configuration/Run a getty.../TTY port`.
3. Set the package source by entering `http://192.168.137.24/dl` in the section: `Build options/Mirrors and Download locations/Primary download site`.
4. Select an external toolchain in the section `Toolchain/Toolchain type` – version: `Arm AArch64 2021.07`.
5. Enable compiler caching - `BR2_CCACHE` in `Build options/Enable compiler cache`. Set the location to `../ccache-br`.
   This configuration was saved as default.

**Further Buildroot Configuration:**

1. Configure appropriate system image generation by selecting in the section: `Filesystem images`
   - `initial RAM filesystem linked into linux kernel`.
2. Add IP address assignment by including the `BR2_PACKAGE_DHCP` package and the `client` suboption (to enable network usage). To add this package, first add the `BR2_PACKAGE_BUSYBOX_SHOW_OTHERS` package.
3. Add the `BR2_PACKAGE_NETPLUG` package (to enable automatic network connection).
4. Select the `BR2_PACKAGE_NTP` package and set the NTP server (to enable system clock synchronization).
5. Set the system name to `BR2_TARGET_GENERIC_HOSTNAME="admin"`.
6. Add packages for formatting:
   - `BR2_PACKAGE_DOSDSTOOLS_MKFS_FAT`
   - `BR2_PACKAGE_E2FSPROGS`
   - `BR2_TARGET_ROOTFS_EXT2_MKFS_OPTIONS`
7. Add packages for mounting devices:
   - `BR2_PACKAGE_UTIL_LINUX_MOUNT`
8. Add packages for file system checking:
   - `BR2_PACKAGE_E2FSPROGS_FSCK`
9. Add packages for resizing partitions:
   - `BR2_PACKAGE_FLOT_RESIZE`
   - `BR2_PACKAGE_E2FSPROGS_RESIZE2FS`
10. Add tools for bootloader preparation:
    - `BR2_PACKAGE_UBOOT_TOOLS_MKIMAGE`
    - `BR2_PACKAGE_HOST_IMX_MKIMAGE`

**U-Boot Configuration:**

In the `Bootloaders` section, the `U-Boot` option was selected. The `Board defconfig` was set to `rpi_4`. After building the system image, the script was modified using the `mkimage` tool:
```
mkimage -T script -C none -n 'Start script' -d boot.txt boot.scr
```

### User System Description

The system configuration was started from the default configuration used for the administrative system. The file system type for the user system was then added. Subsequently, packages for internet usage and those required to run the Tornado server were added. Finally, the created server was placed in a directory that acted as an overlay on the system with the necessary files for running the server in daemon mode. Upon booting the system on the RPi4, it was found that a more advanced version of the start-stop-daemon program was required to start the server, which was added using the `BR2_PACKAGE_START_STOP_DAEMON` package.

### Modifications to Buildroot Configuration, Kernel, and Other Packages

**Buildroot Configuration:**

1. Load the default configuration as for the administrative system.
2. Configure appropriate system image generation by selecting in the section: `Filesystem images`
   - `ext4`.
3. Change the size from 32M to 256M in the file `board/raspberrypi4-64/genimage-raspberrypi4-64.cfg`.
4. Add IP address assignment by including the `BR2_PACKAGE_DHCP` package and the `client` suboption (to enable network usage). To add this package, first add the `BR2_PACKAGE_BUSYBOX_SHOW_OTHERS` package.
5. Add the `BR2_PACKAGE_NETPLUG` package (to enable automatic network connection).
6. Select the `BR2_PACKAGE_NTP` package and set the NTP server (to enable system clock synchronization).
7. Set the system name to `BR2_TARGET_GENERIC_HOSTNAME="stanislaw_tabisz"`.
8. Choose the package that adds an extended start-stop-daemon program: `BR2_PACKAGE_START_STOP_DAEMON`.
9. Select the following packages to enable operation.
10. Add rootfs overlay: `BR2_ROOTFS_OVERLAY=../overlay`.
11. Add the Tornado server:
    - `BR2_PACKAGE_PYTHON3`
    - `Toolchain -> Enable WCHAR support`
    - `BR2_PACKAGE_PYTHON_TORNADO`
    - `BR2_PACKAGE_PYTHON_URLLIB3`

### Testing the Implemented System

After preparing the bootloader and administrative system image, they were loaded into the appropriate locations on the first SD card partition. The `uboot` name was changed to `Image` and loaded as the boot system – an overlay on the bootloader into the `user` directory. System images were then loaded into this folder: the administrative system as `Image.admin` and the user system as `Image.user`. Finally, the processed `boot.scr` file was added to the main directory of the first partition.

The administrative system was then booted, and two partitions were created on the SD card with ext4 formatting using the commands:
```
fdisk /dev/mmcblk0
D
3
D
2
N
P
2
<enter>
+500M
N
P
3
<enter>
+500M
W
```
and
```
mkfs.ext4 /dev/mmcblk0p2
mkfs.ext4 /dev/mmcblk0p3
```
Data from the user system file system was transferred using the command:
```
wget -O - 192.168.145.101:8000/rootfs.ext2 | dd of=/dev/mmcblk0p2 bs=4096
```
Finally, the transferred file system image was resized to the partition size with:
```
resize2fs /dev/mmcblk0p2
```

After these operations, the system was restarted, and the user system was booted. Console messages indicated that the third partition was mounted and the file-serving server was running.

On the departmental computer, a webpage was opened at the address reported by the RPi4 device. After logging in, it was possible to upload and download files contained on the third partition.

### References

**Bootloader Sources:**
- [eLinux Bootroot Lab](https://elinux.org/images/4/4b/Getting-Started-With-Buildroot-Lab-ELC2018.pdf)

**Bootloader Button Configuration Sources:**
- [Denx Wiki](https://www.denx.de/wiki/publish/DULG/to-delete/UBootCmdGroupMisc.html)
- [Digi U-Boot Reference Manual](https://hub.digi.com/dp/path=/support/asset/u-boot-reference-manual/)
- [Denx Wiki Manual](http://www.denx.de/wiki/DULG/Manual)
- [Xilinx Wiki on U-Boot GPIO Driver](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842147/U-Boot+GPIO+Driver)

**Disk Partitioning in Linux Sources:**
- [Linuxiarze Partitions](https://linuxiarze.pl/partycje4/)
- [TLDP fdisk Partitioning](https://tldp.org/HOWTO/Partition/fdisk_partitioning.html)
- [PhoenixNAP Partition Creation](https://phoenixnap.com/kb/linux-create-partition)
- [OpenSource Partitioning](https://opensource.com/article/18/6/how-partition-disk-linux)

**Disk Resizing in Linux Sources:**
- [Resize2fs Command Examples](https://www.thegeekdiary.com/resize2fs-command-examples-in-linux/)

**Tornado Server Creation Sources:**
- [Tornado GitHub Demos](https://github.com/tornadoweb/tornado/tree/master/demos)
- [Raspberry Lab GitHub](https://github.com/H4kan/raspberry_lab/tree/main/app)
- [Alejandro Bernardis Gist](https://gist.github.com/alejandrobernardis/1790864)
- [Fearcat Blog](https://blog.fearcat.in/a?ID=01600-6a465420-e291-4093-a051-b889d3e64ad3)
