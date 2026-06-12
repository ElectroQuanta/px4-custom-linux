# PX4 atop Linux on the mx8mn custom board

This guide provides step-by-step instructions to manually build all components for PX4 atop Linux on the i.MX 8M Nano custom board.

## 1. Prerequisites & Environment

Ensure you have the required host packages and cross-compiler installed:
```bash
# General build dependencies
sudo apt install build-essential git libssl-dev bison flex zlib1g-dev \
                 device-tree-compiler bc swig python3-distutils

# AArch64 cross-compiler
sudo apt install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu

PROJ_DIR=$(pwd)
export CROSS_COMPILE=aarch64-linux-gnu-
export ARCH=arm64
```

---

## 2. ARM Trusted Firmware (ATF)

ATF must be configured to use UART1 as the console.

```bash
git clone https://github.com/nxp-imx/imx-atf.git -b lf_v2.6 --depth 1
cd imx-atf
git am ../patches/atf/*.patch
make PLAT=imx8mn bl31 CROSS_COMPILE=aarch64-linux-gnu- IMX_BOOT_UART_BASE=0x30860000
```

---

## 3. U-Boot

```bash
cd $PROJ_DIR
git clone https://github.com/nxp-imx/uboot-imx.git -b lf_v2022.04 --depth 1
cd uboot-imx
git am ../patches/uboot/*.patch
make imx8mn_evk_defconfig
make -j$(nproc) CROSS_COMPILE=aarch64-linux-gnu-
```

---

## 4. NXP Firmware Blobs

Download and extract the LPDDR4 training firmware. Accept the EULA.

```bash
cd $PROJ_DIR
mkdir firmware && cd firmware
wget https://www.nxp.com/lgfiles/NMG/MAD/YOCTO/firmware-imx-8.20.bin
chmod +x firmware-imx-8.20.bin
./firmware-imx-8.20.bin
```

---

## 5. Boot Image Generation (flash.bin)

Use `imx-mkimage` to combine the binaries.

```bash
cd $PROJ_DIR
git clone https://github.com/nxp-imx/imx-mkimage.git -b lf-6.12.3-1.0.0 --depth 1
cd imx-mkimage/iMX8M

# Copy required files to iMX8M directory
cp ../../uboot-imx/u-boot-nodtb.bin .
cp ../../uboot-imx/spl/u-boot-spl.bin .
cp ../../uboot-imx/arch/arm/dts/imx8mn-evk.dtb .
cp ../../uboot-imx/tools/mkimage mkimage_uboot
cp ../../imx-atf/build/imx8mn/release/bl31.bin .
cp ../../firmware/firmware-imx-8.20/firmware/ddr/synopsys/lpddr4_pmu_train* .

# Build flash.bin
cd ..
make SOC=iMX8MN flash_evk_no_hdmi
```

---

## 6. Linux Kernel

Build the kernel using the project configuration fragment.

```bash
cd $PROJ_DIR
git clone https://github.com/nxp-imx/linux-imx.git -b lf-6.12.3-1.0.0 --depth 1
cd linux-imx
make ARCH=arm64 imx_v8_defconfig
scripts/kconfig/merge_config.sh -m .config ../br-external/linux.config
make -j$(nproc) ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-
```

---

## 7. Root Filesystem (Buildroot)

Use the stripped Buildroot configuration to generate the RootFS.

```bash
cd $PROJ_DIR
git clone https://github.com/buildroot/buildroot.git -b 2025.11.x --depth 1
cd buildroot
make BR2_EXTERNAL=../br-external mx8mn_custom_defconfig
make
```

---

## 8. Custom Device Tree (DTS)

Compile the custom board hardware description.

```bash
# Ensure dtc is installed: sudo apt install device-tree-compiler
dtc -I dts -O dtb -o DTS/imx8mn-evk.dtb DTS/imx8mn-evk.dts 2&>1 /dev/null
```

---

## 9. PX4 Autopilot

1. Get PX4
```bash
git clone https://github.com/ElectroQuanta/PX4-Autopilot.git -b mx8mn-linux --recursive
cd PX4-Autopilot
```

2. Install prerequisites
```bash
python -m venv .px4-venv
source .px4-venv/bin/activate
pip install -r ${PROJ_DIR}/requirements.txt
```

