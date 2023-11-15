---
title: "Firefox Running Extremely Slowly?"
date: "2013-04-05"
categories: 
  - "linux"
  - "rants-raves"
  - "windows"
---

Firefox starts off ok, but soon starts running slower and slower, until it eventually starts to time out on connecting to some pages. The error messages is "the server took too long to respond" however, Firefox might be telling porkies.

If you attempt to access the same URL in Opera, Chrome or, if you must, Internet Explorer, you may find that it is responding quite happily and speedily, while another try in Firefox takes ages to connect or fail again.

You need to sort out a couple of options:

- Edit->Preferences (on Linux) or Tools->Options (I think, on Windows)
- Click Advanced
- Click General tab
- **Uncheck** _Use hardware acceleration when available_
- Click Network tab
- Click Settings button beside _Configure how Firefox connects to the internet_
- **Check** _No proxy_ but be aware that this option might be restricted if you use Firefox at work.
- Click OK button
- Click Close button

You might need to restart Firefox after this, I didn't on Linux, but try a link without restarting first and see what happens. On my system, it works perfectly quickly again.
