---
layout: page-fullwidth
title: Mathematik
published: true
author: mwerner
comments: true
meta_description: "Kleine Rechenaufgaben in LaTeX"
date: 2014-05-13 02:05:04
tags:
    - LaTeX
categories:
    - computer
permalink: /2014/05/13/mathematik
---

Bisher habe ich für Berechnungen in `LaTeX` stets die Pakete [calc][1] oder [fp][2] genutzt.

 Jetzt habe ich gelernt, dass für einfache Berechnungen TeX Bordmittel zur Verfügung stellt.
    
   * `\advance` dient zur Addition und Subtraktion,
   *  `\multiply` zur Multiplikation und
   * `\divide` zur Division
  
Für größere Rechnungen ist dies sicher keine Alternative, da immer nur ein Argument zugelassen ist. Auf die Schnelle hat man aber einen kompatiblen Weg für kleinere
Rechnereien. 

[1]: http://www.ctan.org/pkg/calc
[2]: http://www.ctan.org/pkg/fp
