# Project 2 – Server 1 QEMU/KVM NVMe Lab

## Daily Startup, Connection, Verification and Shutdown Procedure

**Server:** Server 1 – Dell PowerEdge R720\
**Host OS:** Ubuntu 26.04.1 LTS\
**Hypervisor:** QEMU 10.2.1 + KVM\
**Guest OS:** Ubuntu 26.04.1 LTS\
**Guest hostname:** `p2-nvme-guest`\
**NVMe controller:** QEMU NVMe Controller\
**NVMe namespace:** 8 GiB, NSID 1

---

# 1. Lab Architecture

Server 1 contains a physical Ubuntu host running a QEMU/KVM virtual machine. The virtual machine exposes an emulated PCIe NVMe controller and an 8 GiB NVMe namespace to its guest operating system.

```
RHEL Laptop
    |
    | Tailscale Network
    |
    +----------------------------+
    |                            |
    v                            v
Server 1 Host                SSH Client
winteck                      Terminal 3
100.105.92.39                     |
    |                             |
    | QEMU/KVM                     |
    |                             |
    v                             |
QEMU Guest <----------------------+
p2-nvme-guest
    |
    +-- Ubuntu OS Disk
    |   /dev/vda
    |   24 GiB Virtio
    |
    +-- Cloud-init Seed
    |   /dev/vdb
    |   Virtio
    |
    +-- NVMe Controller
        PCI BDF: 00:04.0
        Serial: P2NVME00000001
             |
             v
        NVMe Namespace 1
        /dev/nvme0n1
        8 GiB
```

## 1.1 Physical Server 1

| Property         | Configuration               |
| ---------------- | --------------------------- |
| Hostname         | `winteck`                   |
| Manufacturer     | Dell                        |
| Model            | PowerEdge R720              |
| Operating System | Ubuntu 26.04.1 LTS          |
| Kernel           | 7.0.0-44-generic (at setup) |
| CPU              | 2 × Intel Xeon E5-2660      |
| Logical CPUs     | 32                          |
| RAM              | 129 GiB                     |
| Root filesystem  | Approximately 98 GiB        |
| Virtualization   | KVM                         |
| QEMU             | 10.2.1                      |
| OVMF             | Installed                   |

## 1.2 QEMU Guest

| Property                | Configuration      |
| ----------------------- | ------------------ |
| Hostname                | `p2-nvme-guest`    |
| Operating System        | Ubuntu 26.04.1 LTS |
| Virtual CPU             | 4                  |
| RAM                     | 4 GiB              |
| Firmware                | OVMF UEFI          |
| Accelerator             | KVM                |
| Guest OS disk           | 24 GiB QCOW2       |
| Guest OS disk device    | `/dev/vda`         |
| Cloud-init device       | `/dev/vdb`         |
| NVMe controller         | QEMU NVMe          |
| NVMe namespace          | 8 GiB              |
| Namespace ID            | 1                  |
| Namespace block device  | `/dev/nvme0n1`     |
| NVMe management utility | nvme-cli 2.16      |

## 1.3 Important Host and Guest Separation

There are two different operating systems:

**Physical host:**

```
winteck
```

**Virtual guest:**

```
p2-nvme-guest
```

Commands executed on the host and commands executed inside the guest are not interchangeable.

The QEMU NVMe namespace is exposed to the guest. It is not a physical NVMe SSD installed in Server 1.

The physical host's disks must not be confused with the guest's virtual disks.

---

# 2. Automatic Startup Configuration

We configured a systemd service on the physical Ubuntu host:

```
p2-nvme-guest.service
```

Service location:

```
/etc/systemd/system/p2-nvme-guest.service
```

The service is:

- Enabled for automatic startup.
- Configured to run as the `winteck` account.
- Configured with KVM supplementary-group access.
- Responsible for starting QEMU.
- Configured with a QMP control socket.
- Configured to request graceful guest shutdown.

The QEMU startup script is:

```
/home/winteck/project2-nvme-lab/scripts/start-guest.sh
```

The graceful shutdown helper is:

```
/home/winteck/project2-nvme-lab/scripts/stop-guest.py
```

The QEMU PID file is:

```
/home/winteck/project2-nvme-lab/p2-nvme-guest.pid
```

The QMP control socket is:

```
/run/p2-nvme-guest/qmp.sock
```

## 2.1 What happens when Server 1 boots?

```
Physical Server 1 powers on
          |
          v
Ubuntu Host boots
          |
          v
Network and Tailscale initialize
          |
          v
systemd starts p2-nvme-guest.service
          |
          v
QEMU launches with KVM
          |
          v
OVMF UEFI initializes
          |
          v
Ubuntu Guest boots
          |
          v
NVMe Controller initializes
          |
          v
NVMe Namespace is discovered
          |
          v
/dev/nvme0n1 becomes available
```

