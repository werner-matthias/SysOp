---
title: Ausr(ü/u)stung
layout: page-fullwidth
author: mwerner
show_meta: true
subheadline: Ein Betriebsystem für die Lehre
meta_description: Bevor wir mit dem Code-Schreiben beginnen, brauchen wir die Entwickler-Werkzeuge.
published: true
date: '2017-03-31'
tags:
- aihPOS
- Rust
- JTAG
- EABI
- Debugger
categories:
- aihpos
---

**Inhalt**
- TOC
{:toc}
{% include next-previous-post-in-category %}
 Nachdem im [letzten Teil]({% post_url 2017-03-19-aihpos-1 %}) die Design-Ziel von SOPHIA kurz diskutiert wurden, erläutert dieser Teil die Bereitstellung der Entwicklungswerkzeuge.

## Setup Rust

Die Entwicklung von SOPHIA erfolgt nicht auf dem System, auf dem es später laufen soll -- das gibt es ja noch gar nicht. Vielmehr soll die Entwicklung auf meinem
Arbeitsplatzrechner stattfinden. Das ist in diesem Fall ein MacPro von 2010 mit MacOS. Irgendein Linux-Rechner würde auch gehen, zur Not sogar eine Windows-Maschine.[^1] Wir
können also nicht den normalen Compiler benutzen, sondern einen Crosscompiler. Ein Crosscompiler wird auf einer Plattform ausgeführt (host), und erzeugt Code für eine
andere Plattform (target). Ein "normaler" Compiler ist also ein Crosscompiler, bei dem Host- und Targetplattform "zufällig" identisch sind. 

Glücklicherweise ist Rust -- u.a. durch das  [LLVM-Backend][2], das ja auch von clang benutzt wird -- von vornherein als Crosscompiler
ausgelegt. Etwas komplizierter ist, dass wir _bare metal_ (also für eine Umgebung ohne Betriebssystem) entwickeln, aber dazu später. Installieren wir zunächst Rust. Dazu
gibt es mehrere Wege, man kann den Paketmanager seiner Wahl benutzen. Die offiziell empfohlene Variante läuft über `rustup`. Um `rustup` selbst zu erhalten, sollte man in
einer Shell 

{% terminal %}
$ curl https://sh.rustup.rs -sSf | sh
{% endterminal %} 
 eingeben und den Anweisungen folgen und zunächst die _default-_Installation wählen. `rustup` installiert neben dem eigentlichen Compiler die Standardbibliotheken, das
 Build-Tool `cargo`, sowie Dokumentationen. 

Wir werden jedoch für die Betriebssystementwicklung auch Inline-Assembler, Language Items[^2] und vielleicht einige andere Features brauchen, die noch nicht stabil sind
und deshalb noch keinen Eingang in die stabile Version von Rust gefunden haben. Jedoch gibt es die sogenannte Nightly-Version von Rust, die diese Features hat, die wir
zusätzlich installieren. 

{% terminal %}
$ rustup install nightly
{% endterminal %} 
  
Eine der netten Eigenschaften von Rust ist es, dass verschiedene Versionen und Toolchains sehr feingranular gemanagt werden können und sich nicht in die Quere kommen. Für
jedes Projekt (und sogar Subprojekt) können wir jetzt auswählen, ob wir mit der stabilen oder der nightly Version arbeiten wollen. Wir wählen allerdings die
Nightly-Version als unsere Standardversion. 

{% terminal %}
$ rustup default nightly
{% endterminal %}

## Targets

