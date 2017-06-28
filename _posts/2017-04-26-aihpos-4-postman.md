---
layout: page-fullwidth
title: Please Mr. Postman
subheadline: aihPOS - Ein Betriebsystem für die Lehre
meta_description: "Bevor wir uns dem Design des Kernels widmen, wollen wir Informationen über das System einsammeln:  welchen Raspberry-Typ haben wir, wieviel Speicher
steht zur Verfügung, etc.. Außerdem nutzen wir die Gelegenheit, das Debugging nochmal zu verbessern."
published: true
author: mwerner
date: 2017-04-26 11:04:55
tags:
    - Rust
    - aihpos
categories:
    - aihpos
    - computer
---
**Inhalt**
- TOC
{:toc}
{% include next-previous-post-in-category %}
Bevor wir uns dem Design des Kernels widmen, wollen wir Informationen über das System einsammeln. Auch wenn mein Testboard ein Raspberry Pi 1B+ ist, sollte ein
Betriebssystem dafür doch mindestens so portabel sein, dass es auf verschiedenen Raspberries laufen kann. Dazu wäre es z.B. wichtig zu erfahren, welchen Raspberry-Typ wir
haben, wieviel Speicher zur Verfügung steht und wo die I/O-Hardware liegt.

## Mailboxen

Glücklicherweise kümmert sich um das alles die GPU. Diese hat auch eine Schnittstelle, über die es mit dem ARM reden kann, das sogenannte [Mailbox-System][1]. Mailboxes
sind FIFO-Speicher mit einer Breite von 28 Bit. Jede Mailbox ist in Kanäle unterteilt, die für unterschiedliche Arten von Informationen vorgesehen ist. Es kann pro
Mailbox maximal 16 Kanäle geben. Daten und Kanalinformationen werden mit einem Datenwort übergeben, wobei die Kanalnummer die untersten vier Bit einnimmt und das zu
übertragende Datum die oberen 28. Auch wenn die "größeren" Raspberries eine ganze Menge von Mailboxen haben, interessiert uns im Moment nur die Mailbox 0
(bzw. Mailbox 1 für die Rückantwort). Mailbox 0 hat 9 Kanäle. Für die Datenübertragung muss jeweils auf den passenden Status gewartet werden. Wir machen das im
Moment mit einem Busy Wait. Allerdings kann man auch wesentlich eleganter vorgehen und einen Interrupt auslösen lassen, wenn Daten bereit stehen. 

{% highlight rust linenos %}
{%  github_sample   werner-matthias/aihPOS/blob/master/kernel/src/hal/board/mailbox.rs 0 -1 %}
{% endhighlight %}

## Property Tags

Der für uns interessante ist Kanal 8. Über ihn werden Eigenschaften abgefragt und gesetzt, jeweils über einen _property tag_. Einige dieser Tags werden auch als [ATAG][2]
dem Kernel beim Start zur Verfügung gestellt, aber der Kanal 8 bietet mehr. Diese Property Tag werden -- sowohl bei der Abfrage als auch bei der Antwort -- über eine verkettete Liste übertragen, die folgende Struktur hat:

![]({{site.urlimg}}/tag-list.png){:class="img-responsive"}

Die Tag-ID bestimmt, welche Art von Information übertragen wird. Und auch wenn im Moment noch nicht alle Informationen wichtig sind, definieren wir schon mal alle Tags "auf Vorrat"in einem enum:

{% highlight rust linenos %}
{%  github_sample   werner-matthias/aihPOS/blob/master/kernel/src/hal/board/propertytags.rs 8 93 %}
{% endhighlight %}

Bei der Antwort auf einen Request wird der Frage-Puffer überschrieben. Es muss also dafür gesorgt werden, dass genügend Platz für die Antwort zur Verfügung steht. Je nach
[Tag-Typ][4] ist das unterschiedlich:
{% highlight rust linenos %}
{%  github_sample   werner-matthias/aihPOS/blob/master/kernel/src/hal/board/propertytags.rs 95 190 %}
{% endhighlight %}

Da für die über den Kanal ausgetauschte Information wenige Bytes mitunter nicht ausreichen, werden über die Mailbox Adressen eine Puffers kommuniziert, die die
eigentliche Information enthalten. Selbst eine solche Adresse kann mit 28 Bit nicht vollständig dargestellt werden: Es werden nur die obersten 28 Bit der Adresse werden
übertragen, der Rest als 0 angenommen. Damit muss ein solcher Puffer ein entsprechendes Alignment haben, d.h., seine Adresse muss mit 0000 enden. 

