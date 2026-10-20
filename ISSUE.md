# Draft issue for xorg/driver/xf86-video-mach64

**Title:** Segfault in ATIMach64Detect on Linux/x86: block I/O above 0x3FF not granted by xf86EnableIO

---

PCI Rage XL (1002:4752) crashes in PreInit on every start with "Segmentation fault at address 0x0".

**System**
- Intel STL2 (ServerWorks ServerSet III LE), dual Pentium III, Debian 12 i386
- Kernel 6.1.0-54-686-pae (`CONFIG_X86_IOPL_IOPERM=y`)
- xserver-xorg-core 2:21.1.7-3+deb12u13, xserver-xorg-video-mach64 6.9.7-1
- Rage XL is boot VGA; `I/O ports at 1000 [size=256]`

**Backtrace**
```
ATIMach64Detect   atiprobe.c:143   bus_cntl = inr(BUS_CNTL);
ATIMach64Probe    atiprobe.c:278
ATIMach64ProbeIO  atiprobe.c:454
ATIPreInit        atipreinit.c:644
```
`CPIOBase = 0x1000`, `CPIODecoding = BLOCK_IO`.

**Cause**

On Linux/x86, `hwEnableIO()` in `hw/xfree86/os-support/linux/lnx_video.c` only calls `ioperm(0, 1024, 1)`; nothing calls `iopl(3)`. The first `inl` on the block I/O base (0x1000) raises #GP, which the kernel delivers as SIGSEGV:

```
ioperm(0, 0x400, 1)                     = 0
ioperm(0x28, 0x4, 0)                    = 0
ioperm(0x32, 0x4, 0)                    = 0
ioperm(0x3c, 0x1, 0)                    = 0
ioperm(0x40, 0x1, 0)                    = 0
--- SIGSEGV {si_signo=SIGSEGV, si_code=SI_KERNEL, si_addr=NULL} ---
```

Granting only the block I/O range is not enough: `ATIFindVGA()` then faults on `outb` to 0x46E8 (VGA setup port, atiprobe.c:355).

**Fix**

Calling `iopl(3)` in `ATIMach64ProbeIO()` after block I/O is detected fixes it; the server starts and runs with EXA at 1600x1200. Suggested change (`<sys/io.h>` conflicts with the in/out helpers in xorg `compiler.h`, hence the bare prototype):

```diff
--- a/src/atiprobe.c
+++ b/src/atiprobe.c
@@ -28,6 +28,12 @@
 #include <stdio.h>
 #include <stdint.h>
 
+#if defined(__linux__) && (defined(__i386__) || defined(__x86_64__))
+#include <errno.h>
+/* <sys/io.h> conflicts with xorg compiler.h in/out helpers */
+extern int iopl(int);
+#endif
+
 #include "ati.h"
 #include "atibus.h"
 #include "atichip.h"
@@ -449,6 +455,17 @@
         pATI->CPIOBase = PCI_REGION_BASE(pVideo, 1, REGION_IO);
         pATI->CPIODecoding = BLOCK_IO;
         pATI->PCIInfo = pVideo;
+
+#if defined(__linux__) && (defined(__i386__) || defined(__x86_64__))
+        /*
+         * On Linux/x86 xf86EnableIO() only grants ports 0x000-0x3FF via
+         * ioperm().  Block I/O above 0x3FF and the VGA setup port 0x46E8
+         * then fault on first access, so take full I/O privilege here.
+         */
+        if (iopl(3) != 0)
+            xf86Msg(X_WARNING, MACH64_NAME ": iopl(3) failed: %s\n",
+                strerror(errno));
+#endif
     }
 
     if (!ATIMach64Probe(pATI, pVideo, pATI->Chip))
```

Arguably the server should grant this instead; happy to file it there if preferred.

**Unrelated aside, same function:** the keyboard/timer trap loop in `hwEnableIO()` parses the hex addresses from `/proc/ioports` with `atoi()`, so `0040-0043 timer0` becomes ports 40-43 decimal (0x28). That's why the trace above shows 0x28/0x32/0x3c/0x40 instead of 0x40/0x50/0x60/0x64.

Diagnosed with help from an AI assistant; reproduced and tested on the hardware above.
