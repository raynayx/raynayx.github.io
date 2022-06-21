---
uuid: e324b039-b44e-00ee-409c-194df560296e
title: Setting up Raspberry Pi RP2040 development toolchain(Fedora)
date: 2022-06-20 14:02:53
category: Reveries of a Lost Craftsman
---

# Raspberry Pi RP2040
The Raspberry Pi RP2040 is a microcontroller developed by Raspberry Pi. The MCU is an ARM Cortex M0+ based chip with a number of common peripherals paired with it. The chip also has an interesting Programmable IO subsystem. In this current era of chip shortage, this chip has proven to be the most available.
In this article, I take you through setting up your toolchain in order to write software for this chip.
**_It must be noted that the [official documentation](https://datasheets.raspberrypi.com/pico/raspberry-pi-pico-c-sdk.pdf) for doing this is well written._**

# Prerequisites
In my previous [article](https://raynayx.com/2022/01/07/armToolChain/), I showed how to setup an ARM toolchain. This is one basic prerequite for setting up your system for RP2040 development.
Another required installation is the CMake build tool since the PICO-SDK depends on it.

# CMake Installation
To install CMake run:
```bash
sudo dnf install cmake
```

# Raspberry Pi Pico C/C++ SDK
#### Clone Pi Pico SDK:
Clone the SDK by running:
```bash
git clone https://github.com/raspberrypi/pico-sdk.git ~/opt/pico-sdk
 ```
The SDK will be cloned into the `~/opt/pico-sdk` directory in this example.<br>

Copy [pico_sdk_import.cmake](https://github.com/raspberrypi/pico-sdk/blob/master/external/pico_sdk_import.cmake) from the SDK into your project directory

# Test Pico SDK installation
- Create a directory for the project(you can call it any name).

- Create a file named `main.c` with the following content.
```c
#include <stdio.h>
#include "pico/stdlib.h"

int main() {
    setup_default_uart();
    printf("This works!\n");
    return 0;
}

```
- In your project directory add a `CMakeLists.txt` file with the following initial contents:
```cmake
cmake_minimum_required(VERSION 3.13)

# initialize the SDK based on PICO_SDK_PATH
# note: this must happen before project()
set(PICO_SDK_PATH ~/opt/pico-sdk)	#Set the path to the location of the cloned SDK
include(pico_sdk_import.cmake)
set(MY_TARGET test_pico)		# name of build target

project(my_project)

# initialize the Raspberry Pi Pico SDK
pico_sdk_init()

# rest of your project configs

add_executable(${MY_TARGET}
    main.c
)

# Add pico_stdlib library which aggregates commonly used features
target_link_libraries(${MY_TARGET} pico_stdlib)

# create map/bin/hex/uf2 file in addition to ELF.
pico_add_extra_outputs(${MY_TARGET})
```

- Create a directory inside the project directory with the name `build`.
- Inside a terminal, change into the build directory, thus:
```
cd ./build
```
- Run the `cmake` command inside the `build` directory to create your makefile as follows:
```
cmake ..
```
- Run `make` in the build:
```
make test_pico
```
You should now have a file named `test_pico.elf` and some other formats including `test_pico.uf2`.
This confirms that the SDK setup was successful. Next, we have to get the `elf` file or the `uf2` file into the flash paired with the RP2040.

# Flash the executable onto the RP2040's flash storage
# UF2
One way of doing this is by copying the `uf2` file onto the RP2040 device when it shows up as a mass storage device on the computer. 
Depending on the board you have, you could trigger this in different ways.
For the Pi Pico board, press the `BOOTSEL` button while inserting the USB cable into your computer.
Once the device shows up, copy the `uf2` file and paste it inside the mass storage device. If this is done successfully, the device will reset and start to run your program.
This approach is the simplest way of getting code onto your device.
# ELF