---
layout: post
title: Um-Rust-ung
published: true
author: mwerner
comments: true
date: 2017-05-14 12:05:04
tags: [ ]
categories:
    - aihpos
    - computer
permalink: /2017/05/14/aihpos5-umrustung
---
# Rust auf Nicht-Standard-Plattformen

Nachdem ich im [letzen Teil][1] schon wieder eine Bibliothek per Hand einpflegen musste, hatte ich beschlossen, auf das Rust-eigene Management-Tool `cargo` umzusteigen: Schließlich ist die Einbindung von externen Bibliotheken eine der Stärken von Rust. Dem gegenüber stehen für das SOPHIA-Projekt zwei Nachteile, die die Nutzung von `cargo` erschweren:

  * Unsere Zielplattform wird nicht unterstützt, damit auch nicht die entsprechenden Bibliotheken
  * Das Endprodukt (Kernel-Image) ist &#8222;untypisch&#8220;. Seine Erstellung verlangt Post-Linker-Aktionen, die ebenfalls nicht unterstützt werden.

Für beide Probleme gibt es (mindestens) eine Lösung.

## Xargo

Zur Lösung des ersten Problems gibt es [`xargo`][2], eine `cargo`-Variante für Cross-Umgebungen. Diese übersetzt automatisch die Core-Bibliothek und bei Bedarf auch andere Teile der Standard-Bibliothek und &#8222;schenkt&#8220; dem Compiler die richtige Tool-Chain, indem `rootdir` entsprechend gesetzt wird. Man kann sich damit auch angepasste Versionen der Standard-Bibliothek erstellen &#8212; mal sehen, ob ich das später brauchen werde.

Die Installation geht dismal nicht über Homebrew, sondern schnell und reibungslos über `cargo`:


  
    cargo&nbsp;install&nbsp;xargo
  


`Cargo`/`xargo` gehen von einem bestimmten Directory-Layout und der Existenz von (mindestens) einer Konfigurationsdatei **cargo.toml** aus, die das sogenannte &#8222;[Manifest][3]&#8220; enthält. Dort kann man auch benötigte Bibliotheken einbinden, die dann von `cargo` automatisch geladen und übersetzt werden. Wir wollten ja die **compiler-buildins**-Bibliothek nutzen. Dies geschieht im **cargo.toml** für unseren Kernel im Abschnitt `[dependencies]` und sieht so aus:

[package]
name = "aihPOS"
version = "0.0.2"
authors = ["Matthias Werner &lt;mwerner@informatik.tu-chemnitz.de&gt;"]

[dependencies]
compiler_builtins = { git = "https://github.com/rust-lang-nursery/compiler-builtins", features = ["mem"] }

[profile.dev]
panic = "abort"
lto = false

[profile.release]
panic = "abort"
lto = false
opt-level = 3


