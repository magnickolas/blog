---
title: "Installing Ubuntu 19.10 on Zenbook"
date: 2020-05-16T02:24:08+03:00
draft: true
---

## Wi-Fi
```
sudo apt install network-manager
```

![](https://oplachko.com:500//uploads/upload_d0d16f053a31874363f229b3f4ce99a6.png)

![](https://oplachko.com:500//uploads/upload_d0d16f053a31874363f229b3f4ce99a6.png "Title")

## Window manager
```
sudo apt install i3-wm xinit dmenu
echo "exec i3" > ~/.xinitrc
startx
```


## Bar (Polybar)
```
sudo apt install libiw-dev libpulse-dev libxcb-composite0-dev libxcb1-dev libxcb-ewmh-dev libjsoncpp-dev acpi
git clone https://github.com/polybar/polybar
cd polybar
```


## Get battery status script
```
if [ "$(acpi -b|cut -d':' -f2|cut -d',' -f1|xargs)" = 'Discharging' ]; then 
echo '—'; else echo '⚡'; fi
```


## Screen brightness
```
sudo apt install autoreconf
git clone https://github.com/haikarainen/light
cd light
./autogen.sh 
./configure
make
sudo make install
```

Script for smart brightness altering: @soft_brightness_altering


## Nvidia
```
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt install ubuntu-drivers-common
sudo ubuntu-drivers autoinstall
sudo apt install nvidia-prime
```


## Sound
```
sudo apt install alsa-utils
sudo apt install pulseaudio
sudo apt install pavucontrol
```


## Terminal (Alacritty)
```
sudo apt install cmake pkg-config libfreetype6-dev libfontconfig1-dev libxcb-xfixes0-dev python3 libexpat1-dev  libfontconfig1-dev
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
VERSION='v0.4.2-rc3'
wget https://github.com/alacritty/alacritty/archive/$VERSION.tar.gz
tar -xvpf $VERSION.tar.gz
cd alacritty-$VERSION
cargo deb --install -p alacritty
```


## Terminess font
```
git clone https://github.com/powerline/fonts
cd fonts
./install.sh
mkdir -p .config/fontconfig/conf.d
cd .config/fontconfig/conf.d
wget https://raw.githubusercontent.com/powerline/fonts/master/fontconfig/50-enable-terminess-powerline.conf
fc-cache -vf
```


## Touchpad
```
sudo apt install xserver-xorg-input-synaptics

/etc/X11/xorg.conf.d/30-touchpad.conf

Section "InputClass"
       Identifier "touchpad catchall"
       Driver "synaptics"
       MatchIsTouchpad "on"
       MatchDevicePath "/dev/input/event*"
           Option "TapButton1" "1"
           Option "TapButton2" "3"
           Option "TapButton3" "2"
EndSection
~/.config/i3/toggletouchpad.sh

#!/bin/bash
if synclient -l | grep "TouchpadOff .*=.*0" ; then
    synclient TouchpadOff=1 ;
else
    synclient TouchpadOff=0 ;
fi
```


### Fallback using libinput
```
Section "InputClass"   
  Identifier "touchpad"  
  Driver "libinput"  
  MatchIsTouchpad "on"  
  Option "Tapping" "on"  
EndSection
```


## VSYNC
```
sudo apt install meson ninja
git clone https://github.com/yshui/picom
cd picom/
git submodule update --init --recursive
meson --buildtype=release . build
ninja -C build
ninja -C build install
picom --backend glx --vsync
```


## Bluetooth
If bluetoothctl's scan says "No default controller available"
```
sudo rmmod btusb
sudo modprobe btusb
```


## Suspend on lid close

### /etc/default/grub
```
GRUB_CMDLINE_LINUX_DEFAULT="mem_sleep_default=deep cloud-init=disable"
```


## Multimedia keys (i3 config)
```
pactl list short sinks

bindsym XF86MonBrightnessUp exec light -A 5
bindsym XF86MonBrightnessDown exec light -U 5
bindsym XF86AudioRaiseVolume exec --no-startup-id pactl set-sink-volume 1 +5% #increase sound volume
bindsym XF86AudioLowerVolume exec --no-startup-id pactl set-sink-volume 1 -5% #decrease sound volume
bindsym XF86AudioMute exec --no-startup-id pactl set-sink-mute 1 toggle # mute sound
bindsym XF86TouchpadToggle exec ~/.config/i3/toggletouchpad.sh
bindsym XF86Calculator exec gnome-calculator
```


## Low battery notification

@low_battery_notification


## SSH server

```
sudo apt install openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
hostname -I
```

## Installing open source packages

Use

```
sudo checkinstall
```

instead of

```
sudo make install
```

to be able to manage the package later.


###### tags: `os` `scripts`