### Alignment

In Standard-Rust ein bestimmtes Alignment zu erreichen, ist ohne externe Hilfe unmöglich.[^1] Natürlich können wir im Linkerfile einen statischen Speicherbereich mit entsprechenden Alignment anlegen und von Rust aus nutzen, aber das ist erstens unsicher und zweitens ist dieser Speicher dann für diesen Zweck reserviert. Schöner wäre es, wenn wir den Puffer dynamisch anlegen können, und -- da wir noch keine Heap-Verwaltung haben -- natürlich auf dem Stack.

Glücklicherweise bietet nightly Rust hier Möglichkeiten. Bis vor kurzem musste man tricksen und mit Hilfe von `#[repr(simd)]`  behaupten, dass eine Vektorverarbeitung
vorgenommen wird. Seit neuesten[^2] steht ein eigenes Alignment-Attribut zur Verfügung. Bei seiner Nutzung muss man beachten, dass die Syntax aus dem entsprechenden
[RFC 1358][5] falsch ist, statt z.B. `#[repr(align="16")]` wird das Alignment als (ebenfalls relativ neues) "Attribut-Literal"angegeben:
`#[repr(align(16))]`. Es müssen also gleich zwei Featuregates freigeschaltet werden, `#![feature(repr_align)]` und `#![feature(attr_literals)]`. 

### Tag-Puffer

Die Implementation des Tag-Puffers ist unkompliziert. Man muss lediglich darauf achten, dass die Daten `u32`-Wörter sind, aber die Größen stets in Byte gemessen werden. Entsprechend ist ab und zu eine Multiplikation oder Division mit 4 notwendig, die durch die Shift-Operatoren `<<` realisiert werden.

{% highlight rust linenos %}
{%  github_sample   werner-matthias/aihPOS/blob/master/kernel/src/hal/board/propertytags.rs 192 -1 %}
{% endhighlight %}

### Zuwenig Speicher?

Mit Hilfe der Property-Tags können jetzt gewünschte Informationen erlangt werden. Um nicht mit den "rohen" Puffer-Daten umgehen zu müssen, habe ich
Schnittstellenfunktionen geschrieben:

{% highlight rust linenos %}
{%  github_sample   werner-matthias/aihPOS/blob/master/kernel/src/hal/board/mod.rs 0 -1 %}
{% endhighlight %}

Die Nutzung dieser Funktionen gab für mich zwei Überraschungen:

  1. Bei der Revisionsnummer des Boards hatte ich bei einem Raspberry Pi 1B+ den Wert 0x10 erwartet. Tatsächlich erhielt ich 0x13, der in einigen Verzeichnissen nicht
     gelistet ist. [Hier][6] ist eine vermutlich vollständige Liste; demnach handelt es sich tatsächlich um einen 1B+, aber mit geänderten Leiterplattenlayout. 
  2. Es wurden 256 MByte Speicher berichtet, obwohl der 1B+ doch 512 MByte haben sollte, von denen die GPU standardmäßig lediglich 64 MByte "abzweigt". Eine kurze
     Web-Recherche ergab, dass es sich dabei um einen bekannten Bug der Firmware handelt, für den auch ein entsprechender Bugfix existiert. Wenn man in das
     Boot-Verzeichnis die Datei fixup.dat aus dem originalen Boot-Verzeichnis kopiert, werden korrekt 448 MByte verfügbarer Speicher gemeldet. 

## Kprint
Der Property-Tags-Kanal der Mailbox hat noch viel mehr parat: Mit Hilfe der Property Tags kann nämlich der Framebuffer konfiguriert werden. [Blinksignale][7] zum Debuggen
sind ja gut und schön, aber ein richtiger Text ist doch bequemer.  Zwar hat die Mailbox dafür auch einen eigenen Kanal, aber der existierte bevor die Property Tags hinzu
kamen und kann nicht ganz soviel wie diese.[^3] 

Der Framebuffer ist ein Bereich im Speicher, in dem die Bildschirmdaten pixelweise gespeichert sind.[^4] Er kann für verschiedene Bildschirmauflösungen und Farbtiefen
konfiguriert werden. Der Raspberry-Framebuffer unterscheidet zwischen einer virtuellen und einer physischen Bildschirmauflösung. Erstere muss immer größer oder gleich der
physischen Bildschirmauflösung sein. Die physische Bildschirmauflösung ist das sichtbare Fenster, da über die Parameter `x_offset` und `y_offset` über dem virtuellen
positioniert werden kann. 

