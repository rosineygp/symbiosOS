# symbiosⵀS <!-- omit in toc -->

> Symbiosis Operational System

Share x11, camera and pulseaudio over LXC/LXD containers.

**main features**

- near native graphics acceleration
- nice audio quality
- user profile isolation
- fast startup
- instances can use the same hardware resources (GPU, camera... etc)
- low resources usage

**reason**

- isolate personal and work accounts
- fast switch between accounts (one per workspace)
- performance for video conference
- multiple VPN connections 

# Table of contents <!-- omit in toc -->

- [Host configuration](#host-configuration)
  - [Install dependencies](#install-dependencies)
  - [Configure pulseaudio server](#configure-pulseaudio-server)
- [Profile](#profile)
  - [Create blank profile](#create-blank-profile)
  - [Edit x11 profile](#edit-x11-profile)
    - [Nvidia GPU](#nvidia-gpu)
    - [Intel iGPU](#intel-igpu)
  - [Profile advices](#profile-advices)
- [Guest configuration](#guest-configuration)
  - [Deploy a new guest](#deploy-a-new-guest)
  - [Guest login](#guest-login)
  - [Checking guest](#checking-guest)
    - [Video GLX](#video-glx)
    - [Pulseaudio](#pulseaudio)
    - [Camera](#camera)
- [Terminal Launcher](#terminal-launcher)
  - [Install](#install)
  - [Usage](#usage)

## Host configuration

### Install dependencies

https://linuxcontainers.org/lxd/getting-started-cli/#installation

### Configure pulseaudio server

Append following line at `/etc/pulse/default.pa` and enable tcp module for pulseaudio

```txt
load-module module-native-protocol-tcp auth-ip-acl=127.0.0.1
```

Stop pulseaudio server

```shell
pulseaudio -k
```

> Mapping driver using `unix://` is flickering

## Profile

### Create blank profile

```shell
lxc profile create x11
```

### Edit x11 profile

```shell
lxc profile edit x11
```

Paste the following `yaml`

#### Nvidia GPU

```yaml
config:
  environment.DISPLAY: :0
  environment.PULSE_SERVER: tcp:127.0.0.1:4713
  nvidia.driver.capabilities: all
  nvidia.runtime: "true"
  user.user-data: |
    #cloud-config
    runcmd:
      - 'sed -i "s/; enable-shm = yes/enable-shm = no/g" /etc/pulse/client.conf'
      - 'echo export PULSE_SERVER=tcp:127.0.0.1:4713 | tee --append /home/ubuntu/.profile'
    packages:
      - x11-apps
      - mesa-utils
      - pulseaudio
      - v4l-utils
description: GUI LXD profile
devices:
  PASocket:
    bind: container
    connect: tcp:127.0.0.1:4713
    listen: tcp:127.0.0.1:4713
    type: proxy
  X0:
    bind: container
    connect: unix:@/tmp/.X11-unix/X1
    listen: unix:@/tmp/.X11-unix/X0
    security.gid: "1000"
    security.uid: "1000"
    type: proxy
  mygpu:
    type: gpu
  video0:
    gid: "1000"
    path: /dev/video0
    type: unix-char
name: x11
used_by: []
```

#### Intel iGPU

```yaml
config:
  environment.DISPLAY: :1
  environment.PULSE_SERVER: tcp:127.0.0.1:4713
  raw.idmap: both 1000 1000
  user.user-data: |
    #cloud-config
    runcmd:
      - 'sed -i "s/; enable-shm = yes/enable-shm = no/g" /etc/pulse/client.conf'
      - 'echo export PULSE_SERVER=tcp:127.0.0.1:4713 | tee --append /home/ubuntu/.profile'
    packages:
      - x11-apps
      - mesa-utils
      - pulseaudio
      - v4l-utils
description: GUI LXD profile
devices:
  PASocket:
    bind: container
    connect: tcp:127.0.0.1:4713
    listen: tcp:127.0.0.1:4713
    type: proxy
  X1:
    bind: container
    connect: unix:@/tmp/.X11-unix/X1
    listen: unix:@/tmp/.X11-unix/X1
    security.gid: "1000"
    security.uid: "1000"
    type: proxy
  mygpu:
    gid: "1000"
    type: gpu
  video0:
    gid: "1000"
    path: /dev/video1
    type: unix-char
name: x11
used_by: []
```

### Profile advices

- device `video0` is optional
- pay attention about `uid` and `gid`, it will change as you current user
- `cloud-config` cannot work, it depends of distributution support
- check your current `$DISPLAY` and change if `/tmp/.X11-unix/X1` not work

## Guest configuration

### Deploy a new guest

```shell
lxc launch --profile default --profile x11 ubuntu:20.04 guest01
```

### Guest login

```shell
lxc exec guest01 -- sudo --user ubuntu --login
```

<pre>
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

ubuntu@guest01:~$
</pre>

If you want distinguished colors for different guests `gnome-terminal`.

https://mayccoll.github.io/Gogh/

```shell
gnome-terminal --profile="Jackie Brown" -- bash -c "lxc exec guest01 -- sudo --user ubuntu --login"
```

### Checking guest

#### Video GLX

```shell
glxinfo -B
```
<pre>
direct rendering: Yes
Memory info (GL_NVX_gpu_memory_info):
    Dedicated video memory: 6144 MB
    Total available memory: 6144 MB
    Currently available dedicated video memory: 5228 MB
OpenGL vendor string: NVIDIA Corporation
OpenGL renderer string: GeForce GTX 1060 6GB/PCIe/SSE2
OpenGL core profile version string: 4.6.0 NVIDIA 450.119.03
OpenGL core profile shading language version string: 4.60 NVIDIA
OpenGL core profile context flags: (none)
OpenGL core profile profile mask: core profile

OpenGL version string: 4.6.0 NVIDIA 450.119.03
OpenGL shading language version string: 4.60 NVIDIA
OpenGL context flags: (none)
OpenGL profile mask: (none)

OpenGL ES profile version string: OpenGL ES 3.2 NVIDIA 450.119.03
OpenGL ES profile shading language version string: OpenGL ES GLSL ES 3.20
</pre>

#### Pulseaudio

```shell
pactl info
```

<pre>
Server String: tcp:127.0.0.1:4713
Library Protocol Version: 33
Server Protocol Version: 33
Is Local: no
Client Index: 14
Tile Size: 65472
User Name: rosiney
Host Name: desk
Server Name: pulseaudio
Server Version: 13.99.1
Default Sample Specification: s16le 2ch 44100Hz
Default Channel Map: front-left,front-right
Default Sink: alsa_output.usb-C-Media_Electronics_Inc._USB_Audio_Device-00.analog-stereo
Default Source: alsa_input.usb-C-Media_Electronics_Inc._USB_Audio_Device-00.mono-fallback
Cookie: 9537:bf95
</pre>

#### Camera

```shell
v4l2-ctl --list-devices
```

<pre>
Iriun Webcam (platform:v4l2loopback-000):
        /dev/video0
</pre>

> Sometimes the commands above not working because the `cloud-config` steps not finished, if commands not working try to install manually `x11-apps mesa-utils pulseaudio v4l-utils`

## Terminal Launcher

### Install

Just a small script to call GUI apps without lock terminal or print output

```shell
mkdir -p ~/.local/bin/

cat << EOF > ~/.local/bin/tl
#!/bin/bash
nohup "\$@" &>/dev/null & disown %%
EOF

chmod +x ~/.local/bin/tl
```

Configure autocomplete for `tl`

```shell
cat << EOF >> ~/.bashrc
_tl_completions() {
  COMPREPLY=(\$(compgen -c "\${COMP_WORDS[1]}"))
}

complete -F _tl_completions tl
EOF
```

### Usage

```shell
tl google-chrome
```

> needs to logout/login to reload profile
