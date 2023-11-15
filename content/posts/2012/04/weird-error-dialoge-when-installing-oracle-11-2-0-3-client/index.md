---
title: "Weird Error Dialogue when Installing Oracle 11.2.0.3 Client?"
date: "2012-04-24"
categories: 
  - "oracle"
---

When installing the Pro*C compiler stuff (technical term) from the 11.2.0.3 client install disc (it's in zipfile 4 or 7 in case you need to know!) I hit a strange error almost immediately the installer started the GUI.

I don't have a screen dump, but the error message was the following:

`vbOEL.dunbar-it.co.uk:vbOEL.dunbar-it.co.uk`

In other words, my fully qualified Linux x86-64bit server name, `vbOEL.dunbar-it.co.uk`, twice. Interesting. 

No buttons were present to allow me to carry on, abort etc. Just an OK button which is not much help as it bales out of the installer when clicked.

Googling and DuckDuckGoing (will that ever be a verb?) found me nothing.

As it turned out, I was using the wrong CD as I should have been putting in the 11.2.0.2 version. Once I mounted the correct CD image, I started again. This time I got an error dialoge with a bit more information:

[INS-06101] IP address of localhost could not be determined.
Are you sure you wish to continue?

And this time, I have buttons for YES and NO. I clicked YES and all was well.

It looks like 11.2.0.3 is a tad flaky in the error message/dialogue department.

Cheers, Norm.