Zunächst definieren wir eine Struktur:
{% highlight rust linenos %}
{%  github_sample   werner-matthias/aihPOS/blob/master/kernel/src/debug/framebuffer.rs 12 29 %}
{% endhighlight %}

`screen` ist dabei die Referenz auf den eigentlichen Framebuffer-Speicherbereich, die anderen Felder sollten selbsterklärend sein. `screen` und die resultierende
Speicherzeilenlänge (`pitch`) werden bei der Initialisierung abgefragt, nachdem die Grundparameter vorgegeben werden:

{% highlight rust linenos %}
{%  github_sample   werner-matthias/aihPOS/blob/master/kernel/src/debug/framebuffer.rs 33 87 %}
{% endhighlight %}

Wir wählen einen virtuellen Puffer, der doppelt so hoch ist wie die vertikale Bildschirmauflösung. Damit können wir ein kontinuierliches Scrollen erreichen,[^5] indem wir einen virtuellen Ringpuffer aufbauen, siehe Abschnitt "[Scrollen][8]".

![]({{site.urlimg}}/virphys-fb.png){:class="img-responsive"}
### Vom Pixel zum String

Wenn jetzt ein Pixel in einer bestimmten Farbe dargestellt werden soll, muss entsprechender Wert in den Puffer geschrieben werden. Der Farbwert (im eingestellten Modus) setzt sich folgendermaßen zusammen:

| **Bit** | **24 -- 31** | **16 -- 23** | **8 -- 15** | **0 -- 7** |
|------|:----------:|:-----------:|:---------:|:---------|
|            | Transparenz  | Rotanteil         | Grünanteil  | Blauanteil   |
|            | (nicht genutzt) | (0--255) | (0--255) | (0--255) |
{% highlight rust linenos %}
{%  github_sample   werner-matthias/aihPOS/blob/master/kernel/src/debug/framebuffer.rs 88 90 %}
{% endhighlight %}

Jetzt zeigt es sich von Vorteil, dass wir eine Farbtiefe gewählt haben, die der Wortbreite entspricht: Dadurch wird die Indexierung sehr einfach. Bei 24 Bit Farbtiefe,
die z.B. auch möglich ist, wäre es etwas aufwendiger gewesen.[^6]  Damit wir auch Text ausgeben können, brauchen wir einen Font. Freundlicherweise hat auf
[Stackoverflow][10] jemand einen kompletten ASCII-Font für C bereit gestellt, der für SOPHIA einfach an Rust angepasst und um die deutschen Umlaute erweitert wurde: 

{% highlight rust linenos %}
// Die Daten für diesen Font stammen von http://stackoverflow.com/a/23130671, mit kleinen Modifikationen
pub trait Font {
    fn glyph_width()  -> u32;
    fn glyph_height() -> u32;
    fn glyph_pixel(char: u8, row: u32, col: u32) -> Option<bool>;
}

pub struct SystemFont {}

