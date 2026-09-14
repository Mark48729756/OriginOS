# OriginOS

A real operating system built from scratch — no Linux kernel underneath, no borrowed bootloader, no libc. A Multiboot-compliant bootable image, a from-scratch C kernel, real interrupt-driven hardware drivers, a working network stack, a graphical desktop with a window manager, a RAM filesystem, a terminal, and a self-hosted disk installer that can put OriginOS onto real hardware.

It looks and feels like a calm, minimal macOS-style desktop — a menu bar up top, a clickable dock, draggable windows with traffic-light buttons — but every pixel of it is pushed to the screen by code in this repository.

---

<details>
<summary><strong>Why it used to flicker, and how that got fixed</strong></summary>
<br>

Early builds wrote pixels straight into the hardware framebuffer and redrew the whole screen, line by line, on every tick of a simple keyboard-polling loop — which is exactly what it looks like: visible tearing and flicker on every frame.

The current architecture fixes that properly, not with a hack:

- Every `gfx_*` drawing function writes into an **off-screen buffer in RAM**, never touching real video memory directly.
- `gfx_present()` is the **only** place in the entire kernel that writes to the real framebuffer, and it does it in one fast contiguous pass per frame.
- Frames are built and presented on a **timer interrupt** (the PIT, running at 1000 Hz), driven by real hardware interrupts — not a busy-wait polling loop sitting in place burning cycles.

The result is a clean, tear-free desktop that behaves like a real compositor, not a demo.

</details>

<details>
<summary><strong>Boot & core architecture</strong></summary>
<br>

- **Multiboot-compliant boot path** (`boot/boot.s`) — entry point, Multiboot header, framebuffer mode request handed off from GRUB.
- **GDT (Global Descriptor Table)** (`kernel/gdt.c`, `boot/gdt_flush.s`) — proper flat memory segmentation set up by hand.
- **IDT (Interrupt Descriptor Table)** (`kernel/idt.c`, `boot/idt_flush.s`, `boot/isr.s`) — all 32 CPU exception vectors and all 16 hardware IRQ vectors wired to real assembly stubs, not stubbed out.
- **PIC remapping + PIT timer** (`kernel/pic.c`) — the 8259 Programmable Interrupt Controller is remapped off its BIOS defaults (which collide with CPU exceptions) to a safe range, and the Programmable Interval Timer drives the 1000 Hz frame/tick clock.
- **MTRR support** (`kernel/mtrr.c`) — Memory Type Range Registers, used to mark the framebuffer as write-combining for faster graphics throughput on real hardware.
- **No libc** — string/memory utilities are hand-rolled in `kernel/kstring.c`; `kernel/libgcc_stubs.c` supplies the handful of compiler intrinsics GCC expects even in a freestanding build.
- **RTC support** (`kernel/rtc.c`) — reads the real-time clock chip for the system clock shown in the menu bar.

</details>

<details>
<summary><strong>Input</strong></summary>
<br>

- **PS/2 keyboard, IRQ1-driven** (`kernel/keyboard.c`) — full scancode-to-ASCII layout, Shift, Backspace, and arrow keys, dispatched through real interrupts rather than polled.
- **PS/2 mouse, IRQ12-driven** (`kernel/mouse.c`) — cursor tracking, left/right button clicks, driving window drag and dock interaction.

</details>

<details>
<summary><strong>Graphics</strong></summary>
<br>

- **Framebuffer graphics library** (`kernel/gfx.c`) — primitives for lines, filled circles, rectangles, and text, all rendering into the off-screen back buffer described above.
- **Built-in bitmap font** (`include/font.h`) — a hand-authored 5×7 pixel font baked directly into the kernel binary, no font files required to boot.
- **BMP decoder** (`kernel/bmp.c`) — reads uncompressed 24bpp BMP files (wallpapers, icons, boot screen) out of the initrd.
- **Procedural wallpaper fallback** — if a wallpaper asset is missing, the desktop still renders a gradient-and-hills background generated entirely in code, so a missing asset never breaks the boot.
- **Animated boot screen** (`kernel/boot_screen.c`) — a dedicated full-screen splash shown during startup, before the desktop initializes.

</details>

<details>
<summary><strong>Windowing & desktop</strong></summary>
<br>

- **Window manager** (`kernel/wm.c`) — multiple simultaneous windows, drag-by-titlebar with the mouse, click-to-close on the red traffic-light button, z-ordering and focus tracking so the right window receives keyboard input.
- **Desktop shell** (`kernel/kernel.c`) — wallpaper, top menu bar (with live clock), bottom dock, and the main event loop tying input, windowing, and rendering together.
- **Stack-opening styles** — folders/stacks can open as a grid, a fan, or a list, and the OS remembers the last style you picked.

</details>

<details>
<summary><strong>Filesystem</strong></summary>
<br>

- **RAM-backed filesystem** (`kernel/fs.c`) — real hierarchical folders and files, held in memory. Behaves like a genuine filesystem to every app and terminal command that touches it.
- **initrd loader** (`kernel/initrd.c`) — reads a real `initrd.img` file, built at compile time and handed to the kernel as a GRUB module at boot. Wallpapers, icons, the boot logo, and the installer's own copies of the boot files all live here.

