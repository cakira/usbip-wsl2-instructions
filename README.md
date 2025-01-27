# Adding USB support to WSL 2

This method adds support to WSL 2 by using USBIP on Windows to forward USB packets to USBIP on Linux.
The Linux kernel on WSL 2 does not support USB devices by default. The instructions here will explain how to add USB functionality to the WSL Linux kernel and how to use USBIP to hook devices into Linux.

## USBIP-Win

If you require the latest version of usbip-win you'll have to build usbip-win yourself.

Install git for Windows if you haven't already:
> https://gitforwindows.org/


Open git bash in the start menu and clone usbip-win:
```
$ git clone https://github.com/cezuni/usbip-win.git
```

Follow the instructions to build usbip-win in README.md. You'll have to install Visual Studio (community and other SKU's work), the Windows SDK and the Windows Driver Kit. Install each one in the order that is given in the instructions and don't install more then 1 thing at a time or the Windows SDK or Windows Driver Kit will install in a broken state that took me hours to figure out.

If you don't need the latest version you can get prebuilt versions located here:
> https://github.com/cezuni/usbip-win/releases

## Adding USB support to WSL Linux

Install WSL 2:
> https://docs.microsoft.com/en-us/windows/wsl/wsl2-install

Please take note that you must be running Windows 10 build 18917 or higher as explained in the description. 

You'll need Windows update to function to update to this version. 

If Windows update doesn't work because of various policy restrictions at your work, you can fix this by going into regedit.exe and locating to:
> HKEY_LOCAL_MACHINE\Software\Policies\Microsoft\Windows\WindowsUpdate

and deleting all entries. This will probably auto-refresh in the future with the original settings when you log back onto your domain. I think changing these settings are not good anyway so I won't miss the changes.

Open Ubuntu in WSL by navigating to the start menu and opening Ubuntu

Update sources:
```
~$ sudo apt update
```

