---
title: "Adobe Air Applications Can't Find Passwords etc?"
date: "2011-12-08"
categories: 
  - "linux"
  - "twitter"
---

TweetDeck for Linux, for example. Refused to start on me recently because it couldn't find or access the location where I had saved my encrypted details. The problem is caused by Adobe Air not being able to find a running daemon for Gnome Keyring or KWallet, or a corrupted wallet database.

Full details here [http://kb2.adobe.com/cps/492/cpsid\_49267.html](http://kb2.adobe.com/cps/492/cpsid_49267.html "Adobe  Air Support") - works for me!

In my case, simply restarting KWalletManager worked fine. And restarting TweetDeck of course - it's unable to pick up the fact that the wallet is available.

Cheers, 
Norm.
