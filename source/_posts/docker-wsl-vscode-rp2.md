---
uuid: ddfb9e7e-9f9c-7b98-0821-2a221cef42ea
title: Container-based Embedded Systems Development with VSCode in WSL2 -- for RP2040
date: 2025-02-03 08:38:06
category: Reveries of a Lost Craftsman
excerpt: To turn any directory into a Dev Container directory,
thumbnail:
---

<!-- ##### Table of Content
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
    - [Set up WSL2](#set-up-wsl2)
    - [Install Docker in WSL2](#install-docker-inside-wsl2)
    - [Set up USB Passthrough](#set-up-usb-passthrough)
- [The Dockerfile](#the-dockerfile)
    - [Build Docker Image](#build-docker-image)
    - [Run Container](#run-image-to-get-a-container)
- [Dev Container](#dev-container)
    - [Open Project In Dev Container](#open-project-in-dev-container)
- [Conclusion](#conclusion) -->

# Introduction
Onboarding a new member of your embedded team can be time intensive in terms of setting up the development environment. Sometimes, your setup works while that of your colleague doesn’t because they have some packages installed that are useful for some other project they may be involved in. Other times, you need to set up a system for continuous integration and development on a remote machine. For all of these, if only you could set up a dedicated machine for the project in question, maybe your life would be a tad easier. 
That’s where containers can be useful. Containers can be thought of as very lightweight virtual machines that have virtualized OS functionality as opposed to virtualized hardware as is the case with standard virtual machines. Employing containers allows us to set up consistent isolated development environments across the team or even  with clients. Docker is the most popular containerization platform out there. It runs on Windows(directly), Windows inside Windows Subsystem on Linux(WSL2), macOS and Linux-based OSes among others.

In this article, we will explore setting up a Docker container  that can be used to develop firmware for RP2040 MCUs on WSL2. This setup can then be used inside VSCode to develop firmware and flash the firmware from inside the container.

Apart from the steps to configure WSL2 and USB pass through, every other step can be followed on a Linux distribution to achieve practically the same results.

# Prerequisites

### Set up WSL2
Microsoft has a well documented process for setting this up. Kindly check it out, follow the steps and be back here to continue. Here you go: https://learn.microsoft.com/en-us/windows/wsl/install.


### Install Docker inside WSL2
Once WSL2 is setup, the following commands inside the WSL2 terminal.
Docker has a GUI version called Docker Desktop which has a fairly consistent interface across platforms. You can opt to install that or the CLI version Docker Engine. The set up process is shown here: https://docs.docker.com/engine/install/

- Install Docker Engine for WSL2 Ubuntu https://docs.docker.com/engine/install/ubuntu/

After installing Docker, follow the steps here to manage Docker as non-root user.
- Post installation steps https://docs.docker.com/engine/install/linux-postinstall/

 
### Set up USB Passthrough
When a USB device is connected to your computer running WSL2, that USB device is automatically managed by Windows itself. In order to make the device available inside WSL2, you need a software that can pass it through.
Here's a good guide to get USB passthrough working: https://learn.microsoft.com/en-us/windows/wsl/connect-usb. 

If you prefer a GUI solution(like me), follow this link: https://gitlab.com/alelec/wsl-usb-gui#installation. Setting it up is pretty straightforward and using it is super easy.


# The Dockerfile
The Dockerfile is a blueprint for the isolated environment you want to create. The Docker engine follows the instructions outlined in the Dockerfile and creates an image. You can then run the image with certain parameters to get your isolated development environment –the container.
[This particular Dockerfile](https://github.com/raynayx/rpxContainer/blob/main/Dockerfile) creates an image which has the `pico-sdk`, the `arm-none-eabi` toolchain setup and the `JLink` tools for flashing the firmware to the Raspberry Pi series of MCUs. The image also has [`Invoke`](https://www.pyinvoke.org/)  for managing tasks like building,flashing and debugging the firmware.

You can clone the Dockerfile repository from [here](https://github.com/raynayx/rpxContainer.git).
```dockerfile
FROM fedora:40

ENV REFRESHED_AT=2025-01-28

# Add timezone info  
ENV TZ=Africa/Accra

# Update base
RUN dnf update -y

# Install g++ and dependencies
RUN dnf install -y g++ wget git
    python3-pip python3-invoke
    cmake
    vim ninja-build xz &&\
    #
    # JLink dependencies 
    #
    dnf install -y libXrandr libXfixes libXcursor
    ncurses-compat-libs &&\
    dnf clean all

ARG MAIN_USER=rpx
ARG MAIN_HOME=/home/${MAIN_USER}

RUN useradd -m ${MAIN_USER}

#Download JLink_V812d.rpm
# RUN curl -o ${MAIN_HOME}/opt/JLink.rpm --data "accept_license_agreement=accepted" https://www.segger.com/downloads/jlink/JLink_Linux_x86_64.rpm
ARG JLINK_BIN=JLink.rpm
COPY ${JLINK_BIN} ${MAIN_HOME}/opt/

#install JLink tools
RUN cd ${MAIN_HOME}/opt/ &&\
    dnf install -y --disablerepo=* ./${JLINK_BIN} &&\
    rm ${JLINK_BIN}
  
# Download and install RP2040 Toolchains
# RUN curl -o ~/opt/arm-none-eabi-14.tar.xz \
# https://developer.arm.com/-/media/Files/downloads/gnu/14.2.rel1/binrel/arm-gnu-toolchain-14.2.rel1-x86_64-arm-none-eabi.tar.xz

#Copy and install arm-none-eabi-toolchain
COPY arm-none-eabi-14.tar.xz ${MAIN_HOME}/opt/
RUN cd ${MAIN_HOME}/opt/ && \
    tar -xf arm-none-eabi-14.tar.xz &&\
    mv arm-gnu-toolchain-* arm-none-eabi/ &&\
    rm ./arm-none-eabi-* 

ENV PATH=$PATH:${MAIN_HOME}/opt/arm-none-eabi/bin/

# Clone and setup the RP2040(PICO) SDK
RUN git clone https://github.com/raspberrypi/pico-sdk ${MAIN_HOME}/opt/pico-sdk/

RUN cd ${MAIN_HOME}/opt/pico-sdk/ &&\
    git submodule update --init

# get, build and install picotool
RUN git clone https://github.com/raspberrypi/picotool.git ${MAIN_HOME}/opt/picotool
RUN mkdir ${MAIN_HOME}/opt/picotool_bin
RUN cd ${MAIN_HOME}/opt/picotool/ &&\
    mkdir build
RUN cd ${MAIN_HOME}/opt/picotool/build &&\
    cmake -DCMAKE_INSTALL_PREFIX=${MAIN_HOME}/opt/picotool_bin/ \
    -DPICO_SDK_PATH=${MAIN_HOME}/opt/pico-sdk -DPICOTOOL_FLAT_INSTALL=1 .. 
RUN cd ${MAIN_HOME}/opt/picotool/build &&\
    make install

ENV PICO_SDK_PATH=${MAIN_HOME}/opt/pico-sdk/
ENV CMAKE_CXX_COMPILER=${MAIN_HOME}/opt/arm-none-eabi/bin/arm-none-eabi-g++

#Set the dev directory
WORKDIR ${MAIN_HOME}/dev/

USER ${MAIN_USER}  
```

### Build Docker image
The Dockerfile expects to share the same directory as the following files. 
Get them and name as following:
- [arm-none-eabi-14.tar.xz](https://developer.arm.com/-/media/Files/downloads/gnu/14.2.rel1/binrel/arm-gnu-toolchain-14.2.rel1-x86_64-arm-none-eabi.tar.xz)
- [JLink.rpm](https://www.segger.com/downloads/jlink/JLink_Linux_x86_64.rpm).

To build the image, run the following while replacing `path_to_Dockerfile` with the specific path of the Dockerfile:
```bash
docker buildx build -t rasp/pico -f path_to_Dockerfile .
```
### Run image to get a container
If you want to interact with a Docker container in the terminal, run the following command replacing the `src` flag with the directory of the project you want to work with.
```bash
docker run -it --mount type=bind,src=project/directory/,dst=/home/rpx/dev --privileged -v /dev/bus/usb/:/dev/bus/usb namespace/image_name  /bin/bash
```
The `-v /dev/bus/usb/:dev/bus/usb` passes the USB devices available to the host through to the Docker container. 

# Dev Container
While there are lots of vim and emacs diehards out there, the vast majority of developers spend their time in VS Code. The extensibility of the editor makes it easy to bend it to your will. VSCode Extensions allows for this customizability. The Dev Container extension from Microsoft is among the most useful. You can get it by installing it separately or by installing it as part of a pack of other extensions that allow for remote development.

Inside the VSCode window, click on the Extension tab, search for `Dev Container` and install it.
Dev Container allows VS Code to connect to a docker container as though it was a normal project directory opened inside VS Code. 

To turn any directory into a Dev Container directory, you have to create a `devcontainer.json` file inside a `.devcontainer` directory inside the project directory and configure it. In order to get this to work, you need to point the `devcontainer.json` file to the location of the Dockerfile you intend to use. You can also point it to a prebuilt remote image.
The other configurations include listing the extensions required or useful for the container environment.
For flashing firmware, you will need to pass USB through to the container. This is shown in the  `-- privileged` and `/dev/bus/usb` flags passed in the json file.
This is shown below:
```json
{
    "build":{"dockerfile": "path/to/Dockerfile"},
    "customizations": {
        "vscode": {
            "extensions": [
                "ms-vscode.cpptools-extension-pack",
                "ms-vscode.vscode-serial-monitor",
                "trond-snekvik.gnu-mapfiles",
                "ZixuanWang.linkerscript",
                "ms-vscode.vscode-embedded-tools",
                "mcu-debug.debug-tracker-vscode",
                "marus25.cortex-debug",
                "mcu-debug.peripheral-viewer",
                "mcu-debug.rtos-views"
            ]
        }
    },
    "mounts": ["type=bind,src=/dev/bus/usb,dst=/dev/bus/usb"],
    "runArgs": ["--privileged"]
}
```
In order to get this to work, point the json file to the location of the Dockerfile by editing the `dockerfile` value.

### Open Project in Dev Container
In order to start developing in the Dev Container, open the project directory with the `devcontainer.json` file inside VS Code.
The `Dev Container` extension will prompt you to rebuild the container, reload the window.
Once the build is complete, you can develop inside VS Code as though the environment was totally local.
This is shown below:
<video src="https://github.com/user-attachments/assets/728f1906-61d2-47b6-a99f-e663b43b28cf" controls="controls"></video>

You can do visual debugging through the Cortex Debug extension.
<video src="https://github.com/user-attachments/assets/1d022889-8f8c-42bd-8c4a-9c33c71aed86" controls="controls"></video>

# Sample RP2040 Project Directory Setup
You can clone this [directory](https://github.com/raynayx/rpxPrj.git), follow the instructions in the `README.md` file to setup a project directory that has all required files to get this to work.

# Conclusion
By setting up your development environment, you get the productive VS Code environment and the isolation of a docker container.
Going forward, you can set up things like linting in this isolated development environment to further enhance your development experience.
