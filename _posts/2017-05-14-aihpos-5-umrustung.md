---
layout: page-fullwidth
title: Um-Rust-ung
subheadline: aihPOS - Ein Betriebsystem für die Lehre
published: true
meta_description: "Das Build-System wird auf Rust-Tools umgestellt - make hat ausgedient."
author: mwerner
date: 2017-05-14 
tags:
    - aihpos
    - cargo
    - xargo
categories:
    - aihpos
    - computer
---
**Inhalt**
- TOC
{:toc}
{% include next-previous-post-in-category %}
## Rust auf Nicht-Standard-Plattformen

Nachdem ich im [letzen Teil]({% post_url 2017-04-26-aihpos-4-postman %}) schon wieder eine Bibliothek per Hand einpflegen musste, hatte ich beschlossen, auf das Rust-eigene Management-Tool `cargo` umzusteigen:
Schließlich ist die Einbindung von externen Bibliotheken eine der Stärken von Rust. Dem gegenüber stehen für das SOPHIA-Projekt zwei Nachteile, die die Nutzung von
`cargo` erschweren: 

  * Unsere Zielplattform wird nicht unterstützt, damit auch nicht die entsprechenden Bibliotheken
  * Das Endprodukt (Kernel-Image) ist "untypisch". Seine Erstellung verlangt Post-Linker-Aktionen, die ebenfalls nicht unterstützt werden.

Für beide Probleme gibt es (mindestens) eine Lösung.

### Xargo

Zur Lösung des ersten Problems gibt es [`xargo`][2], eine `cargo`-Variante für Cross-Umgebungen. Diese übersetzt automatisch die Core-Bibliothek und bei Bedarf auch andere Teile der Standard-Bibliothek und "schenkt" dem Compiler die richtige Tool-Chain, indem `rootdir` entsprechend gesetzt wird. Man kann sich damit auch angepasste Versionen der Standard-Bibliothek erstellen -- mal sehen, ob ich das später brauchen werde.

Die Installation geht diesmal nicht über Homebrew, sondern schnell und reibungslos über `cargo`:

{% terminal %}
$ cargo install xargo
{% endterminal %}

`Cargo`/`xargo` gehen von einem bestimmten Directory-Layout und der Existenz von (mindestens) einer Konfigurationsdatei **cargo.toml** aus, die das sogenannte "[Manifest][3]" enthält. Dort kann man auch benötigte Bibliotheken einbinden, die dann von `cargo` automatisch geladen und übersetzt werden. Wir wollten ja die **compiler-buildins**-Bibliothek nutzen. Dies geschieht im **cargo.toml** für unseren Kernel im Abschnitt `[dependencies]` und sieht so aus:

{% highlight toml %}
{%  github_sample werner-matthias/aihPOS/master/kernel/Cargo.toml  0 -1 %}
{% endhighlight %}

