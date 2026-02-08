# Low-latency Linux gaming

> [!WARNING]
> This guide is a fork of original guide by [syzzi (theyareonit)](https://github.com/theyareonit)

This is not a step-by-step instruction manual but rather a collection of resources and things to consider when tuning Linux systems for low-latency, low-jitter gaming.

This guide is targeted toward more experienced users, due to the fact that many settings listed here cannot be blindly applied, and require some understanding of what is going on. That said, I don't claim to be an authority on this subject either, and some information here may be incorrect or misleading.

> [!WARNING]
> Before reading this guide, please go through [PC-Tuning](https://github.com/valleyofdoom/PC-Tuning) to learn how to properly configure your hardware and UEFI/BIOS settings (you can ignore the Windows-specific steps).

# Sections

* [1. Distributions](#1-distributions)
    * [CachyOS](#cachyos)
    * [Gentoo](#gentoo)
* [2. Kernel Parameters](#2-kernel-parameters)
    * [General](#general)
    * [Intel CPUs](#intel-cpus)
    * [AMD CPUs](#amd-cpus)
* [3. Schedulers](#3-schedulers)
* [4. Display servers, compositors, & window managers](#4-display-servers-compositors--window-managers)
    * [X11](#x11)
    * [Wayland](#wayland)
    * [Tearing and latency on Wayland](#tearing-and-latency-on-wayland)
    * [Compositors and window managers](#compositors-and-window-managers)
    * [Practical advice](#practical-advice)
* [5. libinput](#5-libinput)
    * [Mouse sensitivity](#mouse-sensitivity)
    * [Debouncing](#debouncing)
	* [evdev](#evdev)
* [6. NVIDIA GPUs](#6-nvidia-gpus)
* [7. AMD GPUs](#7-amd-gpus)
    * [Environment Variables](#environment-variables)
* [8. Intel GPUs](#8-intel-gpus)
* [9. Audio](#9-audio)
* [10. File systems & storage](#10-file-systems--storage)
* [11. Networking](#11-networking)
* [12. Wine](#12-wine)
* [13. Miscellaneous](#13-miscellaneous)
* [14. Further Reading](#14-further-reading)

# 1. Distributions

## CachyOS

[Website](https://cachyos.org/)

Arch-based distribution that comes with its own optimized repositories and offers custom kernels with various patches (& optimizations such as [LTO](https://www.phoronix.com/review/clang-lto-kernel)). Highly recommended.

I'm not going to go too deep into kernel-related tuning (besides kernel parameters) in this guide because the CachyOS kernels are essentially as good as it gets.

[Information about the CachyOS kernels](https://wiki.cachyos.org/features/kernel/)

[Information about the CachyOS repositories](https://wiki.cachyos.org/features/optimized_repos/)

## Gentoo

[Website](https://www.gentoo.org/)

Source-based distribution which may offer minor performance gains over CachyOS (in exchange for time spent configuring & compiling). Gentoo also now provides pre-built binaries for common packages (including some built specifically for the x86-64-v3 ISA, similar to CachyOS), but it's still more work than CachyOS for relatively little benefit.

# 2. Kernel parameters

Don't use kernel parameters if you don't know what effects they have! This applies doubly so for random kernel parameters you find online with no explanation of what they do.

Some of the parameters listed may compromise the security or stability of your machine.

## General

If you want to disable power saving related settings, I would advise disabling them in both UEFI/BIOS settings as well as kernel parameters. I say this due to stuttering that I experienced in the past, which was caused by having processor C-States disabled in UEFI but not in kernel parameters. I assume that something tried to "override" my UEFI settings.

And similarly to the issue I personally experienced, the impression I get is that you should be redudant with your settings, just to be on the safe side. Even [Intel](https://www.intel.com/content/www/us/en/developer/articles/technical/optimizing-computer-applications-for-latency-part-1-configuring-the-hardware.html) recommends using all 3 of the following parameters: `intel_idle.max_cstate=0`, `processor.max_cstate=0`, and `idle=poll`, despite the fact that `idle=poll` should in theory make the other two redundant. This may just be paranoia though.

| Parameter | Explanation |
| ---       | ---         |
| `mitigations=off` | Disable CPU security mitigations, which often come with a performance penalty. **This will make your system significantly less secure, apply at your own risk.** |
| `nowatchdog` | Disable watchdog timers to [reduce interrupts](https://wiki.archlinux.org/title/Power_management#Disabling_NMI_watchdog). |
| `nosoftlockup` | Don't log backtraces for processes that execute for longer than 120 seconds without yielding. Probably doesn't matter for any games. |
| `audit=0` | Disable the [audit framework](https://wiki.archlinux.org/title/Audit_framework) to marginally reduce overhead (mentioned in section 8 of RHEL's [latency tuning guide](https://access.redhat.com/sites/default/files/attachments/201501-perf-brief-low-latency-tuning-rhel7-v2.1.pdf)). |
| `usbcore.autosuspend=60` | Set USB autosuspend timer to 60 seconds (default: `2`). Alternatively, replace `60` with any other value. A negative value will disable autosuspend entirely. |
| `workqueue.power_efficient=false` | Disable [power-efficient workqueues](https://lwn.net/Articles/731052/) if enabled in your kernel configuration, since it may cause cache misses due to workqueues being scheduled onto different cores. |
| `skew_tick=1` | Skew timer ticks on different cores to [reduce lock contention](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux_for_real_time/7/html/tuning_guide/reduce_cpu_performance_spikes). |
| `threadirqs` | [Thread interrupt handlers](https://wiki.linuxfoundation.org/realtime/documentation/technical_details/threadirq) by default. |
| `tsc=reliable` | Disable clocksource verification for TSC, which may slightly reduce overhead. Not recommended due to the potential for [clock desync](https://lwn.net/Articles/388188/). |
| `preempt=full` | Allow preemption of [almost all](https://wiki.linuxfoundation.org/realtime/documentation/technical_basics/preemption_models) kernel code. |
| `nohz_full=all` | Use tickless mode on all cores wherever possible (specifically, when there is at most one runnable task on a core). Alternatively, you can replace `all` with a range or comma-separated list of cores to make tickless, e.g. `2-5,7,8`. This is [not necessarily beneficial](https://www.kernel.org/doc/Documentation/timers/NO_HZ.txt). In addition, [`isolcpus`](https://access.redhat.com/solutions/480473) can be used to force a specific task to always run tickless. Using these parameters without explicit CPU and IRQ pinning can lead to worse latency and frametimes. Intended for carefully tuned real-time systems, not general desktop gaming. |
| `cpufreq.default_governor=performance` | Set the default [frequency scaling governor](https://wiki.archlinux.org/title/CPU_frequency_scaling#Scaling_governors) to `performance`. |
| `pcie_aspm=off` | Force [ASPM](https://en.wikipedia.org/wiki/Active_State_Power_Management) off for all PCIE devices. Make sure to disable ASPM in UEFI/BIOS settings as well. |
| `processor.max_cstate=0` | Don't allow processor to sleep deeper than the C1 state. This will prevent your CPU from reaching its maximum single-core boost clock (if running stock frequencies), but reduces jitter substantially. This will increase idle power usage. Make sure to disable C-States in UEFI/BIOS settings as well. |
| `idle=poll` | Force CPU into the C0 state (significantly increasing idle power usage). Don’t use if you have hyperthreading/SMT enabled. According to the [Linux kernel documentation](https://www.kernel.org/doc/Documentation/x86/x86_64/boot-options.txt), this parameter provides no performance advantage on CPUs with MWAIT support, which includes all modern x86 CPUs. Despite this, older Intel and RHEL latency tuning guides still recommend it, which makes its real-world usefulness questionable. On modern CPUs, it usually provides no measurable benefit and often increases power consumption and heat while reducing boost headroom. Not recommended unless you are experimenting with very specific real-time workloads. |

## Intel CPUs

Use [`i7z`](https://code.google.com/archive/p/i7z/) to verify that your processor is running at the correct frequency & C-States.

| Parameter | Explanation |
| ---       | ---         |
| `intel_pstate=disable` | [Disable](https://www.kernel.org/doc/Documentation/cpu-freq/intel-pstate.txt) the `intel_pstate` driver. |
| `intel_idle.max_cstate=0` | [Disable](https://docs.kernel.org/admin-guide/pm/cpuidle.html) the `intel_idle` driver and fall back on `acpi_idle`. |
| `modprobe.blacklist=iTCO_wdt` | `nowatchdog` may [fail](https://wiki.archlinux.org/title/Improving_performance#Watchdogs) to disable the Intel TCO watchdog, this is a fix (not necessary on CachyOS). |

## AMD CPUs

Modern AMD CPUs (Zen 2 and newer) generally behave well out of the box, but some tuning can reduce latency variance under sustained gaming load.

AMD systems usually do not benefit from as aggressive power-state disabling as Intel systems. Over-disabling power management may actually *increase* frame-time inconsistency, especially on Zen 3+.

Useful kernel parameters:

| Parameter | Explanation |
| --- | --- |
| `amd_pstate=active` | Forces the modern amd_pstate driver into active mode, allowing the kernel to directly request frequencies instead of relying on ACPI hints. Recommended on Zen 3+ (Ryzen 5000 and newer). |
| `rcu_nocbs=all` | Offloads RCU callbacks from all CPUs, which can reduce jitter on busy cores when combined with `nohz_full`. |
| `pci=noaer` | Disables PCIe Advanced Error Reporting interrupts, which can cause rare but noticeable stutters on some AMD platforms. |

Notes:
- SMT should generally remain enabled unless threads are manually pinned.
- Disabling boost clocks is usually counterproductive on Zen 4+ CPUs.

# 3. Schedulers

* [BORE](https://github.com/firelzrd/bore-scheduler)

* [sched-ext](https://wiki.cachyos.org/configuration/sched-ext/)

If you’re unsure, stick to your distro’s default kernel and scheduler first.  
CachyOS ships multiple kernel variants — including BORE-based ones such as `linux-cachyos-bore` — as well as support for `sched-ext` (SCX) schedulers that can be loaded at runtime via `scx_loader` / `scxctl` and `scx-scheds`.

There is no universally “best” scheduler for gaming. Results depend heavily on the game, CPU, background load (recording, browsers, Discord), and even frame pacing sensitivity. If you experiment, change one thing at a time and benchmark properly.

# 4. Display servers, compositors, & window managers

On Linux, gaming mostly comes down to **X11 vs Wayland**, and which compositor or window manager sits on top of it. There is no single “correct” choice — only trade-offs.

## X11

X11 is the traditional display server and is still widely used for gaming.

**Pros:**
- Predictable behavior
- Mature driver support
- Easy access to tearing and low-latency paths
- Fewer compositor-specific quirks

**Cons:**
- Aging architecture
- Worse isolation and security
- Generally worse handling of modern features like HDR and per-surface synchronization

If something works on X11 but not on Wayland, it’s completely reasonable to keep using X11.

---

## Wayland

Wayland is the modern replacement for X11 and is now usable for gaming, but behavior depends heavily on the compositor and GPU driver.

**Pros:**
- Better frame pacing and synchronization in many cases
- Cleaner input handling
- Improved VRR and multi-monitor behavior on supported setups
- Actively developed

**Cons:**
- Tearing control is compositor-dependent
- Some features are still inconsistent across compositors
- Driver quality (especially proprietary ones) matters a lot

Wayland is no longer “experimental”, but it is also not universally better for every game or setup.

---

## Tearing and latency on Wayland

Wayland is designed around synchronized presentation (no tearing by default). A standardized tearing protocol exists, allowing applications to *request* tearing for lower latency, but the compositor ultimately decides whether to honor it.

In practice:
- Tearing is usually only allowed for fullscreen games
- Some compositors expose explicit tearing or latency options
- Behavior varies significantly between compositors

If tearing is critical for you and Wayland does not behave as expected, X11 remains a valid fallback.

---

### Compositors and window managers

The compositor often matters more than the display server itself.

- **Minimal or lightweight compositors / WMs**  
  (e.g. tiling Wayland compositors or bare X11 window managers)  
  → Less overhead, fewer surprises, usually preferred for competitive or latency-sensitive gaming.

- **Feature-rich desktop environments**  
  (e.g. full compositing, effects, animations)  
  → More convenient, but can introduce latency, frame pacing issues, or compositor-specific bugs.

Disabling unnecessary effects, animations, and background services often matters more than switching display servers.

---

### Practical advice

- If your games run well on your current setup, don’t change it just because something is “newer”.
- Test Wayland and X11 yourself — behavior can differ per GPU, driver version, and compositor.
- Avoid stacking tweaks blindly; change one thing at a time and measure results.
- When in doubt, stability beats theoretical performance gains.

Linux gaming is about choosing the least problematic option *for your hardware and games*, not following a single “best” setup.

# 5. libinput

## Sensitivity and acceleration

To convert mouse sensitivity from a percentage (e.g. `25%`) into a libinput Accel Speed value (e.g. `-0.75`), simply subtract `1` from the percentage (e.g. `0.25 - 1 = -0.75`). Make sure you set your Accel Profile to `flat` as well (to disable acceleration).

However, if you want to separate vertical and horizontal sensitivity (Xorg only), what you have to do is create a file in `/etc/X11/xorg.conf.d/` called something like `50-mouse-sensitivity.conf`, with the following content:

```
Section "InputClass"
	Identifier "All Mice"
	MatchIsPointer "on"
	Driver "libinput"
	Option "AccelProfile" "flat"
	Option "TransformationMatrix" "1 0 0 0 1 0 0 0 1"
EndSection
```

`TransformationMatrix` is a matrix of values that can be used to warp mouse input in [various ways](https://wiki.ubuntu.com/X/InputCoordinateTransformation). But all you need to know for this section is that your mouse's horizontal speed can be controlled with the first value of this matrix, and your mouse's vertical speed can be controlled with the fifth value of this matrix. 

For instance, you can set your mouse to 25% horizontal sensitivity and 50% vertical sensitivity with `Option "TransformationMatrix" "0.25 0 0 0 0.5 0 0 0 1"`.

Instead of `MatchIsPointer`, you can do something like `MatchVendor "Razer"` or `MatchProduct "Viper 8K"` if you don't want the configuration to apply to all mice: in this example, it will search for the substring `"Viper 8K"` within the product's name, so you don't need to know the full, exact name of the mouse that your system sees.

## Debouncing

libinput adds [eager debouncing](https://wayland.freedesktop.org/libinput/doc/latest/button-debouncing.html) to all mice by default. This does not add latency to clicks or releases in most cases, but it prevents you from making very short or very fast clicks, and is completely unnecessary on mice that use optical switches or latch debouncing, as well as mice that already offer configurable debounce time in their software.

To disable it, create the file `/etc/libinput/local-overrides.quirks` (if it does not exist already), and add the following lines to it:

```
[Mouse Debouncing]
MatchUdevType=mouse
ModelBouncingKeys=1
```

## evdev

You can also use `evdev` as a driver instead of libinput on Xorg, which may potentially provide lower latency (haven't tested). For this, you'll need to install a package like `xf86-input-evdev` or similar (*not* `libevdev`).

Your configuration file will look something like this:

```
Section "InputClass"
	Identifier "Mouse"
	MatchIsPointer "on"
	Driver "evdev"
	Option "AccelerationScheme" "none"
	Option "AccelerationProfile" "-1"
	Option "VelocityReset" "30000"
EndSection
```

This is just a basic configuration file to set all pointers to use evdev, to disable mouse acceleration, and to set `VelocityReset` to 30 seconds (meaning that the mouse's subpixel position will only be discarded after 30 seconds of inactivity, rather than the default of 300ms). 

Like with libinput, you can use the `TransformationMatrix` option to set a sensitivity value. However, you can also use the `ConstantDeceleration` option instead, as described [here](https://www.x.org/wiki/Development/Documentation/PointerAcceleration/). This requires deleting the `AccelerationScheme` line.

You may additionally want to force all keyboards to use evdev with `MatchIsKeyboard`.

# 6. NVIDIA GPUs

NVIDIA GPUs generally work well on Linux, but behavior depends heavily on driver version, display server, and compositor.

- On Wayland, acceptable results usually require explicit sync support. In practice, this means using a recent NVIDIA driver (555+), a recent Xwayland version (24.1+), and up-to-date wayland-protocols. Without explicit sync, issues such as flickering, stuttering, or unstable frametimes are common.

- If you encounter rendering or latency issues on Wayland, switching to X11 or forcing Xwayland is a valid and often simpler solution. There is no inherent performance penalty for doing so.

- On X11, NVIDIA tends to be more predictable and stable, especially on older GPUs or older driver versions. For users who prioritize stability over experimentation, X11 remains a safe choice.

# 7. AMD GPUs

### Environment variables

> **Note:** Most Mesa environment variables are intended for debugging or experimentation.  
> On modern Mesa/RADV setups, performance is usually best without forcing anything. Some variables may help specific games, while others can make things worse.

- `RADV_PERFTEST=nggc,sam` — enables experimental RADV paths; can help in some cases but is often unnecessary on newer GPUs and Mesa versions
- `MESA_VK_WSI_PRESENT_MODE=immediate` — requests immediate presentation; may reduce latency, but can be ignored by DXVK or vkd3d-proton

Keep in mind that DXVK and vkd3d-proton may override presentation behavior, so this variable is not always respected.

## 8. Intel GPUs

Intel GPU gaming performance has improved significantly on Xe-LP and Xe-HPG (Arc), but latency tuning options are more limited compared to AMD or NVIDIA.

Recommended kernel parameters if latency variance is observed:

| Parameter | Explanation |
| --- | --- |
| `i915.enable_psr=0` | Disables Panel Self Refresh, which can cause frame pacing issues on some systems. |
| `i915.enable_dc=0` | Disables deep display power states, reducing wake-up latency. |
| `i915.enable_fbc=0` | Disables Frame Buffer Compression, which may introduce latency variance. |

Notes:
- Prefer Vulkan over OpenGL when possible
- Static clocks can help on Intel Arc GPUs
- Avoid outdated i915 tuning guides written before 2022

## 9. Audio

Low-latency audio mainly matters for rhythm games and competitive FPS titles.

PipeWire is recommended over PulseAudio or JACK directly.

General PipeWire recommendations:
- Sample rate: 48000 Hz
- Quantum: 32–64 frames
- Disable suspend-on-idle for active audio devices

Low-latency audio does not require a real-time kernel.  
Ensure `threadirqs` is enabled and avoid USB autosuspend on audio devices if popping or dropouts occur.

## 10. File systems & storage

Storage performance mainly affects load times, but poor configuration can introduce stutter.

File systems:
- ext4 is the safest low-latency default
- XFS performs well for large sequential reads
- btrfs is not recommended unless carefully configured

Recommended ext4 mount options:
- `noatime` — do **NOT** disable write barriers (`barrier=0` / `nobarrier`)

NVMe notes:
- Ensure adequate cooling to prevent thermal throttling
- Disable ASPM in firmware if supported

## Networking

Network sysctl tweaks rarely shorten ping to game servers.

They can help with:
- bufferbloat
- heavy background traffic

But routing, physical distance to the server, and ISP peering matter far more than local sysctl tweaks.

Most networking sysctl tweaks do not reduce latency to game servers. They are mainly useful for handling high throughput or heavy background traffic.

# 12. Wine

As of January 2026, Wine 11.0 is available (released on January 13, 2026). The native Wayland driver has matured a lot, but real-world performance still depends on the game, toolkit, GPU driver, and compositor.

If native Wayland causes issues on your setup, Xwayland remains a perfectly valid fallback.

# 13. Miscellaneous

* [`gpu-screen-recorder`](https://git.dec05eba.com/gpu-screen-recorder/about/) is by far the best recording/clipping/streaming tool for Linux if you don't need too many features. If you need the functionality offered by OBS instead, consider using [`obs-vkcapture`](https://github.com/nowrep/obs-vkcapture), and potentially [`obs-vaapi`](https://github.com/fzwoch/obs-vaapi) if on an Intel or AMD GPU.

* [Don't use `irqbalance`](http://www.alexonlinux.com/why-interrupt-affinity-with-multiple-cores-is-not-such-a-good-thing). Manually setting IRQ affinities is also typically not necessary on Linux either; the kernel does a good job on its own.

* Use [`evhz`](https://git.sr.ht/~iank/evhz) or [`MouseTester`](https://github.com/valleyofdoom/MouseTester) (through Wine) to ensure that your mouse is polling at the correct rate. For `MouseTester`, you may have to hover your cursor over the data plot window to get accurate results.

* Choose a good cursor theme, I like [Windows Classic Dark](https://www.pling.com/p/2060588/).

# 14. Further reading

[CachyOS Discord](https://discord.gg/cachyos-862292009423470592)

[Linux kernel parameters](https://docs.kernel.org/admin-guide/kernel-parameters.html)

[How do I build a real-time audio workstation on Linux?](https://wiki.linuxaudio.org/wiki/system_configuration)

[Low Latency Performance Tuning for Red Hat Enterprise Linux 7](https://access.redhat.com/sites/default/files/attachments/201501-perf-brief-low-latency-tuning-rhel7-v2.1.pdf)