Nach fertiger Installation können wir uns mögliche Targets anzeigen lassen:
{% terminal %}
$rustup target list
  aarch64-apple-ios
  aarch64-linux-android
  aarch64-unknown-fuchsia
  aarch64-unknown-linux-gnu
  arm-linux-androideabi
  arm-unknown-linux-gnueabi
  arm-unknown-linux-gnueabihf
  arm-unknown-linux-musleabi
  arm-unknown-linux-musleabihf
  armv7-apple-ios
  armv7-linux-androideabi
  armv7-unknown-linux-gnueabihf
  armv7-unknown-linux-musleabihf
  armv7s-apple-ios
  asmjs-unknown-emscripten
  i386-apple-ios
  i586-pc-windows-msvc
  i586-unknown-linux-gnu
  i686-apple-darwin
  i686-linux-android
  i686-pc-windows-gnu
  i686-pc-windows-msvc
  i686-unknown-freebsd
  i686-unknown-linux-gnu
  i686-unknown-linux-musl
  mips-unknown-linux-gnu
  mips-unknown-linux-musl
  mips64-unknown-linux-gnuabi64
  mips64el-unknown-linux-gnuabi64
  mipsel-unknown-linux-gnu
  mipsel-unknown-linux-musl
  powerpc-unknown-linux-gnu
  powerpc64-unknown-linux-gnu
  powerpc64le-unknown-linux-gnu
  s390x-unknown-linux-gnu
  sparc64-unknown-linux-gnu
  wasm32-unknown-emscripten
  x86_64-apple-darwin (default)
  x86_64-apple-ios
  x86_64-pc-windows-gnu
  x86_64-pc-windows-msvc
  x86_64-rumprun-netbsd
  x86_64-unknown-freebsd
  x86_64-unknown-fuchsia
  x86_64-unknown-linux-gnu
  x86_64-unknown-linux-musl
  x86_64-unknown-netbsd
{% endterminal %}
  
Eine Kombination wie  `x86_64-apple-darwin` nennt man das [Target-Triple][3], auch wenn diese Bezeichnung mitunter etwas irreführend anmutet, da das Target-Triple auch
mehr (oder weniger) als drei Parameter haben kann. Die generische Form ist `<Architektur><Subarchitektur>-<Hersteller>-<Betriebssystem>-<Binärschnittstelle>`. Betrachten
wir das für uns notwendige Target-Triple genauer.  

### Architektur

Unter Architektur/Subarchitektur wird der Prozessor angegeben, und ggf. noch weitere Informationen wie die Endianess.

Unser Zielsystem, ein Raspberry Pi B+, hat einen ARM1176JZF-S-Prozessor und gehört damit zur ARM11-Familie, was wiederum eine ARMv6-Architektur impliziert. Der ARM1176JZF-S kennt drei verschiedene Befehlssätze, zwischen denen man umschalten kann:

  * den 32-Bit-ARM-Befehlssatz
  * den 16-Bit-Thumb2-Befehlssatz
  * den 8-Bit-Jazelle-Bytecode-Befehlssatz

Die Bitzahl steht hier nicht für die Adressbreite, sondern für die Größe eines Befehls. Da die Dokumentation zum Jazelle-Befehlssatz nicht ohne weiteres öffentlich
zugänglich ist, bleibt für uns die Auswahl zwischen ARM- und Thumb-Befehlssatz. Der Raspberry Pi B+ hat 512 MB Speicher, das ist für die heutigen Verhältnisse nicht
sonderlich viel.[^3] Daher wäre die Nutzung des Thumb-Befehlssatzes eine gute Idee, da hier jeder Befehl nur die Hälfte des Speichers verbraucht.[^4] Da hier weniger Register
und Befehle zur Verfügung stehen, braucht man wiederum für den gleichen Code mehr Befehle, aber erfahrungsgemäß ist die Speichereinsparung trotzdem um die 30%. Vergessen
wir also den ARM-Befehlssatz und schreiben SOPHIA vollständig in Thumb? Nein, die Sache hat einen Haken: Jeder Ausnahmemode (Interruptmodi, Kernelmode, etc.) wird _immer_
in ARM-Befehlsatz ausgeführt. Mit anderen Worten: Mindestens für den Kern sind wir auf den ARM-Befehlssatz angewiesen. Nutzerprogramme und Betriebssystemserver können
dagegen mitunter in Thumb geschrieben werden. 

Wenn man sich die obige Liste von Target-Triplen anschaut, dann fällt auf, dass es zwar armv7,- aber keine armv6-Einträge gibt. Die Einträge, die nur mit
"arm" anfangen, sind eigentlich für alle 32-Bit-ARM, d.h. auch ARMv7. Allerdings kann ARMv7 noch mehr, und ist nicht vollständig abwärtskompatibel.[^5] 

Die Angabe eines Herstellers ist optional. Er wird in unserem Fall ausgelassen oder erhält den Wert "unknown".

### Betriebssystem und Binärschnittstelle

Was das Betriebssystem angeht, so ist die Sache einfach: wir haben keines. Durch die Angabe des Betriebssystems weiß der Compiler, welche Systemrufe möglich sind. Dazu wird z.T. die genutzte Standardbibliothek auf dem System angegeben, gegen die gelenkt wird. So benennt "i686-unknown-linux-gnu" die [GNU C Library][4], während "i686-unknown-linux-musl" die auf statisches Linken optimierte [musl libc][5].

