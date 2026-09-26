# AndLin-Images

This is a fork of [CypherpunkArmory/OCI-Builder](https://github.com/CypherpunkArmory/OCI-Builder),
which builds the OCI root filesystem images used by
[UserLAnd](https://github.com/CypherpunkArmory/UserLAnd) to run Linux distros
under proot or in a VM on Android. All credit for the original Dockerfiles,
workflows and desktop-image assets in this repository goes to
CypherpunkArmory — this fork only adds a new set of images on top of that
work.

This fork exists to build images for **AndLin**, a headless-server-focused
distribution of UserLAnd. AndLin keeps the desktop (`userland-*`) images
CypherpunkArmory already publishes, and adds a parallel set of `andlin-*`
"server" images that drop the X11/VNC/desktop stack entirely, since server
use of a Linux distro on Android doesn't need it — a shell over SSH does.

## Server images

| Distro | Dockerfile                 | Image                          | Workflow                      |
|--------|-----------------------------|---------------------------------|--------------------------------|
| Ubuntu | `Dockerfile.ubuntu_server`  | `ghcr.io/madeye/andlin-ubuntu`  | `build-ubuntu_server.yml`      |
| Debian | `Dockerfile.debian_server`  | `ghcr.io/madeye/andlin-debian`  | `build-debian_server.yml`      |
| Kali   | `Dockerfile.kali_server`    | `ghcr.io/madeye/andlin-kali`    | `build-kali_server.yml`        |
| Arch   | `Dockerfile.arch_server`    | `ghcr.io/madeye/andlin-arch`    | `build-arch_server.yml`        |
| Alpine | `Dockerfile.alpine_server`  | `ghcr.io/madeye/andlin-alpine`  | `build-alpine_server.yml`      |

Each is built by the same reusable `build-image.yml` workflow the desktop
images use, tagged `:latest` and `:YYYYMMDD`, and labelled with the upstream
base image's digest for change detection. Platforms match the corresponding
desktop image (see each workflow for its `platforms` list).

### What's different from the desktop (`userland-*`) images

Kept, unchanged from the desktop variant:

- The same base image per distro (e.g. `FROM ubuntu:latest`).
- systemd + systemd-sysv where the desktop image has them (Ubuntu, Debian,
  Kali) — this is a hard requirement of UserLAnd's AVF VM backend, not a
  desktop convenience; see the comment in each Dockerfile. Alpine still has
  no systemd, matching upstream `Dockerfile.alpine`. Arch keeps systemd too.
- sudo, dropbear (AndLin's SSH server, started via
  `/support/startSSHServer.sh` on port 2022), curl, wget.
- The builder stage that produces `/support/busybox` and
  `/support/libdisableselinux.so`, and the `assets/common/` copy into
  `/support/` with the same permissions.

Dropped (X11/VNC/desktop stack — not needed for headless server use):

- `tightvncserver` / `x11vnc` (VNC server)
- `xterm`, `xfonts-base`, `twm`, `xorg-twm`, `xorg-server-xvfb`, `xorg-xsetroot`
- `libgl1`, `libglx-mesa0`, `mesa-gl` (software GL for the desktop)
- `pulseaudio`
- `expect` (only used to script `vncpasswd`)
- The distro-specific VNC startup script overrides
  (`assets/debian/startVNCServer.sh`, `assets/arch/startVNCServerStep2.sh`,
  `assets/alpine/startVNCServerStep2.sh`) are **not** copied into the server
  images, since the packages they depend on aren't installed. The
  VNC/XSDL scripts under `assets/common/` are still copied in (they're part
  of the shared common asset bundle) but are inert without the packages above
  — harmless dead weight, not a functional path.
- `assets/alpine/addNonRootUser.sh` **is** still copied in for the Alpine
  server image (it registers the shell in `/etc/shells` before `chsh`, which
  the desktop and server images both need); only its sibling
  `startVNCServerStep2.sh` is excluded.

Added — a small, generally useful server baseline:

- `ca-certificates`, `openssh-client`, `less`, `nano`, `procps`, `iproute2`,
  `iputils-ping` (or the distro's equivalent package names, e.g. `iputils`
  and `procps-ng` on Arch).
- Locale generation for `en_US.UTF-8` on the apt-based images (Ubuntu,
  Debian, Kali) and Arch, where glibc's locale machinery supports it. Alpine
  uses musl libc, which doesn't support glibc-style locale generation, so its
  server image just sets `LANG=C.UTF-8` instead of installing a locales
  package.
- `tzdata`.
- `cron` (Ubuntu/Debian/Kali: `cron`; Arch: `cronie`). Not added for Alpine —
  `busybox-suid`, already required, already provides `crond` as a built-in
  applet.
- The OCI label `org.andlin.variant=server` on every server image, so
  tooling can tell them apart from the desktop images at a glance
  (`docker inspect --format '{{ index .Config.Labels "org.andlin.variant" }}'`).

All apt/pacman/apk installs use `--no-install-recommends` (apt) /
`--overwrite` with cache purging (pacman) / `--no-cache` (apk) to keep the
images small, matching the desktop Dockerfiles' existing practice.

## Everything else

See the upstream [CypherpunkArmory/OCI-Builder](https://github.com/CypherpunkArmory/OCI-Builder)
repository for the desktop (`userland-*`) images, their Dockerfiles, and the
shared build infrastructure this fork builds on.
