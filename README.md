# Custom Linux Kernel Firewall Module

A lightweight, dynamic packet filtering firewall implemented as a Linux Kernel Module (LKM). This project demonstrates direct manipulation of the Linux networking stack using **Netfilter Hooks** and creates a custom user-space interface using **Sysfs** and **Kobjects**.

## 📖 Overview

This module sits between the Network Interface Controller (NIC) driver and the Linux IP Stack. It inspects incoming packets (ingress) and decides whether to **ACCEPT** or **DROP** them based on user-defined rules.

Unlike `iptables` or `nftables`, which are generic tools, this is a custom-built solution that demonstrates the internal mechanics of packet filtering, memory management, and kernel-user communication.

## 🚀 Features

*   **Dynamic Packet Filtering:** Drop packets based on **Source IP** or **Source MAC Address**.
*   **Runtime Configuration:** Update rules instantly without unloading/reloading the module.
*   **Global Toggle:** Enable or Disable the firewall logic with a single switch.
*   **Kernel Logging:** Logs dropped packet details to the kernel ring buffer (`dmesg`).
*   **Clean Interface:** Uses a custom `/sys/kernel/` directory for control.

---

## 🏗️ Architecture & Design Decisions

This module was built with specific design choices to adhere to the "Linux Philosophy." Here is the *Why* behind the *How*:

### 1. The Mechanism: Netfilter Hooks
**Choice:** We register a callback function at `NF_INET_PRE_ROUTING`.
*   **Why?** We need to intercept packets **after** the hardware driver (E1000/VirtIO) has received them via DMA, but **before** the Linux IP stack processes them.
*   **Benefit:** This is the earliest point in the software stack to catch a packet. Dropping it here saves CPU cycles because the kernel doesn't waste time parsing the TCP/UDP headers for a packet we intend to delete.

### 2. The Interface: Sysfs (`/sys`)
**Choice:** We created custom files in `/sys/kernel/my_firewall/` rather than using `ioctl` or `module_param`.
*   **Why?** "Everything is a file." Sysfs provides a structured, text-based interface.
    *   `module_param` is too static (hard to change complex strings at runtime).
    *   `ioctl` requires writing a custom C user-space application just to send a command.
*   **Benefit:** The user can control the firewall using standard shell tools like `echo`, `cat`, and scripts, without needing special binaries.

### 3. The Organization: Kobjects
**Choice:** We explicitly created a `kobject` to serve as a directory container.
*   **Why?** In Sysfs, attributes (files) must be attached to a Kobject (directory). We attached our Kobject to `kernel_kobj`.
*   **Benefit:** This places our controls in `/sys/kernel/my_firewall/` instead of polluting the global namespace or hiding them in `/sys/module/`. It categorizes this module correctly as a "Kernel Subsystem" rather than a "Hardware Device."

### 4. Data Handling: `sk_buff`
**Choice:** We inspect the `sk_buff` (Socket Buffer) structure directly.
*   **Why?** The `sk_buff` is the universal carrier of network data in the Linux Kernel.
*   **Benefit:** By using `ip_hdr(skb)` and `eth_hdr(skb)`, we access the raw binary headers efficiently without copying memory.

---

## 🛠️ Installation & Usage

### Prerequisites
*   Linux Kernel Headers (`sudo apt install linux-headers-$(uname -r)`)
*   GCC and Make

### 1. Compile the Module
```bash
make
