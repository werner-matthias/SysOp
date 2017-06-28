---
layout: page-fullwidth
title: Hello Jekyll
meta_description: "Ich verlasse Wordpress und wechsle zu Jekyll."
published: true
author: mwerner
comments: true
date: 2017-06-26 12:16:41
tags:
    - Jekyll
    - Wordpress
categories:
    - Meta
permalink: /2017/06/26/hello-jekyll
---
Wer dieses Blog regelmäßig liest (was bisher nicht viele sein dürften), dem ist vermutlich eine Veränderung aufgefallen. Die Ursache dafür ist,
dass ich mein Blog-System umgestellt habe: Ich bin von Wordpress auf [Jekyll](https://jekyllrb.com) umgestiegen. Jekyll ist ein Generator für statische Webseiten.

Es git einige Gründe, die für den Wechsel sprechen:
- Das Laden einer Seite geht jetzt wesentlich schneller (nach Messungen ungefähr um den Faktor 4..5)
- Statische Seiten sind sicherer als dynamische.[^1]
- Es ist leicht, GitHub-Files mit Syntaxhighlighting einzubinden. Das erspart mir eine Menge von Copy&Paste.

Dem steht ein Nachteil gegenüber: Das Standard-Kommentarsystem von Jekyll ist Disqus, gegen dessen Einsatz ich Bedenken habe, da es eine große Datenkrake ist.
Bis ich eine andere Lösung gefunden habe, können Kommentare nur über Mail an mich oder GitHub-Issues gegeben werden.

[^1]: Ich konnte in letzter Zeit eine ganze Reihe von Versuchen beobachten, meine Wordpress-Installation zu hacken.
