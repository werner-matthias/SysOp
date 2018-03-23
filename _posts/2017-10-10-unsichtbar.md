---
title: "(Un)sichtbar"
layout: page-fullwidth
author: mwerner
show_meta: true
categories:
- small hacks
tags:
- MacOS
- OSX
meta_description: Sichtbarmachen versteckter Dateien auf die einfache Art.
---

MacOS bevormundet seine Nutzer manchmal ganz schön. So werden standardmäßig keine "versteckten" Dateien im Finder dargestellt, also auch sogenannte Dot-Dateien, deren Name mit einem Punkt beginnt.
Bisher habe ich immer, um diese sichtbar zu machen, an den <i>defaults</i> herumgebastelt:

{% terminal %}
$ defaults write com.apple.Finder AppleShowAllFiles true
$ killall Finder
{% endterminal %}
Wenn ich die versteckten Dateien nicht mehr brauchte, musste ich Entsprechendes mit `false` eingeben.

Jetzt habe ich gelernt, dass man die Umschaltung einfach auf Tastendruck haben kann. <kbd>SHIFT</kbd>+<kbd>CMD</kbd>+<kbd>.</kbd> macht den Job.
