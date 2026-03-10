# Fix: Network Disconnections on Proxmox
**Disabling ASPM L1 on RTL8168 NIC (r8169 driver)**
Date: March 2026 | System: Proxmox VE | IP: 192.168.178.100

---

## 1. Problem Description

The Proxmox server experienced intermittent and unpredictable network disconnections, detected via continuous ping:

| Symptom | Observed value | Cause |
|---|---|---|
| Multiple consecutive timeouts | 20–35 seconds of packet loss | NIC sleeping due to ASPM L1 |
| Extreme latency spikes | Up to 900,787 ms | Packets queued during wake-up |
| Normal latency between episodes | 3–13 ms | NIC working normally |

---

## 2. Diagnosis

### 2.1 Identified Hardware

| | |
|---|---|
| Network card | Realtek RTL8168h/8111h (driver r8169) |
| PCI address | 0000:01:00.0 |
| Interface | nic0 → bridge vmbr0 |

### 2.2 Problem Confirmation — dmesg

The kernel log clearly showed that the OS had no ASPM control:

```
r8169 0000:01:00.0: can't disable ASPM; OS doesn't have ASPM control
```

And the link state at boot:

```
r8169 0000:01:00.0 nic0: Link is Down
r8169 0000:01:00.0 nic0: Link is Up - 1Gbps/Full - flow control rx/tx
```

### 2.3 ASPM Active Confirmation — lspci

```bash
lspci -vv -s 0000:01:00.0 | grep -i aspm
```

Output:

```
LnkCtl:  ASPM L1 Enabled; RCB 64 bytes, LnkDisable- CommClk+
```

### 2.4 Why ASPM Causes the Drops

ASPM (Active State Power Management) L1 is a PCIe power-saving feature that puts the NIC into a low-power sleep state during idle periods. When network traffic arrives, the card must "wake up" — a process that can take anywhere from a few milliseconds to hundreds of seconds depending on BIOS firmware behaviour. In this case, the BIOS managed ASPM aggressively and did not yield control to the Linux kernel.

---

## 3. Solution Applied

### 3.1 Why the Kernel Parameter Is Not Enough

The standard `pcie_aspm=off` GRUB parameter **does not work** in this case because the BIOS holds exclusive control over ASPM. Instead, the PCIe register of the NIC must be written directly using `setpci`.

### 3.2 Reading the Current Value

Check the current value of the LnkCtl register (offset `CAP_EXP+10`):

```bash
setpci -s 0000:01:00.0 CAP_EXP+10.w
```

Value returned: **`0142`**

Bit interpretation:
- Bit 0 = ASPM L0s → `0` (disabled)
- Bit 1 = ASPM L1  → `1` (**ENABLED — this is the problem**)

### 3.3 Applying the Fix

Clear bit 1 (L1) while leaving all other bits unchanged: `0142` → `0140`

```bash
setpci -s 0000:01:00.0 CAP_EXP+10.w=0140
```

Immediate verification:

```bash
setpci -s 0000:01:00.0 CAP_EXP+10.w
# Expected output: 0140

lspci -vv -s 0000:01:00.0 | grep LnkCtl
# Expected output: LnkCtl: ASPM Disabled; RCB 64 bytes...
```

### 3.4 Persistence Across Reboots — rc.local

The `setpci` command is volatile and is lost on every reboot. To make it permanent:

```bash
cat > /etc/rc.local << 'EOF'
#!/bin/bash
setpci -s 0000:01:00.0 CAP_EXP+10.w=0140
exit 0
EOF

chmod +x /etc/rc.local
systemctl enable rc-local
systemctl start rc-local
```

Verify the service is running:

```bash
systemctl status rc-local
# Should show: Active: active (exited) ... status=0/SUCCESS
```

---

## 4. Post-Fix Verification

**1. Check the PCIe register:**
```bash
lspci -vv -s 0000:01:00.0 | grep LnkCtl
# Should show: ASPM Disabled
```

**2. Ping test from another host on the network:**
```bash
ping -i 0.2 192.168.178.100
# Expected: stable 3–15 ms latency, no timeouts
```

**3. Verify after reboot:**
```bash
reboot
# After boot:
lspci -vv -s 0000:01:00.0 | grep LnkCtl
# Should still show: ASPM Disabled
```

---

## 5. Notes and Warnings

- This fix is specific to PCI device `0000:01:00.0`. If the NIC is moved to a different slot or the system is reinstalled, verify that the PCI address has not changed before applying.
- The `rc.local` file is executed as root at every boot, before user services start.
- If the BIOS is updated in the future, check whether a native option to disable ASPM has been added — that would be the preferred solution.
- Disabling ASPM will slightly increase the NIC's power consumption, but on a Proxmox server this is negligible and fully acceptable.