**Important:** Automatic guest startup after a normal physical host reboot has already been tested successfully.

Automatic physical power restoration after a complete AC power interruption has not yet been verified. That depends on the Dell BIOS power-recovery configuration.

---

# 3. Daily Startup Procedure

Whenever you switch on physical Server 1, follow this procedure.

## Step 1 – Power on Server 1

Switch on the Dell PowerEdge R720.

Wait for:

- Physical server POST to complete.
- Ubuntu host to boot.
- Network connectivity to become available.
- Tailscale connectivity to become available.
- systemd to start the QEMU guest.

Allow approximately 1–2 minutes initially, then check connectivity.

You do not need to manually start QEMU.

You do not need to manually boot the Ubuntu guest.

## Step 2 – Open RHEL Terminal 1: SSH Tunnel

Open a local terminal on your RHEL laptop.

This terminal will be dedicated to maintaining the SSH tunnel between your laptop and the QEMU guest.

Execute:

```
ssh -o PubkeyAuthentication=no \
    -o PreferredAuthentications=password \
    -L 2222:127.0.0.1:2222 \
    -N \
    winteck@100.105.92.39
```

Enter your normal `winteck` Linux password.

### Expected behaviour

The terminal will remain occupied and display a blinking cursor.

This is expected.

The `-N` option means SSH will not open a remote shell. It will maintain the port-forwarding connection instead.

**Keep Terminal 1 open throughout your learning session.**

Do not press `Ctrl+C` while using the guest.

## Step 3 – Open RHEL Terminal 2: Connect to QEMU Guest

Open a second local terminal on your RHEL laptop.

Execute:

```
ssh -i ~/.ssh/id_ed25519_project2_qemu \
    -p 2222 \
    nvme@127.0.0.1
```

Your dedicated Project 2 SSH private key is used for guest authentication.

Expected prompt:

```
nvme@p2-nvme-guest:~$
```

You are now inside the Ubuntu guest.

**Terminal 2 is your primary learning and practical execution terminal.**

You will use it for:

- Linux commands inside the guest.
- NVMe management commands.
- PCI device inspection.
- Storage-device discovery.
- NVMe Admin command inspection.
- Future controlled test scenarios.
- Python and automation exercises.

---

# 4. Daily Guest Verification

After connecting to the guest, execute the following commands.

## 4.1 Verify guest identity

```
hostname
whoami
hostnamectl
```

Expected hostname:

```
p2-nvme-guest
```

Expected user:

```
nvme
```

## 4.2 Verify block devices

```
lsblk -o NAME,TYPE,SIZE,MODEL,TRAN,MOUNTPOINTS
```

Expected important devices:

| Device         | Expected size         | Purpose             |
| -------------- | --------------------- | ------------------- |
| `/dev/vda`     | 24 GiB                | Ubuntu guest OS     |
| `/dev/vdb`     | Approximately 366 KiB | Cloud-init seed     |
| `/dev/nvme0n1` | 8 GiB                 | NVMe test namespace |

## 4.3 Verify NVMe controller and namespace

```
sudo nvme list
```

Expected device:

```
/dev/nvme0n1
```

Expected controller serial:

```
P2NVME00000001
```

Expected model:

```
QEMU NVMe Ctrl
```

## 4.4 Verify controller identity

```
sudo nvme id-ctrl /dev/nvme0 \
    | grep -E '^(sn|mn|fr|ver)'
```

Expected important values:

```
sn        : P2NVME00000001
mn        : QEMU NVMe Ctrl
fr        : 10.2.1
ver       : 0x10400
```

## 4.5 Verify the NVMe controller PCI function

```
lspci -nn
```

Expected NVMe PCI entry:

```
00:04.0 Non-Volatile memory controller [0108]:
Red Hat, Inc. QEMU NVM Express Controller [1b36:0010]
```

## 4.6 Verify guest memory

```
free -h
```

Approximately 4 GiB should be available to the guest.

---

# 5. Terminal Arrangement

Keep the following terminal arrangement during practical sessions.

## Terminal 1 – SSH Tunnel

**Location:** RHEL laptop local terminal.

Purpose: Maintains the connection between the laptop and guest.

Command:

```
ssh -o PubkeyAuthentication=no \
    -o PreferredAuthentications=password \
    -L 2222:127.0.0.1:2222 \
    -N \
    winteck@100.105.92.39
```

Keep it running.

## Terminal 2 – QEMU Guest

**Location:** RHEL laptop local terminal.

Purpose: Main practical working terminal.

Command:

```
ssh -i ~/.ssh/id_ed25519_project2_qemu \
    -p 2222 \
    nvme@127.0.0.1
```