impl Font for SystemFont {
    fn glyph_width() -> u32 {8}
    fn glyph_height() -> u32 {13}
    fn glyph_pixel(char: u8, row: u32, col: u32) -> Option<bool>  {
        if (row >= Self::glyph_height()) || (col >= Self::glyph_width()) { return None };
        let glyph =
        match char {
            32  =>[0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8],// space
            33 => [0x00u8, 0x00u8, 0x18u8, 0x18u8, 0x00u8, 0x00u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8],// ! 
            34 => [0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x36u8, 0x36u8, 0x36u8, 0x36u8],// "
            35 => [0x00u8, 0x00u8, 0x00u8, 0x66u8, 0x66u8, 0xffu8, 0x66u8, 0x66u8, 0xffu8, 0x66u8, 0x66u8, 0x00u8, 0x00u8],// #
            36 => [0x00u8, 0x00u8, 0x18u8, 0x7eu8, 0xffu8, 0x1bu8, 0x1fu8, 0x7eu8, 0xf8u8, 0xd8u8, 0xffu8, 0x7eu8, 0x18u8],
{% endhighlight %}
  &#8230;
{% highlight rust linenos %}
124 => [0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8],// |
            125 => [0x00u8, 0x00u8, 0xf0u8, 0x18u8, 0x18u8, 0x18u8, 0x1cu8, 0x0fu8, 0x1cu8, 0x18u8, 0x18u8, 0x18u8, 0xf0u8],// }
            // Umlaute
            228 => [0x00u8, 0x00u8, 0x7fu8, 0xc3u8, 0xc3u8, 0x7fu8, 0x03u8, 0xc3u8, 0x7eu8, 0x00u8, 0x00u8, 0xc6u8, 0x00u8],// ä
            252 => [0x00u8, 0x00u8, 0x7eu8, 0xc6u8, 0xc6u8, 0xc6u8, 0xc6u8, 0xc6u8, 0xc6u8, 0x00u8, 0x00u8, 0xc6u8, 0x00u8],// ü
            246 => [0x00u8, 0x00u8, 0x7cu8, 0xc6u8, 0xc6u8, 0xc6u8, 0xc6u8, 0xc6u8, 0x7cu8, 0x00u8, 0x00u8, 0xc6u8, 0x00u8],// ö
            196 => [0x00u8, 0x00u8, 0xc3u8, 0xc3u8, 0xc3u8, 0xc3u8, 0xffu8, 0xc3u8, 0xc3u8, 0xc3u8, 0x66u8, 0x3cu8, 0xc3u8],// Ä
            220 => [0x00u8, 0x00u8, 0x7eu8, 0xe7u8, 0xc3u8, 0xc3u8, 0xc3u8, 0xc3u8, 0xc3u8, 0xc3u8, 0xc3u8, 0x00u8, 0xc3u8],// Ü
            214 => [0x00u8, 0x00u8, 0x7eu8, 0xe7u8, 0xc3u8, 0xc3u8, 0xc3u8, 0xc3u8, 0xc3u8, 0xe7u8, 0xfeu8, 0x3cu8, 0xc3u8],// Ö
            223 => [0xc0u8, 0xc0u8, 0xceu8, 0xc7u8, 0xc3u8, 0xc3u8, 0xc7u8, 0xceu8, 0xc7u8, 0xc3u8, 0xc3u8, 0x67u8, 0x3eu8],// ß
            _ => [0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x06u8, 0x8fu8, 0xf1u8, 0x60u8, 0x00u8, 0x00u8, 0x00u8],
            
        };
        let glyph_row = glyph[row as usize];
        let val: bool = (glyph_row >> col) & 0x1 != 0;
        Some(val)
    }
}
{% endhighlight %}

Ein Textstring kann jetzt im Framebuffer Zeichen für Zeichen, ein Zeichen ggf. interpretiert und dann pixelweise ausgegeben werden. Dabei wird immer Buch über die aktuelle Zeichenposition &#8211; d.h. Spalte und Zeile &#8211; geführt und entsprechend angepasst. Bei der Interpretation beschränken wir uns erst einmal auf die Steuerzeichen für "neue Zeile"und "Tabulator&#8220;.

{% highlight rust linenos %}
pub fn print(&mut self, s: &str) {
        for c in s.chars() {
                self.putchar(c as u8);
        }
    }

    pub fn putchar(&mut self, c: u8) {
        if (self.row+1) * SystemFont::glyph_height() - self.y_offset > self.height {
            self.scroll();
        }
        match c as char {
            '\n' => {
                self.row += 1;
                self.col = 0;
            },
            '\t'  => {
                for _ in 0..4 {
                    self.putchar(' ' as u8);
                }
            },
            _ => {
                let (icol,irow) = (self.col, self.row); // Copy 
                self.draw_glyph(c, icol * (SystemFont::glyph_width()+1), irow * SystemFont::glyph_height());
                
                self.col += 1;
                if self.col * (SystemFont::glyph_width()+1) >= self.width {
                    self.row += 1;
                    self.col = 0;
                }
            }
        }
    }

    pub fn draw_glyph(&mut self, char: u8, x: u32, y: u32) {
        for row in 0..SystemFont::glyph_height() {
            for col in 0..SystemFont::glyph_width() {
                let p = SystemFont::glyph_pixel(char, row, col ) ;
                let color = match p {
                    Some(true) => self.fg_color,
                    _ => self.bg_color
                };
                self.draw_pixel(color, x + SystemFont::glyph_width() - 1 - col, y + SystemFont::glyph_height() - 1 - row)
            }
        }
    }
{% endhighlight %}

Für die formatierte Ausgabe von Werten kann bequemerweise die Rust-Core-Bibliothek genutzt werden. Dazu muss unser Framebuffer den Trait `fmt::Write` implementieren:

{% highlight rust linenos %}
impl<'a> fmt::Write for Framebuffer<'a> { 
    fn write_str(&mut self, s: &str) -> fmt::Result {
        self.print(s);
         Ok(())
    }
}
{% endhighlight %}

