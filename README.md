# sel4-armv8-vmm-manifest
A manifest that allows one to build virtualized seL4 for zcu102 and i.MX8.

All of the code pointed to by `default.xml` is released by TARDEC for contract ASM17-04 under 
DISTRIBUTION STATEMENT A. Approved for public release; distribution unlimited.

## Get the source code
```
mkdir ~/sel4-vmm
cd ~/sel4-vmm
repo init -u https://github.com/dornerworks/sel4-armv8-vmm-manifest.git
repo sync
```

## ZCU102
These instructions assume you are using the [seL4 docker image](https://github.com/SEL4PROJ/seL4-CAmkES-L4v-dockerfiles).

### Build the Linux Kernel
- Install petalinux using the following [instructions](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18841618/PetaLinux+Getting+Started). (I used version 2017.3)
  - Note: We have seen some issues with using 2017.4 and we haven't tested any newer versions.
- Create a project from the petalinux BSP:
```
cd ~
petalinux-create -t project -s xilinx-zcu102-v2017.4-final.bsp
cd xilinx-zcu102-2017.4
```
- Make modifications so that Linux will start a getty for either serial device:
  - Add the following to the end of `project-spec/meta-user/conf/layer.conf`:
  ```
  require conf/distro/include/console.inc
  ```
  - Create the file `project-spec/meta-user/conf/distro/include/console.inc` with the following:
  ```
  SERIAL_CONSOLES_append = " 115200;ttyPS1"
  ```
- Add the vchan module
  - Get the vchan source code
  ```
  cd ~
  git clone https://github.com/dornerworks/vchan_module.git
  ```
  - Create a new module:
  ```
  cd ~/xilinx-zcu102-2017.4
  petalinux-create -t modules -n vchan --enable
  ```
  - Install module files:
  ```
  cp ~/vchan_module/vchan.bb project-spec/meta-user/recipes-modules/vchan/vchan.bb
  cp ~/vchan_module/vchan/* project-spec/meta-user/recipes-modules/vchan/files/
  ```
- Build Linux
```
petalinux-build
```
- Copy built image to seL4 project
```
cp images/linux/Image ~/sel4-vmm/apps/linux/zynqmp/Image1
```
- Generate BOOT.bin
```
petalinux-package --boot --fsbl images/linux/zynqmp_fsbl.elf --pmufw images/linux/pmufw.elf --u-boot
```
- Copy `./BOOT.BIN` to a boot partition on an SD card.

### Build seL4 and applications
- Start the seL4 Docker
```
cd ~/sel4-vmm
make -C /path/to/seL4-CAmkES-L4v-dockerfiles user HOST_DIR=$(pwd)
```
- Install old necessary package
```
sudo pip2 install tempita
sudo pip3 install tempita
```
- Build for the ZCU102
```
make zynqmp_vm_linux_defconfig
make
aarch64-linux-gnu-objcopy -O binary images/sel4arm-vmm-image-arm-zynqmp sel4-vmm
```

### Boot the image
- Connect to both APU UARTs from the ZCU102
  - U-Boot and seL4 use the first UART
  - The VMs all use the second UART
    - The virtual console can be switched with `ctrl+]`
- Copy sel4-vmm to your tftp directory
- Run the image via u-boot
```
dhcp; tftpb 0x10000000 172.192.10.15:sel4-vmm; go 0x10000000
```
