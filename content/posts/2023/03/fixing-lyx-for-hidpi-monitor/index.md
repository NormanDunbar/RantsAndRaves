---
title: "Fixing Lyx for HiDPI Monitor"
date: "2023-03-20T13:53:00Z"
categories: 
  - "blog"
  - "qt"
  - "lyx"
  - "personal"
  - "hidpi"
---

My new laptop has a 4K monitor and my tool of choice for writing my Arduino books, [Lyx](https://www.lyx.org/ "Lyx website"), always displayed the toolbar icons as "*so very tiny, it is hard to see the damned things!*"

After much experimenting with the monitor's scaling, fractional scaling and such like, I found `QT_AUTO_SCREEN_SCALE_FACTOR` which should fix things.

I tested:

```bash
export QT_AUTO_SCREEN_SCALE_FACTOR=1
lyx &
```

and yes, the icons were now usable. Hooray. I just need to make this permanent.

* There is a QT Configuration application installed, but that doesn't have any way, that is obvious, on setting environment variables. A failure!

* I added the export of `QT_AUTO_SCREEN_SCALE_FACTOR` to my own `.bashrc` startup script, which worked as long as I executed Lyx via the command line, but not from the menu. Another partial failure.

* Finally, I added a file named `QT5_HIDPI_SETTING.sh` to `/etc/profile.d`, as root, and edited the contents thusly:

```bash
export QT_AUTO_SCREEN_SCALE_FACTOR=1
```

On a reboot, and obviously, *afer* deleting the setting from my own `.bashrc` file, Running Lyx from the menu gave me the desired, usable, toolbar icons and buttons.


Cheers,
Norm.



