# A Note on This Document

This is a **reconstructed postmortem**, not a live incident log. The investigation happened across **February 16–18, 2026** — three days of intermittent failures. I collected terminal history, journal logs, and notes throughout, then structured this as a clean narrative afterward. The order reflects the logical progression of reasoning, not a precise chronological transcript.

Evidence files are in `proof/`:
- `bug-card1-proof.txt` — `amdgpu` assigned to `card1` instead of `card0`
- `bug-dmesg-card1.txt` — full kernel log, unedited, showing `simpledrm` on minor 0 and `amdgpu` on minor 1
- `bug-gnome-crash.txt` — GDM crash on `card0`, restart, success on `card1`

---

## Background

On **February 13, 2026**, I finished investigating and reporting **Bug #2141741** — a systemd issue where missing initrd files caused kernel panics on NVMe systems. The system was stable afterward. Both kernels were bootable.

Three days later, the login screen stopped appearing.

---

## The Problem

> **February 16:** Black screen on first boot. Power cycle, second boot worked. Noted it but moved on.

> **February 17:** Same pattern, consistently.

> **February 18:** Five consecutive failures before a successful boot.

At that point it was clearly worth investigating properly.

---

## Initial Check: Is the System Running?

During a black screen, switched to a text terminal:

```bash
Ctrl + Alt + F2
```

Text login appeared immediately — the kernel was running, only the graphical session was not.

```bash
$ loginctl list-sessions
SESSION  UID  USER  SEAT   TTY   STATE   IDLE
c1       120  gdm   seat0  tty1  online  yes

$ ps aux | grep gnome-shell
gdm  1734  /usr/bin/gnome-shell  (running on tty1)
```

GDM (the GNOME display manager — the service responsible for showing the login screen and starting desktop sessions) and GNOME Shell were both running under the `gdm` user on `tty1`. The display was blank despite the session being active.

---

## Attempt 1: Screen Blanking

**Hypothesis:** Login screen appears but power-saves to blank before it becomes visible.

GDM runs as its own `gdm` system user with a separate dconf configuration — dconf is GNOME's per-user settings database, similar to a registry. Because GDM is a separate user, its idle and screensaver settings are stored separately from the desktop user's settings. Added a system-wide idle override for the `gdm` user:

```bash
$ sudo nano /etc/dconf/db/gdm.d/00-no-screen-blank
```

```ini
[org/gnome/desktop/session]
idle-delay=uint32 0

[org/gnome/desktop/screensaver]
idle-activation-enabled=false
```

```bash
$ sudo dconf update
$ sudo systemctl restart gdm
```

Login screen appeared after restarting GDM. Rebooted to confirm. Black screen again.

**Result:** Not the cause. Moved on.

---

## Attempt 2: PAM Authentication Failure

**Hypothesis:** Something in the authentication stack is blocking session startup.

```bash
$ journalctl -b 0 --priority=err
pam_unix(gdm-password:auth): conversation failed
```

PAM (Pluggable Authentication Modules) is the Linux subsystem that handles all login authentication — it's what checks passwords when you log in via GDM, `sudo`, SSH, and so on. A `conversation failed` error means PAM started an authentication exchange but got no valid response.

Checked the PAM configuration:

```bash
$ cat /etc/pam.d/common-auth
auth [success=1 default=ignore] pam_sss.so use_first_pass
```

`pam_sss.so` is the module for SSSD — a daemon used for authenticating against remote directory services like LDAP or Active Directory. A few weeks earlier I had installed SSSD while testing a local LDAP setup for a side project. The project was abandoned and I ran `apt remove sssd`, which removes the binary but leaves the entries it wrote in `/etc/pam.d/`. PAM was still trying to call a module that no longer existed.

```bash
$ systemctl status sssd
● sssd.service
   Active: inactive (dead)
```

Commented out the stale `pam_sss.so` lines in the three affected files:

```bash
$ sudo nano /etc/pam.d/common-auth
$ sudo nano /etc/pam.d/common-account
$ sudo nano /etc/pam.d/common-session
```

```bash
$ sudo reboot
$ journalctl -b 0 --priority=err
# No PAM errors
```

Authentication errors were gone. Black screen remained.

**Result:** Fixed an unrelated misconfiguration left over from a previous install. Not the cause of the display issue.

---

## Attempt 3: Waybar Service Errors

```bash
$ systemctl --user status
waybar.service: Failed with result 'exit-code'
waybar[4236]: Failed to acquire required resources
```

