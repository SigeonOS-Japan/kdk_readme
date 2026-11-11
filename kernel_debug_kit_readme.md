# Kernel Debug Kit for macOS — Read Me

Use the Kernel Debug Kit (KDK) to debug kernel-level code, such as custom-built kernel extensions.

## Table of Contents

* [Performing Two-Machine Debugging](#performing-two-machine-debugging)

  * [Identify Device Compatibility](#identify-device-compatibility)

  * [Configure Apple Silicon as a Target Device](#configure-apple-silicon-as-a-target-device)

  * [Configure an Intel-Based Mac as a Target Device](#configure-an-intel-based-mac-as-a-target-device)

  * [Configure the Host Device](#configure-the-host-device)

  * [Trigger the Target Device to Wait for the Debugger](#trigger-the-target-device-to-wait-for-the-debugger)

  * [Connect the Debugger to the Target Device](#connect-the-debugger-to-the-target-device)

  * [Finish Up Your Debugging Session](#finish-up-your-debugging-session)

* [Installing Kernel and Kernel Extension Variants](#installing-kernel-and-kernel-extension-variants)

  * [Install the Variants on an Intel-Based Mac](#install-the-variants-on-an-intel-based-mac)

  * [Removing the Variants](#removing-the-variants)

* [Configuring a Core Dump Server](#configuring-a-core-dump-server)

  * [Configure the Server Device](#configure-the-server-device)

  * [Configure the Target Device](#configure-the-target-device)

  * [Finish Up Core Dump Collection](#finish-up-core-dump-collection)

## Performing Two-Machine Debugging

The KDK supports the debugging of kernel-level code, such as kernel extensions, from a second Mac.

* **Target Device:** The Mac that runs the code you want to debug.

* **Host Device:** The Mac that runs the debugger.

### Identify Device Compatibility

* **Apple Silicon:** Perform two-machine debugging using your Mac’s built-in Ethernet ports.

* **Intel-Based Macs:** Perform two-machine debugging using your Mac’s built-in Ethernet ports, the Ethernet port on the Apple Thunderbolt display, or the Apple Thunderbolt to Gigabit Ethernet adapter. You may also use third-party Thunderbolt Ethernet adapters that support one of the following chipsets:

  * Broadcom C-IV

  * Aquantia AQC107/113

  * Intel 82574L

> \[!CAUTION\]
> You **cannot** perform two-machine debugging using USB Ethernet adapters or wireless networking on any Mac.
>
> You must connect the host and target device to the **same network**, but there are no other restrictions on how the devices connect to that network.

### Configure Apple Silicon as a Target Device

#### Step 1: Modify the Security Configuration

1. Reboot your Mac in **macOS Recovery mode**.

2. Launch **Terminal** and disable System Integrity Protection (SIP):

   ```
   csrutil disable
   
   ```

3. If prompted to lower your system security to “permissive”, enter `y` to accept.

4. Reboot your Mac.

#### Step 2: Identify the Correct Ethernet Device

Run `ifconfig` in Terminal to identify which Ethernet interface connects to the network:

```
ifconfig

```

*Example output (`en1` is the active interface):*

```
en0: flags=8963<UP,BROADCAST,SMART,RUNNING,PROMISC,SIMPLEX,MULTICAST> mtu 1500
	options=60<TSO4,TSO6>
	ether 32:00:13:er:19:e0
	media: autoselect status: inactive
en1: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	options=10b<RXCSUM,TXCSUM,VLAN_HWTAGGING,AV>
	ether 40:6c:8f:5b:a2:96
	inet6 fe80::426c:8fff:fe5b:a296%en2 prefixlen 64 scopeid 0x4
	inet 10.128.19.135 netmask 0xfffff800 broadcast 10.128.23.255
	media: autoselect (1000baseT <full-duplex>)
	status: active

```

#### Step 3: Set the `boot-args`

Set the NVRAM boot arguments:

* `debug=0x44`: Tells the kernel to wait for a debugger when receiving a non-maskable interrupt (NMI).

* `kdp_match_name=enX`: The Ethernet interface identified in Step 2 (e.g., `en1`).

* `wdt=-1`: Disables watchdog monitoring.

```
sudo nvram boot-args="debug=0x44 kdp_match_name=en1 wdt=-1"

```

#### Step 4: Reboot the Device

Reboot your target device. Upon rebooting, you may connect to it from the host machine.

### Configure an Intel-Based Mac as a Target Device

#### Step 1: Modify the Security Configuration

1. Reboot your Mac in **macOS Recovery mode**.

2. If applicable, set **Startup Security Utility** to “Reduced Security”.

3. Launch **Terminal** and disable SIP:

   ```
   csrutil disable
   
   ```

4. Reboot your Mac.

#### Step 2: Identify the Correct Ethernet Device

Run `ifconfig` in Terminal to find your active Ethernet interface (e.g., `en1`).

#### Step 3: Set the `boot-args`

```
sudo nvram boot-args="debug=0x44 kdp_match_name=en1 wdt=-1"

```

#### Step 4: Reboot the Device

Reboot the device to apply changes.

### Configure the Host Device

#### Step 1: Install Xcode and Required Python Packages

Install Xcode, then execute the following commands in Terminal to install required dependencies:

```
xcrun python3 -m pip install --user --ignore-installed macholib
xcrun python3 -m pip install --user --ignore-installed future

```

#### Step 2: Install the KDK

Install the Kernel Debug Kit package (`.pkg`) on the host machine.

### Trigger the Target Device to Wait for the Debugger

The target device halts and waits for an external debugger to attach in the following situations:

1. If the target device **panics**, it automatically halts and waits for an external debugger.

2. If you manually trigger an **NMI** on the target device.

### Connect the Debugger to the Target Device

Launch `lldb` from Terminal and execute `kdp-remote` with the IP address or hostname of the target machine:

```
(lldb) kdp-remote <name_or_ip_address>

```

`lldb` will automatically search Spotlight-indexed folders and `/Library/Developer/KDKs/` on the local system for symbol information. If it cannot find the kernel binary automatically, load it manually:

```
lldb /Library/Developer/KDKs/<KDK Version>/System/Library/Kernels/kernel

```

> \[!NOTE\]
> **Apple Silicon Limitation:** Apple silicon doesn’t support active kernel debugging. You can inspect state when halted due to a panic or NMI, but you **cannot** set breakpoints, continue execution, step into, step over, or step out of instructions.

### Finish Up Your Debugging Session

1. Quit `lldb` to disconnect the host from the target machine.

2. To restore the target device to normal operation:

#### Step 1: Remove `boot-args`

```
sudo nvram -d boot-args

```

#### Step 2: Restore Security Configuration

1. Reboot your Mac into **macOS Recovery mode**.

2. Connect to the internet.

3. Open **Terminal** and run:

   ```
   csrutil enable
   
   ```

   *(Enter `y` if prompted to raise security to "full")*

4. On Intel Macs, set **Startup Security Utility** back to “Full Security” if required.

5. Reboot your Mac.

## Installing Kernel and Kernel Extension Variants

The KDK includes several kernel and extension variants:

* **`kernel` (Release):** Matches the shipping production kernel.

* **`kernel.development` (Development):** Safe for daily use during development, minimal performance overhead.

* **`kernel.kasan` (KASAN):** Enables Kernel Address Sanitizer features. Higher memory/performance overhead; panics on memory safety violations.

> \[!NOTE\]
> Apple silicon does not support installing kernel and kernel extension variants from the KDK.

### Install the Variants on an Intel-Based Mac

#### Step 1: Install the KDK

Install the KDK package on the target device.

#### Step 2: Modify Security Configuration

1. Reboot into **macOS Recovery mode**.

2. Set Startup Security Utility to "Reduced Security".

3. Disable SIP and Authenticated Root:

   ```
   csrutil disable
   csrutil authenticated-root disable
   
   ```

4. Reboot your Mac.

#### Step 3: Identify the System Volume Disk Name

Run `mount` to locate the system volume snapshot mounted at `/`:

```
mount

```

*Example:* `/dev/disk4s5s1 on / (apfs, sealed, local, read-only, journaled)`

Remove the final `sX` suffix from the disk path. In this example, `/dev/disk4s5s1` becomes `/dev/disk4s5`.

#### Step 4: Mount a Live Version of the System Volume

```
mkdir /Users/jappleseed/livemount 
sudo mount -o nobrowse -t apfs /dev/disk4s5 /Users/jappleseed/livemount

```

#### Step 5: Copy Kernel Variants

```
sudo ditto /Library/Developer/KDKs/<KDK Version>/System /Users/jappleseed/livemount/System

```

#### Step 6: Rebuild Kernel Collections and Bless

```
sudo kmutil install --volume-root /Users/jappleseed/livemount --update-all
sudo bless --mount /Users/jappleseed/livemount --bootefi --create-snapshot

```

#### Step 7: Set `kcsuffix` in `boot-args`

Set `kcsuffix` to either `development` or `kasan`:

```
sudo nvram boot-args="debug=0x44 kdp_match_name=en1 wdt=-1 kcsuffix=development"

```

#### Step 8: Reboot and Verify

Reboot the device, then verify using `sysctl`:

```
sysctl kern.osbuildconfig

```

*Expected Output:* `kern.osbuildconfig: development`

### Removing the Variants

#### Step 1: Remove `boot-args`

```
sudo nvram -d boot-args

```

#### Step 2: Identify System Volume Disk

Run `mount` and locate the root volume device name (e.g., `/dev/disk4s5`).

#### Step 3: Mount System Volume

```
mkdir /Users/jappleseed/livemount 
sudo mount -o nobrowse -t apfs /dev/disk4s5 /Users/jappleseed/livemount

```

#### Step 4: Restore Sealed System Snapshot

```
sudo bless --mount /Users/jappleseed/livemount --bootefi --last-sealed-snapshot

```

#### Step 5: Restore Security Configuration

1. Reboot into **macOS Recovery mode**.

2. Connect to the internet.

3. Open Terminal and re-enable security protections:

   ```
   csrutil authenticated-root enable
   csrutil enable
   
   ```

4. Restore Startup Security Utility to "Full Security" if applicable.

5. Reboot your Mac.

#### Step 6: Verify

```
sysctl kern.osbuildconfig

```

*Expected Output:* `kern.osbuildconfig: release`

## Configuring a Core Dump Server

You can capture network core dumps on another Mac during kernel panics.

* **Server Device:** The Mac that receives core-dump files.

* **Target Device:** The Mac that panics and sends core-dump files.

### Configure the Server Device

#### Step 1: Create the PanicDumps Directory

```
mkdir /var/tmp/PanicDumps
chmod 1777 /var/tmp/PanicDumps

```

#### Step 2: Load `kdumpd` Launch Daemon

```
sudo launchctl load -w /System/Library/LaunchDaemons/com.apple.kdumpd.plist

```

### Configure the Target Device

1. Follow standard Target Device setup instructions above.

2. Set `debug=0xc44` and `_panicd_ip` pointing to the server's IP address:

   ```
   sudo nvram boot-args="debug=0xc44 kdp_match_name=en1 wdt=-1 _panicd_ip=10.0.40.2"
   
   ```

3. Reboot the device.

### Finish Up Core Dump Collection

To stop capturing dumps on the server:

```
sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.kdumpd.plist

```