### Scrollen {#scrollen}

Nun muss noch die Scroll-Logik implementiert werden. Die Idee ist sehr einfach: Die eigentliche Verschiebung erfolgt auch die Änderung von `y_offset`, wofür wieder die Mailbox benutzt wird. Sobald das sichtbare Fenster nach unten geschoben wurde, wird eine Bildschirmzeile nach oben in den jetzt verstecken Bereich kopiert. Der noch nicht sichtbare Bereich unten wird von evtl. vorhandenen alten Daten bereinigt. Wenn die maximale Verschiebung erreicht ist, springt der sichtbare Ausschnitt wieder nach oben, was durch das vorherige Kopieren aber wie ein einfaches Scrollen aussieht.

{% highlight rust linenos %}
pub fn scroll(&mut self) {
        // kopiere letzte Zeile
        if self.y_offset > SystemFont::glyph_height() {
            for y in self.y_offset -2*SystemFont::glyph_height()..self.y_offset - SystemFont::glyph_height() {
                for x in 0..self.width {
                    self.screen[(y*self.width +x) as usize] = self.screen[((y+self.height+SystemFont::glyph_height()-2)*self.width +x) as usize];
                }
            }
        }
        if self.y_offset + SystemFont::glyph_height() < self.height { // solange das Fenster noch nicht das Ende des Puffers erreicht hat...
            // lösche ggf. alten Inhalt
            for y in (self.height+self.y_offset)..cmp::min(2*self.height,self.height+self.y_offset+ 2* SystemFont::glyph_height()) {
                for x in 0..self.width {
                    self.screen[(y*self.width +x) as usize] = self.bg_color;
                }
            }
            // versetze Fenster um eine Zeilenhöhe, 
            self.y_offset = self.y_offset + SystemFont::glyph_height();
        } else {
            // sonst gehe zum Pufferbeginn
            self.row = self.row - (self.height / SystemFont::glyph_height()) - 1;
            self.y_offset = 0;
            // lösche letzte Zeile
            for y in self.height - (self.height % SystemFont::glyph_height())..self.height {
                for x in 0..self.width {
                    self.screen[(y*self.width +x) as usize] = self.bg_color;
                }
            }
        }
        let mut prob_tag_buf: PropertyTagBuffer = PropertyTagBuffer::new();
        let mb = mailbox(0);
        prob_tag_buf.init();
        prob_tag_buf.add_tag_with_param(Tag::SetVirtualOffset,Some(&[0,self.y_offset]));
        mb.write(Channel::ATags, &prob_tag_buf.data as *const [u32; BUFFER_SIZE] as u32);
        mb.read(Channel::ATags);
    }
{% endhighlight %}


### (No) Race Codition

Um den Framebuffer zum Debuggen zu nutzen, muss er initialisiert werden. Wo und wann soll das geschehen? Offensichtlich soll er überall zur Verfügung stehen. Da es nicht
sinnvoll ist, eine entsprechende Variable als Funktionsparameter an sämtliche Funktionen weiterzureichen, muss dies Variable global sein, in Rust seit das statisch
(Schlüsselwort `static`). Da aber nicht nur lesend auf den Framebuffer zugegriffen wird, muss sie auch veränderbar (_mutable_) sein. In Rust gilt aber jeder Zugriff auf
eine veränderbare statische Variable als unsicher! Durch einen nebenläufigen Zugriff kann es bei einer solchen Variable nämlich zu Wettlaufsituationen (_race conditions_)
kommen, die dann zu Inkonsistenzen führen. 

Nebenläufiger Zugriff? Denn haben wir doch gar nicht, solange wir keine Interrupts einschalten. Das ist zwar richtig, aber Rust weiß das doch nicht. Es kommt also darauf
an, Rust davon zu überzeugen. Natürlich könnten wir auch einfach alle unsicheren Zugriffe in einem Modul kapseln, aber da das Problem ein grundsätzliches ist, was
vielleicht noch häufiger auftritt, ist eine generische Lösung vorzuziehen. 

Im Gegensatz zu veränderbaren statischen Variablen sind unverändertere (_immutable_) überhaupt kein Problem. Leider ist eine unverändertere Frambuffer-Struktur
nutzlos. Rust kennt aber das Konzept der internen Veränderbarkeit (_Interior mutability_). Die Idee ist, eine unveränderbare Referenz auf einen veränderbaren Wert zu
haben. Damit brauchen wir erst mal kein veränderbares `static` mehr. Allerdings muss Rust immer noch überzeugt werden, dass der Zugriff auf den veränderbaren Wert bei
Nebenläufigkeit sicher ist.

