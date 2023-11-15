---
title: "Thunderbird - Message Filters stop Working on Email Download - But Work Ok Manually."
date: "2014-06-20"
categories: 
  - "linux"
  - "rants-raves"
---

I think this topic has the longest title of all my postings! Never mind. Have you ever started Thunderbird fetching your emails, and encountered a pop-up message that starts off by saying "_The message could not be filtered for folder 'whatever' because writing to folder failed_"? Read on for the cure.

Credit: go here - [https://bugzilla.mozilla.org/show\_bug.cgi?id=931303](https://bugzilla.mozilla.org/show_bug.cgi?id=931303 "https://bugzilla.mozilla.org/show_bug.cgi?id=931303") - then scroll down to Comment 39. A gentleman by the name of Craig Lassen deserves all the kudos for this fix. The problem has been bothering me for some time and I have now fixed it, thanks to this person. Thanks.

The problem manifests when the messages are being downloaded. At some point you will see the pop-up appear and be invited to press the OK button. You will have to do this for every message intended, by a filter, to be moved to a different folder. Only affected folders will show the message.

If you subsequently click _Tools->Run filters on folder_, it will work fine.

The solution - keeping it simple, the following is all you have to do:

- Expand _all_ your folders in the tree on the left side.
- Click on the top entry of the tree - probably your inbox.
- Press the down arrow to visit each folder in turn. Look for "Done" on the lower left on the status bar. When you see that (or "no messages to download"), you can move on.

That's it! At each _affected_ folder in your tree traversal, you will notice a couple of things:

- The words "Building summary file for ...." may appear briefly on the status bar. Followed by "Done".
- The folder name becoming **bold** to indicate unread messages.
- The unread message count updates itself.

Apparently, there has been a minor corruption of some sort in the \*.msf files for the affected folders, and this has resulted in the message filters refusing to write into the affected folders.

Don't thank me, thank Craig!
