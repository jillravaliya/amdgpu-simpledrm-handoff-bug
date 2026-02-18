# When Your GPU Exists But Your Display Manager Can't Find It

![Linux](https://img.shields.io/badge/LINUX-000000?style=for-the-badge&logo=linux&logoColor=FCC624)
![Ubuntu](https://img.shields.io/badge/UBUNTU_24.04-000000?style=for-the-badge&logo=ubuntu&logoColor=E95420)
![Kernel](https://img.shields.io/badge/KERNEL_6.17.0--14-000000?style=for-the-badge&logo=gnu&logoColor=0078D4)
![Bug](https://img.shields.io/badge/BUG_%232142087-000000?style=for-the-badge&logo=hackaday&logoColor=00D26A)

> **Black screen on every boot. GPU detected. Driver loaded. GDM couldn't render — it was initializing against the wrong DRM device.**

---

## What Happened

Starting **February 16, 2026**, Ubuntu 24.04 on an ASUS laptop started failing to display the login screen.

- Black screen on first boot, sometimes second, sometimes fifth
- Power cycling eventually produced a working boot
- Three days later: six consecutive failures with no recovery

---

## Root Cause

### The Short Version

> Two DRM drivers ended up registered for the same physical GPU. GDM initialized against the wrong one, crashed, restarted, and found the correct one on the second attempt. That crash-restart cycle is the black screen.

---

### The Full Picture

At early boot, the kernel registers `simpledrm` to manage the UEFI-provided framebuffer — a memory region set up by firmware that lets the kernel display output before any real GPU driver loads. `simpledrm` is assigned DRM minor 0 — `/dev/dri/card0` — at approximately **t = 0.4s**:

```
[    0.404994] [drm] Initialized simpledrm 1.0.0 for simple-framebuffer.0 on minor 0
```

`simpledrm` is a transitional driver. When a real GPU driver loads for the same hardware, `simpledrm` is expected to unregister and release its minor number so the real driver can take over `card0`.

On this system, `amdgpu` loads at **t = 16.5s**:

```
[   16.512693] [drm] Initialized amdgpu 3.64.0 for 0000:04:00.0 on minor 1
```

`amdgpu` was assigned **minor 1** (`card1`) — not minor 0. `simpledrm` had not released `card0` by the time `amdgpu` registered. There is no message in `dmesg` indicating the handoff was attempted.

Worth noting: **DRM minor numbers are not reassigned after `simpledrm` eventually unregisters.** Once `amdgpu` is assigned minor 1, it keeps it. There is no mechanism that moves it to minor 0 later.
No handoff message. No error. Nothing in the log.
> Full kernel log with both minor assignments: [`proof/bug-dmesg-card1.txt`](./proof/bug-dmesg-card1.txt)

---

### Why the Handoff Does Not Occur

The DRM subsystem has a device takeover mechanism for exactly this scenario — an early-boot framebuffer driver releasing its minor when a real GPU driver loads for the same hardware.

On this system, that mechanism did not trigger. The observed facts from `dmesg`:

- `simpledrm` registered at `t = 0.4s` on minor 0
- `amdgpu` registered at `t = 16.5s` on minor 1
- `dmesg | grep -i "takeover\|handoff"` returns no output

Why it doesn't trigger on this hardware is not something I was able to determine from logs alone. It has been reported for kernel developers to investigate — see the bug report below.

---

### Why GDM Fails to Start

When GDM starts, Mutter enumerates available DRM devices and picks up `card0` first. With `simpledrm` holding `card0`, Mutter tries to initialize a Wayland KMS backend against it and calls `gbm_surface_lock_front_buffer()`. `simpledrm` has no GBM hardware backend — it manages a firmware memory region, not a real GPU. The call returns `ENODEV`:

```
gnome-shell[1743]: Failed to ensure KMS FB ID on /dev/dri/card0:
                   drmModeAddFB failed: No such device
```

Mutter exits. systemd restarts GDM. On the second attempt, `card0` is gone — `simpledrm` has since unregistered. Mutter finds `card1` (`amdgpu`), initializes successfully, and the login screen appears.

This restart cycle is the 5–10 second black screen on every boot.
> Full GDM journal — process `1743` failure on `card0`, process `2509` success on `card1`: [`proof/bug-gnome-crash.txt`](./proof/bug-gnome-crash.txt)

---

### Why the Failure Was Initially Intermittent

Early on the failure did not happen every boot. My best guess based on the pattern: on some boots, GDM started scanning `/dev/dri/` after `simpledrm` had already released `card0`, so Mutter landed on `card1` directly without crashing. As boot timing changed due to other system updates, GDM started consistently while `simpledrm` still held `card0`, making the failure happen every time.

> This is an inference from the observed pattern, not something I was able to confirm with precise measurement.

---

## The Workaround

The simplest fix I found: prevent `simpledrm` from loading at all. With no early-boot DRM driver present, `amdgpu` is the first to register and correctly claims minor 0 (`card0`).

**Step 1 — Edit GRUB config:**

```bash
sudo nano /etc/default/grub
```

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash initcall_blacklist=simpledrm_platform_driver_init"
```

**Step 2 — Apply and reboot:**

```bash
sudo update-grub && sudo reboot
```

**Step 3 — Verify:**

```bash
$ ls /dev/dri/
card0  renderD128

$ dmesg | grep "Initialized.*drm"
[16.512693] [drm] Initialized amdgpu 3.64.0 for 0000:04:00.0 on minor 0
```

**`amdgpu` assigned minor 0 directly. No crash. No restart cycle.**
> Before/after `ls /dev/dri/` and `udevadm` output: [`proof/bug-card1-proof.txt`](./proof/bug-card1-proof.txt)

**Trade-off:**

> No Plymouth boot splash for the first ~16 seconds while `amdgpu` loads. The screen stays blank until the driver is ready. This removes the symptom but does not fix the underlying kernel behavior.

---

## Bug Report

> **Reported to:** Ubuntu kernel team / DRM subsystem
- Status: **New** — awaiting review
**[Launchpad #2142087 — simpledrm fails to hand off /dev/dri/card0 to amdgpu on Picasso/Raven2](https://bugs.launchpad.net/ubuntu/+bug/2142087)**



**Observed symptoms:**
- `amdgpu` registers as `card1` instead of `card0` on every boot
- GDM fails to initialize KMS on `card0` — `drmModeAddFB: No such device`
- 5–10 second black screen on every boot while GDM restarts
- System recovers on second GDM attempt via `card1`

**Evidence attached:**

| File | Contents |
|------|----------|
| [`proof/bug-card1-proof.txt`](./proof/bug-card1-proof.txt) | `udevadm`, `lspci` output — `ls /dev/dri/` before and after workaround |
| [`proof/bug-dmesg-card1.txt`](./proof/bug-dmesg-card1.txt) | Full `dmesg` — `simpledrm` on minor 0, `amdgpu` on minor 1, no handoff messages |
| [`proof/bug-gnome-crash.txt`](./proof/bug-gnome-crash.txt) | Full GDM journal — process `1743` crash on `card0`, process `2509` success on `card1` |

---

## Affected Hardware

- AMD Picasso/Raven 2 [Radeon Vega Series] `[1002:15d8]`
- Kernel `6.17+`
- Ubuntu `24.04` / GDM + GNOME Wayland

---

## Repo Structure

```
README.md          ← You are here
analysis.md        ← Full investigation: 6 attempts, complete failure chain
proof/
  ├── bug-card1-proof.txt      ← amdgpu assigned to card1 instead of card0
  ├── bug-dmesg-card1.txt      ← Full kernel log: simpledrm minor 0, amdgpu minor 1
  └── bug-gnome-crash.txt      ← GDM crash on card0, restart, success on card1
```

> Full investigation with every step and the complete failure chain: **[analysis.md](./analysis.md)**

---
## Connect With Me

I'm actively learning and building in the **systems programming** and **kernel development** space.

- **Email:** jillahir9999@gmail.com
- **LinkedIn:** [linkedin.com/in/jill-ravaliya-684a98264](https://linkedin.com/in/jill-ravaliya-684a98264)
- **GitHub:** [github.com/jillravaliya](https://github.com/jillravaliya)

**Open to:**

- Kernel development mentorship
- Systems programming collaboration
- Technical discussions on DRM/KMS internals
- Open source contribution guidance

---

> **This investigation happened 5 days after Bug #2141741 (the systemd kernel panic bug). Both bugs taught me the same lesson: when something fails silently or randomly, the real bug is always deeper than the visible symptom.**

<div align="center">
<img src="https://readme-typing-svg.herokuapp.com?font=Fira+Code&size=18&pause=800&duration=3000&color=ED1C24&center=true&vCenter=true&width=600&lines=From+Black+Screen+to+DRM+Bug;simpledrm+Handoff+Broken;amdgpu+Stuck+on+card1;Second+Kernel+Bug+Filed" alt="Typing SVG" />
</div>