Waybar is a status bar application designed for Sway and Hyprland — tiling window managers that use a different Wayland compositor than GNOME. It was left over from testing Hyprland a few weeks earlier. Under GNOME it has no valid compositor to connect to and was failing on every boot.

```bash
$ sudo systemctl mask waybar
$ systemctl --user mask waybar
$ sudo reboot
```

Waybar errors gone. Black screen remained.

**Result:** Removed a failing leftover service. Not the cause.

---

## Attempt 4: AMD GPU Firmware Warnings

```bash
$ lspci | grep VGA
04:00.0 VGA compatible controller: AMD Picasso/Raven 2 [Radeon Vega Series]

$ lsmod | grep amdgpu
amdgpu  20094976  5

$ dmesg | grep amdgpu | grep -i error
amdgpu 0000:04:00.0: amdgpu: failed to load ucode RLC_RESTORE_LIST_CNTL(0x29)
amdgpu 0000:04:00.0: amdgpu: psp gfx command LOAD_IP_FW(0x6) failed
```

These looked like potential driver failures. After searching for these specific messages on this hardware, I found multiple reports indicating they are non-fatal warnings — the driver marks them as such and continues loading. The full `dmesg` in `proof/bug-dmesg-card1.txt` shows the driver completing initialization successfully after these lines.

**Result:** Not the cause. Driver was loading correctly.

---

## Attempt 5: Wayland vs X11

**Hypothesis:** The failure is specific to the Wayland session path.

Linux has two display protocols — X11 (the older system) and Wayland (the modern replacement). Ubuntu 24.04 defaults to Wayland. Forcing GDM back to X11 tests whether the issue is Wayland-specific:

```bash
$ sudo nano /etc/gdm3/custom.conf
```

```ini
[daemon]
WaylandEnable=false
```

```bash
$ sudo systemctl restart gdm
$ echo $XDG_SESSION_TYPE
x11
```

Login screen appeared immediately on X11. Rebooted to confirm.

Black screen again. Checked session type from TTY:

```bash
$ echo $XDG_SESSION_TYPE
wayland
```

The setting had reverted. The `gdm3` package rewrites `custom.conf` on certain triggers — the change did not persist across a reboot.

**Result:** X11 worked where Wayland did not. The issue was somewhere in how Wayland initialized the display backend. This narrowed the investigation to Mutter's KMS/DRM initialization path. Mutter is GNOME's compositor and window manager — the component responsible for rendering the desktop and managing GPU access under Wayland.

---

## Breakthrough: Reading the Full Journal

Until this point I had been filtering logs for errors only. Read the full journal from a failed boot without filtering:

```bash
$ journalctl -b 0 | grep -i "gnome-shell\|gdm\|wayland" | tail -40
```

```
Feb 18 10:45:16  gnome-shell[1743]: Added device '/dev/dri/card0' (simpledrm)
Feb 18 10:45:16  gnome-shell[1743]: Integrated GPU /dev/dri/card0 selected as primary
Feb 18 10:45:20  gnome-shell[1743]: Failed to ensure KMS FB ID on /dev/dri/card0:
                                     drmModeAddFB failed: No such device
Feb 18 10:45:21  gnome-shell[2509]: Added device '/dev/dri/card1' (amdgpu)
Feb 18 10:45:21  gnome-shell[2509]: GPU /dev/dri/card1 selected primary
```

Two different process IDs. Two different devices. One crash between them.

- Process `1743`: found `card0`, driver identified as `simpledrm` — crashed
- Process `2509`: new process after restart, found `card1`, driver identified as `amdgpu` — worked

GDM was crashing on `card0` and being restarted by systemd on every boot. systemd is the Linux init system — it manages all services and automatically restarts failed ones. The second GDM attempt always succeeded because it found `card1` instead. That crash-restart cycle was the black screen.

```bash
$ ls -la /dev/dri/
crw-rw----+  1 root video  226, 1  card1
crw-rw----+  1 root render 226, 128  renderD128
```

By the time I checked, only `card1` existed. `card0` had been present during boot, GDM picked it up, crashed, and `card0` was gone by the time GDM restarted.

> Full GDM journal showing both process IDs and the crash sequence: [`proof/bug-gnome-crash.txt`](./proof/bug-gnome-crash.txt)

---

## What the Boot Sequence Shows

```bash
$ sudo dmesg | grep -E "drm|card"
[    0.404994] [drm] Initialized simpledrm 1.0.0 for simple-framebuffer.0 on minor 0
[    0.407532] simple-framebuffer simple-framebuffer.0: [drm] fb0: simpledrmdrmfb frame buffer device
[   16.512693] [drm] Initialized amdgpu 3.64.0 for 0000:04:00.0 on minor 1
[   16.532792] amdgpu 0000:04:00.0: [drm] fb0: amdgpudrmfb frame buffer device
```

