By default, **Windows Subsystem for Linux (WSL2)** runs a stripped-down stock Microsoft Linux kernel. While great for general development and containerized workloads, it lacks built-in drivers for USB IP virtual host controllers (`vhci_hcd`), wireless configuration APIs (`cfg80211`/`mac80211`), and out-of-tree Wi-Fi dongles (like the **Realtek RTL8821AU**).

If you want to run wireless penetration testing, packet injection, and monitor mode natively inside WSL2, you need to compile a **custom WSL2 kernel**, enable **USB/IP**, and compile your adapter's kernel drivers.

This guide walks through the step-by-step setup using **`usbipd-win`** and building a custom wireless kernel.

## Technical Overview & Prerequisites

To pass a physical USB Wi-Fi adapter into WSL2, two core components are required:

1. **Host-Side Bridge (`usbipd-win`):** Windows forwards raw USB packets to the Linux VM over virtual network sockets.
    
2. **Guest-Side Kernel Engine:** The WSL2 Linux kernel must be compiled with `vhci_hcd` support, `mac80211`/`cfg80211` wireless stacks, and the appropriate driver module (`8821au.ko`).

## Step 1: Install and Configure `usbipd-win` on Windows

Open **PowerShell as Administrator** on your Windows host system to install the USB bridging software:

PowerShell

```
# Install usbipd-win via winget
winget install --interactive --exact dorssel.usbipd-win
```

_Note: Restart your terminal or system after installation so environment variables take effect._

### Bind and Attach Your Adapter

Plug in your USB Wi-Fi adapter (e.g., TP-Link RTL8821AU) and identify its **BUSID**:

PowerShell

```
# List connected USB devices
usbipd list
```

Identify your adapter in the output (e.g., `BUSID 1-3`):

Plaintext

```
BUSID  VID:PID    DEVICE                          STATE
1-3    2357:0120  TP-Link Wireless USB Adapter    Not shared
```

Bind the adapter to make it available for sharing, then attach it to your WSL distribution:

PowerShell

```
# Bind the BUSID (Only needs to be done once per port)
usbipd bind --busid 1-3

# Attach the device to your WSL2 instance
usbipd attach --wsl --busid 1-3
```

> **Useful Management Commands:**
> 
> - **Reconnecting:** If you unplug the adapter or restart Windows, simply re-attach it:
>     
>     `usbipd attach --wsl --busid 1-3`
>     
> - **Releasing Control:** To give the adapter back to Windows:
>     
>     `usbipd detach --busid 1-3`
>     

# Step 2: Build a Custom WSL2 Linux Kernel

Because the stock Microsoft kernel lacks USB/IP host controller modules (`vhci_hcd`), attaching the adapter will throw a `Loading vhci_hcd failed` error until a custom kernel is loaded.

### 1. Install Build Dependencies inside WSL2

Launch your WSL2 distribution and install the compilation tools:

Bash

```
sudo apt update && sudo apt install -y \
  build-essential \
  libncurses-dev \
  flex \
  bison \
  libssl-dev \
  bc \
  libelf-dev \
  pkg-config \
  ccache \
  git
```

### 2. Clone the Official Microsoft WSL2 Kernel Source

Clone the matching stable branch of the WSL2 kernel:

Bash

```
mkdir -p ~/kernel-build && cd ~/kernel-build
git clone --depth=1 -b linux-msft-wsl-6.18.y https://github.com/microsoft/WSL2-Linux-Kernel.git
cd WSL2-Linux-Kernel
```

### 3. Configure Kernel Features (`make menuconfig`)

Pull the running WSL config as a baseline and open the configuration menu:

Bash

```
zcat /proc/config.gz > .config
make menuconfig
```

Ensure the following features are selected as **Built-in `[*]`** (press `Y` on each):

- **USB/IP Virtual Host Controller Interface:**
    
    - `Device Drivers` ---> `USB support` --->
        
        - `[*] USB/IP support` (`CONFIG_USBIP_CORE`)
            
        - `[*] VHCI hcd` (`CONFIG_USBIP_VHCI_HCD`)
            
        - `[*] Host Controller drivers` (`CONFIG_USBIP_HOST`)
            
- **Wireless Subsystems:**
    
    - `Networking support` ---> `Wireless` --->
        
        - `[*] cfg80211 - wireless configuration API`
            
        - `[*] Generic IEEE 802.11 Networking Stack (mac80211)`
            

Save the configuration and exit `menuconfig`.

## Step 3: High-Speed Compilation & Driver Installation

Leverage `ccache` and multi-threaded compilation to speed up the kernel build:

Bash

```
# 1. Compile the kernel and install base module structures
make CC="ccache gcc" -j$(($(nproc) + 2)) && \
sudo make modules_install && \
cp arch/x86/boot/bzImage /mnt/c/Users/nikhi/vmlinux-wifi
```

### Compile the Realtek RTL8821AU Driver

Switch to your out-of-tree `8821au` driver source directory and compile it against your newly built kernel headers:

