# Nextation

## Introduction

Nextation is a live Linux distribution based on the Debian Live project. It is meant to replace Thinstation (https://thinstation.org) as a kiosk OS and has been built with the intent to make it work as a drop-in replacement, so it is partially compatible with Thinstation configuration files.

This guide is targeting developers and is designed to provide insights on the custom files and configurations that have been added to the default Debian Live configuration.

Before attempting any code change, make sure to study thoroughly the [Debian Live manual](https://live-team.pages.debian.net/live-manual/html/live-manual/index.en.html).

## Main differences between Thinstation and Nextation

At bootstrap, Thinstation loads and runs the configuration files from the network as root user, before starting the X session. Nextation creates the "tsuser" at boot, populates his home directory with the contents of /etc/skel and starts X. The X autostart script will then execute the script in charge to load and run the network configuration files.

Thinstation uses IceWM or Blackbox window managers as desktop environments, while Nextation uses Xfce4 which is based on the xfwm4 window manager.

Thinstation packages are called "ports" and in most cases they need to be compiled before being installed. Nextation is Debian based and uses the popular .deb package format.

## Supported network configuration parameters

Nextation does not support all of Thinstation's network configuration parameters, but only a subset of them.

Compatible with Thinstation:

- `SESSION_<number>_TYPE`
- `SESSION_<number>_<program>_SERVER`
- `SESSION_<number>_<program>_OPTIONS`
- `NET_TIME_SERVER`

New, Nextation specific:

- `SCREEN_ROTATION`
- `TIME_ZONE`

Indexes run from 0 to 9. `SESSION_<number>_AUTOSTART` is read in `config/includes.chroot/etc/skel/load-config.sh` and is not used.

## Packages

Custom packages are defined in `config/package-lists/my.list.chroot`

- psmisc - makes the "killall" command available, useful to terminate processes
- zstd - allows for compression of the live distro partitions in zstd format (high compression and faster)
- xinput, iio-sensor-proxy - used to rotate
- curl, wget - to fetch files from the internet
- pulseaudio - enables audio
- avahi-utils, avahi-discover - (mDNS, Bonjour, Zeroconf.) allows the machine to be discovered by hostname in the local network
- plymouth-themes - used to change the splash screen shown at boot time
- vim - text editor
- open-vm-tools-desktop - integration with VirtualBox and other virtualization software. virtualbox-guest-x11 is in `config/package-lists/virtualbox.list.chroot`.
- task-xfce-desktop - Xfce desktop environment
- mousepad - graphical text editor
- chromium - browser
- openssh-server - SSH server
- dos2unix - used to convert EOL characters in text files from windows to unix format
- ristretto - image viewer
- tumbler - enables thumbnails in file explorer
- at-spi2-core, gir1.2-atspi-2.0, mousetweaks - enable assistive technology
- onboard, onboard-data - "Onboard" virtual keyboard
- firefox-esr - browser
- spice-vdagent - guest agent for virtual machines
- dconf-editor - view and edit Onboard dconf settings
- libglib2.0-bin - edit dconf from the command line
- mesa-utils - OpenGL utilities
- tigervnc-scraping-server, tigervnc-tools - VNC scraping server and password tool

## General information

### Building of an ISO

```bash
sudo apt install live-build
lb config
sudo lb build
```

### Bootstrap behaviour

Boot arguments in `auto/config` set hostname=nextation and username=tsuser. `config/includes.chroot/lib/live/config/0030-user-setup` creates tsuser at boot. Live-config copies /etc/skel into that home.

`config/includes.chroot/etc/skel/.config/autostart/launch-startup-script.desktop` runs `xterm -e ~/startup.sh`. `startup.sh` sources `fetch-config.sh`, then `load-config.sh`, and writes `startup.log`.

## Configurations

### Loading of Network configuration files

`config/includes.chroot/etc/skel/fetch-config.sh` downloads files from http://10.0.2.2:90/conf. `CONF_HOST` is 10.0.2.2, the VirtualBox NAT host.

It fetches `thinstation.hosts`, `thinstation.network`, `thinstation.conf-<hostname>`, and `thinstation.conf-<MAC without colons>`. The hostname is the first field of the `thinstation.hosts` line whose second field is that MAC.

dos2unix converts each file, then comments and blank lines are removed and the file is sourced. Order is `thinstation.network`, the hostname file, then the MAC file. `load-config.sh` applies the variables.

### VNC

Enabled by adding the tigervnc-scraping-server package. "Scraping" means that it will mirror the current X session, instead of creating a new one (as the "standalone" version of the tigervnc server does).

The server is started by the script `config/includes.chroot/etc/skel/load-config.sh`.

The password can be configured modifying the content of `config/hooks/live/0130-set-vnc-password.hook.chroot`.

### SSH

Enabled by adding the openssh-server package. The ssh daemon will start automatically at boot.

Password authentication is enabled in `config/includes.chroot/etc/ssh/sshd_config.d/enable_passwd_auth.conf`

### NTP

Defined in `config/includes.chroot/etc/skel/load-config.sh`.

It writes `/etc/systemd/timesyncd.conf.d/ntp.conf` from `NET_TIME_SERVER` and restarts systemd-timesyncd.

### Timezone

In `config/includes.chroot/etc/skel/load-config.sh`, `TIME_ZONE` replaces `/etc/localtime` with a symlink to `/usr/share/zoneinfo/$TIME_ZONE`.

### Accessibility features & tools

at-spi2-core, gir1.2-atspi-2.0, and mousetweaks are installed from `config/package-lists/my.list.chroot`. `config/includes.chroot/etc/skel/.profile` sets `org.gnome.desktop.interface toolkit-accessibility` to true. Onboard settings are in Virtual keyboard.

### Screen rotation

Invoked from `config/includes.chroot/etc/skel/load-config.sh` and defined in `config/includes.chroot/etc/skel/rotate-display.sh`.

It uses the value of the `SCREEN_ROTATION` parameter to change the orientation of the screen and of the current pointing device (i.e. mouse).

The `rotate-display.sh` script is standalone and can be also launched while Nextation is running.

Accepted values in `rotate-display.sh` are normal, left, right, and inverted.

### Browser launch

`load-config.sh` starts each session with `xterm -e`, using `SESSION_<n>_TYPE`, `SESSION_<n>_<TYPE>_SERVER`, and `SESSION_<n>_<TYPE>_OPTIONS`. `my.list.chroot` installs chromium and firefox-esr.

`config/includes.chroot_before_packages/usr/bin/chrome` contains the single word chromium. `config/hooks/live/0100-install-snapd.hook.chroot` and `0110-install-chrome-snap.hook.chroot` are commented out.

### Network auto detection

`fetch-config.sh` loops on `curl --silent --head --fail google.com` until the network answers, then records the interface, MAC, and IPv4 addresses used to reach the config host.

### Splash screen

Theme files are in `config/includes.chroot/usr/share/plymouth/themes/logo-mac-style/`. `config/hooks/live/0120-change-plymouth-theme.hook.chroot` runs `plymouth-set-default-theme logo-mac-style`.

`config/includes.chroot/etc/skel/plymouth-preview.sh` shows the splash when run as root. The boot arguments in `auto/config` include quiet splash.

### Virtual keyboard

onboard and onboard-data are in `my.list.chroot`. `config/includes.chroot/etc/skel/.profile` starts Onboard minimized, with auto-show, docking, and the Small.onboard layout.

`config/includes.chroot/etc/skel/.config/autostart/onboard-autostart.desktop` only sets `X-XFCE-Autostart-Override` and does not start Onboard.

### Touchpad tap to click

Defined in `config/includes.chroot/etc/X11/xorg.conf.d/40-libinput.conf`. Option "Tapping" "on" enables tap-to-click for touchpads.
