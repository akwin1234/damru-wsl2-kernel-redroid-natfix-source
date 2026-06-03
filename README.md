# Damru WSL2 Redroid NAT Kernel

This repository is a Damru-focused WSL2 kernel source tree based on Microsoft's `WSL2-Linux-Kernel` commit `308063c46f2741b64f3f9120fd85372ac842d6d8` (`linux-msft-wsl-6.6.114.1`).

It exists so Damru users can reproduce the WSL2 custom kernel used for Redroid plus Docker bridge/NAT support.

## Download Prebuilt Kernel

Most Damru users should use the compiled release asset instead of rebuilding the kernel.

- Release: https://github.com/akwin1234/damru-wsl2-kernel-redroid-natfix-source/releases/tag/v6.6.114.1-damru-redroid-natfix-20260602
- Kernel binary: https://github.com/akwin1234/damru-wsl2-kernel-redroid-natfix-source/releases/download/v6.6.114.1-damru-redroid-natfix-20260602/wsl2-kernel-redroid-natfix-20260602
- SHA256: `1c2a5c2c4737a02b8f81dcd82162727cb5644d194bb9cfb2f9162a9862b03c6e`

PowerShell install example:

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.damru\wsl-kernel" | Out-Null
Invoke-WebRequest `
  -Uri "https://github.com/akwin1234/damru-wsl2-kernel-redroid-natfix-source/releases/download/v6.6.114.1-damru-redroid-natfix-20260602/wsl2-kernel-redroid-natfix-20260602" `
  -OutFile "$env:USERPROFILE\.damru\wsl-kernel\wsl2-kernel-redroid-natfix-20260602"
Get-FileHash -Algorithm SHA256 "$env:USERPROFILE\.damru\wsl-kernel\wsl2-kernel-redroid-natfix-20260602"
```

Then create or update `%UserProfile%\.wslconfig`:

```ini
[wsl2]
kernel=C:\Users\<you>\.damru\wsl-kernel\wsl2-kernel-redroid-natfix-20260602
```

Restart WSL:

```powershell
wsl --shutdown
```

## What Changed

The important change is the checked-in Damru kernel config:

- `damru-redroid-natfix.config`
- `Documentation/damru/wsl2-redroid-natfix.config`

That config keeps the WSL2/Redroid binder support needed by Damru and makes Docker bridge/NAT dependencies built into the kernel instead of relying on missing WSL module files.

Key built-in options include:

- `CONFIG_BRIDGE=y`
- `CONFIG_BRIDGE_NETFILTER=y`
- `CONFIG_STP=y`
- `CONFIG_LLC=y`
- `CONFIG_IP_NF_IPTABLES=y`
- `CONFIG_IP_NF_FILTER=y`
- `CONFIG_IP_NF_NAT=y`
- `CONFIG_IP_NF_TARGET_MASQUERADE=y`
- `CONFIG_IP_NF_TARGET_REDIRECT=y`
- `CONFIG_IP_NF_MANGLE=y`
- `CONFIG_IP_NF_RAW=y`
- `CONFIG_NETFILTER_XT_MATCH_ADDRTYPE=y`
- `CONFIG_NETFILTER_XT_MATCH_CONNTRACK=y`
- `CONFIG_NETFILTER_XT_MATCH_MULTIPORT=y`
- `CONFIG_NETFILTER_XT_MATCH_STATE=y`
- `CONFIG_NETFILTER_XT_TARGET_MASQUERADE=y`
- `CONFIG_NETFILTER_XT_TARGET_REDIRECT=y`

There is also a small compile-compatibility source fix in `tools/lib/bpf/libbpf.c`.

## Build From Source

Build inside WSL/Linux, not Windows. The Linux kernel tree contains filenames that Windows cannot safely check out.

```bash
sudo apt update
sudo apt install -y build-essential flex bison dwarves libssl-dev libelf-dev bc cpio qemu-utils

cp damru-redroid-natfix.config .config
make olddefconfig
make -j"$(nproc)"
```

The built kernel image will be at:

```text
arch/x86/boot/bzImage
```

## Manual Install In WSL2

Copy `arch/x86/boot/bzImage` to a stable Windows path, for example:

```text
C:\Users\<you>\.damru\wsl-kernel\wsl2-kernel-redroid-natfix
```

Then set `%UserProfile%\.wslconfig`:

```ini
[wsl2]
kernel=C:\Users\<you>\.damru\wsl-kernel\wsl2-kernel-redroid-natfix
```

Restart WSL:

```powershell
wsl --shutdown
```

## Verify

Inside the WSL distro:

```bash
zgrep -E 'ANDROID_BINDER|BRIDGE_NETFILTER|XT_MATCH_ADDRTYPE|IP_NF_NAT|TARGET_MASQUERADE' /proc/config.gz
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -t nat -A POSTROUTING -s 172.17.0.0/16 -j MASQUERADE
docker run --rm alpine ping -c 3 8.8.8.8
```

Expected result: Docker containers can reach the internet, and Damru Redroid can use the custom WSL kernel path.

## Upstream

Original source: https://github.com/microsoft/WSL2-Linux-Kernel

Base commit: `308063c46f2741b64f3f9120fd85372ac842d6d8`