Bash

```
cd ~/shared/tools/wifi-pentesting/complied/8821au-20210708

# Clean build directory and compile module
make clean
make CC="ccache gcc" -j$(($(nproc) + 2))

# Install driver module and update module dependencies
sudo make install
sudo depmod -a
```

## Step 4: Configure Distro-Specific or Global Kernel Boot

You can apply your new kernel globally via `.wslconfig` or lock it specifically to your distribution using `/etc/wsl.conf`.

### Option A: Distribution-Specific Configuration (`/etc/wsl.conf`)

To make **only** your pentesting distribution read the custom wireless kernel without affecting other WSL instances:

Open `/etc/wsl.conf` inside your Linux terminal:

Bash

```
sudo nano /etc/wsl.conf
```

Add the following configuration block:

Ini, TOML

```
[wsl2]
kernel=C:\\Users\\nikhi\\vmlinux-wifi

[boot]
systemd=true

[user]
default=h4ck3rfirst
```

### Option B: Global Configuration (`.wslconfig`)

To apply the custom kernel across all WSL2 instances, edit `C:\Users\<Username>\.wslconfig` in Windows PowerShell:

Ini, TOML

```
[wsl2]
kernel=C:\\Users\\nikhi\\vmlinux-wifi
```

## Step 5: Restart WSL and Verify Monitor Mode

1. From **Windows PowerShell**, shut down WSL to load the new kernel image:
    
    PowerShell
    
    ```
    wsl --shutdown
    wsl -d kali-linux
    ```
    
2. Attach the USB dongle from **PowerShell (Admin)**:
    
    PowerShell
    
    ```
    usbipd attach --wsl --busid 1-3
    ```
    
3. Inside your **WSL terminal**, load the module and verify interface recognition:
    
    Bash
    
    ```
    # Load driver module
    sudo modprobe 8821au
    
    # Verify network interfaces
    iwconfig
    ```
    

### Enabling Monitor Mode

With `wlan0` attached, toggle the wireless interface into monitor mode for packet capturing:

Bash

```
sudo ip link set wlan0 down
sudo iw dev wlan0 set type monitor
sudo ip link set wlan0 up

# Verify active mode
iwconfig wlan0
```

## Troubleshooting Quick-Reference

| **Issue**                                   | **Root Cause**                                         | **Solution**                                                                                                    |
| ------------------------------------------- | ------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------- |
| **`Loading vhci_hcd failed`**               | Active kernel lacks USB/IP modules.                    | Boot into custom kernel compiled with `CONFIG_USBIP_VHCI_HCD`.                                                  |
| **`Module 8821au not found`**               | Drivers not installed into `/lib/modules/$(uname -r)`. | Run `sudo make modules_install` in kernel tree, then rebuild driver with `sudo make install && sudo depmod -a`. |
| **`Device with busid is already attached`** | Adapter is locked by another session.                  | Run `usbipd detach --busid <BUSID>` and then re-attach.                                                         |


--- 
if u want to save time
# Guide: Restoring Pre-Compiled Wi-Fi Drivers via Zip File in WSL2

**Scenario:** Deploy a custom WSL2 kernel with wireless support and pre-compiled out-of-tree drivers (like `8821au`) without rebuilding the stack on a new instance.

### Repository Artifacts Overview
* `vmlinux-wifi` — Custom WSL2 kernel binary.
* `6.18.40.1-microsoft-standard-WSL2+.zip` — Archive with compiled modules (`8821au.ko`) and mapping trees.

---

## Step-by-Step Restoration Guide

### Step 1: Deploy and Configure the Custom Kernel on Windows
1. Place `vmlinux-wifi` into a Windows directory (e.g., `C:\WSL-Kernels\vmlinux-wifi`).
2. Configure `.wslconfig` in your Windows user profile:
```ini
[wsl2]
kernel=C:\\WSL-Kernels\\vmlinux-wifi
```

### Step 2: Extract and Move Modules into WSL2
Extract the zip archive and copy the module folder into WSL2:
```bash
sudo mkdir -p /lib/modules/
sudo cp -r 6.18.40.1-microsoft-standard-WSL2+ /lib/modules/
```

### Step 3: Refresh Kernel Module Dependencies
Index the transferred directory structure:
```bash
sudo depmod -a
```

### Step 4: Restart the WSL2 Environment
From Windows PowerShell, reset WSL and attach your Wi-Fi adapter:
```powershell
wsl --shutdown
usbipd attach --wsl --busid <YOUR-BUSID>
```

### Step 5: Verify and Interface Activation
Verify and bring the interface online in your WSL2 terminal:
```bash
iw dev
sudo ifconfig wlan0 up
```

---

## Advanced Usage: Enabling Monitor Mode
Toggle the card into monitor mode:
```bash
sudo ip link set wlan0 down
sudo iw dev wlan0 set type monitor
sudo ip link set wlan0 up
iwconfig wlan0
```