> Full unedited kernel log: [`proof/bug-dmesg-card1.txt`](./proof/bug-dmesg-card1.txt)

DRM (Direct Rendering Manager) is the kernel subsystem that manages GPU access — it's what creates the `/dev/dri/cardX` device files that display servers use to talk to the GPU. DRM minor numbers are assigned in registration order: minor 0 becomes `/dev/dri/card0`, minor 1 becomes `/dev/dri/card1`.

```
t = 0.4s   simpledrm registers → minor 0 → /dev/dri/card0
           Manages the UEFI-provided framebuffer memory region

t = 16.5s  amdgpu loads for PCI 04:00.0
           minor 0 already taken → assigned minor 1 → /dev/dri/card1
```

`simpledrm` is a minimal transitional kernel driver that wraps the framebuffer region set up by UEFI firmware before the real GPU driver loads — it gives the kernel a display surface during early boot. It is intended to release its minor number when a real GPU driver loads for the same hardware, allowing the real driver to take over `card0`. On this system, that did not happen.

```bash
$ dmesg | grep -i "takeover\|handoff"
# No output
```

There is no log entry indicating a handoff was attempted. Whether this is expected behavior for this hardware configuration or a bug is not something I can determine from logs alone — it has been reported for kernel developers to look at.

> Note: DRM minor numbers are not reassigned after `simpledrm` eventually unregisters. Once `amdgpu` is on minor 1, it stays on minor 1.

---

## Why GDM Crashes on `card0`

The error from the journal:

```
gnome-shell[1743]: Failed to ensure KMS FB ID on /dev/dri/card0:
                   drmModeAddFB failed: No such device
```

When Mutter initializes a Wayland KMS backend, it uses GBM (Generic Buffer Management) — a library that allocates GPU memory buffers for rendering. Mutter calls `gbm_surface_lock_front_buffer()` to acquire the framebuffer for rendering. `simpledrm` manages a firmware memory region and has no GBM hardware backend — it cannot service this call. The call returns `ENODEV`, Mutter treats it as fatal and exits.

systemd restarts GDM. On the second attempt, `card0` is no longer present. Mutter finds `card1` (`amdgpu`), which has a real hardware backend. GBM initializes correctly and Wayland starts.

---

## Why the Failure Was Initially Intermittent

Early on the failure did not happen on every boot. My best guess from the observed pattern: on some boots, GDM started scanning `/dev/dri/` after `simpledrm` had already released `card0`, so Mutter landed on `card1` directly without crashing. As boot timing changed due to other system updates, GDM consistently started while `simpledrm` still held `card0`.

> This is an inference from the pattern, not something I confirmed with precise timing measurement.

---

## Attempt 6: AMD DPM Parameter

**Hypothesis:** GPU power management was keeping the GPU in a low-power state during boot, causing initialization timing issues.

DPM (Dynamic Power Management) controls how the GPU scales its clock speed and power consumption. The idea was that maybe the GPU wasn't fully initialized when GDM tried to use it.

```bash
$ sudo nano /etc/default/grub
# Added amdgpu.dpm=1 to GRUB_CMDLINE_LINUX_DEFAULT
$ sudo update-grub && sudo reboot

$ cat /proc/cmdline
... amdgpu.dc=1 amdgpu.dpm=1
```

Still only `card1` in `/dev/dri/`. The minor number assignment happens at driver registration time — DPM settings do not affect it.

**Result:** Not the cause.

---

## The Workaround

The simplest fix I found: prevent `simpledrm` from loading at all. With no early-boot DRM driver registering, `amdgpu` is the first driver to claim a minor number and gets `card0`.

```bash
$ sudo nano /etc/default/grub
```

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash initcall_blacklist=simpledrm_platform_driver_init"
```

`initcall_blacklist` is a kernel boot parameter that prevents a named initialization function from running. `simpledrm_platform_driver_init` is the function that registers `simpledrm` with the DRM subsystem — blocking it means `simpledrm` never loads.

```bash
$ sudo update-grub && sudo reboot
```

```bash
$ ls -la /dev/dri/
crw-rw----+  1 root video  226, 0  card0
crw-rw----+  1 root render 226, 128  renderD128