Diese Datei muss sich im Wurzelverzeichnis eines Projekts oder Subprojekts befinden. Die Quelldateien sind dann in einem Unterverzeichnis **src/**, das weitere Verzeichnisse (mit ggf. eigenen **cargo.toml**-Dateien) befinden können. Beim Start einer Übersetzung mit `cargo build` wird ein Verzeichnis **target** angelegt,  in dem übersetzte Dateien etc. angelegt werden. 

Neben dem Manifest gibt es optional noch eine Konfigurationsdatei, ebenfalls im TOML-Format. Sie heißt **config** und befindet sich in einem Unterverzeichnis **.cargo/**. Wir brauchen sie vor allem, weil die Erstellung des Kernels Compiler- bzw. Linkeroptionen benötigt, die nicht Standard sind:

[build]
traget= "arm-none-eabihf"

[target.arm-none-eabihf]
linker = "arm-none-eabi-gcc"
rustflags = [
  "-C", "code-model=kernel",
  "-C", "link-arg=-nostartfiles",
  "-C", "link-arg=-nostdlib",
  "-C", "link-arg=-Tlayout.ld",
  "-C", "link-arg=-mfloat-abi=hard",
  "-C", "link-arg=-ffreestanding",
  ]

Die Trennung der Steuerung von Compiler-Optionen (in den `[profile.*]`-Abschnitten in **cargo.toml** können auch einige Optionen gesteuert werden) ist etwas unglücklich und wird möglicherweise in künftigen Rust/Cargo-Versionen aufgehoben, aber derzeit muss man damit leben.

## Post-Linker Aktionen

Wenn man sich das bisherige **Makefile** anschaut, dann sieht man, dass der Building-Prozess nicht &#8212; wie sonst meist üblich &#8212; mit dem Linken endet. Dies ist in `cargo` nicht vorgesehen. Zwar kennt `cargo` die Möglichkeit, mit Hilfe eines Build-Scriptes Schritte **vor** der Übersetzung auszuführen; etwas ähnliches für danach wird zwar in unter den Entwicklern diskutiert, wurde aber bisher nicht umgesetzt. Man kann zwar den Linker (in der Target-Datei) umdefinieren, aber ich fürchte, dass eine solche Lösung schnell unübersichtlich wird. Es ein häufig genutzter und sogar empfohlener1 Ansatz ist daher,  `cargo` wiederum von einem `Makefile` aufrufen lassen. Das würde aber  dazu führen, dass Abhängigkeiten z.T. doppelt gepflegt werden müssen. Wenn `cargo`, dann richtig. 

Glücklicherweise ist `cargo` leicht erweiterbar: Wenn `cargo` (oder `xargo`) mit einem unbekannten Befehl gerufen wird, sucht es im Standardpfad nach einem Programm `cargo-`. Dass mache ich mir zunutze und schreibe ein kleines `bash`-Skript `cargo-kernel`, das erst `xargo` aufruft und dann noch den &#8222;Rest&#8220; erledigt:

shift
PROFILE=debug
TARGET=
for opt in "$@"; do
    case $opt in
        --release)
		PROFILE=release
     		;;
	--target=*)
		TARGET=`echo $opt | cut -f2 -d'='` 
    esac
done
BINPATH=./target/$TARGET/$PROFILE
xargo build $@
PKG=`xargo pkgid -q | cut -d# -f2 | cut -d: -f1`
if [ -a $BINPATH/$PKG ]; then
   arm-none-eabi-objcopy $BINPATH/$PKG -O binary  $BINPATH/kernel.img
   arm-none-eabi-objdump -D $BINPATH/$PKG &gt; $BINPATH/$PKG.list
fi

Diese Skript muss irgendwo im Ausführungsfad abgelegt werden. Dann kann man die Übersetzung im Rust-Stil anstoßen:


  
    cargo&nbsp;kernel&nbsp;--target=arm-none-eabihf
  
  
  
      Compiling&nbsp;core&nbsp;v0.0.0&nbsp;(file:///Users/mwerner/.rustup/toolchains/nightly-x86_64-apple-darwin/lib/rustlib/src/rust/src/libcore)
  
  
  
       Finished&nbsp;release&nbsp;[optimized]&nbsp;target(s)&nbsp;in&nbsp;22.87&nbsp;secs
  
  
  
      Compiling&nbsp;compiler_builtins&nbsp;v0.1.0&nbsp;(https://github.com/rust-lang-nursery/compiler-builtins#f3ace110)
  
  
  
      Compiling&nbsp;aihPOS&nbsp;v0.0.2&nbsp;(file:///Users/mwerner/Development/aihPOS/aih_pos/kernel)
  
  
  
    warning:&nbsp;crate&nbsp;`aihPOS`&nbsp;should&nbsp;have&nbsp;a&nbsp;snake&nbsp;case&nbsp;name&nbsp;such&nbsp;as&nbsp;`aih_pos`
  
  
  
      |
  
  
  
      =&nbsp;note:&nbsp;#[warn(non_snake_case)]&nbsp;on&nbsp;by&nbsp;default
  
  
  
     Finished&nbsp;dev&nbsp;[optimized&nbsp;+&nbsp;debuginfo]&nbsp;target(s)&nbsp;in&nbsp;4.8&nbsp;secs
  


Man beachte, dass immer noch `cargo` und nicht `xargo` aufgerufen werden muss. Zwar würde auch `xargo` das Shellskript finden und ausführen, jedoch hat `cargo`/`xargo` einen Lock, der die nebenläufige Ausführung blockiert. `xargo kernel` würde also hängen bleiben.

## Diskussion

Bin ich nun vollständig in der Rust-Welt angekommen? Das hängt von der Betrachtung ab. Ich kann auf `make` verzichten und alles, was ich mit `make` gemacht habe nun mit Rust-Mitteln tun. Andererseits kennt `cargo` noch viele Befehle  &#8212; wie z.B. für das Bauen von Tests und Dokumentationen oder die Arbeit mit Repositories &#8212; die noch nicht berücksichtigt sind. Einiges könnte einfach so funktionieren, aber es besteht dafür keine Garantie.

# GitHub

Wenn ich schon bei einer Reorganisation bin, gehe ich noch einen Schritt weiter: Der alte JTAG-Kernel (siehe [Teil 3][4]) kommt in ein eigenes Verzeichnis, so dass er noch immer mit `make` übersetzt werden kann. Ich nutze ihn als Boot-Kernel, so dass ich Änderungen im Entwicklungskernel schnell über das JTAG-Interface laden und testen kann.

Außerdem habe ich ein [GitHub-Projekt][5] für SOPHIA aufgesetzt. Dort kann der komplette Quellcode abgerufen werden.


  
  
  
  
    
      Die cargo-Entwickler betonen immer wieder, dass cargo kein komplettes Build-Management-Tool ist und auch nicht sein will. &#8629;
    
  


 [1]: http://sysop.matthias-werner.net/?p=450
 [2]: https://github.com/japaric/xargo
 [3]: http://doc.crates.io/manifest.html
 [4]: http://sysop.matthias-werner.net/?p=307
 [5]: https://github.com/werner-matthias/aihPOS