</details>

<details>
<summary><strong>Terminal</strong></summary>
<br>

A real, working shell (`kernel/terminal.c`) sitting directly on top of the RAM filesystem:

| Command | Behavior |
|---|---|
| `ls` | List the current directory |
| `cd` | Change directory |
| `cat` | Print a file's contents |
| `mkdir` | Create a directory |
| `touch` | Create an empty file |
| `echo text > file` | Write text to a file |
| `pwd` | Print working directory |
| `clear` | Clear the terminal screen |
| `whoami` | Print the current user |
| `help` | List available commands |

</details>

<details>
<summary><strong>Storage drivers</strong></summary>
<br>

- **ATA/IDE driver** (`kernel/ata.c`) — talks directly to legacy IDE controllers over I/O ports `0x1F0`/`0x170`, polling PIO, supporting up to 4 legacy channel/drive combinations (primary/secondary × master/slave). This is the same low-level access every BIOS-era OS bootstraps disk I/O through.
- **AHCI/SATA driver** (`kernel/ahci.c`) — a from-spec implementation of the AHCI 1.3 register interface (no ACPI dependency, no vendor-specific quirks beyond what VirtualBox, QEMU, and real ICH-family controllers all agree on), needed because modern and virtualized SATA controllers don't expose legacy IDE emulation.
- **Unified disk API** (`include/ata.h`) — every caller uses one consistent interface; which physical controller a drive actually lives behind is invisible above this layer.

</details>

<details>
<summary><strong>Networking</strong></summary>
<br>

A genuine layered network stack, built up from the wire:

- **PCI enumeration** (`kernel/pci.c`) — scans the PCI bus to find network hardware.
- **Ethernet frame handling** (`kernel/ethernet.c`)
- **ARP** (`kernel/arp.c`) — address resolution
- **IP** (`kernel/ip.c`) — IPv4 packet handling
- **ICMP** (`kernel/icmp.c`) — ping support
- **UDP** (`kernel/udp.c`)
- **DHCP client** (`kernel/dhcp.c`) — full DISCOVER/OFFER/REQUEST/ACK negotiation for automatic IP configuration
- **Network configuration state** (`kernel/netconfig.c`) — tracks the current IP, netmask, and gateway

Three real NIC drivers are included:

- **RTL8139** (`kernel/rtl8139.c`) — the classic Realtek NIC most emulators default to
- **RTL8168/8111** (`kernel/rtl8168.c`) — the common modern Realtek Gigabit family
- **Intel iwl5100** (`kernel/iwl5100.c`) — Intel WiFi Link 5100-series wireless, talking to its CSR block over MMIO

</details>

<details>
<summary><strong>Built-in applications</strong></summary>
<br>

- **Notes** — a real text-editing window with a live cursor, typed input goes straight into the document.
- **Files** — a window onto the RAM filesystem.
- **Terminal** — the full shell described above, one click from the dock.
- **Calculator** (`kernel/calc.c`) — a working calculator app.
- **About** — a static info window.

</details>

<details>
<summary><strong>Sound</strong></summary>
<br>

- **PC speaker driver** (`kernel/speaker.c`) — tone generation through the classic PC speaker, driven through the PIT.

</details>

<details>
<summary><strong>Disk installer</strong></summary>
<br>

`kernel/installer.c` is OriginOS's own installer, and it's a deliberately serious piece of engineering, not a toy:

- Runs as a dedicated full-screen modal (the same pattern as the boot screen) — not a closable desktop window, on purpose, since installing has to work *before* and *without* the desktop even existing, and shouldn't be interruptible mid-write.
- Calls `ata_init()` and lists every drive it actually finds, across both legacy IDE and AHCI.
- Arrow keys and Enter to pick a target drive, Esc to back out safely.
- **Requires typing the literal word `INSTALL`** before anything is written to disk — a single accidental Enter press on the drive picker can't trigger a destructive write.
- Every file the installer writes — the MBR, the patched second-stage bootloader, the kernel binary, the initrd — comes from the **exact same `initrd.img` the running system already booted from**, so the installer always writes the precise bytes of the build it's currently running.
- Patches the second-stage bootloader's on-disk fields (`kernel_lba`, `kernel_sectors`, `initrd_lba`, `initrd_sectors`) to match exactly where *this* install placed the kernel and initrd.
- Writes MBR → patched stage 2 → kernel → initrd, in that order, with a real progress bar.
- Reports success or failure clearly and waits for a keypress before returning control.

</details>

<details>
<summary><strong>Assets</strong></summary>
<br>

Real bitmap assets ship in `assets/`, baked into `initrd.img` at build time:

- 5 desktop wallpapers plus thumbnails (`wallpaper.bmp` through `wallpaper5.bmp` + `_thumb` variants)
- Dock background (`dock_bg.bmp`)
- Boot screen (`bootscreen.bmp`)
- A full dock/app icon set: browser, calculator, calendar, files, folder, notes, settings, system tray, terminal, trash (empty/full), USB, weather

