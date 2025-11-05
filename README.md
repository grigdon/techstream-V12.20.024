# Toyota Techstream (V12.20.024) on Ubuntu 24.04 via VirtualBox

This guide provides step-by-step instructions to configure and run Toyota Techstream V12.20.024 within a pre-configured Windows XP VirtualBox virtual machine on an Ubuntu 24.04 LTS host.

## Prerequisites

Before you begin, ensure you have the following:

- A computer running Ubuntu 24.04 LTS
- A MINI-VCI (J2534) cable

## Step 1: Download the Pre-configured VM

Download the VirtualBox .ova (Open Virtualization Archive) file containing the Windows XP and Techstream installation.

**Download Link:** https://drive.google.com/drive/folders/1fSnWzoqpobXoqWV-TrmPo5CstXtJ0DLC?usp=sharing

## Step 2: Install VirtualBox

Install VirtualBox from the official Ubuntu repositories.

```bash
sudo apt update
sudo apt install virtualbox
```

## Step 3: Install VirtualBox Extension Pack

The Extension Pack is required for USB 2.0 and 3.0 device passthrough, which you need for your MINI-VCI cable.

### Check your VirtualBox version:

```bash
vboxmanage --version
# Example output: 7.0.16_Ubuntur162802
```

### Download the matching Extension Pack:

You must download the Extension Pack that exactly matches your installed version. Go to the [VirtualBox Downloads Page](https://www.virtualbox.org/wiki/Downloads) and find the corresponding file.

For example, if your version is 7.0.16, you would download:

```bash
# This URL is an example. Get the correct one for your version!
wget https://download.virtualbox.org/virtualbox/7.0.16/Oracle_VM_VirtualBox_Extension_Pack-7.0.16.vbox-extpack
```

### Install the Extension Pack:

(Assuming it was downloaded to your ~/Downloads folder)

```bash
cd ~/Downloads
sudo VBoxManage extpack install Oracle_VM_VirtualBox_Extension_Pack-7.0.16.vbox-extpack
```

### Verify the installation:

```bash
VBoxManage list extpacks
```

The output should show the "Oracle VM VirtualBox Extension Pack" as `Usable: true`.

## Step 4: Add Your User to the vboxusers Group

To allow VirtualBox to access your host's USB devices, you must be in the `vboxusers` group.

### Add your user:

```bash
sudo usermod -aG vboxusers $USER
```

### Reboot your computer:

This step is mandatory for the group changes to take effect.

```bash
sudo reboot
```

### Verify after rebooting:

```bash
groups | grep vboxusers
# 'vboxusers' should appear in the list
```

### Verify USB access:

```bash
VBoxManage list usbhost
# This should list your host's USB devices without permission errors
```

## Step 5: Import and Configure the VM

Now, import the downloaded .ova file and configure its USB settings.

1. Open the VirtualBox application
2. Go to **File > Import Appliance...** and select the .ova file you downloaded in Step 1. Follow the prompts to import it
3. Once imported, **do not start the VM yet**
4. Select the techstream VM in the VirtualBox manager and click **Settings**
5. Go to the **USB** tab
6. Check **Enable USB Controller**
7. Select **USB 2.0 (EHCI) Controller** (This is critical for the VCI cable)
8. Click **OK** to save the settings

## Step 6: Start VM and Check for KVM Conflicts

Start the Techstream VM.

- If it starts correctly, proceed to Step 7
- If it fails to start (especially with a kernel-related error), you may have a conflict with KVM (Linux's built-in hypervisor)

### Troubleshooting KVM Conflicts:

KVM modules (like `kvm_intel` or `kvm_amd`) can prevent VirtualBox from running.

Check for loaded KVM modules:

```bash
sudo lsmod | grep kvm
```

If any are listed, remove them. **Warning: Be careful with these commands.**

```bash
# Example for Intel CPUs:
sudo rmmod kvm_intel
sudo rmmod kvm
```

Try starting the VM again. Note that these modules will reload on your next reboot.

## Step 7: Mini-VCI Host Driver Setup (Important!)

By default, Ubuntu's FTDI driver will "claim" the VCI cable, making it `Busy` and unavailable to VirtualBox. You must unload these drivers each time you want to use the cable with the VM.

### Plug in your MINI-VCI cable

Run the following command to identify the device and see the kernel messages:

```bash
lsusb && sudo dmesg | tail -20
```

Look for output indicating the VCI cable was detected. You are looking for two key things:

- An `lsusb` entry like: `Bus 003 Device 015: ID 0403:6001 Future Technology Devices International, Ltd FT232 Serial (UART) IC`
- A `dmesg` entry showing it was attached to a tty port: `[21695.155009] usb 3-7: FTDI USB Serial Device converter now attached to ttyUSB0`

This confirms the host OS (Ubuntu) has loaded its drivers (`ftdi_sio` and `usbserial`). You must unload them:

```bash
sudo modprobe -r ftdi_sio
sudo modprobe -r usbserial
```

### Verify the device is now "Available" to VirtualBox:

```bash
VBoxManage list usbhost | grep -A 10 "M-VCI" # Or another name your cable uses
```

The output should now show `Current State: Available`.

```
Product:            M-VCI
SerialNumber:       A6WJRBFA
Address:            sysfs:/sys/devices/pci0000:00/0000:00:14.0/usb3/3-7//device:/dev/vboxusb/003/015
Current State:      Available
```

**Note:** If it still says `Busy`, unplug the cable, wait a few seconds, plug it back in, and re-run the `modprobe -r` commands.

## Step 8: Connect the VCI to the Live VM

With the host drivers unloaded, you can now "pass" the USB device to the running Windows VM.

1. Ensure the Techstream VM is running
2. Ensure the MINI-VCI cable is plugged into your computer
3. In the VM's window menu bar, click **Devices > USB**
4. Find your VCI cable in the list (e.g., `HX M-VCI [0403]`) and click on it
5. A checkmark will appear next to it, and Windows will make a "device connected" sound

## Step 9: Verify VCI in Windows (Guest Setup)

Inside the Windows XP VM, open the Device Manager.

- Right-click "My Computer" > Properties > "Hardware" tab > "Device Manager"

Expand the **Ports (COM & LPT)** section.

- You should see **USB Serial Port (COM#)**. Note the COM port number (e.g., COM4)
- If you see a yellow exclamation mark (!), you may need to install the FTDI drivers inside Windows

Drivers can be found at: https://ftdichip.com/drivers/vcp-drivers/

Download the drivers (for Windows XP) and install them within the VM, then reboot the VM.

## Step 10: Using Techstream

You are now ready to connect to your vehicle.

1. Launch Techstream from the VM's desktop
2. Connect the VCI cable to your vehicle's OBD-II port
3. Turn the ignition to the "ON" position (engine does not need to be running)
4. In the Techstream software, click **Connect to Vehicle**

Enjoy!
