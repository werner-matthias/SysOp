---
title: Inkscapes verlorener Pfad
meta_description: Das LaTeX-svg-Paket funktioniert nicht mehr. Schuld daran ist Inkscape. 
published: true
author: mwerner
date: 2015-04-21 01:04:12
tags:
    - Inkscape
    - LaTeX
    - OSX
categories:
    - small hacks
---
Ich nutze schon seit Jahren die automatische Einbettung von svg-Grafiken (ich weiß, "svg-Grafiken" ist wie "HIV-Virus") in LaTeX mit Hilfe des `svg`-Pakets.
  
Seit dem letzten Update von Inkscape funktionierte dies nicht mehr, da der Aufruf von Inkscape das Arbeitsverzeichnis ändert.

Natürlich könnte beim Aufruf von

~~~ latex
\includesvg{myfile}
~~~

der Pfad mit angegeben werden, ich wollte aber lieber eine generische Lösung, bei der ich mich nicht darum kümmern muss, wo ich mich gerade befinde.

Das konnte mit Hilfe des Pakets `currfile` gelöst werden. In meiner generischen Präamble steht jetzt:
~~~ latex
\usepackage{svg}
\usepackage{currfile} % Wird für einen Workaround für das Inkscape-Pfadproblem gebraucht
\setsvg{inkscape=inkscape -z -D,svgpath=\currfiledir}
~~~
...und die svg-Einbindung funktioniert wieder.
