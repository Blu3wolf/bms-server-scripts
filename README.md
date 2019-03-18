# Shell scripts for running Falcon BMS

The purpose is to run BMS as server hosting multiplayer sessions. Since we are
using a wine version where drawing primitives is specifically disabled this is
not intended for client use.

## Requirements

A basic understanding of using a UNIX shell is required. Please don't just
copy & paste the commands listed in this README.

### Hardware

* x86\_64 architecture
* 4GB of RAM and 4 CPU cores.
* **No** dedicated GPU or GPU passthrough required using Mesa's [llvmpipe]
  driver

It's recommended to run this in a separate VM dedicated just to Falcon BMS.
Tested using KVM with [PVE] as hypervisor.

### Software

* [Debian GNU/Linux 9.8]
* [contrib] and [backports] sources enabled
* dedicated user for running the scripts and BMS.
* Steam account that owns Falcon 4.0

These scripts have been written and tested with Debian stretch. Though usally
only the `bootstrap.sh` should be distribution specific.

## Usage

Using these scripts is divided into three stages. First we'll bootstrap the
system to be able to run X, VNC and wine using `bootstrap.sh`. After that we'll
install Falcon 4.0 and Falcon BMS using SteamCmd and wine. The last stage is
running BMS and optionally IVC.

SSH into your server/VM and clone this git repository.

```sh
git clone https://github.com/lukrop/bms-server-scripts bms-server
cd bms-server
```

## boostrap.sh

The bootsrap script needs to be run as root or alternativley with sudo.  It
adds the i386 architecture to `dpkg` and installs some packages with `apt`. It
loads the `snd-dummy` kernel module to provide a dummy sound device and makes
sure to load it at each boot. It adds the `$BMS_USER` to the _audio_ and
_video_ groups. Also it will configure and start a VNC server.  `vncserver`
will prompt you do enter a password for connecting to it.

### Variables

* `BMS_USER=bms` user to run Falcon BMS.
* `MANAGE_PACKAGES=1` set to 0 if you don't want the script to install
  packages.
* `VNC_SERVER_ARGS="-localhost yes"` arguments passed to `vncserver`

### Usage

As root `./bootstrap.sh` or as normal user with sudo privileges `sudo
./bootstrap.sh`.

