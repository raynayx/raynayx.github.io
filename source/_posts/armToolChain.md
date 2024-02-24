---
uuid: 4d2861b4-9a60-c79b-d0b9-dcf1fc1680af
title: Installing Arm Embedded GCC on Fedora 35
category: Reveries of a Lost Craftsman
date: 2022-01-07 07:35:37
# tags: RP2040
---

# The Need
Most personal computers these days use CPUs of the x86_64 architecture. In order to build software for them, one needs tools that build binaries compatible with said architecture. A typical flow is writing the software in a compiled language (say C,C++) and then using compilers and linkers to convert the English-like text into machine readable instructions. For the personal computer space, one will need tools like MSVC,Clang or GCC for such a workflow. 

However, as stated earlier, these tools will produce machine readable instructions for the x86_64 architecture. If one needs to build software for any other type of architecture, one needs any of the compiler toolchains for the specific architecture in question. In this text, my concern is with writing code for the ARM architecture which has become very popular. The ARM architecture is found in most mobile phones in the form of the A-series(say Cortex A-76), in microcontrollers (say Cortex M-4) and others.

It follows therefore that to build for the ARM architecture, one will need an ARM specific compiler toolchain. GCC is a common compiler toolchain that is opensource and free. And ARM provides the ARM specific GCC toolchain.
Since, most PCs today are x86_64, you would have to build for ARM on an x86_64 computer -- a process known as ```cross-compiling```.

# Arm Embedded GCC Toolchain
The Embedded GCC Toolchain is made up of compilers,linkers,debuggers and some other utilities that make the work of an embedded software developer simpler.
The default compilers are: ```arm-none-eabi-gcc```,```arm-none-eabi-g++```. There is also a linker --```arm-none-eabi-ld``` and the venerable ```gdb``` debugger in its tasteful ARM flavour as ```arm-none-eabi-gdb```.
In order to make use of them, we need to have them installed on our system. Here is one way that can be done on Fedora(35) as tested.

# The How
In order to install the GCC toolchain for ARM on Fedora(35),
- Go to [developer.arm.com](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm/downloads#) to fetch the latest compiler set which comes as a compressed file.
- Remove any vestiges of previous installs by running:
    ```bash
    $ sudo dnf remove binutils-arm-none-eabi gcc-arm-none-eabi libnewlib-arm-none-eabi
    ```
- If the feedback is ```No packages marked for removal```, that's fine as there are no previous installations on the sysem.
- Then run the following commands:
    ```bash
    $ mkdir ~/opt   #create opt in $HOME
    $ cd ~/opt      #change into opt
    $ tar -xvf *.bz2    #uncompress the file
    $ chmod -R -w gcc-arm-none-eabi-* #
    ```
    <!-- The command on line 3 extracts the compressed file. Make sure the file name in the command matches the file name of the file you downloaded. It may end in `.bz2` as in the example or others like `.tar.xz` or `.tar`. -->

- Add the path to the ```.bashrc``` file to make the toolchain available system wide:
    ```
    export PATH=$PATH:~/opt/gcc-arm-none-eabi-*/bin/
    ```
- Source the ```.bashrc``` file thus:
```
$ source ~/.bashrc
```
- If ```libncurses.so.5``` library is not present on the Fedora system, run:
```bash
$ sudo dnf install ncurses-compat-libs
```

### Confirm installation
After installing, you can run the following in a new terminal window:
```
$ arm-none-eabi-gcc --version
```
If you get a feedback stating:
```arm-none-eabi-gcc (GNU Arm Embedded Toolchain 10.3-2021.10) 10.3.1 20210824 (release)``` with dates that match the version you downloaded, then you have correctly installed the ARM Toolchain.


<!-- It is possible the `arm-none-eabi-gdb` will complain about paths to a specific Python version(Python 3.8 in my case) depending on the version of the toolchain you grab. To fix this, add those paths to your `.bashrc` file thus:
```sh
export PYTHONPATH=/usr/lib64/python3.8
export PYTHONHOME=/usr/lib64/

``` -->

# Next 
After installation, the next step will be to use the shinny new toolchain. We will take a look at doing that pretty soon! 
