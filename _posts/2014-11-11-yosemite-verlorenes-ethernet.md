---
title: 'Yosemite: verlorenes Ethernet'
published: true
meta_description: "Es gibt im Web immer wieder Beschwerden, dass in Yosemite der WiFi-Empfang nicht funktioniert. Dieses Problem habe ich nicht, dafür bricht bei mir das kabelgebundene Ethernet zusammen."
author: mwerner
date: 2014-11-11 09:11:29
tags:
    - OSX
    - Yosemite
    - Treiber
    - Ethernet
categories:
    - small hacks
---
Es gibt im Web immer wieder Beschwerden, dass in Yosemite der WiFi-Empfang nicht funktioniert. Dieses Problem habe ich nicht, dafür bricht bei mir (insbesondere nach
einem Ruhezustand) das kabelgebundene Ethernet zusammen.
  
Genauer gesagt: Obwohl das Netzwerk als funktionsfähig angezeigt wird, funktioniert es nicht mehr. Nicht einmal bei Erneuerung der IP über DHCP hilft. Abhilfe schaft ein Reboot, aber wer will schon ständig seinen Rechner neu booten?

Weder der [hier][1] vorgeschlagene Austausch des IO80211-Kernelmoduls noch das [Löschen][2] der VirtualBox-Module brachten Abhilfe. Dafür gaben mir diese Diskussionen aber die Idee für einen Workaround: Offensichtlich scheint der Intel82574-Treiber nach einiger Zeit zu &#8222;verklemmen&#8220;.

Durch Ent- und Neuladen des entsprechenden Kernelmoduls (als root) funktioniert das Ethernet wieder:

{% terminal %}
$ kextunload /System/Library/Extensions/IONetworkingFamily.kext/Contents/PlugIns/Intel82574L.kext
$ kextload /System/Library/Extensions/IONetworkingFamily.kext/Contents/PlugIns/Intel82574L.kext
{% endterminal %} 

Wenn man diese Zeilen durch ein versagendes &#8222;ping&#8220; zum Standardgateway triggert und das Ganze in einen Cron-Job verpackt, hat man ein Workaround, das bis zu dem hoffentlich baldigen Bugfix von Apple weiterhilft.

 [1]: https://discussions.apple.com/thread/6603587 "Apple Forum"
 [2]: http://ccblog.de/2014/10/26/osx-yosemite-verliert-nach-dem-wakeup-das-netz/ "ccblog"
