# Sanzu

## Overview

Sanzu is a graphical remote desktop solution. It is composed of:

- a server running on Unix or Windows which can stream a X11 or a Windows GUI environment (for now the Unix version is more advanced)
- a client running on Unix or Windows which can read this stream and interact with the GUI environment

It uses modern video codecs like h264/h265 to offer a good image quality and limit its bandwidth consumption. Video compression is done through FFmpeg which allows the use of graphic cards or full featured CPU to achieve fast video compression at low latency. It also allows the use of yuv420 or yuv44 for better graphical details.

You can find more information on the architecture in `doc/architecture.md`.

## Features
- Audio
- Copy/Paste
- TLS
- Kerberos
- PAM
- One way clipboard
- Seamless mode

## Options
```
./sanzu_server --help
./sanzu_client --help
./sanzu_proxy --help
```


## Use case 1: simple client/server
- You'll need to create a X server (for example on number 100). On the server:
```
  Xvfb :100
```
  You can also customize the screen resolution (here, 1920x1080 in 24 bit/pixel):
```
  Xvfb :100 -screen 0 1920x1080x24
```
- Launch the server:
```
  DISPLAY=:100 sanzu_server --config sanzu.toml
```
  or, if the server implements GPU encoding (h264_nvenc here):
```
  DISPLAY=:100 sanzu_server --config sanzu.toml --encoder h264_nvenc
```
- On the server side, launch a window manager through X server
```
  DISPLAY=:100 xfwm4
```
- On the client side, launch the client (./sanzu_client <server ip> <server port>):
```
  ./sanzu_client 192.168.0.1 1122
```
By default, sound is disabled. To enable it, server and client should be launched with option "--audio".

## Use case 2: displaying graphics from local vm, using guest/host shared memory, using direct raw pixels
Here, we will export the display of a linux guest to the host using shm between host and guest to extract pixels. We will also use vsock as sanzu client/server connection
- To do so, you have to launch your preferred vm using additional qemu options:
  - `-device vhost-vsock-pci,guest-cid=1337` : this will assign the vsock cid 1337 to the guest vm
  - `-device ivshmem-plain,memdev=guestmem`: add a shm 'guestmem' between guest and host
  - `-object memory-backend-file,size=64M,share=on,mem-path=/dev/shm/guestmem,id=guestmem`: link 'guestmem' to the `/dev/shm/guestmem` file

Once the VM is running, you can execute the `sanzu_server` in the vm using:
```
sanzu_server --vsock --config 4294967295 --config sanzu.toml --export-video-pci --encoder null --audio --seamless
```
This makes `sanzu_server` listen on the vsock default address, on the default port (1122). It exports raw pixels to the host through the external pci ram backed by the shmem, without compression.
We now run the `sanzu_client` on the host side:
```
sanzu_client 1337 1122 --vsock  --extern-img-source /dev/shm/guestmem --audio
```

This makes `sanzu_client` connect to the vsock cid 1337 (port 1122), which corresponds to our virtual machine. We use the file `/dev/shm/guestmem` to retrieve raw pixels.

## Use case 3: displaying graphics from a remote vm, using guest/host shared memory and sanzu_proxy to use hypervisor video compression capabilities
In this example, we use the same mechanism to extract raw pixels (shm) but we will encode those pixels using `sanzu_proxy` running on the hypervisor. This proxy exposes the same sanzu protocol as the `sanzu_server` (using video compression)
- Reproduce the virtual machine configuration steps as in the use case 2.
- Execute the `sanzu_proxy` on the hypervisor using:
```
sanzu_proxy 1337 1122 --vsock  --extern-img-source /dev/shm/guestmem --audio --encoder h264_nvenc  --listen-address  192.168.1.1 --listening-port 1122
```

You can now run the `sanzu_client` from a remote computer using:
```
sanzu_client  192.168.1.1 1122  --audio
```

## Replacement of ssh -Y
If you have a server, let's say Rochefort, which runs a X server on the display :1234, you can access it with:
```
sanzu_client 127.0.0.1 1337 --proxycommand "ssh rochefort DISPLAY=:1234 sanzu_server --stdio"
```
In this case the connection is done throught ssh so ip and port are useless. The server must have the configuration file "/etc/sanzu.toml" present or specified with the "--config" flag.


## Compilation
### Debian
Packages required:

```
apt install build-essential cargo libasound2-dev ffmpeg libavutil-dev libclang-dev \
    libkrb5-dev libx264-dev libx264-dev libxcb-render0-dev libxcb-shape0-dev \
    libxcb-xfixes0-dev libxdamage-dev libxext-dev x264 xcb libavformat-dev \
    libavfilter-dev libavdevice-dev libpam0g-dev libdbus-1-dev
```

