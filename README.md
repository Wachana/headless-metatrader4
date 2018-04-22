# MetaTrader 4 Terminal in wine

This image has all dependencies which required to run MetaTrader 4 Terminal with Myfxbook EA in wine, but can be used for some other EAs/scripts.

## Prepare distribution with MetaTrader 4

1. Install appropriate (branded) MT4 terminal locally and close it if opened after installation
1. Run it with [`/portable`](https://www.metatrader4.com/en/trading-platform/help/userguide/start_comm) argument to create structure of Data directory inside the directory with the terminal
1. Close the terminal
1. Delete all temporary and unrequired files from the directory with the terminal (`metaeditor.exe`, `terminal.ico`, `Sounds` dir, log files, etc)
1. Edit a file `myfxbook_ea-example.ini` (only Login, Password, Server fields usually) and save it in the root of the directory of the terminal by name `myfxbook_ea.ini`
1. Edit a file `Myfxbook-example.set` and save it in directory `MQL4/Presets/` of the terminal by name `Myfxbook.set`
1. If you plan extend the `Dockerfile` and add the terminal to the image on build step, then make `mt4.tar.bz2` file by any appropriate tool. E.g. `tar cfj ../mt4.tar.bz2 *`. Make sure that no root folder is exist in the archive.

## Setup host system

Unfortunately, MetaTrader 4 can't work without real GUI desktop, thus you should install an desktop application in the host OS. Let's do it with Xfce and TightVNC. And fortunately, cheapest droplet of DigitalOcean ($5/mon) is enough for running the task. If you haven't account in DigitalOcean you can register by my referral [link](https://m.do.co/c/8a6e11b01bba) to get $10 credit.

```bash
sudo apt-get update
sudo apt-get install tightvncserver xfce4 xfce4-goodies
vncserver :1
```

Set a password for connection and test the connection to `<host>:5901` from your machine by a VNC client. These steps inspired by [an article](https://medium.com/google-cloud/linux-gui-on-the-google-cloud-platform-800719ab27c5), if you catch any issue try to read the article first.

Stop the vncserver because we will create a service soon: `vncserver -kill :1`.

Next, create an user to run the container, "monitor" for example. The user should have UID 1000 because it will map to the container where UID 1000 is used for running MetaTrader app. The user should be sudoer to start privileged container.

```bash
useradd -u 1000 -s /bin/bash -mU monitor
adduser monitor sudo
```

Then set a password for the user to access `sudo` command: `passwd monitor`.

Let's create a service to start VNC server automatically after reboot. Copy the following into `/etc/init.d/vncserver`:

```bash
#!/bin/sh -e
### BEGIN INIT INFO
# Provides:          vncserver
# Required-Start:    networking
# Required-Stop:
# Default-Start:     3 4 5
# Default-Stop:      0 6
### END INIT INFO

# The username:group that will run VNC
export USER="root"

# The display that VNC will use
DISPLAY=":1"
# The Desktop geometry to use.
GEOMETRY="1024x768"

OPTIONS="-geometry ${GEOMETRY} ${DISPLAY}"

. /lib/lsb/init-functions

case "$1" in
start)
    log_action_begin_msg "Starting vncserver for user '${USER}' on ${DISPLAY}"
    su ${USER} -c "/usr/bin/vncserver ${OPTIONS}"
    # Give access to X Server for other users
    su ${USER} -c "DISPLAY=${DISPLAY} xhost +localhost"
    ;;
stop)
    log_action_begin_msg "Stoping vncserver for user '${USER}' on ${DISPLAY}"
    su ${USER} -c "/usr/bin/vncserver -kill ${DISPLAY}"
    ;;
restart)
    $0 stop
    $0 start
    ;;
esac

exit 0
```

Make the script executable with `chmod +x /etc/init.d/vncserver`. Then run `update-rc.d vncserver defaults`. And finally `/etc/init.d/vncserver start`. These steps inspired by [an article](http://www.abdevelopment.ca/blog/start-vnc-server-ubuntu-boot/), if you catch any issue try to read the article first.

## Run the container

Login by root and build the container:

```bash
docker build -t myfxbook .
```

Login by user "monitor" and type:

```bash
sudo docker run -it --rm \
    --privileged \
    -u $UID \
    -e :1 \
    -v /tmp/.X11-unix:/tmp/.X11-unix:ro \
    -v /path/to/prepared/mt4/distro:/home/winer/.wine/drive_c/mt4 \
    myfxbook
```

You can use `-d` parameter instead of `-it` to move the process to background.

Base image is Ubuntu, therefore if you want to debug the container then add `--entry-point bash` parameter to the `docker run` command.

## Extend image

You can make your own `Dockerfile` inherited from this image and copy a particular distribution of MetaTrader 4 Terminal to image on build phase. For this task env variables `$USER` and `$MT4DIR` are acceptable.
