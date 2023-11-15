---
title: "HPLIP Stops Working After Linux Mint 17.2 Upgrade"
date: "2015-08-26"
categories: 
  - "linux"
---

I was able to use HPLIP with Mint 17.1 but when I upgraded to 17.2, I had the problem where attempting to run the utility caused nothing at all to happen. The solution:

```bash
sudo apt-get install hplip-gui
```

Works perfectly for me now.
