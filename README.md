# GPU Passthrough for Proxmox LXC Container

This comprehensive guide walks you through setting up NVIDIA GPU passthrough to LXC containers in Proxmox VE. This enables containerized applications to directly access GPU hardware for AI/ML workloads, gaming, or other GPU-accelerated tasks.

## Initial Proxmox Setup

### Post-Installation Script (Recommended for New Installs)
For new Proxmox installations, it's highly recommended to run the community post-install script which optimizes repositories, updates sources, and applies essential configurations:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/post-pve-install.sh)"
```

### Essential System Updates and Dependencies
Update the system and install necessary packages including kernel headers, build tools, and monitoring utilities:

```bash
apt update && apt upgrade -y 
```
```bash
apt install pve-headers-$(uname -r)
```
```bash
apt install build-essential
```
```bash
apt install software-properties-common
```
```bash
apt install make
```

> [!NOTE]
> Make sure no errors on this piece, otherwise won't be able to continue later.

```bash
update-initramfs -u
```

```bash
reboot
```

## NVIDIA Driver Installation

### Download NVIDIA Drivers
Get the appropriate NVIDIA drivers for your specific GPU from the [official NVIDIA driver page](https://www.nvidia.com/en-us/drivers/) and download directly to your Proxmox host:

```bash
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/580.82.09/NVIDIA-Linux-x86_64-580.82.09.run
```

### Install Drivers with DKMS Support
Make the driver executable and install with DKMS (Dynamic Kernel Module Support) for automatic rebuilding across kernel updates:

```bash
chmod +x NVIDIA-Linux-x86_64-580.82.09.run
```

```bash
./NVIDIA-Linux-x86_64-580.82.09.run --dkms
```
> [!NOTE]
> Installation Steps:
> - MIT/GPL
> - Alternate method of installing the NVIDIA drivers was detected - Continue
> - Building kernel modules - Be patient
> - No to 32bit libraries
> - Rebuild initramfs
> - No to Nvidia X driver

> [!WARNING]  
> If you encounter a compiler version mismatch error during installation (`cc: error: unrecognized command-line option '-ftrivial-auto-var-init=zero'`), install GCC 12:

```bash
apt update
apt install gcc-12 g++-12 -y
export CC=/usr/bin/gcc-12
```

Reboot once drivers installed

```bash
reboot
```

## GPU Device Configuration

### Identify NVIDIA Device Numbers
List all NVIDIA device files to identify the major and minor device numbers. These numbers are crucial for configuring container GPU access permissions:

```bash
ls -al /dev/nvidia*
```

**Example output - Record these device numbers for configuration:**
```bash
root@ai-node1:~# ls -al /dev/nvidia*
crw-rw-rw- 1 root root 195,   0 Sep 10 10:16 /dev/nvidia0
crw-rw-rw- 1 root root 195, 255 Sep 10 10:16 /dev/nvidiactl
crw-rw-rw- 1 root root 510,   0 Sep 10 10:16 /dev/nvidia-uvm
crw-rw-rw- 1 root root 510,   1 Sep 10 10:16 /dev/nvidia-uvm-tools

/dev/nvidia-caps:
total 0
drwxr-xr-x  2 root root     80 Sep 10 10:16 .
drwxr-xr-x 19 root root   9520 Sep 10 10:36 ..
cr--------  1 root root 235, 1 Sep 10 10:16 nvidia-cap1
cr--r--r--  1 root root 235, 2 Sep 10 10:16 nvidia-cap2
root@ai-node1:~# 
```

**Important:** Note the major device numbers: `195` (nvidia0), `510` (uvm), `235` (nvidia-cap*)

## LXC Container Setup

### Create LXC Container
Create an LXC container but **DO NOT start it yet**. 
- **Recommended OS:** Debian (same base as Proxmox) 
- **Also verified:** Ubuntu
- **Pro tip:** Create a privileged LXC container as some applications/marketplaces require additional permissions that unprivileged containers cannot provide

For privileged container creation guide: [LXC Privileged Container Guide](https://github.com/en4ble1337/proxmox-tools/blob/main/lxc-privledged-ct.md)

### Configure Container GPU Access
Once the container is created, modify its configuration file. In this example, the LXC container ID is `105`. Add the GPU passthrough configuration:

```bash
nano /etc/pve/lxc/105.conf
```

**Append the following configuration (using device numbers from previous step):**
```bash
lxc.cgroup2.devices.allow: c 195:* rwm
lxc.cgroup2.devices.allow: c 235:* rwm
lxc.cgroup2.devices.allow: c 510:* rwm
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-modeset dev/nvidia-modeset none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-caps/nvidia-cap1 dev/nvidia-caps/nvidia-cap1 none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-caps/nvidia-cap2 dev/nvidia-caps/nvidia-cap2 none bind,optional,create=file
```

**Note:** The `.allow:` numbers are derived from `ls -al /dev/nvidia*` output. This configuration is for a single GPU system.

### Multi-GPU Systems (Reference Only)
For systems with multiple GPUs, additional mount entries are required for each GPU device:

```bash
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidia1 dev/nvidia1 none bind,optional,create=file
lxc.mount.entry: /dev/nvidia2 dev/nvidia2 none bind,optional,create=file
lxc.mount.entry: /dev/nvidia3 dev/nvidia3 none bind,optional,create=file
```

**Reference:** Above example shows a system with 4 GPUs (0,1,2,3), requiring additional mount entries for each device.

## Container Driver Installation

### Start Container and Copy Drivers
Save the configuration and start the LXC container, then copy the NVIDIA drivers into the container:

```bash
pct start 105
```

**Copy drivers to container (container must be running):**
```bash
pct push 105 NVIDIA-Linux-x86_64-580.82.09.run /root/NVIDIA-Linux-x86_64-580.82.09.run
```

### Install Drivers Inside Container
Enter the container and install the identical drivers (version must match host drivers):

```bash
pct enter 105
chmod +x NVIDIA-Linux-x86_64-580.82.09.run
./NVIDIA-Linux-x86_64-580.82.09.run --no-kernel-modules
```

## Verification

### Test GPU Access
Reboot the container and verify NVIDIA GPU recognition:

```bash
nvidia-smi
```

If `nvidia-smi` returns GPU information, the passthrough configuration is successful.

## Next Steps

### Docker with NVIDIA Support
For containerized GPU workloads, set up Docker with NVIDIA runtime support:
📖 **Instructions:** [Docker NVIDIA Setup Guide](https://github.com/en4ble1337/ai-linux-tools/blob/main/docker-nvidia.md)

---

## Troubleshooting Tips

- Ensure driver versions match exactly between host and container
- Verify device numbers match your system's output from `ls -al /dev/nvidia*`
- For multiple GPUs, add mount entries for all GPU devices
- Use privileged containers for applications requiring extended permissions
- Reboot both host and container after driver installations