$ dmesg | grep "Initialized.*drm"
[   16.512693] [drm] Initialized amdgpu 3.64.0 for 0000:04:00.0 on minor 0
```

`amdgpu` on minor 0 — `card0`. No crash. No restart cycle. Login screen appears once `amdgpu` finishes loading (~16 seconds).

**Trade-off:** No Plymouth boot splash (the graphical boot animation) for the first ~16 seconds. The screen is blank until `amdgpu` is ready. This removes the symptom but does not address the underlying behavior in the kernel.

> Before/after `ls /dev/dri/` and `udevadm` output: [`proof/bug-card1-proof.txt`](./proof/bug-card1-proof.txt)

---

## Stability

Five consecutive reboots over two days after applying the workaround. Zero failures.

---

## Complete Sequence

```
UEFI provides framebuffer memory region
    ↓
simpledrm registers (t=0.4s) → /dev/dri/card0 (minor 0)
    ↓
[~16 seconds pass]
    ↓
amdgpu loads for PCI 04:00.0
minor 0 already taken → amdgpu assigned minor 1 → /dev/dri/card1
    ↓
systemd starts GDM
Mutter enumerates /dev/dri → picks up card0
    ↓
Mutter → gbm_surface_lock_front_buffer(card0) → ENODEV
Mutter exits
    ↓
[Black screen — GDM restarting]
    ↓
GDM restarts → card0 now gone → Mutter finds card1 (amdgpu)
GBM initializes → Wayland starts → login screen appears
```

---

## Bug Report

**[Launchpad #2142087](https://bugs.launchpad.net/ubuntu/+bug/2142087)** — `simpledrm` does not release `/dev/dri/card0` before `amdgpu` registers on this hardware, causing GDM to crash on every boot.

**Hardware:** AMD Picasso/Raven 2 `[1002:15d8]`, kernel `6.17.0-14-generic`, Ubuntu 24.04

**Observed behavior:**
- `simpledrm` registers on minor 0 at `t=0.4s`
- `amdgpu` registers on minor 1 at `t=16.5s`
- No handoff messages in `dmesg`
- GDM crashes on `card0` every boot, recovers on `card1`

**Workaround:**
```bash
GRUB_CMDLINE_LINUX_DEFAULT="... initcall_blacklist=simpledrm_platform_driver_init"
```

**Status:** New — awaiting review.

---

## What I Took Away From This

The symptom — random black screen on boot — looked like it could be caused by a dozen different things. Most of the early attempts fixed real issues (stale PAM config, leftover Waybar service) but none of them were the actual cause.

> The actual cause only became visible when I stopped filtering logs and read everything from a failed boot.

Each step narrowed the problem down to the right layer:

- **System is running** → not a kernel panic or hard freeze
- **PAM cleaned up** → not an authentication issue
- **X11 works, Wayland doesn't** → problem is in the Wayland display initialization path
- **Full journal** → two process IDs, two different devices, one crash in between
- **`dmesg`** → two drivers registered for the same physical GPU, no handoff between them

The biggest lesson for me personally: filtering logs for `--priority=err` hid the most important information. The actual failure was logged at info level — two driver registrations with different minor numbers, no error, no warning. It only looked wrong when you read the full sequence.

> The workaround removes the symptom but not the underlying behavior. That is what the bug report is for — the rest is for someone with deeper kernel knowledge to investigate.

---

## System Status After Workaround

> Everything below confirms the workaround is stable and applied correctly.

```bash
$ uname -r
6.17.0-14-generic

$ ls -la /dev/dri/
crw-rw----+  1 root video  226, 0  card0    ← amdgpu, minor 0
crw-rw----+  1 root render 226, 128  renderD128

$ cat /proc/cmdline
... initcall_blacklist=simpledrm_platform_driver_init

$ systemctl status gdm
● gdm.service - GNOME Display Manager
     Active: active (running)

$ loginctl list-sessions
SESSION  UID  USER         SEAT   TTY   STATE   IDLE
3        1000 jillravaliya seat0  tty2  active  no
```

**Zero boot failures across five consecutive reboots over two days.**

---

## Connect

I am currently looking for internship opportunities in systems programming, Linux, or low-level development. If you found this write-up useful or want to discuss it, feel free to reach out.

- **Email:** jillahir9999@gmail.com
- **LinkedIn:** [linkedin.com/in/jill-ravaliya-684a98264](https://linkedin.com/in/jill-ravaliya-684a98264)
- **GitHub:** [github.com/jillravaliya](https://github.com/jillravaliya)

---

> *This investigation happened five days after Bug #2141741 — a systemd kernel panic on NVMe systems. Both pointed to the same habit: when something fails silently or inconsistently, the visible symptom is rarely where the actual problem is.*