Every asset can be swapped out just by replacing the file of the same name in `assets/` — and if one is ever missing, the relevant piece of the UI falls back to procedural art instead of failing the build.

</details>

<details>
<summary><strong>Project structure</strong></summary>
<br>

```
boot/boot.s            — entry point, Multiboot header, framebuffer mode request
boot/gdt_flush.s        — loads the GDT, reloads segment registers
boot/idt_flush.s        — loads the IDT
boot/isr.s               — 32 ISR + 16 IRQ assembly stubs
boot/linker.ld            — links the kernel at the 1 MB mark
boot/mbr.s / stage2.s      — the installable on-disk BIOS bootloader

include/ + kernel/        — paired .h/.c per subsystem:
  gfx           — back buffer, drawing primitives, text, lines, circles
  font.h         — the embedded 5x7 bitmap font
  gdt            — Global Descriptor Table
  idt            — Interrupt Descriptor Table + IRQ dispatcher
  pic            — 8259 PIC remap + PIT timer
  mtrr           — Memory Type Range Registers
  rtc            — real-time clock
  keyboard       — IRQ1 keyboard, full layout
  mouse          — IRQ12 PS/2 mouse
  fs             — RAM filesystem (folders/files)
  wm             — window manager (drag, z-order, close)
  terminal       — command shell over the RAM-FS
  kstring        — hand-rolled string utilities (no libc)
  io.h            — I/O port access
  multiboot.h     — multiboot_info structure
  bmp            — uncompressed 24bpp BMP decoder
  boot_screen    — startup splash
  initrd         — initrd.img reader
  speaker        — PC speaker tone generation
  calc           — calculator app
  ata            — legacy IDE driver
  ahci           — AHCI/SATA driver
  pci            — PCI bus enumeration
  ethernet / arp / ip / icmp / udp / dhcp / netconfig — network stack, layer by layer
  rtl8139 / rtl8168 / iwl5100 — NIC drivers
  installer       — the on-disk installer described above
  power.h          — power management definitions

kernel/kernel.c    — the desktop itself: wallpaper, menu bar, dock, main event loop
kernel/libgcc_stubs.c — compiler intrinsics needed for a freestanding build

grub.cfg           — GRUB bootloader configuration
Makefile            — builds OriginOS.iso and runs it in QEMU
tools/              — build-time helper scripts
assets/             — bitmap assets (see above)
```

</details>

<details>
<summary><strong>Controls once it's running</strong></summary>
<br>

- **Mouse** — click a dock icon to open an app (Files / Notes / About / Terminal / Calculator); click-and-drag a window's title bar to move it; click the red traffic-light button to close it.
- **Keyboard** — typing goes to whichever window currently has focus (the most recently opened or clicked one). In the terminal, commands run on Enter and Backspace erases; in Notes, typing edits the document directly. `Esc` closes the focused window.

</details>

<details>
<summary><strong>Booting on real hardware</strong></summary>
<br>

`OriginOS.iso` is built as a **hybrid BIOS+UEFI image** — a USB stick written from this ISO boots on UEFI systems, not just through CSM/legacy compatibility mode.

That said, this only covers **booting the USB stick itself**. The bootloader that gets installed *onto a disk* by OriginOS's own installer (`boot/mbr.s` → `boot/stage2.s`) is still pure BIOS/real-mode code (`int 13h`/`int 10h` calls) with no UEFI equivalent — no GPT, no EFI System Partition, no `.efi` binary. In short:

- The **install USB** can boot in UEFI mode.
- A **disk that OriginOS was installed to** still only boots in Legacy/CSM mode.

If the target machine won't offer a Legacy/CSM option for the installed disk (or has Secure Boot enabled, which blocks any unsigned GRUB or hand-rolled boot code just as hard), go into firmware setup (usually Del/F2/F10/Esc at power-on) and:

- Enable **Legacy Boot** or **CSM** — even just as a per-drive option, it doesn't need to be global.
- Disable **Secure Boot**.

Without both of those, firmware simply won't recognize the installed disk as bootable at all and will bounce straight back to its own menu.

</details>

<details>
<summary><strong>Kernel panics & error codes</strong></summary>
<br>

OriginOS doesn't attempt to recover from a CPU-level fault — once the processor raises an exception, the kernel's own state can no longer be trusted, so instead of limping along it stops cleanly and shows a full-screen report: a sad face, the numeric exception vector, its plain-English name, and where to look it up.

The panic screen covers all 32 standard x86 exception vectors (0–31), straight from the kernel's own interrupt table. There's no tier of "worse" errors here: a page fault (14) is reported exactly as clearly and seriously as a double fault (8) — each one just points to a different root cause.

</details>

<details>
<summary><strong>Known limitations / where to go next</strong></summary>
<br>

- **Font coverage** — currently uppercase letters, digits, and basic punctuation only.
- **No persistent RAM-FS** — the in-memory filesystem resets on every reboot; files aren't yet written back to disk.
- **More applications** — an image viewer and a few other dock apps are natural next additions.
- **VBE via real BIOS calls** — for flexible resolution selection instead of a fixed mode.
- **Window resizing** — windows currently only drag; there's no resize handle yet.

</details>
