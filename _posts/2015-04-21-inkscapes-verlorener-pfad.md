---
layout: page
title: Inkscapes verlorener Pfad
published: true
author: mwerner
comments: true
date: 2015-04-21 01:04:12
tags:
    - Inkscape
    - LaTeX
    - OSX
categories:
    - computer
permalink: /2015/04/21/inkscapes-verlorener-pfad
---
Ich nutze schon seit Jahren die automatische Einbettung von svg-Grafiken (ich weiß, &#8222;svg-Grafiken&#8220; ist wie &#8222;HIV-Virus&#8220;) in LaTeX mit Hilfe des `svg`-Pakets.
  
Seit dem letzten Update von Inkscape funktionierte dies nicht mehr, da der Aufruf von Inkscape das Arbeitsverzeichnis ändert.

Natürlich könnte beim Aufruf von

\includesvg{myfile}

der Pfad mit angegeben werden, ich wollte aber lieber eine generische Lösung, bei der ich mich nicht darum kümmern muss, wo ich mich gerade befinde.

Das konnte mit Hilfe des Pakets `currfile` gelöst werden. In meiner generischen Präamble steht jetzt:

\usepackage{svg}
\usepackage{currfile} % Wird für einen Workaround für das Inkscape-Pfadproblem gebraucht
\setsvg{inkscape=inkscape -z -D,svgpath=\currfiledir}

&#8230;und die svg-Einbindung funktioniert wieder.
