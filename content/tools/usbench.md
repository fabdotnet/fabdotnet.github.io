+++
title = "USBench — USB Hardening Audit Bench (PROBE / TARGET)"
wip = true  # shows a red DRAFT badge next to the title (see extend_head.html)
date = 2026-10-18T00:00:00Z
draft = false
tags = ["tool", "usb", "hardening", "badusb", "pentest", "freebsd", "windows"]
summary = "A PROBE emulates USB device classes while a TARGET monitor records how the host reacts — which driver binds, which effect appears — across Linux, FreeBSD and Windows, with a campaign tool to prove a hardening change before/after."
+++

## Summary

**USBench** is a two-sided bench for auditing how a machine handles USB devices. A **PROBE** host presents itself *as* a USB device and emulates the standard classes (keyboard/HID, mass storage, network, serial, audio…). A **TARGET** monitor runs on the machine under test and records, in real time, what the enumeration actually triggered: which class was enumerated, whether a driver bound to it, and which observable effect appeared (a network interface came up, a volume auto-mounted, a HID keyboard was accepted, a serial port was created).

The point is not to run an exploit. It is to answer a measurable question for a given host and a given hardening policy: *of all the USB classes an attacker could present, which ones does this machine actually act on?* A **campaign** tool then compares a baseline run against a hardened run and tells you, class by class, what the control neutralised and what it left exposed.

