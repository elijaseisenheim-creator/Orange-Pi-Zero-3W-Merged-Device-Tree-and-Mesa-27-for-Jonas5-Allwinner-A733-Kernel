# How To Build

## Warning: This is highly experimental and may not work for everyone, please try with caution!

*1. Download the Armbian Image* 
---
Go to Jonas5 Github repository and download the latest Armbian Desktop Image for the Orange Pi 4 Pro.
Flash the Image to an SD Card and boot it once, it should make it to the Desktop but without working USB, if it doesn't, give it a few Minutes or more because it resizes the SD Card on the first boot.
Download the merged DTB and GPU firmware files from this repository.

*2. Integrating the new DTB*
---
Disconnect the now booted Orange Pi from power and mount it's SD Card root file system on your computer, this only works on BSD or Linux, because of the ext4 filesystem Linux uses.
Once mounted, copy the new DTB to the SD Card with these commands, you may need root access:

- sudo cp /pathtoyourdtb/merged.dtb /pathtoyourmountedsdcard/boot/sun60i-a733-orangepi-4-pro.dtb
- sudo cp /pathtoyourdtb/merged.dtb /pathtoyourmountedsdcard/boot/dtb-6.18.62-sun60iw2/allwinner/sun60i-a733-orangepi-4-pro.dtb

Next, we need to copy the missing GPU firmware files to the root filesystem, use this command:

- sudo cp -r /pathtofirmware/powervr /pathtoyourmountedsdcard/lib/firmware/

Finally, we can eject the SD Card from the computer and put it back into the OrangePi, it should boot up to a pixelated spinning animation, this is the default Desktop Environment trying to start up, but failing because of the GPU Drivers.
After waiting a few seconds you can press "Ctr + Alt + F3" to switch to tty3, it is best to have an Ethernet USB Card connected for the next steps, I haven't tested WiFi so far as the WiFi Module on my OrangePi is unfortunatly broken.
If the spinning animation with the OrangePi or Radxa logo stays during boot, you can reboot the OrangePi by either unplugging and plugging it back in or by switching to tty2 and typing in "reboot".

Now we can rebuild the initramfs and cache with these commands:

- sudo update-initramfs -u -k "$(uname -r)"

- sudo ldconfig

Unfortunatly the OrangePi getting stuck on initramfs seems to happen randomly at startup, I will be looking into this problem in the near future and update the repository if I find a solution.

If the commands were successfull, you can restart the OrangePi with "sudo reboot".

Now you can try reloading the powervr drivers after entering tty3 again with:
- sudo modprobe -r powervr

and

- sudo modprobe powervr


*3. Installing Sway*
---
First, just install sway and xwayland through the normal package manager and start sway from tty3.
It should now start up, but still report only software rendering, you can check this by typing:

- glxinfo -B

and

- eglinfo -B 

If both say llvmpipe as the renderer, you are running in software mode.

If that is the case, we have to build Mesa 27 with the imagination driver and zink enabled.

*4. Building Mesa 27*
---
Clone the official github Mesa 27 repository and install all build depedencies:

- git clone https://gitlab.freedesktop.org/mesa/mesa.git

- cd mesa

As for the build depedencies, refer to, you may have to add the normal Debian 13 repositories before installing them:
- https://docs.mesa3d.org/install.html

To build mesa, we have to set some Flags:

- meson setup build --wipe \
  -Dllvm=enabled \
  -Dbuildtype=release -Db_ndebug=true \
  -Dvulkan-drivers=imagination \
  -Dgallium-drivers=zink \
  -Dplatforms=wayland,x11 \
  -Dglx=dri -Degl=enabled -Dgbm=enabled \
  -Dllvm=enabled -Dprefix=/usr

and now we can start the builing process:

- ninja -C build
- sudo ninja -C build install
- sudo ldconfig

*Note:* this may take some time, if you have a version of the OrangePi with less than 4GB of RAM, you might consider creating a swap file.

*5. Checking for Acceleration*
---
After the build is finished, we can reboot, enter tty3 again, reload the powervr drivers and start sway again.
It should now say zink or PowerVR when checking for the renderer:

- vulkaninfo --summary | grep -E 'deviceName|driverName'
- eglinfo -B | grep -i -E 'renderer|vendor'
- glxinfo -B | grep -i renderer


Some applications may need to have set the icd.json file manually to start, just add this in front of every command that requires that: 
- VK_DRIVER_FILES=/usr/share/vulkan/icd.d/powervr_mesa_icd.aarch64.json 