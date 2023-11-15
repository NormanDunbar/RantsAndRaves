---
title: "Internet Explorer Won't Upload Files to MOS?"
date: "2013-05-02"
categories: 
  - "oracle"
  - "rants-raves"
---

{{< alert theme="info" >}}
This post is from 2013 when Internet Explorer was [still] a thing. I rather suspect nobody uses IE any more!
{{< /alert >}}

Are you forced to use Internet Explorer at work? Are you, like me, forced to use an old, insecure, broken version of IE at work, because it's the Government Standard version? And are you, like me, unable to upload evidence files to My Oracle Support?

- You need to go to _Tools_, then _Internet Options_.
- On the _Security_ tab, click the _Custom Level_ button.
- Now find and enable the _Include Local Directory Path_ option.
- OK your way back out, restart IE, and Robert is your mother's brother.

This works fine and has been tested on IE7, IE8, IE9 and possibly (but untested) IE10. Other browsers are not affected, but I'm told that Chrome also has problems.

Firefox, on the other hand, just works.

Hope this helps, unlike me, you may be able to find and click the above options. Our (Government) security policies have that option disabled and I'm _not allowed_ to change it. I'm _not allowed_ to use Firefox either. Go figure.

> Security - sometimes it's there to stop you doing your job.

Cheers.