3. Build
```bash
export BR_BIN=${PROJ_DIR}/buildroot/output/host/bin
export BR_SYSROOT=${PROJ_DIR}/buildroot/output/host/aarch64-buildroot-linux-gnu/sysroot
make nxp_mx8mn-linux_default \
CMAKE_ARGS="-DCMAKE_C_COMPILER=${BR_BIN}/aarch64-buildroot-linux-gnu-gcc \
-DCMAKE_CXX_COMPILER=${BR_BIN}/aarch64-buildroot-linux-gnu-g++ \
-DCMAKE_LINKER=${BR_BIN}/aarch64-buildroot-linux-gnu-ld \
-DCMAKE_AR=${BR_BIN}/aarch64-buildroot-linux-gnu-ar \
-DCMAKE_NM=${BR_BIN}/aarch64-buildroot-linux-gnu-nm \
-DCMAKE_OBJCOPY=${BR_BIN}/aarch64-buildroot-linux-gnu-objcopy \
-DCMAKE_OBJDUMP=${BR_BIN}/aarch64-buildroot-linux-gnu-objdump \
-DCMAKE_STRIP=${BR_BIN}/aarch64-buildroot-linux-gnu-strip \
-DCMAKE_SYSROOT=${BR_SYSROOT}"
```

---

## 10. SD Card Preparation

### Partitioning
1. **P1 (BOOT)**: FAT32, ~64MB.
2. **P2 (ROOTFS)**: Ext4, remaining space.

### Deployment
Flash the boot image (requires `seek=32` for i.MX8M):
```bash
cd ${PROJ_DIR}
SD_DEV=/dev/sda # replace appropriately (e.g., /dev/mmcblk0)
SD_BOOT=${SD_DEV}1
SD_ROOT=${SD_DEV}2
sudo dd if=imx-mkimage/iMX8M/flash.bin of=${SD_DEV} bs=1k seek=32 conv=fsync
sync
```

Copy Linux image and DTB to BOOT partition:
```bash
sudo mount ${SD_BOOT} /mnt
sudo cp linux-imx/arch/arm64/boot/Image /mnt
sudo cp DTS/imx8mn-evk.dtb /mnt
sync
sudo umount ${SD_BOOT}
```

Populate RootFS (using `rootfs.tar`):
```bash
PX4_DEST_DIR=/mnt/home/px4
PX4_BUILD_DIR=PX4-Autopilot/build/nxp_mx8mn-linux_default
PX4_CFG_DIR=px4-cfg
sudo mkfs.ext4 -L rootfs ${SD_ROOT}
sudo mount ${SD_ROOT} /mnt
sudo tar -xpf buildroot/output/images/rootfs.tar -C /mnt
sudo mkdir -p ${PX4_DEST_DIR}/bin ${PX4_DEST_DIR}/etc
sudo rsync -rlptD --delete ${PX4_BUILD_DIR}/bin/ ${PX4_DEST_DIR}/bin/ 
sudo rsync -rlptD --delete ${PX4_BUILD_DIR}/etc/ ${PX4_DEST_DIR}/etc/ 
sudo cp ${PX4_CFG_DIR}/run.sh ${PX4_CFG_DIR}/mx8mn_mc.config ${PX4_DEST_DIR}/ 
sudo cp ${PX4_CFG_DIR}/S99px4 /mnt/etc/init.d/
sync
sudo umount /mnt
```


---

## 11. Booting the System

Once the SD card is ready, insert it into the board and power it up. It should
boot straight into Linux.

**Run PX4**
```bash
cd /home/px4
./run.sh
INFO  [px4] mlockall() enabled. PX4's virtual address space is locked into RAM.
INFO  [px4] assuming working directory is rootfs, no symlinks needed.

______  __   __    ___ 
| ___ \ \ \ / /   /   |
| |_/ /  \ V /   / /| |
|  __/   /   \  / /_| |
| |     / /^\ \ \___  |
\_|     \/   \/     |_/

px4 starting.
``` 

## 12. Notes
1. Ensure all components are tightly secured, especially the GPS and the battery
2. When testing outside, always perform a calibration first, and wait for the
   GPS signal to set (at least 15 satellites)
3. Sometimes a X/Y position control error may occur. Wait a little for it to
   vanish.
4. To force drone arming from the PX4 console:

   ```bash
   commander arm -f
   ``` 

