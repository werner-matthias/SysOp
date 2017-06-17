---
layout: post
title: OS X und NFS
published: true
author: mwerner
comments: true
date: 2015-10-27 01:10:27
tags:
    - El Captain
    - OSX
categories:
    - computer
permalink: /2015/10/27/os-x-und-nfs
---
&#8230;ist auch in El Captain (OS X 10.10) immer noch ein schwieriges Paar. Ein &#8222;normaler&#8220; NFS-Mount (wie in anderen UNIXen) schlägt in der Regel fehl.

In diesem Fall sollten folgende Optionen probiert werden:
  



  
    mount&nbsp;-t&nbsp;nfs&nbsp;-o&nbsp;resvport,nolocks,locallocks,intr,soft,wsize=32768,rsize=3276&nbsp;SERVER:/MOUNTPOINT&nbsp;PATH
  



  
&nbsp;