Die Binärschnittstelle beschreibt, wie z.B. die Parameterübergabe bei Funktionsrufen erfolgt. Theoretisch  könnten wir uns hier auch unsere eigenen Konventionen ausdenken; es empfiehlt sich aber, an das den EABI-Standard  ([_embedded-application binary interface_][6]) zu halten, der für verschiedene Architekturen nicht nur die Aufrufkonventionen festlegt, sondern auch noch z.B. Dateiformate. Für ARM unterscheidet man noch durch einen Suffix ("hf") zur ABI, ob Gleitkommaoperationen durch einen (On-Chip-)Coprozessor ausgeführt werden sollen. Da ein Kernel i.d.R. keine Gleitkomma-Operationen ausführen muss, ist das eigentlich uninteressant. Da aber der Raspberry den VFPv2-Gleitkomma-Prozessor enthält, kann diese Option ruhig gewählt werden.

## Ein eigenes Target

### Architekturbeschreibung

Nach der Diskussion im letzten Abschnitt bräuchten wird also das Target-Triple **arm-none-eabihf** (und ggf. später **arm-aihpos-eabihf** und/oder
**thumpv6-aihpos-eabihf**). Leider gibt es dieses Triple (noch) nicht. Deshalb müssen wir eine Beschreibung dafür anlegen.[^6] Die Beschreibung ist eine JSON-Datei, und
würde in unserem Fall etwa so aussehen: 

~~~ json
{%  github_sample werner-matthias/aihPOS/blob/master/jtag/arm-none-eabihf.json  0 -1 %}
~~~

Die meisten Punkte sollten selbsterklärend sein, außer wahrscheinlich "data-layout". Hier werden Eigenschaften des Datenlayouts[^7], insbesondere Alignment in kompakter Form dargestellt, wobei einiges redundant zu den anderen Punkten ist:
  - e: little endiang
  - m:e: ELF-Mangling, private Symbole erhalten einen .L-Prefix
  - p:32:32: Größe und Alignment für Pointer, hier jeweils 32 Bit
  - v128:64:128: Für 128-Bit-Vektoren ist das Alignment 64 Bit, bevorzugt 128 Bit
  - a:0:32: Kein Alignment für Aggregattypen (wie struct), aber 32 Bit bevorzugt
  - n32: Native Größe eines Integer ist 32 Bit
  - S64: Stack hat Alignment von 64 Bit
 
### Linker

Dieses JSON-File kopieren wir in unser Projektverzeichnis, wir werden es später ständig brauchen. In ihm haben wir auch spezifiziert, dass für das Linken der
GCC-Arm-Cross-Linker benutzt werden soll. Diesen müssen wir installieren, genau genommen die ganze Suite von GCC-Arm-Cross-Tools, wozu z.B. auch `arm-none-eabi-objcopy`
oder `arm-none-eabi-readelf` gehört. Ich installiere die Tools über den bei mir vorhandenen Paketmanager `brew`, YMMV: 
{% terminal %}
$ brew tap nitsky/stm32
$ brew install arm-none-eabi-gcc
{% endterminal %}
  
### Core

Wie schon mehrmals betont, müssen wir auf den Einsatz der`Rust`-Standard-Bibliothek verzichten. Damit man aber nicht ganz auf dem Trockenen sitzt, hat Rust seine
Standard-Bibliothek so strukturiert, dass es einen Teil gibt, der frei von externen Referenzen und Betriebssystemrufen ist: `core`. Aber natürlich gibt es core nicht
nicht für unser Taget-Triple, daher müssen wir uns die Bibliothek selbst übersetzen.