Install prerequisites to build Linux kernel:
```
~$ sudo apt install build-essential flex bison libssl-dev libelf-dev libncurses-dev autoconf libudev-dev libtool
````

Clone the WSL 2 kernel. For this compilation we will put the source in the default folder WSL2-Linux-Kernel in the user home directory:
```
~$ cd
~$ git clone https://github.com/microsoft/WSL2-Linux-Kernel.git
~$ cd WSL2-Linux-Kernel/
```

Find out the name of your Linux kernel:
```
~/WSL2-Linux-Kernel$ uname -r
5.10.16.3-microsoft-standard-WSL2
```

Sometimes the name of the tag doesn't exactly match the name of your linux kernel. So, use the command `git tag` together with `grep` to find the tag that best matches your kernel:
```
~/WSL2-Linux-Kernel$ git tag | grep 5.10
linux-msft-wsl-5.10.16.3
v2.6.25.10
v2.6.35.10
v3.15.10
v4.15.10
```
Clearly, linux-msft-wsl-5.10.16.3 is the desired tag.

Checkout your version of the kernel:
```
~/WSL2-Linux-Kernel$ git checkout linux-msft-wsl-5.10.16.3
```

Copy in your current kernel configuration:
```
~/WSL2-Linux-Kernel$ cp /proc/config.gz config.gz
~/WSL2-Linux-Kernel$ gunzip config.gz
~/WSL2-Linux-Kernel$ mv config .config
```

Run menuconfig to select what kernel modules you'd like to add:
```
~/WSL2-Linux-Kernel$ sudo make menuconfig
```

Navigate in menuconfig to select the USB kernel modules you'd like. These suited my needs but add more or less as you see fit:
```
Device Drivers->USB support[*]
Device Drivers->USB support->Support for Host-side USB[M]
Device Drivers->USB support->Enable USB persist by default[*]
Device Drivers->USB support->USB Modem (CDC ACM) support[M]
Device Drivers->USB support->USB Mass Storage support[M]
Device Drivers->USB support->USB/IP support[M]
Device Drivers->USB support->VHCI hcd[M]
Device Drivers->USB support->VHCI hcd->Number of ports per USB/IP virtual host controller(8)
Device Drivers->USB support->Number of USB/IP virtual host controllers(1)
Device Drivers->USB support->USB Serial Converter support[M]
Device Drivers->USB support->USB Serial Converter support->USB FTDI Single Port Serial Driver[M]
Device Drivers->USB support->USB Physical Layer drivers->NOP USB Transceiver Driver[M]
Device Drivers->Network device support->USB Network Adapters[M]
Device Drivers->Network device support->USB Network Adapters->[Deselect everything you don't care about]
Device Drivers->Network device support->USB Network Adapters->Multi-purpose USB Networking Framework[M]
Device Drivers->Network device support->USB Network Adapters->Multi-purpose USB Networking Framework->CDC Ethernet support (smart devices such as cable modems)[M]
Device Drivers->Network device support->USB Network Adapters->Multi-purpose USB Networking Framework->Host for RNDIS and ActiveSync devices[M]
```

Build the kernel and modules with as many cores as you have available (-j [number of cores]). This may take a few minutes depending how fast your machine is:
```
~/WSL2-Linux-Kernel$ make -j 12 && make modules_install -j 12
```

After the build completes you'll get a list of what kernel modules have been installed. Mine looks like:
```
  INSTALL drivers/hid/hid-generic.ko
  INSTALL drivers/hid/hid.ko
  INSTALL drivers/hid/usbhid/usbhid.ko
  INSTALL drivers/net/mii.ko
  INSTALL drivers/net/usb/cdc_ether.ko
  INSTALL drivers/net/usb/rndis_host.ko
  INSTALL drivers/net/usb/usbnet.ko
  INSTALL drivers/usb/class/cdc-acm.ko
  INSTALL drivers/usb/common/usb-common.ko
  INSTALL drivers/usb/core/usbcore.ko
  INSTALL drivers/usb/serial/ftdi_sio.ko
  INSTALL drivers/usb/phy/phy-generic.ko
  INSTALL drivers/usb/serial/usbserial.ko
  INSTALL drivers/usb/storage/usb-storage.ko
  INSTALL drivers/usb/usbip/usbip-core.ko
  INSTALL drivers/usb/usbip/vhci-hcd.ko
  DEPMOD  5.10.16.3-microsoft-standard-WSL2
```

Build USBIP tools:
```
~/WSL2-Linux-Kernel$ cd tools/usb/usbip
~/WSL2-Linux-Kernel/tools/usb/usbip$ ./autogen.sh
~/WSL2-Linux-Kernel/tools/usb/usbip$ ./configure
~/WSL2-Linux-Kernel/tools/usb/usbip$ sudo make install -j 12
```

Copy USBIP tools libraries to location that USBIP tools can get to them:
```
~/WSL2-Linux-Kernel/tools/usb/usbip$ sudo cp libsrc/.libs/libusbip.so.0 /lib/libusbip.so.0
````

Make a script in your home directory to modprobe in all the drivers. Be sure to modprobe in usbcore and usb-common first. I called mine startusb.sh. Mine looks like:
```
#!/bin/bash
sudo modprobe usbcore
sudo modprobe usb-common
sudo modprobe hid-generic
sudo modprobe hid
sudo modprobe usbnet
sudo modprobe cdc_ether
sudo modprobe rndis_host
sudo modprobe usbserial
sudo modprobe usb-storage
sudo modprobe cdc-acm
sudo modprobe ftdi_sio
sudo modprobe usbip-core
sudo modprobe vhci-hcd
echo $(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')
```
The last line spits out the IP address of the Windows host. Helpful when you are attaching devices from it.
If you see some crap about /bin/bash^M: bad interpreter: No such file or directory it's because you have cr+lf line endings. You need to convert your shellscript to just lf.

Mark it as executable:
```
~$ sudo chmod +x startusb.sh
```

Replace the Linux kernel with the one just compiled. Before running the command below, replace `<your-user>` with your username.
```
cp ~/WSL2-Linux-Kernel/vmlinux /mnt/c/Users/<your-user>/
```

And create a file .wslconfig in `C:\Users\<your-user>\` with the following contents, always replacing `<your-user>` with your actual username:

```
[wsl2]
kernel=C:\\Users\\<your-user>\\vmlinux
```

Restart WSL. In a CMD window in Windows type:
```
C:\Users\rpasek>wsl --shutdown
```

Open WSL again by going to start menu and opening Ubuntu and run your script:
```
~$ ./startusb.sh
```

Check in dmesg that all your USB drivers got loaded:
```
~$ sudo dmesg
````

If so, cool, you've added USB support to WSL. 

## Using USBIP-Win and USBIP on Linux

This will generate a list of usb devices attached to Windows:
```
C:\Users\rpasek\usbip-win-driver>usbip list -l
```

The busid of the device I want to bind to is 1-220. Bind to it with: 
```
C:\Users\rpasek\usbip-win-driver>usbip bind --busid=1-220
```

Now start the usbip daemon. I start in debug mode to see more messages:
```
C:\Users\rpasek\usbip-win-driver>usbipd --debug
```

Now on Linux get a list of availabile USB devices:
```
~$ sudo usbip list --remote=172.30.64.1
```

The busid of the device I want to attach to is 1-220. Attach to it with:
```
~$ sudo usbip attach --remote=172.30.64.1 --busid=1-220
```

Your USB device should be usable now in Linux. Check dmesg to make sure everything worked correctly:
```
~$ sudo dmesg
```

## Couple of tips

* You need to bind to the base device of a composite device. Binding to children of a composite device does not work.
* You can't bind hubs. Sorry, it would be really cool but it doesn't work.
* If you'd like to unbind so you can access the device in Windows again: 
  1. Go into Device Manager in Windows,
  2. In the View menu, select "Show hidden devices"
  3. Find the USB/IP STUB device under System devices,
  4. Right click and select Update driver
  5. Click Browse my computer for driver software
  6. Click Let me pick from a list of available drivers on my computer
  7. Select the original driver that Windows uses to access the device
* Sometimes USBIP on Windows can't attach to a device. Try moving the device to a different hub and binding again. You can move the device back after you bind as binding sticks through attach/detach cycles.
* I had some trouble on one of my machines getting composite devices to show up in usbip list on the Windows side. To get around this:
  1. Download Zadig (https://zadig.akeo.ie) and run it
  2. Go to options and select "List All Devices" and deselect "Ignore Hubs or Composite Parents"
  3. Select your USB composite device in the list
  4. Install the libusbK driver
  5. Now your device should show up in the usbip list
  6. I don't really know why this works. It might be that the Windows default driver captures the composite device in a way that USBIP-Win can't see it and installing the libusbK frees it. USBIP-Win will essentially overwrite the libusbK driver with the USBIP-Win driver so you might be able to select any driver (not just libusbK). Either way it's probably a bug that needs to be worked out in USBIP-Win
