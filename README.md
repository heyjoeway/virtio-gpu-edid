# virtio-gpu-edid

High-resolution, high-refresh display modes for QEMU/Proxmox VMs that use
**virtio-gpu**. QEMU's own EDID offers a 1280x800 @ 75 Hz preferred mode, with
everything else at 50–60 Hz. This replaces it with an EDID offering every common
resolution, ultrawides included, at 60–360 Hz, loaded by the kernel at boot with
`drm.edid_firmware=`. Every compositor then sees the same mode list: GNOME,
KDE, Gamescope, and Steam's resolution and refresh pickers in Game Mode.

> [!NOTE]
> Written alongside
> [zeroqn/proxmox-native-context](https://github.com/zeroqn/proxmox-native-context)
> (Proxmox host setup for AMD virtio-gpu native context) and
> [heyjoeway/sunshine-dmabuf-fix](https://github.com/heyjoeway/sunshine-dmabuf-fix)
> (fixes Sunshine's Vulkan encoding in such guests), for streaming VM desktops
> at high refresh rates. This tool only depends on virtio-gpu, though.

Neither QEMU nor Proxmox can change the refresh rates in QEMU's EDID or supply
your own, so the replacement happens inside the guest.

Tested, with every mode listed in the compositor:

| Guest | Host GPU (virtio-gpu mode) | Desktop |
|---|---|---|
| Ubuntu 26.04 LTS | Renoir iGPU (AMD native context) | GNOME on Wayland: 159 of 161 modes offered |
| Bazzite (deck image, Fedora 44 base) | RX 6700 XT (AMD native context) | Gamescope and KDE Plasma |

Both are Proxmox VE 9 guests. The EDID only depends on the virtio-gpu kernel
driver, so plain virgl or Venus guests should behave the same, but they're
untested. Supported systems for `install`: Ubuntu 26.04 LTS and Bazzite.

## Install

```sh
git clone https://github.com/heyjoeway/virtio-gpu-edid.git
cd virtio-gpu-edid
sudo ./vm-edid install                # then reboot
./vm-edid status                      # after the reboot
```

The default EDID is bundled as [`vm-edid.bin`](vm-edid.bin), so the defaults
need no build step or Python. The generator is deterministic, so you can check
the bundled file against a fresh build (SHA-256 starts with `8b3830d8ea85`):

```sh
python3 make_edid.py -o /tmp/vm-edid.bin && cmp /tmp/vm-edid.bin vm-edid.bin && echo identical
```

For a custom EDID, run `./vm-edid build [options]` as your user first; it
writes `build/vm-edid.bin`, which `install` prefers over the bundled file.
`sudo ./vm-edid uninstall` removes the boot configuration again.

## Default modes

Every resolution at every refresh rate (156 modes), plus the preferred mode, as CVT reduced
blanking v2 timings:

| Aspect | Resolutions |
|---|---|
| 16:9 | 1920x1080, 2560x1440, 3840x2160, 5120x2880 |
| 16:10 | 1920x1200, 2560x1600 |
| 21:9 and similar | 2560x1080, 3440x1440, 3840x1600, 5120x2160 |
| 32:9 | 3840x1080, 5120x1440, 7680x2160 |

Refresh rates: 60, 75, 100, 120, 144, 165, 175, 180, 200, 240, 280 and 360 Hz.
The desktop starts at the preferred mode, 1024x768 @ 60. Pick your real mode
once in KDE's display settings or Steam's display settings; both remember it
per monitor, so later boots come up in it.

**Why 1024x768:** virtio-gpu deletes any mode flagged preferred unless it's
1024x768 or within 16 pixels of the scanout size QEMU reports. That size starts
as the device's `xres`/`yres` (1280x800 unless set) but can follow the console
client, for example noVNC with remote resizing. Without a preferred mode,
compositors and the boot console start at the biggest mode in the list: here
that was 7680x2160 @ 360 Hz, which left the boot screen panned and coincided
with boot taking minutes. 1024x768 is the one size that's always kept. For a
different preferred mode, pin the scanout size on the host and tell the
generator; the generator refuses a preferred mode that would be deleted:

```sh
# Proxmox host, /etc/pve/qemu-server/<vmid>.conf (append to any existing args:)
args: -set device.vga.xres=2560 -set device.vga.yres=1440
```
```sh
# VM
./vm-edid build --preferred 2560x1440@120 --host-size 2560x1440
```

Customize with `build` options (see `make_edid.py --help`):

```sh
./vm-edid build --resolutions 2560x1440,3440x1440 --rates 60,120,165,240
./vm-edid build --max-clock-mhz 2000   # drop the most extreme modes
python3 make_edid.py --list          # print the modes without writing a file
```

## How it works

- **EDID layout:**
  - The base EDID block carries only the preferred mode. Its detailed timing
    format tops out at a 655 MHz pixel clock.
  - Every other mode goes into DisplayID 1.3 extension blocks as Type I
    timings, five per 128-byte block. The default matrix is 33 blocks (4 KB).
  - DisplayID 2.0 Type VII timings would also work for the kernel, but
    libdisplay-info doesn't decode them, so they aren't used.
- **No inferred modes:** there's no range-limits descriptor and no
  "continuous frequency" flag. Without them the kernel adds no inferred
  in-between modes, so the mode list is exactly the matrix.
- **The runtime override doesn't work:** virtio-gpu reads its EDID only when
  the driver starts or when the host signals a display change. The debugfs
  `edid_override` entry, followed by a `detect`, is accepted but never used.
  So the EDID has to be loaded at boot.

## What `install` does

| | Ubuntu 26.04 | Bazzite |
|---|---|---|
| EDID file | `/usr/lib/firmware/./vm-edid.bin` | `/etc/firmware/./vm-edid-<hash>.bin` (`/usr` is read-only) |
| Kernel arguments | `/etc/default/grub.d/99-vm-edid.cfg`, then `update-grub` | `rpm-ostree kargs`: `drm.edid_firmware=…` and `firmware_class.path=/etc/firmware` |
| Initramfs | hook `/etc/initramfs-tools/hooks/vm-edid`, then `update-initramfs -u` | `rpm-ostree initramfs-etc --track` (virtio-gpu is built into the kernel and starts early) |

The connector is detected automatically: it's the one connected `Virtual-N`
output; override it with `--connector`.

On Bazzite the file is named after its content (`vm-edid-<hash>.bin`).
`rpm-ostree initramfs-etc` keeps reusing its initramfs overlay while the set of
tracked paths stays the same, and `--force-sync` did not pick up a changed file
in testing: the kernel kept loading the first EDID ever installed. A new name
per EDID forces a new overlay. Reinstalling untracks other EDIDs under
`/etc/firmware/edid/` and replaces the kernel argument. Old EDID files are
kept: every rpm-ostree transaction rebuilds the overlay from the tracked list,
and a file that some deployment still tracks must not disappear.

Bazzite applies the new deployment while shutting down: ostree's finalize
step, which systemd kills after 5 minutes. On a slow (here: rotational) disk,
finalizing took 6.5 minutes, so the update was silently dropped and the old
deployment booted again (`journalctl -b -1 -u ostree-finalize-staged` shows
"Stopping timed out"). Raising that one service's stop timeout fixed it:

```sh
sudo mkdir -p /etc/systemd/system/ostree-finalize-staged.service.d
printf '[Service]\nTimeoutStopSec=30min\n' |
  sudo tee /etc/systemd/system/ostree-finalize-staged.service.d/10-timeout.conf
sudo systemctl daemon-reload
```

Then let the shutdown run; don't force-stop the VM while it finalizes.

After rebooting, `./vm-edid status` shows the EDID the kernel actually
loaded (name, preferred mode, resolution count) and whether it matches the
installed file.

## Surviving updates

- **Ubuntu:**
  - The grub drop-in and the firmware file aren't owned by any package.
  - The initramfs hook copies the EDID into every rebuilt initramfs.
- **Bazzite:** kernel arguments and `initramfs-etc` files carry over to every
  new deployment.
- **Different hardware:** the argument names the virtio-gpu connector, so on
  other hardware (for example GPU passthrough) it matches nothing and does
  nothing.

## Limitations

- **The refresh rate is a target on virtio-gpu, not a guarantee.** There's no
  real scanout clock: a page flip completes when the host has processed the
  frame. The compositor paces itself to the mode's refresh rate, but the
  achieved frame rate depends on the host. With Proxmox's VNC console, QEMU
  also reads every frame back for the console. If a high mode doesn't reach
  its rate in Moonlight's statistics, check the QEMU process's CPU on the host.
- **The top modes are there for completeness.** 7680x2160 @ 360 Hz is a
  7.2 GHz pixel clock. Use `--max-clock-mhz` or a shorter list if the menus
  get too long.
- Changing modes still needs a reboot, because of the runtime limitation
  above.

## Files

```
make_edid.py   EDID generator (Python 3, no dependencies)
vm-edid        build/install/uninstall/status (bash)
vm-edid.bin    the bundled default EDID (make_edid.py's default output)
```

## License

MIT, see [LICENSE](LICENSE).
