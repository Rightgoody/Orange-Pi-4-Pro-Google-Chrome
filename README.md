# Orange Pi 4 Pro — Google Chrome WebGL

Date: 3 November 2026
Board: Orange Pi 4 Pro (Allwinner A733, kernel family sun60iw2)
System: Orange Pi 1.1.0 Resolute (Ubuntu 26.04), XFCE on X11
Browser: Google Chrome 156.0.8078.47
GPU: Imagination PowerVR B-Series BXM-4-64

## What broke

WebGL stopped working in Google Chrome, then came back only as a very slow software renderer. The WebGL Aquarium, which used to be smooth, ran at a crawl. The board image itself still had a working GPU. Chrome was the piece that changed.

## What caused it

On 31 July 2026 the Orange Pi image installed Google Chrome 151.0.7922.71. On 26 October 2026 at 00:05, apt upgraded that package to Chrome 156.0.8078.47, in the same batch as Docker, Cursor, LibreWolf, and VLC. Mesa, the kernel, and the PowerVR driver were not upgraded that night.

The Orange Pi image build does not leave Chrome’s launcher alone. For a Resolute desktop image, `orangepi-build` downloads Google’s current Chrome package and then rewrites `/opt/google/chrome/google-chrome`. The source is `external/config/sources/families/sun60iw2.conf` on the `next` branch of https://github.com/orangepi-xunlong/orangepi-build. The edit adds these flags to every launch:

```
--use-gl=angle --use-angle=vulkan --enable-features=Vulkan,VulkanFromANGLE --ignore-gpu-blocklist
```

That file belongs to the `google-chrome-stable` package. The 26 October upgrade replaced it with Google’s stock launcher, so the Vulkan flags were gone.

This GPU driver speaks OpenGL ES and Vulkan. It does not provide desktop OpenGL. Without the image’s flags, Chrome on X11 probes desktop OpenGL, fails to bring up the `sunxi-drm` driver, fails a Zink (OpenGL-on-Vulkan) fallback, and settles on Mesa `llvmpipe`, which draws with the CPU. Current Chrome also no longer invents a software WebGL context when that happens, so WebGL first disappeared. Turning the GPU blocklist off later made WebGL exist again, but still on `llvmpipe`, which is why the aquarium was extremely slow.

`chrome://gpu` will say “WebGL: Hardware accelerated” even in that state. The line that matters is **GL_RENDERER**. `llvmpipe` means the CPU. `PowerVR` means the GPU.

## What was checked

- `glxinfo` reported `llvmpipe` (CPU OpenGL). Xorg’s own log still said `glamor X acceleration enabled on PowerVR B-Series BXM-4-64`.
- `eglinfo` reported `PowerVR B-Series BXM-4-64`, OpenGL ES 3.2, build `24.2@6603887`. The GPU was fine for programs that use the Orange Pi EGL libraries in `/usr/local/lib`.
- Chrome’s GPU process, before the fix, was full of threads named `llvmpipe-0` through `llvmpipe-7` and was using more than a whole CPU core.
- Chrome settings had not turned hardware acceleration off.
- The stock launcher on disk was Google’s original last line: `exec -a "$0" "$HERE/chrome" "$@"` with no Vulkan flags.

## Attempts that did not restore GPU speed

These were useful for diagnosis. They are not the fix.

1. `--use-gl=angle --use-angle=gles --ignore-gpu-blocklist --disable-gpu-sandbox`  
   The first time this was typed, Chrome was already running, so the new flags were ignored and the old window stayed open. The terminal still printed `glx: failed to create dri3 screen`, `failed to load driver: sunxi-drm`, and `Zink requires the nullDescriptor feature`. Those lines are the desktop-GL probe failing. They can appear even when Vulkan WebGL is working.

2. `chrome://flags/#ignore-gpu-blocklist` set to Enabled.  
   This made WebGL show up again, on the CPU. The aquarium was slow. The saved GPU report `about-gpu-2026-11-03T05-24-43-133Z.txt` showed **GL_RENDERER** `llvmpipe` and a command line that contained only `--ignore-gpu-blocklist`.

3. A full quit, then `--use-gl=angle --use-angle=gles --ignore-gpu-blocklist`.  
   The flags did apply (`about-gpu-2026-11-03T05-26-50-370Z.txt`, display type `ANGLE_OPENGLES`). Chrome still chose Mesa’s software OpenGL ES 3.2 (`llvmpipe`, Mesa 26.0.3), not the PowerVR driver. Chrome’s GPU process loaded Ubuntu’s `libGL.so` and never switched to the Imagination Vulkan driver.

4. `chrome://flags/#use-angle`  
   That entry is not in Chrome 156’s flags page. The switch still exists, but only as a launch argument.

## The fix that worked

Launch Chrome the way the Orange Pi image did:

```bash
google-chrome --use-gl=angle --use-angle=vulkan --enable-features=Vulkan,VulkanFromANGLE --ignore-gpu-blocklist
```

Quit any already-running Chrome first. A second `google-chrome` command only opens a window in the existing process and ignores new flags. Check with:

```bash
pkill -f /opt/google/chrome/chrome
sleep 1
pgrep -af /opt/google/chrome/chrome || echo "chrome is stopped"
```

After this launch, WebGL performance matched the GPU again. `--use-angle=vulkan` is the important difference from the GLES attempts. This board’s fast path for Chrome is ANGLE on the Imagination Vulkan driver (`libVK_IMG.so`), which is what the image build selects.

## How that was made the default

The image only patches the launcher while the image is being built. It does not survive the next `apt upgrade`. Two pieces were installed so normal launches and future Chrome upgrades keep the flags.

1. `/opt/google/chrome/google-chrome` was edited so its last line is again:

```
exec -a "$0" "$HERE/chrome" --use-gl=angle --use-angle=vulkan --enable-features=Vulkan,VulkanFromANGLE --ignore-gpu-blocklist "$@"
```

`/usr/bin/google-chrome` and `/usr/bin/google-chrome-stable` both point at this file, so the menu and a terminal get the same flags.

2. `/usr/local/sbin/orangepi-chrome-gpu` reapplies that line if an upgrade puts Google’s stock launcher back. `/etc/apt/apt.conf.d/99-orangepi-chrome-gpu` runs that script after apt package operations. Copies of both files are in this folder.

A temporary menu launcher in `~/.local/share/applications/google-chrome.desktop` was created while the system file could not be edited, then removed after the real launcher was patched, so the flags are not passed twice.

## How to confirm after a restart

Quit Chrome completely, open it from the menu, and open `chrome://gpu`.

- **Command Line** includes `--use-angle=vulkan`.
- **Vulkan** is enabled.
- **GL_RENDERER** names the PowerVR GPU, not `llvmpipe`.

The `sunxi-drm` and Zink messages in the terminal can still show up. They are the desktop OpenGL probe. They are not the WebGL path once Vulkan is active.
