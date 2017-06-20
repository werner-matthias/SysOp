---
layout: post
title: Ausr(ü/u)stung
published: true
author: mwerner
comments: true
date: 2017-03-31 02:03:29
tags:
    - Rust
    - SOPHIA
categories:
    - aihpos
    - computer
permalink: /2017/03/31/aihpos-2-ausrustung
---
Nachdem im [letzten Teil][1] die Design-Ziel von SOPHIA kurz diskutiert wurden, erläutert dieser Teil die Bereitstellung der Entwicklungswerkzeuge.
  


# Setup Rust

Die Entwicklung von SOPHIA erfolgt nicht auf dem System, auf dem es später laufen soll &#8212; das gibt es ja noch gar nicht. Vielmehr soll die Entwicklung auf meinem Arbeitsplatzrechner stattfinden. Das ist in diesem Fall ein MacPro von 2010 mit MacOS. Irgendein Linux-Rechner würde auch gehen, zur Not sogar eine Windows-Maschine.1 Wir können also nicht den normalen Compiler benutzen, sondern einen Crosscompiler. Ein Crosscompiler wird auf einer Plattform ausgeführt (host), und erzeugt Code für eine andere Plattform (target). Ein &#8222;normaler&#8220; Compiler ist also ein Crosscompiler, bei dem Host- und Targetplattform &#8222;zufällig&#8220; identisch sind.

Glücklicherweise ist Rust &#8212; u.a. durch das  [LLVM-Backend][2], das ja auch von clang benutzt wird &#8212; von vornherein als Crosscompiler ausgelegt. Etwas komplizierter ist, dass wir _bare metal_ (also für eine Umgebung ohne Betriebssystem) entwickeln, aber dazu später. Installieren wir zunächst Rust. Dazu gibt es mehrere Wege, man kann den Paketmanager seiner Wahl benutzen. Die offiziell empfohlene Variante läuft über `rustup`. Um `rustup` selbst zu erhalten, sollte man in einer Shell
  



  
    curl&nbsp;https://sh.rustup.rs&nbsp;-sSf&nbsp;|&nbsp;sh
  
 eingeben und den Anweisungen folgen und zunächst die 

_default-_Installation wählen. `rustup` installiert neben dem eigentlichen Compiler die Standardbibliotheken, das Build-Tool `cargo`, sowie Dokumentationen.

Wir werden jedoch für die Betriebssystementwicklung auch Inline-Assembler, Language Items2 und vielleicht einige andere Features brauchen, die noch nicht stabil sind und deshalb noch keinen Eingang in die stabile Version von Rust gefunden haben. Jedoch gibt es die sogenannte Nightly-Version von Rust, die diese Features hat, die wir zusätzlich installieren.


  
     rustup&nbsp;install&nbsp;nightly
  


Eine der netten Eigenschaften von Rust ist es, dass verschiedene Versionen und Toolchains sehr feingranular gemanagt werden können und sich nicht in die Quere kommen. Für jedes Projekt (und sogar Subprojekt) können wir jetzt auswählen, ob wir mit der stabilen oder der nightly Version arbeiten wollen. Wir wählen allerdings die nightly-Version als unsere Standardversion.


  
     rustup&nbsp;default&nbsp;nightly
  


# Targets

Nach fertiger Installation können wir uns mögliche Targets anzeigen lassen:
  



  
    rustup&nbsp;target&nbsp;list
  
  
  
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
  
  
  
    x86_64-apple-darwin&nbsp;(default)
  
  
  
    x86_64-apple-ios
  
  
  
    x86_64-pc-windows-gnu
  
  
  
    x86_64-pc-windows-msvc
  
  
  
    x86_64-rumprun-netbsd
  
  
  
    x86_64-unknown-freebsd
  
  
  
    x86_64-unknown-fuchsia
  
  
  
    x86_64-unknown-linux-gnu
  
  
  
    x86_64-unknown-linux-musl
  
  
  
    x86_64-unknown-netbsd
  
  
  
  



  
Eine Kombination wie  `x86_64-apple-darwin` nennt man das [Target-Triple][3], auch wenn diese Bezeichnung mitunter etwas irreführend anmutet, da das Target-Triple auch mehr (oder weniger) als drei Parameter haben kann. Die generische Form ist `---`. Betrachten wir das für uns notwendige Target-Triple genauer.

## Architektur

Unter Architektur/Subarchitektur wird der Prozessor angegeben, und ggf. noch weitere Informationen wie die Endianess.

Unser Zielsystem, ein Raspberry Pi B+, hat einen ARM1176JZF-S-Prozessor und gehört damit zur ARM11-Familie, was wiederum eine ARMv6-Architektur impliziert. Der ARM1176JZF-S kennt drei verschiedene Befehlssätze, zwischen denen man umschalten kann:

  * den 32-Bit-ARM-Befehlssatz
  * den 16-Bit-Thumb2-Befehlssatz
  * den 8-Bit-Jazelle-Bytecode-Befehlssatz

