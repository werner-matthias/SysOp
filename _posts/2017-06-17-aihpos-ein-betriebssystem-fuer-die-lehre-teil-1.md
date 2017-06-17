---
layout: post
title: 'aihPOS - another incompatible and hackneyed Pi Operating System'
published: true
author: mwerner
comments: true
date: 2017-03-19 12:03:55
tags:
    - Rust
    - SOPHIA
categories:
    - aihpos
    - computer
permalink: /2017/03/19/aihpos-ein-betriebssystem-fuer-die-lehre-teil-1
---
# Hintergrund

In meinem Kurs [Betriebssysteme][1] verwende ich ab und zu Code-Abschnitte, die die Implementation verschiedener Konzepte des Betriebssystementwurfs demonstrieren. Diese Code-Abschnitte sind bisher sehr uneinheitlich, und setzen sich aus verschiedenen Quellen zusammen:

  * Freier Code aus &#8222;echten&#8220; Betriebssystemen wie z.B. Linux;
  * &#8222;Unechter&#8220; Code (Code, zwar prinzipiell funktioniert, aber so nie einem Betriebsystem eingesetzt wurde), vornehmlich in C und Modula-2;
  * Pseudocode.

Um die Didaktik des Kurses zu verbessern, möchte ich ein Betriebssystem extra für diese Veranstaltung schreiben, so dass möglichst viele Konzepte direkt im Code wiederzufinden sind. Außerdem soll  **aihPOS** &#8212; so der Name des Projekts &#8212; später als Codebasis bei verschiedenen Forschungsprojekten in unser Gruppe nützlich sein.

&#8222;aihPOS&#8220; steht für _**a**nother **i**ncompatible and **h**ackneyed **P**i **O**perating **S**ystem. _Obwohl dieses Akronym einiges über das Betriebssystem aussagt, habe ich es doch in erster Linie gewählt, weil es rückwärts geschrieben den Name meiner Tochter ergibt: Sophia. Daher werde ich den Betriebssystemnamen ab sofort so schreiben: SOPHIA. 🙂

# Ziele

Bevor man ein Projekt der Größe eines Betriebssystems beginnt, und sei es auch nur ein so kleines wie SOPHIA, muss man sich über die Designziele im Klaren sein. Ich nenne zunächst einmal einige Ziele, die ich ausdrücklich **nicht **habe:

  * Geschwindigkeit: immer, wenn es einen Widerspruch zwischen **Performance** und einem klaren Design gibt, werde ich das klare Design bevorzugen. Schließlich soll SOPHIA niemals produktiv eingesetzt werden, sondern zur Demonstrations- und Testzwecken dienen.
  * **Kompatibilität** zu POSIX, Windows o.ä.
  * Einsatz auf komplexer Hardware wie z.B. **x86**-PCs.

Vielmehr habe ich folgende Designziele:

  1. Wesentliche Konzepte von Betriebssystemen sollen entsprechende **Abstraktionen** im Code wiederfinden.
  2. Das Betriebssystem soll eine **klare Struktur** haben und dem **[Mikrokernel][2]**-Ansatz folgen.
  3. Als Zielhardware wird der **Raspberry Pi** benutzt. Er ist günstig zu haben, so dass Studierende leicht mit Modifikationen des Codes herumspielen können, ohne ihren teuren Laptop oder Desktop zu riskieren. Außerdem besitzt unser Labor bereits einen Klassensatz Raspberry Pi 1B+ (mit einer [Gertboard][3]-Erweiterung).

# Sprache

Traditionell werden Betriebssysteme in Assembler oder C geschrieben. Theoretisch kann für das Schreiben von Betriebssysteme nahezu jede Sprache genutzt werden (und z.T. werden sie auch tatsächlich genutzt). Häufig muss zwar  &#8222;im Notfall&#8220; etwas Assembler-Code eingebunden werden (völlig ohne geht es in der Regel sowieso nicht, da viele Feinheiten der unterliegenden Maschine sich nicht in höheren Programmiersprachen abbilden), aber es gibt Betriebssysteme in Java oder Lisp. Jedoch eignen sich die meisten Sprachen nicht sonderlich gut für eine Bare-Metal-Entwicklung, wie ein Betriebssystem sie erfordert: Die Sprachen bringen ihr eigenes Laufzeitsystem mit, das wiederum Systemrufe des Betriebssystem nutzt. Diese stehen aber bei der Entwicklung eines Betriebssystems ja gar nicht zur Verfügung. Entsprechend können einige Sprachkonzept nicht genutzt werden. Noch mehr Systemrufe sind meist in den Bibliotheken zu finden, so dass viele Bibliotheksfunktionen ebenfalls nicht genutzt werden können und ggf. nachimplementiert werden müssen.