Diese Datei muss sich im Wurzelverzeichnis eines Projekts oder Subprojekts befinden. Die Quelldateien sind dann in einem Unterverzeichnis **src/**, das weitere
Verzeichnisse (mit ggf. eigenen **cargo.toml**-Dateien) befinden können. Beim Start einer Übersetzung mit `cargo build` wird ein Verzeichnis **target** angelegt,  in dem
übersetzte Dateien etc. angelegt werden.

Neben dem Manifest gibt es optional noch eine Konfigurationsdatei, ebenfalls im TOML-Format. Sie heißt **config** und befindet sich in einem Unterverzeichnis **.cargo/**. Wir brauchen sie vor allem, weil die Erstellung des Kernels Compiler- bzw. Linkeroptionen benötigt, die nicht Standard sind:

{% highlight toml %}
{%  github_sample   werner-matthias/aihPOS/blob/master/kernel/.cargo/config 0 -1 %}
{% endhighlight %}

Die Trennung der Steuerung von Compiler-Optionen (in den `[profile.*]`-Abschnitten in **cargo.toml** können auch einige Optionen gesteuert werden) ist etwas unglücklich und wird möglicherweise in künftigen Rust/Cargo-Versionen aufgehoben, aber derzeit muss man damit leben.

### Post-Linker Aktionen

Wenn man sich das bisherige **<tt>Makefile</tt>** anschaut, dann sieht man, dass der Building-Prozess nicht -- wie sonst meist üblich -- mit dem Linken endet. Dies ist
in `cargo` nicht vorgesehen. Zwar kennt `cargo` die Möglichkeit, mit Hilfe eines Build-Scriptes Schritte **vor** der Übersetzung auszuführen; etwas ähnliches für danach
wird zwar in unter den Entwicklern diskutiert, wurde aber bisher nicht umgesetzt. Man kann zwar den Linker (in der Target-Datei) umdefinieren, aber ich fürchte, dass eine
solche Lösung schnell unübersichtlich wird. Es ein häufig genutzter und sogar empfohlener[^1] Ansatz ist daher, `cargo` wiederum von einem `Makefile` aufrufen lassen. Das
würde aber  dazu führen, dass Abhängigkeiten z.T. doppelt gepflegt werden müssen. Wenn `cargo`, dann richtig. 

Glücklicherweise ist `cargo` leicht erweiterbar: Wenn `cargo` (oder `xargo`) mit einem unbekannten Befehl gerufen wird, sucht es im Standardpfad nach einem Programm `cargo-`. Dass mache ich mir zunutze und schreibe ein kleines `bash`-Skript `cargo-kernel`, das erst `xargo` aufruft und dann noch den "Rest" erledigt:

{% highlight bash %}
{%  github_sample   /werner-matthias/aihPOS/blob/master/bin/cargo-kernel 0 -1 %}
{% endhighlight %}

Diese Skript muss irgendwo im Ausführungsfad abgelegt werden. Dann kann man die Übersetzung im Rust-Stil anstoßen:

{% terminal %}
$ cargo kernel --target=arm-none-eabihf
   Compiling core v0.0.0 (file:///Users/mwerner/.rustup/toolchains/nightly-x86_64-apple-darwin/lib/rustlib/src/rust/src/libcore)
      Finished release [optimized] target(s) in 22.87 secs
   Compiling compiler_builtins v0.1.0 (https://github.com/rust-lang-nursery/compiler-builtins#f3ace110)
   Compiling aihPOS v0.0.2 (file:///Users/mwerner/Development/aihPOS/aih_pos/kernel)
      warning: crate `aihPOS` should have a snake case name such as `aih_pos`
      |
      = note: #[warn(non_snake_case)] on by default
   Finished dev [optimized + debuginfo] target(s) in 4.8 secs
{% endterminal %} 

Man beachte, dass immer noch `cargo` und nicht `xargo` aufgerufen werden muss. Zwar würde auch `xargo` das Shellskript finden und ausführen, jedoch hat `cargo`/`xargo` einen Lock, der die nebenläufige Ausführung blockiert. `xargo kernel`würde also hängen bleiben.

### Diskussion

Bin ich nun vollständig in der Rust-Welt angekommen? Das hängt von der Betrachtung ab. Ich kann auf `make` verzichten und alles, was ich mit `make` gemacht habe nun mit
Rust-Mitteln tun. Andererseits kennt `cargo` noch viele Befehle -- wie z.B. für das Bauen von Tests und Dokumentationen oder die Arbeit mit Repositories -- die noch nicht
berücksichtigt sind. Einiges könnte einfach so funktionieren, aber es besteht dafür keine Garantie. 

## GitHub

Wenn ich schon bei einer Reorganisation bin, gehe ich noch einen Schritt weiter: Der alte JTAG-Kernel (siehe [Teil 3](/2017/04/10/aihpos-3-lebenszeichen)) kommt in ein eigenes Verzeichnis, so dass er noch immer mit `make` übersetzt werden kann. Ich nutze ihn als Boot-Kernel, so dass ich Änderungen im Entwicklungskernel schnell über das JTAG-Interface laden und testen kann.

Außerdem habe ich ein [GitHub-Projekt](https://github.com/werner-matthias/aihPOS) für SOPHIA aufgesetzt. Dort kann der komplette Quellcode abgerufen werden.
{% include next-previous-post-in-category %}

[^1]:  Die cargo-Entwickler betonen immer wieder, dass cargo kein komplettes Build-Management-Tool ist und auch nicht sein will.
    
 [2]: https://github.com/japaric/xargo
 [3]: http://doc.crates.io/manifest.html

