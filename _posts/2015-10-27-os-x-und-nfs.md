---
title: OS X und NFS
published: true
author: mwerner
teaser: ...sind auch in El Captain (OS X 10.10) immer noch ein schwieriges Paar.
date: 2015-10-27 01:10:27
tags:
    - OSX
    - El Captain
    - nfs
categories:
    - small hacks
---
Ein &#8222;normaler&#8220; NFS-Mount (wie in anderen UNIXen) schlägt in der Regel fehl.

In diesem Fall sollten folgende Optionen probiert werden:

{% terminal %}
$ mount -t nfs -o resvport,nolocks,locallocks,intr,soft,wsize=32768,rsize=3276 SERVER:/MOUNTPOINT PATH
{% endterminal %}