Um mit Nebenläufigkeit umzugehen, kennt Rust zwei Traits: Send und `Sync`. Es sind sogenannte Marker-Traits, die selbst keine Methoden verlangen. `Send` sagt aus, dass es
bei nebenläufiger Nutzung eines Wertes keine _race condition_ gibt, während `Sync` dies für die nebenläufige Nutzung seiner Referenz zusichert. Beide Traits sind selbst
wieder unsicher, was heißt, dass der Compiler die Einhaltung der Garantien nicht überprüfen kann. 

{% highlight rust linenos %}
use core::cell::UnsafeCell;

pub struct NoConcurrency<T: Sized> {
    data: UnsafeCell<T>,
}

unsafe impl<T: Sized + Send> Sync for NoConcurrency<T> {}
unsafe impl<T: Sized + Send> Send for NoConcurrency<T> {}

impl<'b, T> NoConcurrency<T>{
    pub const fn new(data: T) -> NoConcurrency<T> {
        NoConcurrency{
            data: UnsafeCell::new(data)
        }
    }

    pub fn set(&self, data: T) {
        let r = self.data.get();
        unsafe { *r = data;}
    }

    pub fn get(&self) -> &mut T {
        let r =self.data.get();
        unsafe { &mut (*r)}
    }
}
{% endhighlight %}

## Der Linker zickt mal wieder

Bei der Übersetzung des Codes entsteht ein Linkerfehler:

{% terminal %}  
    error: linking with `arm-none-eabi-gcc` failed: exit code: 1
    /Users/mwerner/Development/aihPOS/xargo/aih_pos/src/hal/board/framebuffer.rs:198: undefined reference to `__aeabi_uidiv`
    /Users/mwerner/Development/aihPOS/xargo/aih_pos/src/hal/board/framebuffer.rs:201: undefined reference to `__aeabi_uidivmod`
{% endterminal %}  

Eine kurze Recherche zeigt, wo er herkommt: Unser Prozessor kann keine Ganzzahldivision (aber Gleitkommadivision!) und der Compiler verlässt sich darauf, dass die
Plattform entsprechenden Code zur Verfügung stellt, was wir aber bisher nicht machen. Da es einige dieser Befehle gibt, verzichte ich darauf, sie alle per Hand zu
definieren und greife auf die [Compiler-Buildins-Bibliothek][11] zurück, die in der "Rust Language Nursery" zur Verfügung gestellt wird. 

Nun habe ich schon zwei Rust-Bibliotheken (core und compiler-buildins), die händisch eingepflegt werden, wenn es zu Updates bei ihnen oder dem Compiler kommt. Höchste
Zeit, das Projekt auf Rust-Tools umzustellen, die dies automatisch erledigen. Dies wird das Thema des nächsten Beitrags sein. 
{% include next-previous-post-in-category %}

 [1]: https://github.com/raspberrypi/firmware/wiki/Mailboxes
 [2]: http://www.simtec.co.uk/products/SWLINUX/files/booting_article.html#appendix_tag_reference
 [3]: http://sysop.matthias-werner.net/wp-content/uploads/2017/05/tag-list.png
 [4]: https://github.com/raspberrypi/firmware/wiki/Mailbox-property-interface
 [5]: https://github.com/rust-lang/rfcs/blob/master/text/1358-repr-align.md
 [6]: http://elinux.org/RPi_HardwareHistory#Which_Pi_have_I_got.3F
 [7]: http://sysop.matthias-werner.net/?p=307#morsezeichen
 [8]: #scrollen
 [9]: http://sysop.matthias-werner.net/wp-content/uploads/2017/04/virphys-fb.png
 [10]: http://stackoverflow.com/a/23130671
 [11]: https://github.com/rust-lang-nursery/compiler-builtins

[^1]: Das ist nicht ganz präzise: Rust hält das "natürliche" Alignment ein, das ggf. durch `#[repr(packed)]` unterlaufen werden kann.
[^2]: Version 6d841da vom 22. April 2017 
[^3]: Eine weitere Alternative für das Debuggen wäre auch noch die Nutzung der UART. 
[^4]: Diese Darstellung ist etwas vereinfacht. 
[^5]: Genau genommen bräuchte man dafür nur ein oder zwei "Reservezeilen", aber so sind wir etwas flexibler. 
[^6]: Es wäre dann vermutlich besser, wenn ein `u8`-Array genutzt würde. 
