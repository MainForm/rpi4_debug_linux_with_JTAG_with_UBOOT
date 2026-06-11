# Raspberry Pi 4 Buildroot Linux with U-Boot

이 저장소는 Raspberry Pi 4 64-bit 환경에서 Buildroot로 Linux 이미지를 만들고,
U-Boot를 거쳐 커널을 부팅하기 위한 외부 Buildroot tree입니다. UART 콘솔과
JTAG 디버깅을 염두에 두고 설정했습니다.

## 구성 요약

- Target: Raspberry Pi 4 Model B, 64-bit
- Build system: Buildroot 2024.02
- Kernel defconfig: `bcm2711`
- Buildroot defconfig: `configs/raspberrypi4_64_defconfig`
- Board files: `board/raspberrypi4-64/`
- Output directory: `output/raspberrypi4_64/`

## 주요 설정

### Raspberry Pi firmware

`board/raspberrypi4-64/config_4_64bit.txt`가 Raspberry Pi firmware의
`config.txt`로 설치됩니다.

중요한 설정은 다음과 같습니다.

```txt
kernel=u-boot.bin
dtoverlay=disable-bt
enable_uart=1
arm_64bit=1
```

`kernel=u-boot.bin`으로 Raspberry Pi firmware가 Linux `Image`를 직접 실행하지
않고 U-Boot를 먼저 실행하게 합니다.

`dtoverlay=disable-bt`는 Bluetooth를 끄고 GPIO14/GPIO15의 PL011 UART를
Linux의 `ttyAMA0` 콘솔로 쓰기 위해 사용합니다. 이 설정 덕분에 U-Boot와 Linux
UART 디버깅 경로를 단순하게 유지할 수 있습니다.

### Kernel command line

`board/raspberrypi4-64/cmdline.txt`는 firmware가 직접 Linux를 부팅할 때 쓰는
cmdline입니다.

```txt
root=/dev/mmcblk0p2 rootwait console=tty1 console=serial0,115200
```

UART 콘솔을 안정적으로 사용하기 위해 `console=ttyAMA0,115200`처럼 특정 Linux
장치명을 직접 고정하지 않고 `console=serial0,115200`을 사용합니다. Raspberry Pi
firmware가 현재 UART overlay 설정에 맞춰 `serial0`을 실제 콘솔 UART로 연결해
줍니다.

주의: U-Boot가 Linux를 부팅할 때는 이 파일이 자동으로 사용되지 않습니다.
U-Boot 경유 부팅에서는 U-Boot의 `bootargs`가 Linux command line이 됩니다.

### U-Boot 자동 부팅

U-Boot 설정은 `board/raspberrypi4-64/uboot.fragment`로 추가합니다.

```conf
CONFIG_BOOTCOMMAND="setenv bootargs root=/dev/mmcblk0p2 rootwait console=tty1 console=ttyAMA0,115200 earlycon=pl011,mmio32,0xfe201000 loglevel=8; fatload mmc 0:1 ${kernel_addr_r} Image; booti ${kernel_addr_r} - ${fdt_addr}"
```

이 `bootcmd`는 다음 순서로 동작합니다.

1. Linux `bootargs`를 설정합니다.
2. boot 파티션에서 `Image`를 `${kernel_addr_r}`로 로드합니다.
3. Raspberry Pi firmware가 overlay를 적용해서 U-Boot에 넘겨준 `${fdt_addr}`를
   그대로 사용합니다.
4. `booti`로 64-bit Linux kernel을 실행합니다.

중요한 점은 DTB를 SD카드에서 다시 로드하지 않는다는 것입니다. SD카드의 raw
`bcm2711-rpi-4-b.dtb`를 직접 로드하면 firmware가 적용한 UART overlay가 빠질 수
있습니다. 그래서 `${fdt_addr}`를 사용합니다.

### Boot partition image

`board/raspberrypi4-64/post-image.sh`에서는 boot 파티션 이미지에 Linux kernel
`Image`가 들어가도록 다음 항목을 명시적으로 추가합니다.

```bash
FILES+=( "Image" )
```

U-Boot의 `bootcmd`가 boot 파티션에서 `Image`를 `fatload`로 읽기 때문에 이 파일이
반드시 boot 파티션에 포함되어야 합니다.

## 빌드 방법

프로젝트 루트에서 실행합니다.

```bash
make raspberrypi4_64_defconfig
make -C output/raspberrypi4_64
```

빌드 결과는 아래에 생성됩니다.

```txt
output/raspberrypi4_64/images/
```

주요 산출물:

- `sdcard.img`: SD카드에 기록할 전체 이미지
- `boot.vfat`: boot 파티션 이미지
- `u-boot.bin`: Raspberry Pi firmware가 실행할 U-Boot
- `Image`: Linux kernel image
- `rootfs.ext2`: root filesystem

## SD카드 기록

`sdcard.img`를 SD카드 전체 디바이스에 기록합니다. `/dev/sdX1` 같은 파티션이
아니라 `/dev/sdX` 전체 디바이스를 사용해야 합니다.

```bash
lsblk -o NAME,PATH,SIZE,TYPE,MODEL,SERIAL,TRAN,MOUNTPOINTS

sudo umount /dev/sdX1 2>/dev/null
sudo umount /dev/sdX2 2>/dev/null

sudo dd if=output/raspberrypi4_64/images/sdcard.img \
  of=/dev/sdX bs=4M conv=fsync status=progress

```

## U-Boot에서 수동 부팅

자동 부팅이 실패하면 U-Boot 프롬프트에서 다음 명령으로 수동 부팅할 수 있습니다.

```bash
mmc list
mmc dev 0
fatls mmc 0:1

setenv bootargs 'root=/dev/mmcblk0p2 rootwait console=tty1 console=ttyAMA0,115200 earlycon=pl011,mmio32,0xfe201000 loglevel=8'
fatload mmc 0:1 ${kernel_addr_r} Image
booti ${kernel_addr_r} - ${fdt_addr}
```

`Starting kernel ...` 이후 로그가 보이지 않으면 Linux console이 잘못 잡힌
것입니다. U-Boot에서 직접 넘기는 `bootargs`에는 `serial0` 대신 실제 Linux 장치명
`ttyAMA0`를 사용해야 합니다.
