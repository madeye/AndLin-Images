# ServerBox-Images

The root filesystem images for [ServerBox](https://github.com/madeye/ServerBox), which turns a
used Android phone into a Linux server. Each image is a headless Linux distribution with an SSH
server that ServerBox runs under PRoot or, on phones with the Android Virtualization Framework, in
a VM.

This repository started as a fork of
[CypherpunkArmory/OCI-Builder](https://github.com/CypherpunkArmory/OCI-Builder), which builds the
images for [UserLAnd](https://github.com/CypherpunkArmory/UserLAnd). The Dockerfiles, the build
workflow and the shared assets come from that work; ServerBox drops the desktop images and keeps
only headless server ones.

## Images

| Distro | Dockerfile          | Image                             | Platforms                                            |
|--------|---------------------|-----------------------------------|------------------------------------------------------|
| Ubuntu | `Dockerfile.ubuntu` | `ghcr.io/madeye/serverbox-ubuntu` | amd64, arm64, arm/v7                                 |
| Debian | `Dockerfile.debian` | `ghcr.io/madeye/serverbox-debian` | amd64, arm64, arm/v7, 386                            |
| Kali   | `Dockerfile.kali`   | `ghcr.io/madeye/serverbox-kali`   | amd64, arm64                                         |
| Arch   | `Dockerfile.arch`   | `ghcr.io/madeye/serverbox-arch`   | amd64, arm64, arm/v7                                 |
| Alpine | `Dockerfile.alpine` | `ghcr.io/madeye/serverbox-alpine` | amd64, arm64, arm/v7, 386                            |

Each distro has a `build-<distro>.yml` workflow that calls the reusable `build-image.yml`. Images
are tagged `:latest` and `:YYYYMMDD`, and also pushed to Docker Hub when `DOCKERHUB_USERNAME` and
`DOCKERHUB_TOKEN` are set. A build runs when its Dockerfile, `input/` or `assets/common/` changes on
`master`, on demand, and weekly on Mondays; the weekly run is skipped when the upstream base image's
digest hasn't changed since the last build (it's stored as a label on the image).

## What's in an image

- The distro's base image (e.g. `FROM ubuntu:latest`; Arch uses `menci/archlinuxarm`, which
  covers amd64, arm64 and arm/v7 in one image).
- systemd and systemd-sysv on Ubuntu, Debian, Kali and Arch: the AVF VM backend boots the image
  with systemd, so this is a requirement, not a convenience. Alpine has no systemd and runs under
  PRoot only.
- sudo, curl, wget, and dropbear, ServerBox's SSH server, started by
  `/support/startSSHServer.sh` on port 2022.
- `/support/busybox` and `/support/libdisableselinux.so`, built in a separate builder stage so the
  toolchain doesn't ship in the image, and the scripts from `assets/common/` in `/support/`.
  Alpine layers `assets/alpine/addNonRootUser.sh` over the common one, since Alpine needs the
  shell registered in `/etc/shells` before `chsh`.
- A small server baseline: `ca-certificates`, `openssh-client`, `less`, `nano`, `procps`,
  `iproute2`, `iputils-ping` (or the distro's equivalents, e.g. `iputils` and `procps-ng` on
  Arch), `tzdata`, and cron (`cron` on Ubuntu/Debian/Kali, `cronie` on Arch; Alpine's
  `busybox-suid` already provides `crond`).
- An `en_US.UTF-8` locale on the glibc distros. Alpine uses musl, which doesn't support glibc-style
  locale generation, so it sets `LANG=C.UTF-8` instead.
- The label `org.serverbox.variant=server`.

There is no X11, VNC or desktop stack. Installs use `--no-install-recommends` (apt), cache purging
(pacman) or `--no-cache` (apk) to keep the images small.

## Building locally

```sh
docker buildx build --platform linux/arm64 -f Dockerfile.debian -t serverbox-debian .
```
