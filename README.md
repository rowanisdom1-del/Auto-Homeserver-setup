# Auto-Homeserver-Setup

One script that turns a fresh Raspberry Pi into a fully working home server: **Immich** (photos), **Jellyfin** (movies/TV), shared over **Samba** locally and the internet via **Tailscale Funnel** — all from five questions, no manual browser wizards.

Tested on a Raspberry Pi 5 (8GB) with Raspberry Pi OS Lite 64-bit and a USB 3 external drive.

## What it does

- Detects, formats (with a confirmation step — see [safety](#safety)), and permanently mounts your external drive
- Installs Docker, Immich, and Jellyfin
- Works around the Raspberry Pi 5's 16KB kernel page size, which otherwise crash-loops Jellyfin's .NET runtime
- Completes Jellyfin's first-run setup and adds Movies/TV libraries via its API — no clicking through a browser wizard
- Sets up a Samba share for the media folder
- Installs Tailscale, logs it in, and turns on Funnel so the server is reachable outside your network
- Installs a small permanent service so that **after any future reboot, everything reconnects on its own within about a minute** — no manual `tailscale funnel` command needed ever again

## Usage

Clone it or just grab the script directly:

```bash
git clone https://github.com/rowanisdom1-del/Auto-Homeserver-setup.git
cd Auto-Homeserver-setup
```

Then copy it to the Pi and run it:

```bash
scp media_server_setup.sh USER@yourhost.local:~/
ssh USER@yourhost.local
chmod +x media_server_setup.sh
sudo ./media_server_setup.sh
```

You'll be asked once for: username, hostname, timezone, a password (used for the Immich DB, Jellyfin admin, and Samba), the Jellyfin display name, and which port to expose via Funnel (defaults to Jellyfin's `8096`).

The script then runs unattended through several required reboots. Watch progress any time with:

```bash
tail -f /var/lib/media-server-setup/log.txt
```

It's safe to re-run by hand if anything goes wrong partway — every step checks whether it already completed and skips it if so:

```bash
sudo /usr/local/sbin/media-server-setup.sh
```

## Safety

Before formatting anything, the script prints the exact drive model and size it detected and requires you to type `WIPE` (all caps) to continue — typing anything else cancels with nothing touched. This confirmation always happens during your first, manual, foreground run, before any of the required reboots — see [kinks](#kinks-we-hit-building-this) for why that matters.

## What still needs you

A few things can't be safely automated:

1. **Disable key expiry** for the Pi at [login.tailscale.com/admin](https://login.tailscale.com/admin) → Machines → your Pi → ⋯. Without this, Tailscale eventually drops the device and everything stops working until you log in again.
2. Confirm **MagicDNS** is on under the admin console's DNS tab.
3. **Copy your actual media files** into `/mnt/storage/media/movies` and `/tv`, named so Jellyfin can match them:
   - Movies: `Title (Year)/Title (Year).ext`
   - TV: `Show/Season 01/Show - S01E01.ext`

## Kinks we hit building this

Real issues found while actually running this on hardware, in case you hit the same:

- **`zram0` and `loop0` look like real disks to `lsblk`.** The drive-detection filter now explicitly excludes anything named `loop*` or `zram*`, not just the SD card.
- **A destructive confirmation prompt can hang forever if it ever runs unattended.** The script auto-resumes after each required reboot via a systemd service with no terminal attached — if the one prompt that needs a human (`WIPE`) ever landed *after* a reboot, it would sit there waiting for input nobody could give it, with no visible error. Fixed by moving that step to always run during the very first, manual invocation, before the first automatic reboot.
- **`ssh` will refuse to connect after you reflash the SD card.** Your computer remembers the old host key; the new OS has a new one. Fix: `ssh-keygen -R yourhost.local`, then reconnect and accept the new key.
- **Tailscale Funnel doesn't turn itself on.** Installing and logging into Tailscale isn't the same as exposing a port — that needs its own explicit `tailscale funnel --bg <port>` step, which is easy to forget (we did, the first time).
- **Funnel doesn't necessarily survive every reboot on its own.** A small permanent systemd service (`media-server-funnel.service`, installed by the setup script and left in place forever) waits for Tailscale to come back up after any boot and re-applies the funnel automatically.

## Requirements

- Raspberry Pi (tested on Pi 5) running Raspberry Pi OS Lite 64-bit, freshly flashed with SSH + your user enabled via Raspberry Pi Imager
- One external USB drive, already plugged in, and no others connected
- Internet access on the Pi

## License

Use it, change it, break it, it's yours.

Use it, change it, break it, it's yours.
