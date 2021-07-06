# buildroot-qemu-aarch64
Buildroot for ARM64 on qemu

## Version
Based on buildroot 2021.02.1, last updated 2021.05.07

## Clone repository with submodules
You must clone this project with
```
git clone --recursive https://github.com/schenkmi/buildroot-qemu-aarch64.git
```

## Add buildroot as git submodule

cd buildroot-qemu-aarch64
git submodule add git://git.buildroot.net/buildroot buildroot

cd buildroot
git tag --list

git checkout tags/2021.02.1 -b 2021.02.1
cd ..
git add --all buildroot
git commit -m "using tag 2021.02.1 of buildroot" buildroot
git push


### submodule update
```
cd /work/buildroot-qemu-aarch64
git submodule update --remote --recursive
cd buildroot
git tag --list
git checkout tags/2021.05 -b 2021.05
cd ..
git commit -m "using tag 2021.05 of buildroot" buildroot
```


## Build instructions plain Linux

You can build the image with
```
cd buildroot-qemu-aarch64
mkdir -p out && cd out
make -C ../buildroot O="$(pwd)" BR2_EXTERNAL="../buildroot-external" qemu_aarch64_virt_defconfig
make
```

PATH="/work/buildroot-qemu-aarch64/out/host/bin:${PATH}"
qemu-system-aarch64 -M virt -cpu cortex-a53 -nographic -smp 1 -kernel out/images/Image -append "rootwait root=/dev/vda console=ttyAMA0" -netdev user,id=eth0 -device virtio-net-device,netdev=eth0 -drive file=out/images/rootfs.ext4,if=none,format=raw,id=hd0 -device virtio-blk-device,drive=hd0


qemu-system-aarch64 \
  -M virt \
   -cpu cortex-a53 \
   -serial stdio \
    -smp 1  \
  -kernel out/images/Image \
  -drive file=out/images/rootfs.ext4,if=none,format=raw,id=hd0 -device virtio-blk-device,drive=hd0 \
  -append "root=/dev/vda console=ttyAMA0,115200" \
  -netdev user,id=ethernet.0,hostfwd=tcp::2222-:22,hostfwd=tcp::2223-:69 \
  -device rtl8139,netdev=ethernet.0 \
  -name Versatile_ARM_EXT


  export PATH="/work/buildroot-qemu-aarch64/out/host/bin:${PATH}"
  exec qemu-system-aarch64 -M virt -cpu cortex-a53 -nographic -smp 1 -kernel Image -append "rootwait root=/dev/vda console=ttyAMA0" -netdev user,id=eth0 -device virtio-net-device,netdev=eth0 -drive file=rootfs.ext4,if=none,format=raw,id=hd0 -device virtio-blk-device,drive=hd0  ${EXTRA_ARGS}
  )





## Build instructions using Docker

The first step is to build the Docker image and create a container. You can specify a directory in your hosts system which will be mounted inside the container. This allows you to copy the built system easily. The current directory must contain the provided Dockerfile which will be compiled into an image. The directory called `build` will contain the sources and the images which will be built later.
```
mkdir -p build
docker build -t br-optee-rpi .
docker create -it --name br-optee-rpi --mount type=bind,source="$(pwd)",destination=/build br-optee-rpi
```

You can start the container if it was created successfully. This command gives you an interactive shell inside the container.
```
docker start -ia br-optee-rpi
```

The next step is to create a build directory called `out` which will contain the downloaded package source files and the output images. We will use this directory for an out-of-tree Buildroot build, by adding the `O=` parameter to the make command. We have to specify the directories where the external trees are stored, which can be done by adding the `BR2_EXTERNAL=` parameter to make (We can specify multiple directories by using `:` as a separator). The default config file is called `rpi3_defconfig` which is inside the `configs` directory.
```
mkdir -p out && cd out
make -C ../buildroot O="$(pwd)" BR2_EXTERNAL="../buildroot-external" rpi3_defconfig
```

After the configuration has been finalized you can issue the make command to start building the sources. This can take a long time, so be patient.
```
make
```

## Menuconfig
Precondition: you are inside out/ folder
```
make -C ../buildroot O="$(pwd)" BR2_EXTERNAL="../buildroot-external" menuconfig
```

## Update default configuration
```
make -C ../buildroot O="$(pwd)" BR2_EXTERNAL="../buildroot-external" update-defconfig
```

## Build an SDK
```
make -C ../buildroot O="$(pwd)" BR2_EXTERNAL="../buildroot-external" sdk
```

## Flashing
Please change /dev/sdX with the correct mounting point of your sd card.
```
sudo dd if=images/sdcard.img of=/dev/sdX bs=1M conv=fdatasync status=progress
```

## Console log
```
sudo picocom -b 115200 /dev/ttyUSB0
```
## Signed FIT images

### Create RSA 4096 key set
```
cd buildroot-external/board/rpi3
mkdir keys
openssl genrsa -F4 -out "keys/dev.key" 4096
openssl req -batch -new -x509 -key "keys/dev.key" -out "keys/dev.crt"
```
### Copy U-BOOT DTB for target platform
This step needs to be done for every updated U-BOOT to have the DTB in sync with U-BOOT.

```
cd out/images
cp ../build/uboot-2021.01/arch/arm/dts/bcm2837-rpi-3-b.dtb u-boot-bcm2837-rpi-3-b.dtb
```
### Create a signed FIT
The mkimage will place the public key into the DTB. This DTB will be used to build the U-BOOT.
```
cd out/images
cp -R ../../buildroot-external/board/rpi3/keys .
cp ../../buildroot-external/board/rpi3/rpi3_bcm2837_fit.its .
cp ../build/uboot-2021.01/arch/arm/dts/bcm2837-rpi-3-b.dtb u-boot-bcm2837-rpi-3-b.dtb
PATH=../host/bin:$PATH mkimage -f rpi3_bcm2837_fit.its -K u-boot-bcm2837-rpi-3-b.dtb -k ./keys -r image.fit
cp u-boot-bcm2837-rpi-3-b.dtb ../../buildroot-external/board/rpi3/
```
### Configure to use external DTB for U-BOOT (EXT_DTB=)
```
cd out
make menuconfig
U-BOOT Options
(EXT_DTB=${BR2_EXTERNAL_RPI_OPTEE_PATH}/board/rpi3/u-boot-bcm2837-rpi-3-b.dtb) Custom make options
```

## Testing

OP-TEE can be tested with the xtest application
```
xtest
```
