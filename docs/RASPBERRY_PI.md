# Raspberry Pi notes

Build and runtime notes for running camstream on a Raspberry Pi. There is no x86-specific code in `src/`; the Pi-relevant paths are CSI capture through `rpicam-vid`, the V4L2 M2M hardware encoder and the mDNS responder.

## Supported setups

| Device | Camera | Encoder | Notes |
|---|---|---|---|
| Pi Zero 2 / 3 / 4 | CSI IMX219/477/708 via `rpicam-vid` | hardware `/dev/video11` (bcm2835-codec) | lowest CPU |
| Pi 5 | same CSI | no hardware H.264 block, use libx264 | `--encoder sw`, or `auto` falls back on its own |
| x86 / any Linux | USB V4L2 `/dev/video0` | libx264 | `--source v4l2` |
| no camera | `--test` synthetic pattern | libx264 | for development |

Testing status: the code builds warning-free and the test suite passes on x86_64, and the aarch64 cross build of both binaries is verified. The V4L2 M2M encoder path follows the bcm2835-codec driver source but has not been run on Pi hardware in this revision.

## Build on Pi

```sh
sudo apt update && sudo apt install -y build-essential libssl-dev libsrtp2-dev libx264-dev rpicam-apps
git clone --recursive https://github.com/salman-cc/cam.git camstream && cd camstream
make -j4
./build/camstream --source csi --width 1280 --height 720 --fps 30 --encoder auto --listen 0.0.0.0 --http-port 8080
```

For the libpeer variant:

```sh
sudo apt install -y cmake python3-jsonschema python3-jinja2
git submodule update --init --recursive
make camstream-libpeer -j4
./build/camstream-libpeer --source csi --width 1280 --height 720 --fps 30 --listen 0.0.0.0 --http-port 8080
```

## Cross compile from x86_64

Using `zig cc`:

```sh
pip install ziglang
zig cc -target aarch64-linux-gnu -O2 server.c mongoose.c -o web_server_pi   # minimal server

make camstream-libpeer CMAKE=cmake \
  LIBPEER_CMAKE_ARGS="-DCMAKE_TOOLCHAIN_FILE=cmake/zig-aarch64.cmake" \
  CC="zig cc -target aarch64-linux-gnu"                                       # full binary
```

The minimal server needs only libc, and libpeer cross-builds with its vendored mbedtls/libsrtp2, so neither needs a Pi sysroot.

## Pi-specific behavior

- **CSI capture**: `src/media/csi_source.c` spawns `rpicam-vid --codec yuv420 --flush -o -` via `posix_spawn` with close-on-exec descriptors and a 1 MB pipe. libcamera pads luma rows to 64 bytes and chroma to 32; the source reports the stride and the encoder strips it.
- **Hardware encoder**: `src/media/encoder_v4l2m2m.c` probes `/dev/video11` first (bcm2835-codec on Pi 0-4), retries EAGAIN/EBUSY 3 times, and `auto` falls back to libx264 on Pi 5.
- **mDNS**: `src/net/mdns.c` publishes `camstream.local` and works alongside Avahi or systemd-resolved.
- **libpeer**: pinned at `5b849de`, ECDSA certificates (`CONFIG_DTLS_USE_ECDSA=1`), which take milliseconds to generate on a Pi where RSA would take seconds. `src/lpstream/lp_sdp.c` sanitizes browser answers before libpeer's fixed-size buffers see them.

## systemd

`packaging/camstream.service` and `packaging/camstream-libpeer.service`: `KillMode=mixed`, `SupplementaryGroups=video`, no `User=` by default so a fresh install works, with a commented `camstream` user for a dedicated account. Install with `sudo make install` / `sudo make install-libpeer`.

## Pitfalls

| Symptom | Fix |
|---|---|
| `v4l2: cannot open /dev/video0` | `sudo usermod -aG video $USER`, re-login, check `ls /dev/video*` |
| `VIDIOC_STREAMON errno 22` with CSI | use `--source csi`; `/dev/video0` on a CSI sensor is raw Bayer |
| `csi: camera busy: PID ...` | `sudo fuser -v /dev/video0`, stop the other libcamera app |
| `m2m ... errno 11 EAGAIN` | hardware encoder refused; `auto` falls back to libx264 |
| `http: cannot listen 8080` | `sudo systemctl stop camstream`, or find the holder with `ss -ltnp 'sport=:8080'` |
| browser shows `PR_END_OF_FILE_ERROR` | `https://` was typed; use `http://<pi-ip>:8080/` |
| `camstream.local` does not resolve | enable mDNS on the client (`resolvectl mdns eth0 yes` or Avahi), or use the IP from `/api/status` |
| high CPU with no viewer | only expected for `test`/`stdin` sources; CSI plus the hardware encoder idles near zero |
| minimal server returns 404 | check `web_root/` exists and `root_dir` in `server.c` points at it |

## Firewall

```sh
sudo ufw allow 8080/tcp && sudo ufw allow 50000:50007/udp
# or firewalld:
sudo firewall-cmd --permanent --add-port=8080/tcp --add-port=50000-50007/udp
sudo firewall-cmd --reload
```

## Single cable between Pi and laptop

With no DHCP server, both sides fall back to link-local `169.254.x.x` addresses and mDNS still resolves `camstream.local` over the same cable. On Pi OS Bookworm (NetworkManager):

```sh
nmcli con show
sudo nmcli con mod "Wired connection 1" ipv4.method auto ipv4.link-local enabled ipv4.dhcp-timeout infinity
sudo nmcli con up "Wired connection 1"
```
