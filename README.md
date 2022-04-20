# PetaLinux Guide: How to Create a PetaLinux Environment (2019.1) <!-- omit in toc -->
### By: Grace Palenapa <!-- omit in toc -->
### Updated 04.20.2022 <!-- omit in toc -->

---------------------

## Table of Contents <!-- omit in toc -->
- [Installing PetaLinux](#installing-petalinux)
- [Creating a Petalinux Project from Custom Hardware Design](#creating-a-petalinux-project-from-custom-hardware-design)
- [Creating a Petalinux Project from Prebuilt .bsp File](#creating-a-petalinux-project-from-prebuilt-bsp-file)
- [Accessing the Hardware using UIO driver](#accessing-the-hardware-using-uio-driver)
- [Setting Environment Variables to Access Xilinx Licenses](#setting-environment-variables-to-access-xilinx-licenses)
  - [Accessing Floating Licenses in Linux](#accessing-floating-licenses-in-linux)
- [Additional Reference Links](#additional-reference-links)

---------------------

## Installing PetaLinux 

1. Add i386 using the package management system, dpkg.
    ```
    sudo dpkg --add-architecture i386
    ```

2. Change Ubuntu's shell from dash to bash.
    ```
    sudo dpkg-reconfigure dash
    ```

3. Install necessary packages.
    ```
    sudo apt-get update

    sudo apt-get install -y gcc git make net-tools libncurses5-dev tftpd zlib1g-dev libssl-dev flex bison libselinux1 gnupg wget diffstat chrpath socat xterm autoconf libtool tar unzip texinfo zlib1g-dev gcc-multilib build-essential libsdl1.2-dev libglib2.0-dev zlib1g:i386 screen pax gzip gawk`\
    ```

4. Create TFTP Server
    - configure the TFTP server as shown below (had some trouble here, so adjust as needed?)
        ```
        service tftp 
        {
        protocol = udp 
        port = 69 
        socket_type = dgram 
        wait = yes 
        user = nobody 
        server = /usr/sbin/in.tftpd 
        server_args = /tftpboot 
        disable = no
        }
        ```

    - make /tftpboot and assign proper permissions
        ```
        sudo mkdir /tftpboot
        sudo chmod -R 777 /tftpboot
        sudo chown -R nobody /tftpboot 
        ```

5. Make the PetaLinux directory with needed permissions.
    ```
    sudo mkdir -p ~/PetaLinux/2019.1/
    sudo chmod -R 755 /tools/Xilinx/PetaLinux/2019.1/
    sudo chmod 777 ./Downloads/petalinux-v2019.1-final-installer.run
    sudo chown -R <user>:<user> /tools/Xilinx/PetaLinux/2019.1/
    ```

6. Run PetaLinux Installer. 
    ```
    ~/Downloads/petalinux-v2019.1-final-installer.run /tools/Xilinx/PetaLinux/2019.1/
    ```

7. Run PetaLinux.
    ```
    source /tools/Xilinx/PetaLinux/2021.2/settings.sh
    ```

    - now you can run PetaLinux Commands in your project environment

***NOTE: PetaLinux Installation Guide indicates to run the PetaLinux installer in /opt/pkg/ folder. I ran the installer in ~/PetaLinux/2019.1/, and it works fine for me.***

---------------------

## Creating a Petalinux Project from Custom Hardware Design 

1. Build Vivado project like normal
    - make sure these are configured for the Zynq in Vivado 
      - one Triple Timer Counter (required)
      - external memory controller with at least 32 MB of memory (required)
      - UART for serial console (required)
      - non-volatile memory (optional)
      - ethernet (optional, essential for ethernet access)
    - run synthesis, implementation, and generate bitstream
    - export hardware to Xilinx SDK, include bitstream

2. Run PetaLinux using below command
    ```
    source ~/PetaLinux/2019.1/settings.sh
    ```

3. Create a PetaLinux project. I usually make mine in the same directory as the Vivado project
    ```
    petalinux-create --type project --template zynq --name <your-petalinux-project>
    ```

4. Change to PetaLinux project directory 
    ```
    cd <path-to-petalinux-project-root>
    ```

5. Import hardware description 
   ```
   petalinux-config --get-hw-description <path-to-SDK-folder-with-hdf-file>
   ```

6. Create an application to run on PetaLinux
    ```
    petalinux-create --type apps --name <your-app> --template c++ --enable
    ```
    - go to `<petalinux-project-path>/project_spec/meta-user/recipes_apps/` to find the application source code

7. Go the user-packages submenu using the below command and confirm that your-app is selected
    ```
    petalinux-config -c rootfs
    ```
    - the --enable flag in step 6 should already do this

8. Use `petalinux-config` command to edit more settings
    - `-c kernel` to edit the kernel
    - `-c rootfs` to edit root file system settings
    - To determine the boot method, refer to __Booting the FPGA__ in step 11
    - If using the SD card for the root file system, refer to __Booting the FPGA: Boot via JTAG__ in step 11 for formatting and partitioning the SD card.

9.  Build the application and then the whole project
    ```
    petalinux-build -c <your-app>
    petalinux-build
    ```

10. To create a new boot image file, run...
    ```
    petalinux-package --boot --fsbl images/linux/zynq_fsbl.elf --fpga images/linux/system.bit
    ```

11. Booting the FPGA 

    > Boot via JTAG 
    > 
    > 1. Enter configuration GUI through `petalinux-config` command 
    > 2. Go to `Subsystem AUTO Hardware Settings --> Advanced bootable images storage settings`
    > 3. Change the `boot image settings --> image storage media` setting to `primary flash`
    > 4. Execute the below code to boot up PetaLinux on the FPGA.
    >   ```
    >   petalinux-boot --jtag --kernel --fpga
    >   ```
    >
    > Boot via SD card
    > 
    > 1. Follow the first three steps in __Boot via JTAG__ and choose `primary SD` instead
    > 2. Format and partition the SD card
    >    - Leave 2MB-8MB of free space before the first partition (I used 4MB).
    >    - Format a minimum of 60MB in fat32 for the first partition.
    >    - Format the remaining free space in ext4.
    >
    >    - ***NOTE: The fat32 partition will hold the kernel image and the device tree, and the ext4 partiton will hold the root file system.***
    > 
    > 3. Copy and extract the necessary files into the partitions. The mount point may be different than /media/BOOT and /media/rootfs unless you create those folders and mount the SD card to those points.
    >   ```
    >   sudo cp /images/linux/image.ub /media/BOOT                  ; /media/BOOT refers 
    >                                                                   to the fat32 partition
    >   sudo cp /images/linux/system.dtb /media/BOOT
    >   sudo cp /images/linux/BOOT.bin /media/BOOT 
    >   sudo tar xvf /images/linux/rootfs.tar.gz -C /media/rootfs   ; /media/rootfs/ refers 
    >                                                                   to the ext4 partition
    >   ```
    > ***NOTE: You can also copy `/images/linux/rootfs.cpio` into the `/media/rootfs/` partition instead of extracting the files.
    >
    > 4. Umount/Eject SD card from your computer.

12. When PetaLinux is booted and running on the board, access the on-board PetaLinux Environment through the serial port.
    ```
                                                    ; execute these commands if needed
    lsusb                                           ; to see if your board is connected to
                                                        your machine
    dmesg | grep -i cp210 | grep -i tty             ; to find which USB tty port the board is 
                                                        connected to
    ```
    - Open a serial port terminal and configure it as such:
      - Port: that tty port you found
      - Bits: 8
      - Stopbits: 1
      - Parity: none
      - Flow control: none
  
13. Login with username `root` and password `root`, unless changed otherwise in the `petaliunx-config` GUI.
14. To run the application, run the below code 

    ```
    ls /usr/bin/
    /usr/bin/your-app
    ```

---------------------

## Creating a Petalinux Project from Prebuilt .bsp File

link to .bsp documentation and installer [here](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/embedded-design-tools.html)

Prebuilt example projects can be found on above on the Xilinx PetaLinux Documentation page. To build the project from the .bsp file and run the prebuilt images, execute the below command.

```
petalinux-create --type project -s <path-to-bsp-file>
petalinux-boot --jtag --prebuilt 3                          ; to boot the FPGA via JTAG
petalinux-boot --qemu --prebuilt 3                          ; to emulate the FPGA on the computer, doesn't actually boot the FPGA
```

---------------------

## Accessing the Hardware using UIO driver

UIO = Userspace I/O
Make sure you do all of these to connect the UIO driver to your peripheral.

This section is based on Vivado project with a simple custom IP that allows the user to turn the on-board LEDs on and off.

- Use `petalinux-config -c kernel` to select the UIO driver capabilities.
  - Go to `Device Drivers --> Userspace I/O drivers`.
  - Select `Userspace platform driver with generic irq and dynamic memory`
  - Exit the configuration GUI.
- Go into the `<your-petalinux-environment>/project-spec/meta-user/recipes-kernel/linux/linu-xlnx/user...cfg`, and insert the following lines into the file.

    ```
    CONFIG_UIO = y
    CONFIG_UIO_PDRV = y
    CONFIG_UIO_PDRV_GENIRQ = y
    ```

- Go into the `<your-petalinux-environment>/project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi`, and make the file look like the below code to add to the device tree.

    ```
    /include/ "system-conf.dtsi"
    / {
        led_ctrl@43c00000 {
            compatible = "axi_gpio_0, generic-uio, ui_pdrv"; 
            status = "okay";
            };

        chosen {      
            bootargs = "console=ttyPS0,115200 earlyprintk uio_pdrv_genirq.of_id=generic-uio";
        }; 
    };

    &led_ctrl_0 {
        compatible = "generic-uio";
    };
    ```
    - Adjust as needed depending on with IP block you want to use with the UIO driver.
    - Other tutorials online used the AXI GPIO.

- To add other device drivers, edit this device tree file and the user...cfg file.
 
To make sure the UIO device driver is installed...
- Boot up petalinux and check the `/dev` folder for `uio0` or others if have multiple hardware using that driver.
- You can also check the `/sys/class/` folder to check for the UIO driver
- Go into `/sys/class/uio/uio0/map/map0/` and use the following commands to view the mapping info.

    ```
    more addr
    more name
    more offset
    more size
    ```

To turn on and off the LEDs, use the following commands and observe the output.

```
devmem 0x43c00000 w 0x1     ; turns on the first light
devmem 0x43c00000 w 0x2     ; turns on the second light
devmem 0x43c00000 w         ; turns on the first two lights
```

You can also use the peekpoke program that's prebuilt into your PetaLinux project environment, which will write the indicated value to your specified register.

```
poke 0x43c00000 0x1     ; turns on the first light
peek 0x43c00000         ; view status of the register
poke 0x43c00000 0xF     ; turns on all the lights
```

## Setting Environment Variables to Access Xilinx Licenses

### Accessing Floating Licenses in Linux

Use the following commands in the terminal to reset your environment variable to your `~/.bashrc`.

```
$ echo "export VAR="value"" >> ~/.bashrc && source ~/.bashrc 
```

To access the floating point Xilinx licenses, enter...

```
$ echo "XILINXD_LICENSE_FILE=<floating-point-path>" >> ~/.bashrc
$ source ~/.bashrc
```

---------------------

## Additional Reference Links

[PetaLinux Tools User Guide: Installation Guide](https://www.xilinx.com/support/documents/sw_manuals/petalinux2014_2/ug976-petalinux-installation.pdf)

[PetaLinux Tools Documentation: Reference Guide](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2021_1/ug1144-petalinux-tools-reference-guide.pdf)

__MOST HELPFUL__
[Design Flow for a Custom FPGA Board in Vivado and PetaLinux](https://www.hackster.io/news/design-flow-for-a-custom-fpga-board-in-vivado-and-petalinux-b998c0b4f9f7)

[Installing Vivado, Vitis, & PetaLinux 2021.2 on Ubuntu 18.04](https://www.hackster.io/whitney-knitter/installing-vivado-vitis-petalinux-2021-2-on-ubuntu-18-04-0d0fdf)

[Custom Application Creation in PetaLinux on the Zynqberry](https://www.hackster.io/news/custom-application-creation-in-petalinux-on-the-zynqberry-c946ec2f32f5)

__TO ACCESS PETALINUX THROUGH SERIAL PORT__
[Install and Run Ubuntu 20.04 on Xilinx Zynq-7000 SoC ZC706 Through a Bootable SD Card, and Access It Through the Serial Port](https://enes-goktas.medium.com/install-and-run-ubuntu-20-04-on-zynq-7000-soc-zc706-evaluation-board-c5fef2423c98)

__USING UIO LINUX DRIVER TO ACCESS CUSTOM IP AXI REGISTERS__
[GPIO and Petalinux (part one listed here)](https://www.linkedin.com/pulse/gpio-petalinux-part-1-roy-messinger?trk=pulse-article_more-articles_related-content-card)

[How to Set and Unset Environment Variables on Linux](https://devconnected.com/how-to-set-and-unset-environment-variables-on-linux/)