| | |
|---|---|
| **Repository** | [github.com/fabdotnet/USBench](https://github.com/fabdotnet/USBench) |
| **Version** | 0.99.4 (pre-1.0) |
| **License** | Apache-2.0 |
| **PROBE** | Python 3, Linux gadget ConfigFS — needs a USB Device Controller (e.g. Raspberry Pi Zero W, dwc2) |
| **TARGET** | Linux, FreeBSD, Windows — Python *or* a dependency-free shell / PowerShell monitor |

## Motivation

USB is a standing attack surface on endpoints: BadUSB keystroke injection, rogue network adapters that reroute traffic, auto-mounting storage, and so on. Plenty of offensive hardware exists to *present* those devices. What is usually missing on the defensive side is a way to **measure the result**: once you apply a control — USBGuard, a blocked class, `authorized_default=0`, a Windows device-installation policy — did it actually stop the host from binding a driver, or did it only stop the device from *working* while the driver still loaded?

USBench was built to make that answer reproducible and comparable. One PROBE sequence, the same every time; a TARGET monitor that emits a structured log and a self-contained report; and a diff that turns "it feels more locked down now" into a per-class **neutralized / still active / regression** verdict with a non-zero exit code when a high-risk effect remains — usable in CI.

## How it works

```
   PROBE  ──────────── USB cable ────────────▶  TARGET
   (emulates device classes)                    (device under test)
   USBench-probe.py                           USBench-target.py / -linux.sh
   Raspberry Pi with a UDC (dwc2)               / -bsd.sh / .ps1
```

- The **PROBE** (`USBench-probe.py`) uses the Linux USB gadget subsystem (ConfigFS) to enumerate as one class at a time, cycle through all of them, or present a **composite** device with several functions at once — the shape real multi-function tools take. A host-only x86 PC cannot do this: a USB *Device* Controller is required, which is why a Raspberry Pi Zero W (or similar dwc2/dwc3 board) is used.
- The **TARGET monitor** snapshots the system's view of USB, network, storage, input, serial, audio, printer and video, then diffs those snapshots every poll and timestamps each change.

The decisive signal is the **USB** line: a device shows up on the target *even when no driver binds* (`drv=-`). "Enumerated but ignored" versus "a driver bound" (`drv=usb-storage`, `drv=cdc_ether`, …) is the core of the analysis — a control that blocks the driver is doing its job; one that only hides the effect is not.

## Cross-platform TARGET monitors

The target side is deliberately portable, because the machine being audited is rarely the one you would choose. Every monitor emits the **same structured output and the same verdict**, so a single campaign can mix operating systems against one PROBE.

| Platform | Monitor | Notes |
|---|---|---|
| **Linux** | `USBench-target.py`, `USBench-target-linux.sh` | sysfs/procfs. The `.sh` has **zero dependencies** (busybox-friendly) for appliances and embedded targets without Python. |
| **FreeBSD** | `USBench-target.py`, `USBench-target-bsd.sh` | The `.sh` uses only the **base system** (`usbconfig`, `sysctl`, `ifconfig`, `geom`) — nothing to install on the audited host. |
| **Windows** | `USBench-target.ps1` | PowerShell 5.1, CIM/WMI, no extra modules. Server 2016+ / 10 / 11. |

When Python is available the `.py` monitor covers both Linux and FreeBSD (it auto-detects the OS). When it is not — a locked-down appliance, a FreeBSD box with no `pkg` access — the shell monitor for that OS gives the same result without touching the system.

## Validation

This release was validated under QEMU/KVM, each OS driven through the same hot-plug sequence (HID keyboard, mouse, mass storage, USB network, USB audio, USB serial), in a clean run and in a hardened run:

| OS | Result |
|---|---|
| Debian 12, Debian 13 | all classes enumerated, correct Linux driver bound each time |
| Fedora 44 | Cloud images ship without `kernel-modules`: only HID and storage bind until the modules package is installed — a useful reminder that a minimal image under-reports the real attack surface |
| FreeBSD 15.1 | validated with both the Python monitor and the base-system `.sh` |
| Windows Server 2022 | the five Windows USB drivers bound; a device blocked by policy is now reported as *enumerated without driver* rather than vanishing from the report |

Hardening was then applied on each OS (Linux `authorized_default=0`, FreeBSD `hw.usb.disable_enumeration=1`, a Windows device-installation policy) and the campaign diff confirmed the neutralisation with a clean exit code. Physical-hardware validation (a real Raspberry Pi PROBE against real targets) is the next step; the figures above are from virtual UDCs and QEMU USB devices.

## The campaign: proving a control

```bash
# Baseline: capture the target's reaction to the PROBE sequence
sudo ./USBench-campaign.py run --tag baseline --duration 90 --probe-cmd "ssh probe@usb-probe ..."

# Apply a control (USBGuard, blocked class, policy...), then:
sudo ./USBench-campaign.py run --tag usbguard --duration 90 --probe-cmd "..."

# Compare
./USBench-campaign.py diff --before baseline --after usbguard
```

`diff` labels every effect **✓ neutralized**, **• still active** or **✗ regression**, and exits with code **2** if any high-risk effect remains (a volume auto-mounted, a HID keyboard accepted, a network interface up, or a driver bound). That exit code makes it a regression gate: a hardening baseline that silently loosens between two OS releases shows up as a failed check.

## Evidence, by design

For a write-up you need proof, not just a verdict. Each PROBE function leaves an identifiable, timestamped marker keyed by a campaign `--tag` (traceable device descriptors, a benign HID banner, a network/serial banner, a labelled storage volume with a witness file), and the TARGET report cross-references them: each USB class marked **POSITIVE** or **NEGATIVE**, followed by the concrete evidence for every effect plus raw log and `dmesg` extracts. Same `--tag` on both sides pairs the two reports. The intrusive behaviours (HID typing in particular) are opt-in and off by default.

## Design decisions

- **The TARGET is non-invasive.** The shell monitors install nothing on the audited machine; the FreeBSD one stays entirely within the base system.
- **One schema everywhere.** Linux, FreeBSD and Windows emit the same event format and verdict, so cross-OS campaigns compare 1:1 (driver *names* differ by OS and are reported on both sides, but classes and effects line up).
- **Driverless devices stay visible.** A device blocked by policy is still reported as *enumerated, no driver bound* — otherwise an effective control would look like "nothing happened", which is exactly the case you most want to see documented.
- **CI-friendly.** The diff's exit code is the contract; everything else is human-readable.

## Status and limitations

- **Pre-1.0** (0.99.x): the output format is stabilising but may still change before 1.0.
- **PROBE needs a UDC.** ConfigFS covers classes with a native gadget function. Arbitrary/malformed descriptors would need raw-gadget or Facedancer (out of scope here).
- **Validated on virtual hardware.** Physical-Pi validation is pending.
- **macOS**: no monitor yet.

## Getting started

```bash
git clone https://github.com/fabdotnet/USBench.git
cd USBench
# TARGET (the machine under test) — Linux example, no Python needed:
sudo sh target/USBench-target-linux.sh -i 0.3 -l target.log -T baseline -m
# PROBE (Raspberry Pi with a UDC):
sudo ./USBench-probe.py --all --duration 8 --watch-dmesg
```

The README documents the PROBE wiring, the per-OS monitors, the evidence options and the full campaign workflow.

## References

- [USBench repository](https://github.com/fabdotnet/USBench) — source, README, [LICENSE](https://github.com/fabdotnet/USBench/blob/main/LICENSE) (Apache-2.0)
- [Linux USB Gadget ConfigFS](https://www.kernel.org/doc/html/latest/usb/gadget_configfs.html) — how the PROBE presents classes
- [USBGuard](https://usbguard.github.io/) — a Linux USB authorization control to test against
- [FreeBSD `usbconfig`(8)](https://man.freebsd.org/cgi/man.cgi?usbconfig) — the base-system tool the BSD monitor builds on

## Disclaimer

USBench is intended for **authorized** security testing — hardening and audit — on systems you own or are explicitly permitted to test. The PROBE emulates attacker-style USB devices; do not point it at hardware you are not authorized to assess.

## Credits

Developed by Fabien MAISONNETTE (FabDotNET).