Every script has some **configuration** variables which you can adjust to your
liking. See the [Configuration](#configuration) for details.

Override default configuration:

```sh
BMS_USER=myuser MANAGE_PACKAGES=0 ./bootstrap.sh
```

## install.sh

This script will download a [patched version of wine], Falcon BMS 4.33 U1 Setup
and incremental installers from U2 to U5 via torrent.  It will per default use
[SteamCmd] to install Falcon 4.0. A Steam account which owns the game is
**required**. If you own Falcon 4.0 differently (e.g. GoG, CD-ROM) you'll have
to manually install it in the _WINEPREFIX_ using wine.

The script will download a tar archive with wine-4.3-nodraw precompiled. If you
don't trust my binaries you can clone the repository and build it yourself.
Remember to build win32 and win64, winehq recommends doing that in separate x86
and x86\_64 build systems, e.g. docker or LXC.

### Variables

* `DOWNLOAD_BMS=1` Set to 0 if you don't want the script to download BMS via
  torrent. In that case you'll have to place the installer files into the
  `sources/` directory yourself.
* `DOWNLOAD_WINE=1` Set to 0 to disable downloading the precompiled wine
  binaries. In that case you'll have to place the wine binaries into
  `wine/wine-4.3-nodraw` yourself.
* `INSTALL_FALCON_4=1` Whether Falcon 4.0 is going to be installed using
  SteamCmd.
* `INSTLAL_BMS=1` If we actually want to install BMS.
* `TWEAK_CONFIG=1` Adjust options in _Falcon BMS.cfg_ for optimal performance.
* `RESIZE_TEXTURES=1` Set to 0 to disable resizing the textures to 1x1.
* `SEED_TIME=15` Time to seed the torrent in minutes. Set to 0 to stop seeding
  after the download is finished.

### Usage

After bootstrapping you should have a VNC server running with jwm as window
manager. Per default the VNC server will only accept connections from localhost
so you'll want to use a SSH tunnel to connect with your VNC client of choice.
Substitute _user_ and _host_ with your server information:

```sh
ssh -L 5901:localhost:5901 user@host
```

After that you can connect with your VNC client to `localhost:5901` and should
be rewarded with a jwm session. Left click anywhere and choose _Terminal_ from
the menu. Using the shell navigate to the directory of these scripts and run
`./install.sh`.

With the default configuration the script will now download the patched wine
binaries and place them into `wine/wine-4.3-nodraw`. After that it will
download SteamCmd and prompt for Steam account credentials. If you have
SteamGuard active you'll have to also input the code sent to you by Mail. The
script doesn't store your credentials it simply passes them on to SteamCmd.
SteamCmd will download and install Falcon 4.0.

In the next step the script will download and extract the BMS installer files
to `sources/`. If the files are successfully extracted the Falcon BMS Setup
will launch. Just click your way through it. Do **not** change the installtion
path. When the U1 Setup is finished the setup wizards for the incremental
updates 2 to 5 will automatically start. As with the main Setup just mash that
"Next" button.

Now the script will adjust options in `User/Config/Falcon BMS.cfg` to optimize
performance, disabling eye-candy as much as possible. In the last step the
script will use `mogrify` to resize most of the .DDS and .GIF textures to a
size of 1x1. This reduces disk usage, speeds up loading times and memory
footprint.

## start.sh

This script launches Falcon BMS and optionally the IVC Server using the patched
wine version.

### Variables

* `BMS_LOGFILE=logs/bms.log` logfile location
* `BMS_THEATER="Korea KTO"` Theater with which BMS will start up
* `BMS_ARGS="-window"` arguments passed to BMS
* `START_IVC=1` whether to start the IVC server
* `PROMETHEUS_METRICS=0` scrape the FPS values from the BMS log and export them
  with prometheus-node-exporter
* `PROMETHEUS_FILE="/var/lib/prometheus/node-exporter/bms-fps.prom"` prom file
  location. This file needs to be writable for `$BMS_USER`.

### Usage

If you don't need to change any default options it's as easy as doing
`./start.sh`.

In case you want to run a different theater and export FPS metrics to
prometheus:

```sh
BMS_THEATER="Lorik's Korea 1.11" PROMETHEUS_METRICS=1 ./start.sh
```

# Appendix

## BMS graphics settings

Don't forget to configure the graphics settings in-game after you started BMS
for the first time. I recommend the following settings:

![Graphics Settings](graphics_settings.png?raw=true "Graphics Settings")

Notice the grass slider still beeing at the recommended default position. I
experienced issues with the AI not beeing able to taxi properly at Kunsan when
the slider was all the way to the left.

## Monitoring

### Locally

You can check the output of the terminal window running the `start.sh` script
or alternativley check the log file.

```sh
tail -f bms-server.log
```

### Prometheus node-exporter

Currently there is a small helper script located in `prometheus` which reads
the contents of the server log where the current FPS are written to and creates
the output for a `.prom` file, exposing `bms_fps` as gauge value.

If the output of the script is redirected into a `.prom` file it can be made
available for scraping by node-exporter.

As root you could create such a file in `/var/lib/prometheus/node-exporter` and
make it writeable for the group of your bms user:

```sh
touch /var/lib/prometheus/node-exporter/bms-fps.prom
chown root:$BMS_USER /var/lib/prometheus/node-exporter/bms-fps.prom
chmod g+w /var/lib/prometheus/node-exporter/bms-fps.prom
```

## License

Licensed under the MIT license. See LICENSE file.


[PVE]: https://git.lukrop.com/lukrop/bms-server-scripts
[Debian GNU/Linux 9.8]: https://www.debian.org/
[contrib]: https://wiki.debian.org/SourcesList#Component
[backports]: https://backports.debian.org/
[llvmpipe]: https://www.mesa3d.org/llvmpipe.html
[patched version of wine]: https://github.com/lukrop/wine/tree/bms-nodraw
[SteamCmd]: https://developer.valvesoftware.com/wiki/SteamCMD#Windows