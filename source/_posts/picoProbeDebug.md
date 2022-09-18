---
uuid: 1a981ce5-61eb-f5f2-5d03-93c13f92d786
title: picoProbeDebug
date: 2022-09-18 12:38:53
category: Reveries of a Lost Craftsman
tags:
thumbnail:
---
# Introduction
In my previous [article](https://raynayx.com/2022/06/20/piPicoSetup/), I described how to go about setting up your development environment for writing code for the RP2040 on Fedora(mostly 35 and 36).<br>

In the conclusion of that article, I mentioned that there are two main ways of getting your binary from the build computer onto the flash paired with the RP2040 MCU.
The first was to use the UF2 format. This approach is fairly straight forward. All it requires is for you to hold down the BOOTSEL button on the Pi Pico board while inserting the connected USB cable into your computer. This makes the board show up as a USB mass storage device. You then drag and drop the binary with `.uf2` extension into the device. This action causes the MCU to reset. After this, it starts to run your code.<br>

You will agree that this approach is not exactly friendly to multiple code flashes per programming session.
# JTAG/SWD
The other way of going about flashing onto RP2040 based boards is to use the universal `JTAG` or more precisely `SWD` interface.
## JTAG
The Joint Test Action Group(JTAG)

## SWD
ARM Serial Wire Debug


# Picoprobe
The people at Raspberry have created a way of flashing and debugging an RP2040 using another RP2040 based board. The binary for doing this is called the `Picoprobe`. 
The steps are described in their [documentation](https://datasheets.raspberrypi.com/pico/getting-started-with-pico.pdf). I will attempt to summarize the whole process here.

## OpenOCD
For the Picoprobe to work, we need to setup a software called Open On-Chip Debugger(OpenOCD).

The makers of the RP2040 have a fork of OpenOCD with modifications to make it work with Picoprobe with almost no modifications.
- To set it up, we need to install a few dependencies as follows:
```
sudo dnf install automake autoconf texinfo libtool libusb-devel libftdi-devel
```
- We then need to clone the OpenOCD repository into a directory of our choosing. I have decided to clone mine into `~/opt/`
```
git clone https://github.com/raspberrypi/openocd.git --branch rp2040 --depth=1 --no-single-branch
```
- Change directory into the newly created `openocd` directory and run the following commands one after the other:
```
$ ./bootstrap
$ ./configure --enable-picoprobe
$ make -j4
$ sudo make install
```
To confirm if the above step was successful, run the following command:
```
$ openocd -f interface/picoprobe.cfg -f target/rp2040.cfg -s tcl
```
If you get the LIBUSB_ERROR_ACCESS do the following to fix it:

__Fixing the LIBUSB_ERROR_ACCESS problem__
- Check if /etc/udev/rules.d/60-openocd.rules
- Create the file /etc/udev/rules.d/60-openocd.rules and paste the following text in there:
```
# Raspberry Pi Picoprobe
ATTRS{idVendor}=="2e8a", ATTRS{idProduct}=="0004", MODE="660", GROUP="plugdev", TAG+="uaccess"
```
- Unplug the Picoprobe and plug it back in

Run the following again:
```
$ openocd -f interface/picoprobe.cfg -f target/rp2040.cfg -s tcl
```
OpenOCD should run and display at least the version of your installation. It will return immediately with an error due to the absence of the Picoprobe device.

## Flash Picoprobe via UF2
We need to load the Picoprobe firmware onto one of our RP2040 boards. This is the board that will act as the hardware debugger to help us program and debug other RP2040 boards.

To do this, we need to clone the Picoprobe repository made available by the folks at Raspberry Pi.
- Clone the Picoprobe repository into any directory of your choice:
```
git clone https://github.com/raspberrypi/picoprobe.git
```
- Change into the picoprobe directory and create a `build` directory:
```
cd picoprobe
mkdir build
```
- Open the `CMakeLists.txt` file in a text editor and add the following line to specify the path of the `pico-sdk` on your computer:
```
set(PICO_SDK_PATH ~/opt/pico-sdk)
```
You are welcome to change the pico-sdk path to match the path on your computer.

- Change into the `build` directory and run the following to build the picoprobe:
```
cd build
cmake ..
make
```

The picoprobe binary should be built after a few moments.

- Find the `picoprobe.uf2`.
- If you had already plugged in the USB cable for your Pi Pico, remove it.
- Hold down the BOOTSEL button on the Pico board while inserting the USB.
- The board should show up as a mass storage device.
- Copy the `picoprobe.uf2` file on to the storage device. The board should reset and start running your code.

The Pico board you used is now a hardware debugger ready to program and debug your other RP2040 based boards!

# Picoprobe to Target Device wiring
In order to program an RP2040 based device using the Picoprobe, both hardware have to wired in a certain way.
The following is a specific example in which the target(device to be programmed) is a Pi Pico board.

Picoprobe | Target
----------| ------
GND | GND
GPIO 2 |SWCLK
GPIO 3 |SWDIO
GPIO 4/UART1 TX |GP1/UART0 RX
GPIO 5/UART1 RX |GP0/UART0 TX
VSYS|VSYS

## Testing the connection
In order to verify that the connection works as we desire, we need to connect the picoprobe's USB to the host computer. We then lauch OpenOCD by running the following:
```
$ openocd -f interface/picoprobe.cfg -f target/rp2040.cfg -s tcl
```

If the connections are done in the right fashion, OpenOCD will present us with a series of texts. The last ones will be about starting gdb server and listening for gdb connections on some port. The default is port 3333.

_If you receive any error messages, verify your connections and try again. 
If it persists, try using short wires between the two boards._

## arm-none-eabi-gdb
GDB is a software debugger. It allows us to send debug instructions to a device being debugged.
To test if our setup works correctly, we can start GDB and try to send commands to our target hardware.

- Launch another terminal window or tab and start the ARM flavour of GDB by running:
```
$ arm-none-eabi-gdb
```

Once we are presented with the GDB prompt, we need to connect to the GDB server started by OpenOCD.
To do this, we type the following:
```
target extended-remote localhost:3333
```

This will connect with the OpenOCD GDB server. Now we can talk to our target board.

To program the flash paired with the RP2040, change into the directory that has the binary of interest and run the following:
```
load test_pico.elf
```
After the programming is complete, your target can be debugged.
This can be done at the terminal by typing commands. However, Visual Studio Code(VS Code) has some extensions that allow clicking to achieve most of the things one will want in a debugging session. The actions common in step through debugging include 

Next, we will set up VS Code to do this.

# Tying it all together in VS Code

Install VS Code. You can do this by grabbing the `.rpm` or by adding the VS Code repo and install it using `dnf`. The VS Code web pages show how to do this.

Launch VS Code and click on the `Extensions` icon on the sidebar.
Search `Cortex-Debug` by `marus25` and install it.

Still in the Extensions view, search for and install `C/C++` by Microsoft. This will enhance the experience of writing C/C++ programs since it will provide perks like `Intellisense` and function peeking among other things.

Create a new directory and open it inside VS Code. It can be done as follows from the terminal:
```
mkdir blink
cd blink
code .
```

- Once the directory is open inside VS Code, create a sub-directory name `.vscode` in the root of the earlier directory.

- Inside the `.vscode` directory, create a file named `launch.json`.

- Copy and paste the following text into the `launch.json` file:
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Pico Debug",
            "cwd": "${workspaceRoot}",
            "executable": "${command:cmake.launchTargetPath}",
            "request": "launch",
            "type": "cortex-debug",
            "servertype": "openocd",
            // This may need to be arm-none-eabi-gdb depending on your system
            "gdbPath" : "arm-none-eabi-gdb",
            "device": "RP2040",
            "configFiles": [
                "interface/picoprobe.cfg",
                "target/rp2040.cfg"
            ],
            "svdFile": "${env:PICO_SDK_PATH}/src/rp2040/hardware_regs/rp2040.svd",
            "runToMain": true,
            // Work around for stopping at main on restart
            "postRestartCommands": [
                "break main",
                "continue"
            ],
            "searchDir": ["<PATH TO TCL>"],
        }
    ]
}
```
- Create another file name `settings.json` inside `.vscode` and paste in the following text:
```json
{
    // These settings tweaks to the cmake plugin will ensure
    // that you debug using cortex-debug instead of trying to launch
    // a Pico binary on the host
    "cmake.statusbar.advanced": {
        "debug": {
            "visibility": "hidden"
        },
        "launch": {
            "visibility": "hidden"
        },
        "build": {
            "visibility": "default"
        },
        "buildTarget": {
            "visibility": "hidden"
        }
    },
    "cmake.buildBeforeRun": true,
    "C_Cpp.default.configurationProvider": "ms-vscode.cmake-tools",
    "cortex-debug.openocdPath": "<PATH TO OPENOCD>"
}
```
The texts inside the `settings.json` and `launch.json` are from Shawn Hymmel's article on the same topic. I find them very useful. That article can be found [here](https://www.digikey.be/en/maker/projects/raspberry-pi-pico-and-rp2040-cc-part-2-debugging-with-vs-code/470abc7efb07432b82c95f6f67f184c0).


- Enable printing over USB or UART inside the `CMakeLists.txt` file. The `0` turns off the particular function while the `1` turns it on. In this case, printing over USB is disabled while printing over UART is enabled.
```cmake
pico_enable_stdio_usb(${MY_TARGET} 0)
pico_enable_stdio_uart(${MY_TARGET} 1)
```

## Debug with VS Code
In order to debug the code, click on the `Run and Debug` tab.
At the top of the sidebar that shows up, click on the `Run` icon.
This will compile the code and upload it to the target board. The target board will reset and run from startup. Right before execution hits `main` the Picoprobe debugger will halt execution and await instructions from you.
VS Code will provide you will a little toolbar which displays buttons like `Continue`,`Step Over`,`Step Into`, `Step Out` and others.
These buttons allow us to do the step through debugging we set out to do.