Expected prompt:

```
nvme@p2-nvme-guest:~$
```

## Terminal 3 – Physical Server 1 Host

**Location:** RHEL laptop local terminal.

Purpose: Host administration and QEMU management.

Command:

```
ssh winteck@100.105.92.39
```

Expected prompt:

```
winteck@winteck:~$
```

Terminal 3 is optional during normal learning sessions. Use it when host administration is necessary.

---

# 6. Server 1 Host Administration Commands

These commands must be executed on the physical Ubuntu host, not inside the guest.

## 6.1 Check guest service status

```
sudo systemctl status p2-nvme-guest.service --no-pager -l
```

## 6.2 Check whether the guest is running

```
systemctl is-active p2-nvme-guest.service
```

Expected:

```
active
```

## 6.3 Check whether automatic startup is enabled

```
systemctl is-enabled p2-nvme-guest.service
```

Expected:

```
enabled
```

## 6.4 Check QEMU process

```
pgrep -af qemu-system-x86_64
```

## 6.5 Check QEMU PID

```
cat ~/project2-nvme-lab/p2-nvme-guest.pid
```

## 6.6 Check SSH forwarding port

```
ss -ltnp | grep ':2222'
```

Expected listener:

```
127.0.0.1:2222
```

## 6.7 Check service journal

```
sudo journalctl -u p2-nvme-guest.service -n 50 --no-pager
```

## 6.8 Check service logs from the current host boot

```
sudo journalctl -u p2-nvme-guest.service -b --no-pager
```

## 6.9 Check guest serial output

```
tail -n 50 ~/project2-nvme-lab/logs/p2-nvme-guest-serial.log
```

## 6.10 Check QEMU errors

```
tail -n 50 ~/project2-nvme-lab/logs/p2-nvme-guest-qemu.log
```

---

# 7. Starting and Stopping the Guest Manually

Normally, the systemd service handles guest startup automatically.

However, when required, use the following commands from **Terminal 3 on Server 1**.

## 7.1 Start the guest

```
sudo systemctl start p2-nvme-guest.service
```

Verify:

```
systemctl is-active p2-nvme-guest.service
```

## 7.2 Stop the guest gracefully

```
sudo systemctl stop p2-nvme-guest.service
```

The service invokes the Python shutdown helper, which communicates through QMP and sends an ACPI system-powerdown request.

Ubuntu then performs its normal shutdown sequence.

This avoids abruptly terminating QEMU.

Verify:

```
systemctl is-active p2-nvme-guest.service
```

Expected:

```
inactive
```

## 7.3 Restart the guest

```
sudo systemctl restart p2-nvme-guest.service
```

Verify:

```
sudo systemctl status p2-nvme-guest.service --no-pager
```

**Important:** Never start the QEMU guest manually with `start-guest.sh` while systemd is already managing it. Always use `systemctl` to avoid running multiple QEMU processes against the same virtual disks.

---

# 8. Ending a Daily Learning Session

Closing the guest SSH terminal does not shut down the QEMU guest.

This is the normal behaviour.

## Option A – Continue running the guest

If you have finished learning but want the guest to remain available:

1. Exit Terminal 2:
   ```
   exit
   ```
2. Keep Terminal 1 open if you will reconnect soon.
3. The QEMU guest continues running in the background.
4. You can reconnect through a new Terminal 2 whenever required.

## Option B – Shut down the guest

If you want to stop the guest and release its resources:

1. Connect to Server 1 through Terminal 3:
   ```
   ssh winteck@100.105.92.39
   ```
2. Execute:
   ```
   sudo systemctl stop p2-nvme-guest.service
   ```
3. Verify:
   ```
   systemctl is-active p2-nvme-guest.service
   ```
4. Close Terminal 1 after the guest has stopped.

The next time you manually start the guest, use:

```
sudo systemctl start p2-nvme-guest.service
```

---

# 9. Physical Server 1 Reboot Behaviour

The systemd service is enabled for automatic startup.

When the physical host is rebooted normally:

```
Server 1 reboot
      |
      v
Ubuntu host starts
      |
      v
systemd starts QEMU guest
      |
      v
Ubuntu guest starts
      |
      v
NVMe controller initializes
      |
      v
NVMe namespace is discovered
      |
      v
/dev/nvme0n1 becomes available
```

We have already tested this exact sequence successfully.

## After a normal physical host reboot

You only need to:

1. Wait for Server 1 to become reachable through Tailscale.
2. Re-establish Terminal 1 SSH tunnelling.
3. Open Terminal 2 and connect to the guest.
4. Verify `/dev/nvme0n1`.

You do not need to start QEMU manually.

---

# 10. Troubleshooting

## Issue 1 – Cannot SSH into Server 1

Check:

- Is the physical Dell server powered on?
- Has Ubuntu completed boot?
- Is the server connected to the network?
- Is Tailscale connected?
- Is the Tailscale IP still correct?

Try:

```
ssh winteck@100.105.92.39
```

## Issue 2 – SSH tunnel cannot connect

On Server 1, check:

```
sudo systemctl status p2-nvme-guest.service --no-pager
```

Check the port:

```
ss -ltnp | grep ':2222'
```

If the service is inactive and Server 1 is healthy:

```
sudo systemctl start p2-nvme-guest.service
```

Do not manually execute the QEMU startup script.

## Issue 3 – Cannot SSH into the guest

Check that Terminal 1 is still running.

Then, on RHEL Terminal 2:

```
nc -vz 127.0.0.1 2222
```

If the connection fails, check the host service and SSH-forwarding listener.

## Issue 4 – Guest SSH host-key warning

The guest SSH host key is stored in the RHEL laptop's `~/.ssh/known_hosts`.

If SSH reports that the host identification has changed, do not blindly remove the old entry.

First verify the new ED25519 fingerprint from the physical Server 1 host:

```
ssh-keyscan -t ed25519 -p 2222 127.0.0.1 2>/dev/null \
    | ssh-keygen -lf -
```

Only update the laptop's known-host entry after confirming the fingerprint through a trusted path.

## Issue 5 – Guest service fails to start

On Server 1:

```
sudo systemctl status p2-nvme-guest.service --no-pager -l
```

Check:

```
sudo journalctl -u p2-nvme-guest.service -n 100 --no-pager
```

Check QEMU error log:

```
tail -n 100 ~/project2-nvme-lab/logs/p2-nvme-guest-qemu.log
```

Do not create another VM or overwrite the virtual disk images while investigating.

---

# 11. Important Safety Rules

1. Do not format or partition the physical Server 1 host disks.
2. Do not confuse physical host storage with guest virtual storage.
3. Keep the guest OS disk (`vda`) separate from the NVMe test namespace (`nvme0n1`).
4. Never delete or overwrite `ubuntu-cloud-os.qcow2`.
5. Never delete or overwrite `nvme-namespace-1.raw`.
6. Do not manually start QEMU when systemd is already managing the guest.
7. Do not force-terminate the QEMU process during normal operation.
8. Do not perform destructive NVMe commands unless the specific test is planned, authorized and explained.
9. Keep the RHEL laptop's internal NVMe protected; use read-only inspection only.
10. QEMU results must not be presented as physical enterprise SSD qualification evidence.

---

# 12. Quick Daily Reference

This is the short version to follow every day.

### A. Power on Server 1

Wait for Ubuntu and Tailscale to become available.

### B. RHEL Terminal 1 – Connect SSH tunnel

```
ssh -o PubkeyAuthentication=no \
    -o PreferredAuthentications=password \
    -L 2222:127.0.0.1:2222 \
    -N \
    winteck@100.105.92.39
```

Keep it open.

### C. RHEL Terminal 2 – Connect to guest

```
ssh -i ~/.ssh/id_ed25519_project2_qemu \
    -p 2222 \
    nvme@127.0.0.1
```

### D. Verify NVMe

```
hostname
sudo nvme list
lsblk
```

### E. Learn and perform planned activities

Use Terminal 2 as your primary practical environment.

### F. Finish your session

Close the guest SSH connection using:

```
exit
```

The guest continues running.

To shut it down completely, use Terminal 3:

```
sudo systemctl stop p2-nvme-guest.service
```

---

# 13. Final Configuration Status

| Item                               | Status     |
| ---------------------------------- | ---------- |
| Physical Server 1 setup            | Completed  |
| Ubuntu host installation           | Completed  |
| QEMU installation                  | Completed  |
| KVM acceleration                   | Verified   |
| OVMF UEFI                          | Configured |
| Ubuntu guest installation          | Completed  |
| NVMe controller attachment         | Verified   |
| 8 GiB namespace discovery          | Verified   |
| NVMe CLI installation              | Completed  |
| Identify Controller                | Verified   |
| Identify Namespace                 | Verified   |
| SMART / Health log                 | Retrieved  |
| NVMe Error Information log         | Retrieved  |
| systemd automatic startup          | Verified   |
| Graceful shutdown                  | Verified   |
| Physical host reboot recovery      | Verified   |
| Namespace rediscovery after reboot | Verified   |
| Unexpected AC power recovery       | Not tested |

**Final operating procedure:**

Power on Server 1 → Open RHEL Terminal 1 → Open RHEL Terminal 2 → Connect to `p2-nvme-guest` → Verify `/dev/nvme0n1` → Begin the planned Project 2 work.

**Server 1 QEMU/KVM infrastructure is configured and operational.**