Trotzdem werde ich für SOPHIA **nicht** C benutzen, und das im wesentlichen aus drei Gründen:

  * Entsprechend der obigen Anforderungen sollen sich Betriebssystemkonzept gut in Code-Abstraktionen wiederfinden. Dies ist bei C fast so wenig der Fall, wie in Assembler.1
  * C ist eine alte Sprache, die viele modernen Konzepte der [Typen- und Speicher- und Nebenläufigkeitssicherheit][4] nicht kennt.  Daher ist das &#8222;Rumspielen&#8220; mit dem Code (vergleiche Designziel 3) relativ gefährlich.
  * So ein Projekt ist immer eine gute Gelegenheit, eine neue Sprache zu erlernen (oder wenigstens zu vertiefen)

Es stellt sich die Frage, welche anderen Sprachen möglich wären. Die beiden derzeitigen Mainstream-Sprachen Java2 und Python kommen hier nicht infrage, da sie soweit von der Maschine entfernt sind, dass die Brücke zwischen Maschinen- und Sprachabstraktionen nicht sehr intuitiv wäre. Es gibt in meiner Gruppe zwar ein Projekt, ein Python-Betriebssystem zu schreiben, aber dies ist noch nicht so weit, dass es den hier gestellten Designzielen entspricht.

Es gibt einige Sprachen, die für sich in Anspruch nehmen systemnahe zu sein. Die bekanntesten sind wohl C++, D, Forth, Go und Rust. Von diesen scheidet Forth aus, da es ein bißchen **zu** maschinennah ist. Außerdem werden Abstraktionen in der Regel auf Datenstrukturen abgebildet, wo Forth etwas limitiert ist. Die anderen vier Sprachen sind aus dieser Perspektive zweifellos alle geeignet.

Daher habe ich weitere Kriterien betrachtet, die sicherlich sowohl in Auswahl als auch Attribuierung z.T. etwas subjektiv sind:


  
    
    
    
    
      C++
    
    
    
       D
    
    
    
       Go
    
    
    
       Rust
    
  
  
  
    
      Bare-metal Programmierung
    
    
    
      +
    
    
    
    
    
    
      &#8211;
    
    
    
      +
    
  
  
  
    
      Speichersicherheit
    
    
    
      &#8211;3
    
    
    
      +
    
    
    
      4
    
    
    
      +
    
  
  
  
    
      Nebenläufigkeitssicherheit
    
    
    
      &#8211;
    
    
    
    
    
    
      +
    
    
    
      +
    
  
  
  
    
      Toolstabilität
    
    
    
      +
    
    
    
      &#8211;
    
    
    
       +
    
    
    
    
  
  
  
    
      Subjektive Neuheit
    
    
    
      5,
    
    
    
      &#8211;
    
    
    
       +
    
    
    
    
  


Als Ergebnis dieser Betrachtungen werde ich SOPHIA in Rust schreiben. Mir erscheint auch Rust am meisten eine neue Sichtweise zu vermitteln.6  Es gibt ein bekanntes Zitat von [Alan J. Perlis][5]:
  



  A language that doesn’t affect the way you think about programming is not worth knowing.- Alan J. Perlis



  
Dies ist der persönliche Hintergrund dieser Entscheidung. C++ mit Programmierdisziplin hätte es genauso gut &#8212; und vermutlich an manchen Stellen einfacher &#8212; gemacht, aber Rust bringt mir neue Ideen bei.

# Betriebssystementwicklung

An dieser Stelle will ich nochmal auflisten, bei der Betriebssystementwicklung anders ist, als bei der Entwicklung (vieler) anderer Software:

  * Man hat es nicht mit abstrakter, sondern konkreter Hardware zu tun. Dadurch muss man sich mit vielen technischen Details des Prozessors und der Gesamtplattform beschäftigen, die man sonst ignorieren kann.
  * Die Dienste eines Betriebssystems stehen nicht zur Verfügung. Es gibt keine z.B. Speicherverwaltung und oder Dateisysteme. Daher stehen die meisten Funktionen der Standardbibliothek nicht zur Verfügung, weil diese letztendlich Systemfunktionen rufen.
  * Durch die beiden obigen Umstände werden im Design keine Abstraktionen &#8222;aufgezwungen&#8220;. Sie müssen erst erschaffen werden.
  * Damit ein Betriebssystem auch zum Laufen kommt, muss man sich Gedanken über den Bootstrapping-Prozess und damit über Binärformate und die Toolchain machen.

