---
title: "Three Useful Laptop Scripts"
subtitle: ""
date: 2020-09-20T20:44:14+03:00
draft: true
description: "Some scripts to increase comfort using Linux with a tiling manager."

tags: [scripts, i3]
categories: [sharing]

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: ""
featuredImagePreview: ""

toc:
  enable: true
math:
  enable: true
lightgallery: false
license: ""
---

In this post, I want to share some of the scripts I once wrote for myself after installing Linux on my notebook.  
My configuration on it is [Ubuntu][ubuntu] + [i3wm][i3] + [Polybar][polybar].<!--more-->

All the presented scripts should work fine with POSIX-compliant shells as long as their listed dependencies work.

By the way, the process of preparing this post forced me to refactor and improve a bit the scripts I haven't touched quite a long time. This is where the advantages of open-source come in. :smirk:

## Low battery warning

An annoying problem I was facing is that every time I hadn't noticed how the notebook had been running out of battery and then it shut down.

Making a simple notification could be the solution to this problem. However, it has not to be annoying.
More specifically:

- The system shouldn't notify me too often
- A notification shouldn't appear under any conditions if the battery's charging

---

Script:

```shell
#!/bin/sh

export DISPLAY=:0

get_battery_level() {
    level=$(acpi -b | cut -d',' -f2 | xargs)
    level=${level%'%'}
    echo $level
}

is_charging() {
    status=$(acpi -b)
    [ "$status" = "${status%'Discharging'*}" ]
}

# Notify if the battery level is less than some value
# (default 10%)
BATTERY_THRESHOLD=${1:-10}
# Make some pause between notifications
# (default 20 mins)
NOTIFY_DELAY=${2:-1200}

CACHE_FILE="/tmp/tmp.battery_notify_timestamp"

if [ -e $CACHE_FILE ]; then
  latest_notify_time=$(cat $CACHE_FILE)
fi

cur_battery_level=$(get_battery_level)
timestamp=$(date +'%s')

echo $cur_battery_level
echo $latest_notify_time

if (! is_charging) && [ \
     $cur_battery_level -lt $BATTERY_THRESHOLD -a \
       \( -z "$latest_notify_time" -o \
          $timestamp -gt $((latest_notify_time+NOTIFY_DELAY)) \
       \) \
   ]; then
    notify-send -u critical "Battery is low"
    echo $timestamp > $CACHE_FILE
else
    if [ -z $latest_notify_time ]; then
      latest_notify_time=$timestamp
    fi
    echo $latest_notify_time > $CACHE_FILE
fi
```

Dependencies:

- [ACPI][acpi]
- [Notify-send][libnotify]

---

To periodically run it I used crontab:

`* * * * * ~/scripts/battery_notification.sh >/dev/null 2>&1`

## Soft altering of screen brightness

When I made a simple script to change screen brightness by some constant value, I noticed that the lowest possible nonzero brightness wasn’t low/flexible enough in a dark environment.

Thus I needed some strategy of accounting low brightness level values. I decided to use a multiplier strategy if the additive one doesn’t apply to the current brightness level.

---

Script:

```shell
#!/bin/sh

if_expr() {
    [ ! $(echo "$1" | bc) -eq 0 ]
}

soft_alter() {
    CUR_LEVEL=$(light)
    MIN_STEP=0.01

    CMD=$1
    DELTA=$2
    SCALE=$3

    case $CMD in
        inc)
            EXPR=$( (if_expr $CUR_LEVEL'<'$DELTA) &&
                       ((if_expr $CUR_LEVEL+$MIN_STEP'>'$CUR_LEVEL*$SCALE) &&
                           echo $CUR_LEVEL+$MIN_STEP ||
                           echo $CUR_LEVEL*$SCALE
                       ) ||
                       echo $CUR_LEVEL+$DELTA
                 ) ;;
        dec)
            EXPR=$( (if_expr $CUR_LEVEL'>='$DELTA*$SCALE) &&
                       ((if_expr $CUR_LEVEL'<='$MIN_STEP) &&
                           echo $CUR_LEVEL ||
                           echo $CUR_LEVEL-$DELTA
                       ) ||
                       echo $CUR_LEVEL/$SCALE
                 ) ;;
    esac
    NEW_LEVEL=$(printf "%.2f\n" $(awk "BEGIN { print "$EXPR" }"))
    echo $NEW_LEVEL
    return 0
}

if [ "$#" -ge 1 -a \
     "$#" -le 3 -a \
     \( "$1" = "inc" -o \
        "$1" = "dec" \) ];
then
    CMD=$1
    DELTA=${2:-5}
    SCALE=${3:-1.5}
    light -S $(soft_alter $CMD $DELTA $SCALE)
else
    echo "Usage:"
    echo "  sh $(basename "$0") [CMD] [DELTA:5] [SCALE:1.5]"
    echo "Parameters:"
    echo "  CMD    {up, down}"
    echo "  DELTA  Brightness alter value"
    echo "  SCALE  Brightness scale coefficient for soft altering"
    echo "         in case of small brightness"
fi
```

Dependencies:

- [bc][bc]
- [Awk][awk]
- [Light][light]

---

On my notebook, I've bound this script to the corresponding keys in i3wm config:

```
bindsym XF86MonBrightnessUp exec --no-startup-id ~/scripts/change_brightness.sh inc
bindsym XF86MonBrightnessDown exec --no-startup-id ~/scripts/change_brightness.sh dec
```

## Charging indicator in status bar

The last one is a tiny script for a status bar that indicates whether the AC adapter is plugged in and charging the battery.

---

Script:

```shell
#!/bin/sh

status=$(acpi -b)
if [ "$status" != "${status%'Discharging'*}" ]; then
    echo ''
else
    echo '⚡'
fi
```

Dependencies:

- [ACPI][acpi]

---

In order to embed it in [Polybar][polybar] I added a new module named `battery-status` to `modules-right` and initialized it:

```
[module/battery-status]
type = custom/script
exec = ~/.config/polybar/get_battery_status.sh
interval = 5
```

[bc]: https://git.yzena.com/gavin/bc
[awk]: https://github.com/onetrueawk/awk
[light]: https://github.com/haikarainen/light
[i3]: https://i3wm.org/
[ubuntu]: https://ubuntu.com/
[polybar]: https://github.com/polybar/polybar
[acpi]: https://sourceforge.net/projects/acpiclient/
[libnotify]: https://gitlab.gnome.org/GNOME/libnotify
