# MetaTrader 4 Terminal in wine

This image has all dependencies which required to run MetaTrader 4 Terminal with Myfxbook EA in wine, but can be used for some other EAs/scripts.

## Prepare distribution with MetaTrader 4

1. Install appropriate (branded) MT4 terminal locally (yep, you can do it on Windows) and close it if opened after installation
1. Run the terminal with [`/portable`](https://www.metatrader4.com/en/trading-platform/help/userguide/start_comm) parameter to create a structure of Data directory inside the directory with the terminal
1. Close the terminal
1. Delete all temporary and unrequired files from the directory with the terminal (`metaeditor.exe`, `terminal.ico`, `Sounds` dir, log files, etc)
1. If you required for Myfxbook EA then
    1. [Install the EA](https://www.myfxbook.com/help/connect-metatrader-ea)
    1. Edit a file `myfxbook_ea-example.ini` (only Login, Password, Server fields usually) and save it in the root of the directory of the terminal by name `myfxbook_ea.ini`
    1. Edit a file `Myfxbook-example.set` and save it in directory `MQL4/Presets/` of the terminal by the name `Myfxbook.set`

## Configure the host system

The cheapest droplet of DigitalOcean ($5/mon) is enough for running the container. If you haven't account in DigitalOcean you can register by my referral [link](https://m.do.co/c/8a6e11b01bba) to get $10 credit.

A user in the image who runs MetaTrader app has UID 1000 (you can change it by `--build-arg USER_ID=NNNN` of `docker build`), so you may need to create a user in the host OS to map with it. Let's name it "monitor" for example. The user should be in docker group to run the container.

```bash
useradd -u 1000 -s /bin/bash -mU monitor
adduser monitor docker
```

## Run the container

Login by user "monitor" and type:

```bash
docker run -d --rm \
    --cap-add=SYS_PTRACE \
    -v /path/to/prepared/mt4/distro:/home/winer/.wine/drive_c/mt4 \
    nevmerzhitsky/mt4-for-myfxbook-ea
```

Or do it by root but add `--user 1000` parameter to command.

Without `--cap-add=SYS_PTRACE` parameter you will can't attach any EA to chart and run any script. (I think this is due to checking for any debugger attached to the terminal - protection from sniffing by MetaQuotes.) If your EA/script doesn't work even with `--cap-add=SYS_PTRACE` then replace it with `--privileged` parameter and try again. But this decrease security thus does it at your own risk! Instead of this, you can investigate which `--cap-add` values will fix you EA/script.

You can use `-it` parameters instead of `-d` to move the main process to the foreground. Worth noting, the main process of the container (script `run_mt.sh`) will not catch `Ctrl+C` properly as it does for SIGTERM from `docker stop`. This may lead to abnormal termination of the terminal.

A base image is Ubuntu, therefore if you want to debug the container then add `--entry-point bash` parameter to the `docker run` command.

## Extending the image

You can make your own `Dockerfile` inherited from this image and copy a particular distribution of MetaTrader 4 Terminal to an image on build phase. For this task, env variables `$USER` and `$MT4DIR` are acceptable.

You can make an archive with the content from section "Prepare distribution with MetaTrader 4" by an appropriate tool. E.g. `cd mt4-distro; tar cfj ../mt4.tar.bz2 *`. Make sure that no root folder exists in the archive. Then you can extract the archive into the image by instruction `ADD mt4.tar.bz2 $MT4DIR`.

## Troubleshooting

If you see this error in the container logs:

```
X Error of failed request:  BadAccess (attempt to access private resource denied)
  Major opcode of failed request:  129 (MIT-SHM)
  Minor opcode of failed request:  3 (X_ShmPutImage)
  Serial number of failed request:  458
  Current serial number in output stream:  458
```

Then try to add `--ipc=host` parameter to the `docker run` command due to [a comment](https://github.com/osrf/docker_images/issues/21#issuecomment-239334515). But this decrease security thus does it at your own risk!