In den nächsten Folgen zu SOPHIA werde ich diese Aspekte besprechen. Was ich dagegen hier nicht machen werde, ist eine Einführung in die Rust- oder die Assembler-Programmierung. Jedoch sind dazu nützliche Informationen in den folgenden Links zu finden.

# Links

  * **Rust** 
      * [The Rust Programming Language][6] (the book)
      * [Rust Referenz][7]
      * [Rust Core Library][8]
      * [The Rustonomicon][9] (&#8222;Unsafe&#8220; Programmierung in Rust)
  * **ARM** 
      * [ARM1176JZF-S Technical Reference Manual &#8211; ARM Infocenter][10]
      * [ARM Systems Developer Guide][11] (etwas veraltet, aber trotzdem sehr hilfreich)
      * [ARM assembler in Raspberry Pi][12]
      * [Exploring ARM inline assembly in Rust][13]
  * **Raspberry Pi** 
      * [Seite der Raspberry Pi Foundation][14]
      * Broadcom: [BCM2835 ARM Peripherals][15]
      * David Welch: [Raspberry Pi ARM based bare metal examples][16]
  * **Betriebssystemprojekte in Rust** 
      * [Redox][17] &#8211; ein schon ziemlich ausgewachsenes UNIX-ähnliches Betriebssystem (x86 7)
      * [Ironkernel][18] (wohl erster BS-Ansatz in Rust, ARM)
      * Philipp Oppermann&#8217;s blog: [Writing an OS in Rust][19] (x86)
      * [IntermezzOS][20] (x86)

&nbsp;


  
  
  
  
    
      Man kann schließlich auch in den meisten Assemblern so etwas wie Structs definieren. &#8629;
    
    
       Nein, ich habe Java nie ernsthaft in Betrachtung gezogen, was aber auch an meiner Abneigung gegenüber dieser Sprache liegen mag. &#8629;
    
    
       Man kann in C++ sehr sicher schreiben, wenn man eine ausreichende Selbstdisziplin an den Tag legt und verschiedene Idiome nicht benutzt. Im Gegensatz dazu erzwingt z.B. Rust sicheren Code. Dort, wo man unsichere (weil mächtigere) Konstrukte braucht, muss es explizit gemacht werden. &#8629;
    
    
      Go bekommt in dieser Kategorie bei mir Punktabzug für den vorhandenen Null-Pointer &#8629;
    
    
      Eigentlich wollte ich hier erst ein Minuszeichen setzen, habe es mir dann anders überlegt: Ich habe C++ lange vor Boost und C++11 gelernt, und seitdem nie grundlegend &#8222;aufgeholt&#8220;. &#8629;
    
    
      Go landet da auf Platz 2, zumal ich schon mal in Occam reingeschnuppert hatte, und D erscheint mir z.T. bewußt &#8222;unoriginell&#8220; zu entworfen zu sein (was in anderen Kontext einen Vorteil darstellt). &#8629;
    
    
      ARM ist geplant, aber bisher nur teilweise implementiert. &#8629;
    
  


 [1]: http://osg.informatik.tu-chemnitz.de/lehre/os
 [2]: https://en.wikipedia.org/wiki/Microkernel
 [3]: https://www.gertbot.com
 [4]: http://joeduffyblog.com/2015/11/03/a-tale-of-three-safeties/
 [5]: https://de.wikipedia.org/wiki/Alan_J._Perlis
 [6]: https://doc.rust-lang.org/book/
 [7]: https://doc.rust-lang.org/reference.html
 [8]: https://doc.rust-lang.org/core/index.html
 [9]: https://doc.rust-lang.org/nomicon/
 [10]: http://infocenter.arm.com/help/topic/com.arm.doc.ddi0301h/ddi0301h_arm1176jzfs_r0p7_trm.pdf
 [11]: http://mazsola.iit.uni-miskolc.hu/~drdani/docs_arm/36_Elsevier-ARM_Sy.pdf
 [12]: http://thinkingeek.com/arm-assembler-raspberry-pi/
 [13]: http://embed.rs/articles/2016/arm-inline-assembly-rust/
 [14]: https://www.raspberrypi.org
 [15]: https://www.raspberrypi.org/wp-content/uploads/2012/02/BCM2835-ARM-Peripherals.pdf
 [16]: https://github.com/dwelch67/raspberrypi
 [17]: https://www.redox-os.org
 [18]: https://github.com/wbthomason/ironkernel
 [19]: http://os.phil-opp.com
 [20]: http://intermezzos.github.io