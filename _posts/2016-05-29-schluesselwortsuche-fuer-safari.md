---
layout: post
title: Schlüsselwortsuche für Safari
published: true
author: mwerner
comments: true
date: 2016-05-29 10:05:04
tags:
    - Safari
categories:
    - computer
permalink: /2016/05/29/schluesselwortsuche-fuer-safari
---
Eines der Features, die ich bei Safari gegenüber Firefox immer vermisst habe, sind Schlüsselwortsuchen. Beispielsweise erhalte ich bei meiner Firefox-Installation, wenn ich

ctan beamer

in die Adresszeile eingebe, eine Liste mit allen LaTeX-Paketen von der Seite des [Comprehensive TEX Archive Network][1]s, die etwas mit Beamer zu tun haben.

Zwar kennt Safari die &#8222;schnelle Website-Suche&#8220;, die auf [OpenSearch][2] basiert, aber diese ist

  1. nur sehr umständlich lokal für neue Seiten anpassbar (es ist einfacher, wenn die Seiten-Designer selbst  ein  mitgeben);
  2. sind die Schlüsselworte nicht frei wählbar.

Ich habe mich daher nach Plugins umgesehen. Am geeignetsten für meine Zwecke ist [SafariKeyworkSearch][3]. Es ist zwar sehr schlicht, aber dafür leicht nutz- und anpassbar. Ein kleiner Nachteil ist, dass dieses Plugin seine Daten zusammen mit den Cookies und Webdaten speichert. Wenn man &#8212; wie ich das regelmäßig mache &#8212; diese Daten &#8222;global bereinigt&#8220; (also löscht), sind auch die Sucheinstellungen futsch. Jedoch lassen sie sich auch einfach extern speichern und laden, so dass dies keine zu große Katastrophe darstellt. Irgendwann schreibe ich vielleicht mal ein Skript, welches die Sache automatisiert.

Wen es interessiert, dies sind meine Sucheinträge:

{
 "gg":"https://startpage.com/do/search?query=@@@","Default":"gg",
 "az":"http://www.amazon.de/s/ref=nb_sb_noss?__mk_de_DE=%C5M%C5%u017D%D5%D1&url=search-alias%3Daps&field-keywords=@@@",
 "cs":"http://citeseer.ist.psu.edu/search?q=@@@&sort=rel",
 "ctan":"http://www.ctan.org/search/?phrase=%%%",
 "gb":"http://books.google.com/books?oe=UTF-8&um=1&hl=de&q=@@@",
 "ggg":"http://www.google.com/search?q=@@@",
 "gm":"http://maps.google.com/maps?oi=map&q=@@@",
 "gn":"https://www.google.de/search?tbm=nws&q=@@@",
 "gp":"http://images.google.de/images?hl=de&q=@@@&gbv=1",
 "gr":"http://www.canoo.net/services/Controller?input=%%%&MenuId=Search&service=canooNet&lang=de",
 "gs":"http://scholar.google.de/scholar?q=@@@&hl=de&lr=",
 "l":"http://dict.leo.org/ende?lp=ende&lang=de&searchLoc=0&cmpType=relaxed&sectHdr=on&spellToler=on&chinese=both&pinyin=diacritic&search=%%%&relink=on",
 "li":"http://www.linguee.de/deutsch-englisch/search?source=auto&query=%%%",
 "lt":"https://translate.google.com/#en/de/%%%",
 "so":"http://stackoverflow.com/search?q=@@@",
 "st":"http://de.statista.com/statistik/suche/?q=@@@",
 "w":"http://de.wikipedia.org/w/index.php?title=Spezial%3ASuche&search=%%%",
 "wa":"http://www.wolframalpha.com/input/?i=@@@",
 "we":"http://en.wikipedia.org/wiki/Special:Search/%%%",
 "wq":"http://de.wikiquote.org/w/index.php?title=Spezial%3ASuche&search=%%%",
 "wqe":"http://en.wikiquote.org/w/index.php?title=Special%3ASearch&search=%%%",
 "yt":"http://youtube.com/results?search_query=@@@",
 "yy":"http://www.yandex.com/yandsearch?text=%%%&lr=87"
}

&nbsp;

&nbsp;

 [1]: http://www.ctan.org/
 [2]: http://www.opensearch.org/Specifications/OpenSearch/1.1#OpenSearch_description_document
 [3]: http://safarikeywordsearch.aurlien.net