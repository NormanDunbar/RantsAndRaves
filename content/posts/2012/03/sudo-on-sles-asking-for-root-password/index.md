---
title: "Sudo on SLES Asking for Root Password?"
date: "2012-03-21"
categories: 
  - "linux"
---

It's a post-installation configuration that hasn't been done. The `/etc/sudoers` file has the following section in it:

{{< alert theme="info" >}}
In the default (unconfigured) configuration, sudo asks for the root password.
This allows use of an ordinary user account for administration of a freshly
installed system. When configuring sudo, delete the two following lines:

```text
Defaults targetpw   # ask for the password of the target user i.e. root
ALL ALL=(ALL) ALL   # WARNING! Only use this together with 'Defaults targetpw'!
```
{{< /alert >}}

The administrator needs to edit `/etc/sudoers`, using the `visudo` command, and comment out the final two lines above, or remove them altogether. Once done, individual user accounts can be added as follows:

```text
oracle ALL=(ALL) ALL
norman ALL=(ALL) ALL
```

Alternatively,  groups can be given access as follows:

```text
%dba ALL=(ALL) ALL
%oinstall ALL=(ALL) ALL
```

Substituting the appropriate levels of commands and privileges of course!