{% include alert warning=' _Die folgenden Schritte sind bei Einsatz von `xargo`, wie in der Folge ["Um-Rust-ung"](https://werner-matthias.github.io/SysOp/aihpos/computer/aihpos-5-umrustung) wird gezeigt, unnötig.<br/>
Ich lasse sie aber aus "historischen" Gründen stehen._' %}

Dazu holen wir uns den Quellcode in unser Projektverzeichnis: 

{% terminal %}
$ cd ~/aihpos
$ git clone https://github.com/hackndev/rust-libcore
{% endterminal %}
  
 Jetzt können wir die Core-Bibliothek übersetzen. Dazu schreiben wir das Target-JSON-File in das Hauptverzeichnis der Core-Quellfiles und starten die Übersetzung mit `cargo`:
{% terminal %}
$ cd rust-libcore
$ cp ../arm-none-eapihf.json .
$ cargo build --release --target=arm-none-eapihf
    Compiling rust-libcore v0.0.3 (file:///Users/mwerner/aihpos/rust-libcore)
    Finished release [optimized] target(s) in 47.20 secs
{% endterminal %}
  
Unter rust-libcore/target/arm-none-eabihf/release finden wir die fertige Bibliothek libcore.rlib.

## Debugger

Haben wir damit alles zusammen? Für die eigentliche Entwicklung schon. Aber ich möchte es mir etwas bequemer machen und nicht ständig die SD-Karte wechseln
müssen. Glücklicherweise hat unserer Prozessor den Coprozessor CP14. Dieser ermöglicht die Nutzung eines [JTAG-Interfaces][7] zum Debuggen. Früher war JTAG-Hardware sehr
teuer, die Preise kamen schnell in den vierstelligen Euro-Bereich. Das hat sich zum Glück geändert. Ich nutze den [j-link EDU][8] der Firma Segger, den man (als
Universität) bereits für knapp 60 € erwerben kann. Mein Freund und Kollege Jan Richling hat mir noch einen Adapter zur Verfügung gestellt, den er für sein
Raspberry-Pi2-Praktikum einsetzt. Jedoch weisen der Raspberry 1B+ und der 2 bzgl. JTAG keine Unterschiede auf, so dass ich den Adapter auch für meinen Pi einsetzen kann. 

![]({{site.urlimg}}/jlink.png){:class="img-responsive"} ![]({{site.urlimg}}/adapter.png){:class="img-responsive"}

Um diese Hardware nutzen zu können, brauche ich das Software-Gegenstück in Form des [OpenOCD][11] (Open On-Chip Debugger). Die Installation übernimmt wieder `brew`:
{% terminal %}
$ brew install openocd
{% endterminal %}

{% include next-previous-post-in-category %}

[^1]: Die Entwickler-Tools, die wir gleich besprechen, sind i.d.R. eher auf unixoide Systeme ausgerichtet, so dass es bei Windows meist etwas Mehraufwand gibt.
[^2]: Damit können Funktionen zur Verfügung gestellt werden, die sonst die in der Standardbibliothek sind, aber refactor zur Rust-Runtime gehören.
[^3]: Zum Vergleich: Ich war in meiner Jugend sehr stolz, dass ich den Speicher meines ersten Computers -- einen ZX 81 -- mit 64 kB(!) sehr weit ausgebaut hatte.
[^4]: Genau genommen nur die meisten, da es seit Thumb2 einige 32-Bit-Ausnahmen gibt.
[^5]: Der Raspberry 3 hat mit einem ARMv8-Prozessor, der eine 64-Bit-Architektur hat und im 64-Bit-Modus einen völlig anderen Befehlssatz (A64), der hier aber nicht weiter betrachtet wird.
[^6]: Eine häufig genutzte Alternative wäre, "arm-unknown-linux-gnueabihf" oder "arm-unknown-linux-musleabihf" zu nehmen und ein paar Basisfunktionen der C-Bibliothek wie z.B. memcpy nachzuimplementieren.
[^7]: Eine genauere Beschreibung von möglichen Layout-Eigenschaften findet sich in der LLVM-Dokumentation

 [1]: http://sysop.matthias-werner.net/aihpos-ein-betriebssystem-fuer-die-lehre-teil-1/
 [2]: https://de.wikipedia.org/wiki/LLVM
 [3]: https://clang.llvm.org/docs/CrossCompilation.html#target-triple
 [4]: https://en.wikipedia.org/wiki/GNU_C_Library
 [5]: http://www.musl-libc.org
 [6]: http://infocenter.arm.com/help/topic/com.arm.doc.ihi0036b/IHI0036B_bsabi.pdf
 [7]: https://en.wikipedia.org/wiki/JTAG
 [8]: https://www.segger.com/j-link-edu.html
 [9]: http://sysop.matthias-werner.net/wp-content/uploads/2017/04/jlink.png
 [10]: http://sysop.matthias-werner.net/wp-content/uploads/2017/04/adapter.png
 [11]: http://openocd.org