### Alpine
Required packages:

```
apk add cmake g++ ffmpeg protoc ffmpeg-dev x264 \
    xcb-util linux-pam-dev curl bash \
    krb5-dev alsa-lib-dev opus-dev clang-dev libx11-dev
```

### Archlinux
Required packages:

```
pacman -S rust libxtst libxext ffmpeg libxdamage libxfixes pkgconf clang \
    gcc llvm
```

### Redhat based
Required packages:

```
yum install alsa-lib-devel ffmpeg-devel compat-libxcb gcc-c++ protobuf-compiler \
    libxcb-devel libX11-devel dbus-devel clang-devel krb5-devel opus-devel \
    libavdevice pam-devel
```

### Windows cross compilation from archlinux
You can follow the steps described in the Docker file in `build/Dockerfile-windows`

### Windows binaries compilation from windows
Compilation example is shown in the github action. To reproduce those steps:
- install classic development environment installed on your windows system (visual studio, ...)
- install the Vcpkg package
- prepare vcpkg:
```
vcpkg integrate install
```
- install ffmpeg package
```
vcpkg install ffmpeg:x64-windows
```
- run cargo on the sanzu workspace:
```
cargo build --release -p sanzu --no-default-features
```

## Launching binaries on windows
- Download the ffmpeg binaries library from official repositories
- either from https://github.com/BtbN/FFmpeg-Builds/releases file ffmpeg-n4.4-latest-win64-gpl-shared-4.4.zip
- or from https://www.gyan.dev/ffmpeg/builds/ file ffmpeg-4.4.1-full_build-shared.7z
- decompress
- copy `sanzu_client`or `sanzu_server` in the `bin` sub folder of the ffmpeg decompressed folder
- execute client or server using previously described commands

## Testing with TLS
Generate CA / server / client certificates (for test purposes, you can use gen_ca_and_certs.sh)
Add the following example configuration to sanzu.toml:
```
[tls]
server_name = "localhost"
ca_file = "/home/user/certs/rootCA.crt"
auth_cert = "/home/user/certs/localhost.crt"
auth_key = "/home/user/certs/localhost.key"
# allowed_client_domains = ["domain.local"]
```
Run the server:
```
   sanzu_server --config sanzu.toml --port 1122 --address 127.0.0.1
```

And the client:
```
   sanzu_client 127.0.0.1 1144 --audio --tls-ca ./certs/rootCA.crt --tls-server-name localhost
```


## Server configuration file
### video
#### max_fps
This option limits the maximum FPS outputted by the server. This can be useful to limit video throughput in case of fast client & server.
#### max_stall_img
On the server side, if no motion is detected, the encoder is stopped after `max_stall_img` frames count. As most video encoders are progressive, the encoding of a fixed image will enhance the client quality with time. A too low value will result in premature pause in the video encoding process and the video client will be stalled to a bad image quality. A too high value will make the encoder continually consume graphic resources.
#### ffmpeg_options_cmd
Optional: command line to execute each time an encoder is created. It gives a chance to do some tweaks at run time, for example change the process affinity. The output of the command is passed to FFmpeg encoder options. Thus, the command can also tweak the encoder, for example the physical GPU on which the encoder is executed.
#### control_path
Path of a control socket. This socket reacts to client connection by restarting the current encoder. This is used to hot restart video encoder, for example to do some dynamic graphic cards load balancing.
### audio
#### sample_rate
The default sample rate at which the server will capture the sound.
#### max_buffer_ms
The size (in ms) of the sound buffer. The value must be large enough to overcome network lags and avoid sound shuttering. The value must not be too large to avoid desynchronization between image and sound.
### export_video_pci
Used when server is in virtual machine, to export raw frames through shared memory instead of network. Used when the video encoding is done out of the virtual machine.
### ffmpeg
#### globals
FFmpeg video codec options used for all encoders.
#### codec_name
FFmpeg options used for the libx264 codec_name.
Those options can be retrieved from the FFmpeg command line:
`ffmpeg -h encoder=code_cname`

## Known issues
- If connection is flappy, keyboard events might be sent with a delay. Consequence is identical to sticky keys, with input repetition.
- If you are using an X11 server and you have keyboard layout issues, you might need to explicitly set your keyboard layout by using [setxkbmap](https://linux.die.net/man/1/setxkbmap) on the server.
