---
title: Braking News
subheadline: Ein Betriebsystem für die Lehre
meta_description: 'So ist das mit "unstable" Versionen\: Kaum kommt man nach zwei Wochen von einer Reise zurück, schon funktioniert die Übersetzung nicht mehr -- in diesem ist
    Fall der Heap kaputt.'
published: true
author: mwerner
date: 2017-07-20
tags:
    - aihPOS
    - Rust
    - Allokator
    - Heap
categories:
    - aihpos
---
**Inhalt**
- TOC
{:toc}
{% include next-previous-post-in-category %}
## SOPHIA, wir haben ein Problem
### Missing Feature
Für fast zwei Wochen war ich in Bulgarien. Danach erhielt ich beim ersten Übersetzungsversuch des SOPHIA-Codes eine Fehlermeldung:
{% terminal %}
error[E0557]: feature has been removed
 --> libs/kalloc/src/lib.rs:1:12
  |
1 | #![feature(allocator)] 
  |            ^^^^^^^^^

error: The attribute `allocator` is currently unknown to the compiler and may have meaning added to it in the future (see issue #29642)
 --> libs/kalloc/src/lib.rs:2:1
  |
2 | #![allocator]
  | ^^^^^^^^^^^^^
  |
  = help: add #![feature(custom_attribute)] to the crate attributes to enable

error: aborting due to 2 previous errors
{% endterminal %}

Natürlich fiel die Fehlermeldung nicht einfach so vom Himmel, ich hatte vorher -- wie ich das regelmäßig tue -- meine Toolchain mit `rustup update` aktualisiert.
Selbstverständlich hätte ich jetzt meine Toolchain auch wieder zurück setzen können, aber ich möchte, dass SOPHIA mit der jeweilig aktuellen Toolchain übersetzbar ist.
Und da ich "nightly" nutze, können solche abrupten Wechsel schon mal vorkommen. Also wird erstmal nichts aus der Erweiterung unseres Betriebssystem, sondern es heißt, den bestehenden Code wieder
lauffähig zu machen.

### Auf der Suche nach dem verlorenen Attribut
Wie die Fehlermeldung angibt, wurde das Feature `#![allocator]` entfernt. Was nun? Ein `rustc --explain E0557` hilft hier nicht weiter, und das angegebene Issue in GitHub
beschäftigt sich mit Customer-Attributen. Vielleicht braucht man das Attribut nicht mehr, es reicht, dass die Funktionen vorhanden sind?
Ein einfaches Auskommentieren hilft nicht:
{% terminal %}
error: no #[default_lib_allocator] found but one is required; is libstd not linked?

error: cannot continue compilation due to previous error

error: Could not compile `aihPOS`.
{% endterminal %}
Aha, sollte das Attribut einfach bloß umbenannt worden sein? Leider nein, `#![allocator]` war ein modulweites Attribut, im Gegensatz zu `#[default_lib_allocator]`. Im
"[Unstable Book](https://doc.rust-lang.org/unstable-book/)" ist `#[default_lib_allocator]` nicht aufgeführt. Allerdings hilft der
GitHup-[Issue](https://github.com/rust-lang/rust/issues/27389) zu  `#![allocator]` weiter: Es verweist auf einen anderen Issue,
[#42727](https://github.com/rust-lang/rust/pull/42727), der wiederum auf anderes verweist, u.a. Issue [#32838](https://github.com/rust-lang/rust/issues/32838) und
die RFCs [RFC 1974](https://github.com/rust-lang/rfcs/blob/master/text/1974-global-allocators.md) und
[RFC 1398](https://github.com/rust-lang/rfcs/blob/master/text/1398-kinds-of-allocators.md). Daraus ergibt sich ein stimmiges Bild: Das Allokator-Interface wurde
überarbeitet und durch eine neue API ersetzt. 

Im Folgendem wird beschrieben, wie das Allokator-Interface nun aussieht. Die entsprechenden Aussagen aus der Folge ["Wir machen einen Haufen"]({% post_url 2017-06-07-aihpos-heap %}) dieser
Reihe sind als überholt zu betrachten.

## Auf zu neuen Ufern
### Allokator-API
Ein Allokator (im allgemeinen) ist eine Struktur, die den `Allocator`-Trait implementiert. Der Trait ist zwar im Crate `alloc` definiert, aber derzeit[^1]
noch nicht in der offiziellen [Dokumentation](https://doc.rust-lang.org/alloc/index.html). Die Quellfiles sind aber ausführlich kommentiert.

Der Trait besteht aus folgenden Funktionen:
~~~ rust
pub unsafe trait Allocator {
    unsafe fn alloc(&mut self, layout: Layout) -> Result<*mut u8, AllocErr>;

    unsafe fn dealloc(&mut self, ptr: *mut u8, layout: Layout);
	
    fn oom(&mut self, _: AllocErr) -> !;
	
    fn usable_size(&self, layout: &Layout) -> (usize, usize);
	
    unsafe fn realloc(&mut self,
                      ptr: *mut u8,
                      layout: Layout,
                      new_layout: Layout) -> Result<*mut u8, AllocErr>;
					  
    unsafe fn alloc_zeroed(&mut self, layout: Layout) -> Result<*mut u8, AllocErr>;
					  
    unsafe fn alloc_excess(&mut self, layout: Layout) -> Result<Excess, AllocErr>;
	
    unsafe fn realloc_excess(&mut self,
                             ptr: *mut u8,
                             layout: Layout,
                             new_layout: Layout) -> Result<Excess, AllocErr>;
							 
    unsafe fn grow_in_place(&mut self,
                            ptr: *mut u8,
                            layout: Layout,
                            new_layout: Layout) -> Result<(), CannotReallocInPlace>;
							
    unsafe fn shrink_in_place(&mut self,
                              ptr: *mut u8,
                              layout: Layout,
                              new_layout: Layout) -> Result<(), CannotReallocInPlace>;

    fn alloc_one<T>(&mut self) -> Result<Unique<T>, AllocErr>
        where Self: Sized;
	
    unsafe fn dealloc_one<T>(&mut self, ptr: Unique<T>)
        where Self: Sized;

    fn alloc_array<T>(&mut self, n: usize) -> Result<Unique<T>, AllocErr>
        where Self: Sized;

    unsafe fn realloc_array<T>(&mut self,
                               ptr: Unique<T>,
                               n_old: usize,
                               n_new: usize) -> Result<Unique<T>, AllocErr>
        where Self: Sized;

    unsafe fn dealloc_array<T>(&mut self, ptr: Unique<T>, n: usize) -> Result<(), AllocErr>
        where Self: Sized;
}
~~~
Müssen jetzt wirklich fünfzehn Funktionen implementiert werden? Und die neuen Typen, `Layout`, `AllocErr`, `Excess`, `CannotReallocInPlace`?
Tatsächlich sieht die Sache viel freundlicher aus: Nur zwei der Funktionen sind obligatorisch, nämlich `alloc()` und `dealloc()`. Alle anderen haben sinnvolle
Standardimplementationen (_defaults_).

`Layout` ist eine Struktur, die einen Speicherrequest beschreibt. Sie hat u.a. die beiden Methoden:
~~~ rust
pub fn size(&self) -> usize;
~~~
und
~~~ rust
pub fn align(&self) -> usize;
~~~

Der Summentyp[^2] `AllocErr` ist -- wie der Name vorschlägt -- für den Fehlerbericht bei der Allozierung zuständig. Er ist wie folgt definiert:
~~~ rust
#[derive(Clone, PartialEq, Eq, Debug)]
pub enum AllocErr {
    Exhausted { request: Layout },
    Unsupported { details: &'static str },
}
~~~
Alle anderen Dinge braucht man zunächst einmal nicht (können aber später zur besseren Effizienz genutzt werden). Es ist somit jetzt einfach, die Kompatibilität zum früheren
Interface wieder herzustellen.

Die Allokator-API gilt für alle Allokatoren, Rust unterstützt Hierarchien von Allokatoren. Am Ende wird irgendwann die Speicherverwaltung des Betriebssystems von die
Standardbibliothek gerufen. Existiert kein solches (wie in unserem Fall) oder wird nur die Standardbibliothek nicht eingebunden, aber trotzdem Crates benutzt, die
Allozierungen vornehmen wollen, muss ein globaler Allokator angegeben werden. Dies wird einfach gemacht, indem einer Instanz einer Allokator-Struktur mit dem Attribut
`#[global_allocator]` dazu erklärt wird. Warum bei der Fehlermeldung oben dieses Attribut -- das übrigens auch noch keinen Eintrag im
[Unstable Book](https://doc.rust-lang.org/unstable-book/) hat -- nicht genannt wurde, sondern `#[default_lib_allocator]` ist mir nicht ganz klar; vermutlich ist das bloß
noch nicht stabilisiert.

### Am Rande des Speichers
Ich nutze die Gelegenheit und implementiere eine eigene Heapverwaltung. Dazu nutze ich das *Boundary-Tag-Verfahren* [^3]. Das hat den Vorteil, dass die Wiedereingliederung
von zurückgegebenen Speicher eine konstante Komplexität \\(\mathcal{O}(1)\\) hat, jedoch auf Kosten des Speicherverbrauchs: Jeder belegte Speicherabschnitt hat zwei Tags
Verwaltungsinformation, in denen die Größe des Abschnitts und Statusflags stehen. In meiner Implementation ist ein Tag ein `usize` (genauer: eine `struct` mit einem
`usize`), so dass jeweils pro Block 8 Bytes Overhead entstehen. Da es durch die Alignmentanforderungen ohnehin zu Verschnitt kommt, halte ich dies für vertretbar.

![]({{site.urlimg}}/boundary.png){:class="img-responsive"}

Um einen Speicherbereich zu belegen, wird einfach die Freispeicherliste durchsucht: 
~~~ rust
{% github_sample werner-matthias/aihPOS/blob/5fbe76b768b21e2b28277a212a8f898bbfb94f74/kernel/src/mem/heap/mod.rs 72 80 %}
~~~
Die eigentliche "Arbeit" liegt bei der `allocate`-Methode des Speicherabschnitts: Hier wir überprüft, ob der Abschnitt als Ganzes belegt werden soll, oder ob er
aufgeteilt wird. Im letzteren Fall wird die Teilung durchgeführt; der abgetrennte Bereich nimmt den Platz des ursprünglichen Abschnittes in der Liste ein.
~~~ rust
{% github_sample werner-matthias/aihPOS/blob/5fbe76b768b21e2b28277a212a8f898bbfb94f74/kernel/src/mem/heap/memory_region.rs 224 260 %}
~~~
Die Komplexität der Reservierung ist immer noch \\(\mathcal{O}(n)\\), wobei \\(n\\) die Anzahl der freien Speicherblöcke ist.
Im Unterschied zur vorherigen Lösung muss bei der Wiedereingliederung die Liste *nicht* durchsucht werden:
~~~ rust
{% github_sample werner-matthias/aihPOS/blob/5fbe76b768b21e2b28277a212a8f898bbfb94f74/kernel/src/mem/heap/mod.rs 82 104 %}
~~~
Das ist natürlich nur ein des Gesamtcodes, der vollständige Code ist wieder
auf [GibHub](https://github.com/werner-matthias/aihPOS/tree/5fbe76b768b21e2b28277a212a8f898bbfb94f74/kernel/src/mem/heap) zu finden.

## Blick zurück in Freude 
Nachdem ich die mein neuer Allokator tadellos lief, stellte ich fest, dass die von mir früher verwendete Version 0.2.7 der `linked_list_allocator`-Bibliothek gar nicht mehr die aktuelle ist. Die
derzeit neuste Version ist 0.4.1 und hat die neue Allokator-API schon vollständig umgesetzt. Danke an den Autor Philipp Oppermann, von dem auch das Blog
"[Writing an OS in Rust](https://os.phil-opp.com)" stammt. Ich werde zwar in SOPHIA bei meinem Boundary-Tag-Allokator bleiben, es ist aber gut zu wissen, dass es eine Alternative gibt
und dass die Community so schnell reagiert.


{% include next-previous-post-in-category %}


[^1]: &nbsp;18. Juli 2017

[^2]: Ich bin mir nicht sicher, was die beste Bezeichnung im Deutschen für `enum` ist. Als C-Programmierer denkt man da an "Aufzählungstyp", was aber in Rust in die Irre
        führt. "Summentyp" ist typentheoretisch korrekt, klingt aber in meinen Ohren trotzdem eigenartig.

[^3]: D. Knuth, “The Art of Computer Programming”, 2nd Auflage, Addison Wesley, 1973, Seiten 441f.

[^4]: Natürlich ist es ein leichter Performance-Verlust, aber Performance ist ja ausdrücklich *kein* [Design-Ziel]({% post_url 2017-03-19-aihpos-1 %}) in SOPHIA. 
