---
title: "Does Your Volume Control Freeze on KDE?"
date: "2011-11-16"
categories: 
  - "linux"
---

I've been using KDE4 on OpenSuse (11.4 currently) for some time and never had any problems. After a recent patch fix to the system, the volume control buttons on my Dell Vostro stopped responding. Normally pressing volume up or volume down worked instantaneously. Stop and go still worked fine, but not volume.

What would happen is that the on screen indicator would appear after about 45 seconds or so - but the volume wouldn't change until it did, then, it would go to maximum or minimum - depending on which button I'd been hammering for 45 seconds! :-)

A similar thing would happen if I tried to use the Volume control applet in the task bar. Hmmm.

The solution is fairly simple:

- Stop the kmix process by right-clicking it in the task bar, and choosing the _quit_ option.
- `cd ~/.kde4/share/apps `
- `rm -R kmix*`
- Restart Kmix by pressing ALT-F2 and typing `kmix` into the dialogue that appears. After a few seconds, you get your volume control icon back in the task bar and everything works fine.

The problem appears to have been caused by a configuration change which left problems when old config was found in the assorted config files - but don't quote me on that!

Cheers.  
Norm