Die Bitzahl steht hier nicht für die Adressbreite, sondern für die Größe eines Befehls. Da die Dokumentation zum Jazelle-Befehlssatz nicht ohne weiteres öffentlich zugänglich ist, bleibt für uns die Auswahl zwischen ARM- und Thumb-Befehlssatz. Der Raspberry Pi B+ hat 512 MB Speicher, das ist für die heutigen Verhältnisse nicht sonderlich viel.3 Daher wäre die Nutzung des Thumb-Befehlssatzes eine gute Idee, da hier jeder Befehl nur die Hälfte des Speichers verbraucht.4 Da hier weniger Register und Befehle zur Verfügung stehen, braucht man wiederum für den gleichen Code mehr Befehle, aber erfahrungsgemäß ist die Speichereinsparung trotzdem um die 30%. Vergessen wir also den ARM-Befehlssatz und schreiben SOPHIA vollständig in Thumb? Nein, die Sache hat einen Haken: Jeder Ausnahmemode (Interruptmodi, Kernelmode, etc.) wird _immer_ in ARM-Befehlsatz ausgeführt. Mit anderen Worten: Mindestens für den Kern sind wir auf den ARM-Befehlssatz angewiesen. Nutzerprogramme und Betriebssystemserver können dagegen mitunter in Thumb geschrieben werden.

Wenn man sich die obige Liste von Target-Triplen anschaut, dann fällt auf, dass es zwar armv7,- aber keine armv6-Einträge gibt. Die Einträge, die nur mit &#8222;arm&#8220; anfangen, sind eigentlich für alle 32-Bit-ARM, d.h. auch ARMv7. Allerdings kann ARMv7 noch mehr, und ist nicht vollständig abwärtskompatibel.5

Die Angabe eines Herstellers ist optional. Er wird in unserem Fall ausgelassen oder erhält den Wert &#8222;unknown&#8220;.

## Betriebssystem und Binärschnittstelle

Was das Betriebssystem angeht, so ist die Sache einfach: wir haben keines. Durch die Angabe des Betriebssystems weiß der Compiler, welche Systemrufe möglich sind. Dazu wird z.T. die genutzte Standardbibliothek auf dem System angegeben, gegen die gelenkt wird. So benennt &#8222;i686-unknown-linux-gnu&#8220; die [GNU C Library][4], während &#8222;i686-unknown-linux-musl&#8220; die auf statisches Linken optimierte [musl libc][5].

