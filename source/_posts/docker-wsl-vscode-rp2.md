---
uuid: ddfb9e7e-9f9c-7b98-0821-2a221cef42ea
title: VS Code for Container-based Embedded Systems Development in WSL2 -- For RP2040
date: 2025-02-03 08:38:06
category: Reveries of a Lost Craftsman
excerpt: To turn any directory into a Dev Container directory,
thumbnail:
---

# Introduction
Onboarding a new member of your embedded team can be time intensive in terms of setting up the development environment. Sometimes, your setup works while that of your colleague doesn’t because they have some packages installed that are useful for some other project they may be involved in. Other times, you need to set up a system for continuous integration and development on a remote machine. For all of these, if only you could just set up a dedicated machine for the project in question, maybe your life will be a tad easier. 
That’s where containers can be useful. Containers can be thought of as very lightweight virtual machines that have virtualized OS functionality as opposed to virtualized hardware as is the case with standard virtual machines. Employing containers allows us to set up consistent isolated development environments across the team or even  with clients or partners. 

Docker is the most popular containerization platform out there. It runs on Windows(directly), Windows inside Windows Subsystem Linux(WSL2), macOS and Linux among others. In this article, my goal is to set up a Docker container  that can be used to develop firmware for the RP2040 MCUs from Raspberry Pi. You can then connect to this container inside VS Code,develop firmware and flash the firmware from inside the container.

If you’re using a Linux based system, you can skip the following steps and go to installing docker on your Linux box.

# Prerequisites
If you’re using Windows, you have two paths. You can install Docker directly on Windows and skip to the Dockerfile section –the direct path. Alternatively, you can set up WSL2 and then install Docker inside WSL2. But, why choose the straightforward approach when there’s a convoluted route? 
I will show you how to get it done with WSL2 since that allows us to use relatively the same steps on Linux-based systems.

## Set up WSL2
Microsoft has a well documented process for setting this up. Kindly check it out, follow the steps and be back here to continue. Here you go: https://learn.microsoft.com/en-us/windows/wsl/install.

## Install Docker inside WSL2
Docker has a GUI version called Docker Desktop which has a fairly consistent interface across platforms. You can opt to install that or the CLI version Docker Engine. The set up process is shown here: https://docs.docker.com/engine/install/

## Set up USB Passthrough
If you're following along in WSL2, you will need a tool that makes your USB devices available inside the WSL2 environment. Here's a good guide to get it working: https://learn.microsoft.com/en-us/windows/wsl/connect-usb. 

If you prefer a GUI solution(like me), follow this link: https://gitlab.com/alelec/wsl-usb-gui#installation. Installing it is pretty straightforward and using it is super easy.


# The Dockerfile
The Dockerfile is a blueprint for the isolated environment you want to create. The Docker engine follows the instructions outlined in the Dockerfile and creates an image. You can then run the image with certain parameters to get your isolated development environment –the container.
[This Dockerfile](https://github.com/raynayx/rpxContainer/blob/main/Dockerfile) creates an image which has the `pico-sdk`, the `arm-none-eabi` toolchain setup and the `JLink` tools for flashing the firmware to the Raspberry Pi series of MCUs. The image also has [`Invoke`](https://www.pyinvoke.org/)  for managing tasks like building,flashing and debugging the firmware.

## Build Docker image
To build the image, do:
```bash
docker buildx build -t namespace/image_name -f path_to_Dockerfile .
```
## Run image to get a container
You can then run the image to get a container this way:
```bash
docker run -it --mount type=bind,src=project/directory/,dst=/home/rpx/dev --privileged -v /dev/bus/usb/:/dev/bus/usb namespace/image_name  /bin/bash
```
The `-v /dev/bus/usb/:dev/bus/usb` passes the USB devices available to the host through to the Docker container. 

# Dev Container
While there are lots of vim and emacs diehards out there, the vast majority of developers spend their time in VS Code. The extensibility of the editor makes it easy to bend to your will. The tools that allow for this are referred to as Extensions. The Dev Container extension from Microsoft is among the most useful. You can get it by installing it separately or by installing it as part of a pack of other extensions that allow for remote development.
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
## Open Project in Dev Container
In order to start developing in the Dev Container, open the project directory with the `devcontainer.json` file inside VS Code.
The `Dev Container` extension will prompt you to rebuild the container, reload the window.
Once the build is complete, you can develop inside VS Code as though the environment was totally local.
This is shown below:
<video src="https://github.com/user-attachments/assets/728f1906-61d2-47b6-a99f-e663b43b28cf" controls="controls"></video>

You can do visual debugging through the Cortex Debug extension.
<video src="https://github.com/user-attachments/assets/1d022889-8f8c-42bd-8c4a-9c33c71aed86" controls="controls"></video>

# Conclusion
- Things you can do going forward
