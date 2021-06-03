# symbiosOS

Share x11, camera and pulseaudio over LXC/LXD containers.

## Host configuration

### Install dependencies

https://linuxcontainers.org/lxd/getting-started-cli/#installation

### Configure pulseaudio server

Append tcp module for pulseaudio at `/etc/pulse/default.pa`

```txt
load-module module-native-protocol-tcp auth-ip-acl=127.0.0.1
```

After it execute:

```shell
pulseaudio -k
```

> Mapping driver using `unix://` is flickering.

## Profile

### Create blank profile

```shell
lxc create profile x11
```

### Nvidia Profile

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

> device `video0` is optional.
>
> Be careful about `uid` and `gid`.

## Guest configuration

### Deploy a new Guest

```shell
$ lxc launch --profile default --profile x11 ubuntu:20.04 guest01
```

### Guest login

```shell
$ lxc exec guest01 -- sudo --user ubuntu --login

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

ubuntu@guest01:~$
```

If you want distinguished colors for different guests `gnome-terminal`.

https://mayccoll.github.io/Gogh/

```shell
$ gnome-terminal --profile="Jackie Brown" -- bash -c "lxc exec guest01 -- sudo --user ubuntu --login"
```

### Checking guest

Sometimes the commands not working because the `cloud-config` steps not finished.

- Video GLX

```shell
$ glxinfo -B

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
```

- Pulseaudio

```shell
$ pactl info

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
```

- Camera

```shell
$ v4l2-ctl --list-devices

Iriun Webcam (platform:v4l2loopback-000):
        /dev/video0
```

## Terminal Launcher

Call gui apps in terminal without lock or output.

```shell
mkdir -p ~/.local/bin/

cat << EOF > ~/.local/bin/tl
#!/bin/bash
nohup "\$@" &>/dev/null & disown %%
EOF

chmod +x ~/.local/bin/tl
```

> needs to logout/login to reload profile

Usage

```shell
$ tl google-chrome
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

> needs to logout/login to reload profile