Die Binärschnittstelle beschreibt, wie z.B. die Parameterübergabe bei Funktionsrufen erfolgt. Theoretisch  könnten wir uns hier auch unsere eigenen Konventionen ausdenken; es empfiehlt sich aber, an das den EABI-Standard  ([_embedded-application binary interface_][6]) zu halten, der für verschiedene Architekturen nicht nur die Aufrufkonventionen festlegt, sondern auch noch z.B. Dateiformate. Für ARM unterscheidet man noch durch einen Suffix (&#8222;hf&#8220;) zur ABI, ob Gleitkommaoperationen durch einen (On-Chip-)Coprozessor ausgeführt werden sollen. Da ein Kernel i.d.R. keine Gleitkomma-Operationen ausführen muss, ist das eigentlich uninteressant. Da aber der Raspberry den VFPv2-Gleitkomma-Prozessor enthält, kann diese Option ruhig gewählt werden.

# Ein eigenes Target

## Architekturbeschreibung

Nach der Diskussion im letzten Abschnitt bräuchten wird also das Target-Triple **arm-none-eabihf** (und ggf. später **arm-aihpos-eabihf** und/oder** ****thumpv6-aihpos-eabihf**). Leider gibt es dieses Triple (noch) nicht. Deshalb müssen wir eine Beschreibung dafür anlegen.6 Die Beschreibung ist eine JSON-Datei, und würde in unserem Fall etwa so aussehen:

{
 "llvm-target": "arm-none-eabihf",
 "target-endian": "little",
 "target-pointer-width": "32",
 "os": "none",
 "env": "eabihf",
 "vendor": "unknown",
 "arch": "arm",

 "linker": "arm-none-eabi-gcc",
 "linker-flavor": "gnu",
 "data-layout": "e-m:e-p:32:32-i64:64-v128:64:128-a:0:32-n32-S64",
 "executables": true,
 "relocation-model": "static",
 "no-compiler-rt": true
}

Die meisten Punkte sollten selbsterklärend sein, außer wahrscheinlich &#8222;data-layout&#8220;. Hier werden Eigenschaften des Datenlayouts7, insbesondere Alignment in kompakter Form dargestellt, wobei einiges redundant zu den anderen Punkten ist:


  
    
      
        
          e: little endiang
        
        
          m:e: ELF-Mangling, private Symbole erhalten einen .L-Prefix
        
        
          p:32:32: Größe und Alignment für Pointer, hier jeweils 32 Bit
        
        
          v128:64:128: Für 128-Bit-Vektoren ist das Alignment 64 Bit, bevorzugt 128 Bit
        
        
          a:0:32: Kein Alignment für Aggregattypen (wie struct), aber 32 Bit bevorzugt
        
        
          n32: Native Größe eines Integer ist 32 Bit
        
        
          S64: Stack hat Alignment von 64 Bit
        
      
    
  


## Linker

Dieses JSON-File kopieren wir in unser Projektverzeichnis, wir werden es später ständig brauchen. In ihm haben wir auch spezifiziert, dass für das Linken der GCC-Arm-Cross-Linker benutzt werden soll. Diesen müssen wir installieren, genau genommen die ganze Suite von GCC-Arm-Cross-Tools, wozu z.B. auch `arm-none-eabi-objcopy` oder `arm-none-eabi-readelf` gehört. Ich installiere die Tools über den bei mir vorhandenen Paketmanager `brew`, YMMV:


  
    brew&nbsp;tap&nbsp;nitsky/stm32
  
  
  
    brew&nbsp;install&nbsp;arm-none-eabi-gcc
  


## Core

Wie schon mehrmals betont, müssen wir auf den Einsatz der Rust-Standard-Bibliothek verzichten. Damit man aber nicht ganz auf dem Trockenen sitzt, hat Rust seine Standard-Bibliothek so strukturiert, dass es einen Teil gibt, der frei von externen Referenzen und Betriebssystemrufen ist: `core`. Aber natürlich gibt es core nicht nicht für unser Taget-Triple, daher müssen wir uns die Bibliothek selbst übersetzen. Dazu holen wir uns den Quellcode in unser Projektverzeichnis:


  
    cd&nbsp;~/aihpos
  
  
  
    git&nbsp;clone&nbsp;https://github.com/hackndev/rust-libcore
  
 Jetzt können wir die Core-Bibliothek übersetzen. Dazu schreiben wir das Target-JSON-File in das Hauptverzeichnis der Core-Quellfiles und starten die Übersetzung mit 

`cargo`:
  



  
    cd&nbsp;rust-libcore
  
  
  
    cp&nbsp;../arm-none-eapihf.json&nbsp;.
  
  
  
    cargo&nbsp;build&nbsp;--release&nbsp;--target=arm-none-eapihf
  
  
  
    Compiling&nbsp;rust-libcore&nbsp;v0.0.3&nbsp;(file:///Users/mwerner/aihpos/rust-libcore)
  
  
  
    Finished&nbsp;release&nbsp;[optimized]&nbsp;target(s)&nbsp;in&nbsp;47.20&nbsp;secs
  



  
Unter rust-libcore/target/arm-none-eabihf/release finden wir die fertige Bibliothek libcore.rlib.

# Debugger

Haben wir damit alles zusammen? Für die eigentliche Entwicklung schon. Aber ich möchte es mir etwas bequemer machen und nicht ständig die SD-Karte wechseln müssen. Glücklicherweise hat unserer Prozessor den Coprozessor CP14. Dieser ermöglicht die Nutzung eines [JTAG-Interfaces][7] zum Debuggen. Früher war JTAG-Hardware sehr teuer, die Preise kamen schnell in den vierstelligen Euro-Bereich. Das hat sich zum Glück geändert. Ich nutze den [j-link EDU][8] der Firma Segger, den man (als Universität) bereits für knapp 60 € erwerben kann. Mein Freund und Kollege Jan Richling hat mir noch einen Adapter zur Verfügung gestellt, den er für sein Raspberry-Pi2-Praktikum einsetzt. Jedoch weisen der Raspberry 1B+ und der 2 bzgl. JTAG keine Unterschiede auf, so dass ich den Adapter auch für meinen Pi einsetzen kann.

[][9]  [][10]

Um diese Hardware nutzen zu können, brauche ich das Software-Gegenstück in Form des [OpenOCD][11] (Open On-Chip Debugger). Die Installation übernimmt wieder `brew`:
  



  
    brew&nbsp;install&nbsp;openocd
  



  
  
  
  
    
      Die Entwickler-Tools, die wir gleich besprechen, sind i.d.R. eher auf unixoide Systeme ausgerichtet, so dass es bei Windows meist etwas Mehraufwand gibt. &#8629;
    
    
      Damit können Funktionen zur Verfügung gestellt werden, die sonst die in der Standardbibliothek sind, aber refactor zur Rust-Runtime gehören. &#8629;
    
    
      Zum Vergleich: Ich war in meiner Jugend sehr stolz, dass ich den Speicher meines ersten Computers &#8212; einen ZX 81 &#8212; mit 64 kB(!) sehr weit ausgebaut hatte. &#8629;
    
    
      Genau genommen nur die meisten, da es seit Thumb2 einige 32-Bit-Ausnahmen gibt. &#8629;
    
    
      Der Raspberry 3 hat mit einem ARMv8-Prozessor einen noch anderen Befehlssatz, der hier aber nicht weiter betrachtet wird. &#8629;
    
    
      Eine häufig genutzte Alternative wäre, &#8222;arm-unknown-linux-gnueabihf&#8220; oder &#8222;arm-unknown-linux-musleabihf&#8220; zu nehmen und ein paar Basisfunktionen der C-Bibliothek wie z.B. memcpy nachzuimplementieren. &#8629;
    
    
      Eine genauere Beschreibung von möglichen Layout-Eigenschaften findet sich in der LLVM-Dokumentation &#8629;
    
  


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
