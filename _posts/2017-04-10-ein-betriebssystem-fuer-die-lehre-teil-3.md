---
layout: post
title: Lebenszeichen
published: true
author: mwerner
comments: true
date: 2017-04-10 09:04:10
tags:
    - Rust
    - SOPHIA
categories:
    - aihpos
    - computer
permalink: /2017/04/10/ein-betriebssystem-fuer-die-lehre-teil-3
---
Nachdem wir unsere [Toolchain][1] installiert haben, wollen wir den ersten &#8222;Kernel&#8220; bauen. Er soll nichts anderes machen, als den Pi für JTAG vorbereiten. Dafür müssen wir uns mit Details unserer Plattform herumschlagen, insbesondere mit der Beschreibung der [I/O-Hardware][2], dem Speicher-Layout und dem Boot-Prozess.

# Boot-Prozess

Unsere ARM-CPU ist nicht die einzige CPU auf dem Raspberry. Vielmehr gibt es einen weiteren Prozessor, die [GPU VideoCore][3]® IV. Anders als die Bezeichnung vermuten lässt, ist die GPU nicht nur für die Grafiksteuerung verantwortlich, sondern übernimmt z.B. auch den Boot-Prozess.1 Wenn der Strom eingeschaltet wird, übernimmt erst mal die GPU die Regie. Sie bootet von einem internen ROM, lädt dann von der SD-Karte die Datei bootcode.bin. Diese kann enthält einen Treiber, um Dateien im ELF-Format lesen. Als nächstes wird die eigentliche GPU-Firmware aus der Datei start.elf geladen. Die Firmware liest u.a. die Datei config.txt (so vorhanden) und konfiguriert das Board entsprechend, und lädt anschließend die Datei kernel.img auf die (aus AMR-Sicht) Adresse 0x08000. Von dieser Adresse aus startet dann auch der ARM.

# Speicher und Formate

## Layout und Linker

Die Datei kernel.img ist ein Binärfile, das unmittelbar Maschinencode enthält, also keine ELF-Datei oder ein anderes höheres Format. Wir müssen also erstens dafür sorgen, dass unser &#8222;Kernel&#8220; von der Adresse 0x08000 lauffähig ist, und zweitens, dass er als reine Binärdatei vorliegt. Beim ersten hilft der Linker. In der &#8222;normalen&#8220; Softwareentwicklung bekommt man vom Linker nicht viel mit, er macht ohne viel Aufsehen das, was man von ihm erwartet. Da wir hier aber ganz bestimmte Anforderungen haben, müssen wir es ihm mit einem [Linker-Skript][4] mitteilen. Unser Linker-Skript ist (noch) sehr einfach:

ENTRY(kernel_main)
SECTIONS
{
    /* Starts at LOADER_ADDR. */
    . = 0x8000;
    .text :
    {
        KEEP(*(.text.kernel_main))
        *(.text)
    }	
    .bss :
    {
        bss = .;
        *(.bss)
    }
}

Es sagt im Wesentlichen, dass unsere Einsprungstelle in den Code kernel_main heißt,2 und dass der Code an Adresse 0x0800 beginnt und zwar mit kernel_main.

## Binärfile

Um aus dem vom Linker generierten ELF-File ein Binärfile zu machen, nutzen wir [`objcopy`][5], genauer die Cross-Variante arm-none-eabi-objcopy. Der Name objcopy ist ein klares Understatement, das dieses Programm nicht nur kopiert, sondern eine Vielzahl von Transformationen ausführen kann. Wir nutzen es allerdings nur dazu, aus dem ELF-File den Binärcode zu generieren, eine Aufgabe, die auf Plattformen _mit_ einem Betriebssystem i.d.R. der Lader übernimmt.

## Makefile

Eigentlich hat Rust sein eigenes Build-System `cargo`, mit dem wir schon die core-Bibliothek gebaut haben. Jedoch ist `cargo` nicht sonderlich gut für bare-metal Programme geeignet. Daher werden wir auf die klassische Form des Makefiles zurückgreifen.3
  
Unser Makefile geht davon aus, dass wir folgendes Verzeichnis-Layout haben:

aihpos
  |
  +- rust-corelib
  |
  +- jtag
      |
      +-- src
      +-- obj
      +-- build


In src sind alle Quelldateien. Die Objektdateien liegen in obj und alle anderen generierten Dateien in build. Das Makefile selbst befindet sich in jtag und sieht so aus:

BUILDDIR=build
SRCDIR=src
OBJDIR=obj
BUILDDIR=build
OBJCOPY=arm-none-eabi-objcopy
OBJDUMP=arm-none-eabi-objdump
RUSTC=rustc
LIBCORE=../rust-libcore/target/arm-none-eabihf/release/libcore.rlib
RUSTFLAGS= --target $(TARGET) -C panic=abort -g --crate-type staticlib --extern core=$(LIBCORE)
LINKFLAGS= -O0 -g -Wl,-gc-sections -mfpu=vfp -mfloat-abi=hard -march=armv6zk -mtune=arm1176jzf-s -nostdlib

TARGET=arm-none-eabihf
MAIN=kernel

IMAGE=$(BUILDDIR)/$(MAIN).img
LIST=$(BUILDDIR)/$(MAIN).list
ELF=$(BUILDDIR)/$(MAIN).elf

vpath %.rs $(SRCDIR)
vpath %.o $(OBJDIR)

SOURCES= main.rs 
OBJ= $(addsuffix .o, $(basename $(SOURCES)))

.PHONY: clean 

all: $(IMAGE) $(LIST)

main.o:	main.rs panic.rs

$(IMAGE): $(ELF)
	$(OBJCOPY) $(ELF) -O binary $(IMAGE)

$(LIST): $(IMAGE)
	$(OBJDUMP) -d $(ELF) &gt; $@

$(ELF): $(OBJ) 
	arm-none-eabi-gcc $(LINKFLAGS) -Tsrc/layout.ld  $(addprefix $(OBJDIR)/, $(OBJ)) -o $@

%.o: %.rs 
	$(RUSTC) $(RUSTFLAGS) $&lt; -o $(OBJDIR)/$@

clean:
	rm -f $(OBJDIR)/*
	rm -f $(BUILDDIR)/*


Nach der Diskussion dürfte es nicht weiter überraschend sein. Einzig das Ziel LIST haben wir noch nicht besprochen. Dies ist der mit Hilfe von objdump disassemblierte Code des erzeugten ELF-Files. Er kann manchmal bei der Fehlersuche nützlich sein.

# Hardware

## JTAG

Der Pi hat 54  universelle Kommunikationspins, die als Ausgabe (Standard beim Einschalten), Eingabe oder mit jeweils bis zu sechs Spezialfunktionen (Alternativfunktionen 0&#8230;5) beschaltet werden kann. Für JTAG werden fünf Steuerleitungen gebraucht, die jeweils auf zwei verschiedene Pins

  * Data Input (_TDI_), serieller Eingang, auf Pin 4 in der Alternativfunktion 5 oder Pin 26  in der Alternativfunktion 4
  * Test Data Output (_TDO_), serieller Ausgang,  auf Pin 5 in der Alternativfunktion 5 oder Pin 24  in der Alternativfunktion 4
  * Test Clock (_TCK_). Das Taktsignal,  auf Pin 13 in der Alternativfunktion 5 oder Pin 25  in der Alternativfunktion 4
  * Test Mode Select (_TMS_), Steuerleitung, auf Pin 12 in der Alternativfunktion 5 oder Pin 27  in der Alternativfunktion 4
  * Test Reset (_TRST_), Reset, auf Pin 12 in der Alternativfunktion 5 oder Pin 22  in der Alternativfunktion 4

Der Adapter (siehe letzter Beitrag) nutzt die Pin 22 bis 27, entsprechen müssen diese auf Alternativfunktion 4 gesetzt werden. Dies geschieht über die GPIO-Select-Register, genauer über GPIOSEL2. Jeweils drei aufeinanderfolgende Bits wählen die Funktion für ein GPIO-Pin aus.

Außerdem muss vorher für diese Pins die Pull-up/down-Steuerung deaktiviert werden. Dabei folgen wir der Beschreibung auf Seite 101 die ARM-Peripherie-Dokumentation.

## LED

Damit wir sehen, dass unser Programm auch läuft, wollen wir die grüne LED des Raspberrys blinken lassen. Diese ist an den GPIO-Pin 47 angeschlossen. Die GPIO-Pins sind in zwei Sätze unterteilt, Satz 1 für Pin 0&#8230;31, und Satz 2 für Pin 32&#8230;53. Pin 47 ist also Pin 15 (wenn man mit Null anfängt zu zählen) des zweiten Satzes.

# Software

## Das erste Rust-Programm

Wir haben jetzt alle Informationen zusammen, um eine Software zu schreiben, die den Raspberry auf einen JTAG-Zugriff vorzubereiten. Wie schon im [Teil 1 dieser Reihe][6] diskutiert, soll Rust benutzt werden, mit &#8212; wenn nötig &#8212; etwas Assembler. In der Regel wird der Startup-Code in Assembler geschrieben, siehe z.B. [hier][7]. Das ist im Allgemeinen auch eine gute Idee: jede Rust-Funktion (oder C-Funktion) hat einen Prolog, in dem die Rücksprungadresse auf dem Stack gesichert wird. Allerdings ist zu Beginn ja noch gar kein Stack angelegt. Dies und andere Initialisierungen macht das Assemblerprogramm, ehe es in den in der höheren Programmiersprache geschriebenen Code ruft.

Allerdings ist ein solches Assembler-Modul in unserem Fall nicht nötig: Einerseits können wir Assembler-Befehle direkt in den Rust-Code einbetten, andererseits können wir unsere Startfunktion mit dem Attribute #[naked] versehen, so dass der Compiler keinen Prolog oder Epilog anlegt.

#![feature(asm,lang_items,core_intrinsics,naked_functions)]
#![no_std]

// Hardware-Adressen
const GPIO_BASE: u32 = 0x20200000;
const GPFSEL2:    *mut u32 = (GPIO_BASE+0x08) as *mut u32;
const GPSET1:     *mut u32 = (GPIO_BASE+0x20) as *mut u32;
const GPCLR1:     *mut u32 = (GPIO_BASE+0x2C) as *mut u32;
const GPPUD:      *mut u32 = (GPIO_BASE+0x94) as *mut u32;
const GPPUDCLK0:  *mut u32 = (GPIO_BASE+0x98) as *mut u32;

fn sleep(value: u32) {  
    for _ in 1..value {
        unsafe { asm!(""); }  // Hack: Compiler kann Schleife nicht wegoptimieren
    }
} 

#[no_mangle]   // Name wird für den Export nicht verändert
#[naked]       // keinen Prolog
pub extern fn kernel_main() {
    // Setze den Stackpointer
    unsafe{ asm!("mov sp, #0x8000");  }
    
    // Pull up/down abschalten
    unsafe{ *GPPUD = 0 };
    sleep(150);
    unsafe{ *GPPUDCLK0 = (1 &lt;&lt; 22) | (1 &lt;&lt; 23) | (1 &lt;&lt; 24) | (1 &lt;&lt; 25) | (1 &lt;&lt; 26) | (1 &lt;&lt; 27) };
    sleep(150);
    unsafe{ *GPPUDCLK0 = 0 };

    // GPIO Pins 22 .. 27 auf alternative Funktion 4 (= 011) setzen
    let mut selection: u32 = unsafe{ *GPFSEL2};
    selection = selection & !((0b111 &lt;&lt;  6)  | (0b111 &lt;&lt;  9) | (0b111 &lt;&lt;  12) | (0b111 &lt;&lt;  15) | (0b111 &lt;&lt;  18) | (0b111 &lt;&lt;  21) );
    selection = selection | (0b011 &lt;&lt;  6) | (0b011 &lt;&lt; 9) | (0b011 &lt;&lt;  12) | (0b011 &lt;&lt;  15) | (0b011 &lt;&lt;  18) | (0b011 &lt;&lt;  21);
    unsafe {  *GPFSEL2 = selection};

    // Als Lebenszeichen lassen wir die grüne LED blinken
    let led_on  = GPSET1;
    let led_off = GPCLR1; 
    loop {
        unsafe { *(led_on) = 1 &lt;&lt; 15; }
        sleep(50000);
        unsafe { *(led_off) = 1 &lt;&lt; 15; }
        sleep(50000);
    }
} 


## Panik!

Nun übersetzen wir dieses Programm.
  



  
    make
  
  
  
    rustc&nbsp;--target&nbsp;arm-none-eabihf&nbsp;-C&nbsp;panic=abort&nbsp;-g&nbsp;--crate-type&nbsp;staticlib&nbsp;--extern&nbsp;core=../rust-libcore/target/arm-none-eabihf/release/libcore.rlib&nbsp;src/main.rs&nbsp;-o&nbsp;build/main.o
  
  
  
    error:&nbsp;language&nbsp;item&nbsp;required,&nbsp;but&nbsp;not&nbsp;found:&nbsp;`panic_fmt`
  
  
  
    error:&nbsp;aborting&nbsp;due&nbsp;to&nbsp;previous&nbsp;error
  
  
  
    make:&nbsp;***&nbsp;[main.o]&nbsp;Error&nbsp;101
  
 Hoppla! Der Linker hat ein undefiniertes Symbol. Ursache dafür ist, dass die Core-Bibliothek nicht völlig frei von externen Referenzen ist. Tust muss wissen, wie es sich in einem Ausnahmefall (Exception) verhalten soll, um evtl. sogar eine Recovery zu ermöglichen, und das ist plattform-spezifisch. Wenn allerdings wie in unserem Fall der der „Kernel“ selbst crashed, ist eine Recovery hoffnungslos. Es ist daher hinreichend, die undefinierten Symbole (es sind nämlich noch ein paar) einfach durch Dummy-Funktionen zu definieren. Dazu legen wir noch eine Datei panic.rs an:

use core::intrinsics;

#[lang = "eh_personality"] extern fn eh_personality() {}

#[no_mangle]
pub extern fn __aeabi_unwind_cpp_pr0() -&gt; ()
{
 loop {}
}

#[no_mangle]
pub extern fn __aeabi_unwind_cpp_pr1() -&gt; ()
{
 loop {}
}

#[allow(non_snake_case)] #[no_mangle]
pub extern "C" fn _Unwind_Resume() -&gt; ! {
 loop {}
}

#[lang = "panic_fmt"]
#[no_mangle]
pub extern fn rust_begin_panic(_msg: core::fmt::Arguments, 
                               _file: &'static str,
                               _line: u32) -&gt; ! {
 unsafe { intrinsics::abort() }
}


Außerdem fügen wir main.rs noch eine Zeile hinzu:

include!("panic.rs");

Dieses Macro funktioniert ähnlich wie das `#include` in C und macht die Datei panic.rs logisch zu einem Bestandteil von main.rs. Dieses Vorgehen ist ein bisschen „unrustig“, da eigentlich in Rust kein Modul in mehr als einer Datei sein soll (dafür ruhig mehrere Module in einer Datei), aber ich habe diese Variante gewählt, um den eigentlichen Funktionscode halbwegs „rein“ zu halten ohne extra ein neues Modul anlegen zu müssen. Nun läuft die Übersetzung durch. Anschließend kopieren wir da Binär-Image auf die SD-Karte,
  



  
     make
  
  
  
    rustc&nbsp;--target&nbsp;arm-none-eabihf&nbsp;-C&nbsp;panic=abort&nbsp;-g&nbsp;--crate-type&nbsp;staticlib&nbsp;--extern&nbsp;core=../rust-libcore/target/arm-none-eabihf/release/libcore.rlib&nbsp;src/main.rs&nbsp;-o&nbsp;obj/main.o
  
  
  
    arm-none-eabi-gcc&nbsp;-O0&nbsp;-g&nbsp;-Wl,-gc-sections&nbsp;-mfpu=vfp&nbsp;-mfloat-abi=hard&nbsp;-march=armv6zk&nbsp;-mtune=arm1176jzf-s&nbsp;-nostdlib&nbsp;-Tsrc/layout.ld &nbsp;obj/main.o&nbsp;-o&nbsp;build/kernel.elf
  
  
  
    arm-none-eabi-objcopy&nbsp;build/kernel.elf&nbsp;-O&nbsp;binary&nbsp;build/kernel.img
  
  
  
    arm-none-eabi-objdump&nbsp;-d&nbsp;build/kernel.elf&nbsp;>&nbsp;build/kernel.list
  
  
  
    cp&nbsp;build/kernel.img&nbsp;/Volume/boot/
  



  
Wenn der Raspberry mit dieser SD gestartet wird, fängt die grüne LED tatsächlich an zu blinken.

# Mit dem Pi reden

Jetzt soll die JTAG-Lommunikation getestet werden. Dazu schreiben wir zwei openocd-Konfigurationsdateien4:

interface jlink


# Broadcom 2835 on Raspberry Pi
telnet_port 4444
gdb_port 3333
adapter_khz 0

if { [info exists CHIPNAME] } {
set _CHIPNAME $CHIPNAME
} else {
set _CHIPNAME raspi
}

reset_config none

if { [info exists CPU_TAPID ] } {
set _CPU_TAPID $CPU_TAPID
} else {
set _CPU_TAPID 0x07b7617F
}
jtag newtap $_CHIPNAME arm -irlen 5 -expected-id $_CPU_TAPID

set _TARGETNAME $_CHIPNAME.arm
target create $_TARGETNAME arm11 -chain-position $_TARGETNAME


In einem Terminal kann jetzt der OpenOCD-Server gestartet werden:
  



  
    openocd&nbsp;-f&nbsp;jlink.cfg&nbsp;-f&nbsp;raspi.cfg
  
  
  
    Open&nbsp;On-Chip&nbsp;Debugger&nbsp;0.10.0+dev-00093-g6b2acc0&nbsp;(2017-03-28-11:17)
  
  
  
    Licensed&nbsp;under&nbsp;GNU&nbsp;GPL&nbsp;v2
  
  
  
    For&nbsp;bug&nbsp;reports,&nbsp;read
  
  
  
    http://openocd.org/doc/doxygen/bugs.html
  
  
  
    adapter&nbsp;speed:&nbsp;1000&nbsp;kHz
  
  
  
    none&nbsp;separate
  
  
  
    Info&nbsp;:&nbsp;auto-selecting&nbsp;first&nbsp;available&nbsp;session&nbsp;transport&nbsp;&#8222;jtag&#8220;.&nbsp;To&nbsp;override&nbsp;use&nbsp;&#8218;transport&nbsp;select&nbsp;&#8218;.
  
  
  
    raspi.arm
  
  
  
    Info&nbsp;:&nbsp;No&nbsp;device&nbsp;selected,&nbsp;using&nbsp;first&nbsp;device.
  
  
  
    Info&nbsp;:&nbsp;J-Link&nbsp;V10&nbsp;compiled&nbsp;Jan&nbsp;9&nbsp;2017&nbsp;17:48:51
  
  
  
    Info&nbsp;:&nbsp;Hardware&nbsp;version:&nbsp;10.10
  
  
  
    Info&nbsp;:&nbsp;VTarget&nbsp;=&nbsp;3.335&nbsp;V
  
  
  
    Info&nbsp;:&nbsp;clock&nbsp;speed&nbsp;1000&nbsp;kHz
  
  
  
    Info&nbsp;:&nbsp;JTAG&nbsp;tap:&nbsp;raspi.arm&nbsp;tap/device&nbsp;found:&nbsp;0x07b7617f&nbsp;(mfg:&nbsp;0x0bf&nbsp;(Broadcom),&nbsp;part:&nbsp;0x7b76,&nbsp;ver:&nbsp;0x0)
  
  
  
    Info&nbsp;:&nbsp;found&nbsp;ARM1176
  
  
  
    Info&nbsp;:&nbsp;raspi.arm:&nbsp;hardware&nbsp;has&nbsp;6&nbsp;breakpoints,&nbsp;2&nbsp;watchpoints
  



  
Nun kann ein weiteres Terminalfenster geöffnet werden, in dem ein Telnet gestartet wird.
  



  
    telnet&nbsp;localhost&nbsp;4444
  
  
  
    Connected&nbsp;to&nbsp;localhost.
  
  
  
    Escape&nbsp;character&nbsp;is&nbsp;&#8218;^]&#8216;.
  
  
  
    Open&nbsp;On-Chip&nbsp;Debugger
  
  
  
    >&nbsp;halt
  
  
  
    target&nbsp;halted&nbsp;in&nbsp;ARM&nbsp;state&nbsp;due&nbsp;to&nbsp;debug-request,&nbsp;current&nbsp;mode:&nbsp;Supervisor
  
  
  
    cpsr:&nbsp;0x600001d3&nbsp;pc:&nbsp;0x000082e0
  
  
  
    >
  



  
Jetzt kann man viele interessante Dinge machen. Beispielsweise können Register ausgelesen oder Speicherstellen gelesen oder geschrieben werden:
  



  
    >&nbsp;reg&nbsp;r1
  
  
  
    r1&nbsp;(/32):&nbsp;0x00007FD4
  
  
  
    >&nbsp;mww&nbsp;0x2020002C&nbsp;0x8000
  
  
  
    >&nbsp;mww&nbsp;0x20200020&nbsp;0x8000
  



  
Die letzten beiden Befehle schalten die grüne LED aus- und wieder an. Der vermutlich wichtigste Befehl ist aber load_image. Damit entfällt das Kopieren eines neuen Images auf die SD-Karte. Die Datei (wahlweise auch im ELF-Format) kann direkt auf den Pi gebracht werden.

Für ein noch etwas detaillierteres Debuggen kann openocd mit dem GNU-Debugger gekoppelt werden. Im Konfigurationsfile wurde der Port 3333 dafür freigegeben. Natürlich darf nicht der lokale Debugger genutzt werden5, sondern der ARM-Cross-Debugger:
  



  
    arm-none-eabi-gdb&nbsp;-q&nbsp;build/kernel.elf
  
  
  
    Reading&nbsp;symbols&nbsp;from&nbsp;build/kernel.elf&#8230;done.
  
  
  
    (gdb)&nbsp;target&nbsp;remote&nbsp;localhost:3333
  
  
  
    Remote&nbsp;debugging&nbsp;using&nbsp;localhost:3333
  
  
  
    0x000082e0&nbsp;in&nbsp;core::mem::swap&nbsp;(x=0x7f98,&nbsp;y=0x7fd4)&nbsp;at&nbsp;~/Development/aihPOS/Code/rust-libcore/rust/src/libcore/mem.rs:448
  
  
  
    448&nbsp;pub&nbsp;fn&nbsp;swap(x:&nbsp;&mut&nbsp;T,&nbsp;y:&nbsp;&mut&nbsp;T)&nbsp;{
  
 Wer will, kann den Debugger auch in eine GUI oder IDE einbinden, z.B. mit der 

[`gdbgui`][8].
  
[][9]Nun kann man auch ohne Betriebssystem (fast) so bequem debuggen, wie man es von der Entwicklung von Desktop-Programmen gewohnt ist. Zwar ist die Arbeit über das JTAG-Interface etwas langsam, aber immer noch viel schneller als wenn man ständig die SD-Karte wechselt.

## Morsezeichen {#morsezeichen}

Manchmal will man eine schnelle Rückmeldung haben, ob eine bestimmte Stelle im Programm erreicht wurde oder ob ein Fehler aufgetreten ist. Solange wir keine Konsole haben, können wir die LED dafür nutzen. Dazu wird unser Blinkprogramm in ein eigenes Modul ausgelagert und parameterisiert:

#![allow(dead_code)]
// Hardware-Adressen
const GPIO_BASE: u32 = 0x20200000;
const GPSET1: *mut u32 = (GPIO_BASE+0x20) as *mut u32;
const GPCLR1: *mut u32 = (GPIO_BASE+0x2C) as *mut u32;

#[derive(Clone,Copy)]
#[repr(u32)]
pub enum Bc {
    Long =  75000,
    Short = 25000,
    Pause = 50000,
}

type BlinkSeq = &'static [Bc];

/* Blinksequenzen */
// einmal blinken
pub const BS_DUMMY: BlinkSeq =   &[Bc::Long];
// Lang, dann ein-, zwei, oder dreimal kurz
pub const BS_ONE: BlinkSeq   =   &[Bc::Long,Bc::Short];
pub const BS_TWO: BlinkSeq   =   &[Bc::Long,Bc::Short,Bc::Short];
pub const BS_THREE: BlinkSeq =   &[Bc::Long,Bc::Short,Bc::Short,Bc::Short];
// SOS (für Panik, wenn keine Konsole zur Verfügung steht: • • •  – – –  • • •
pub const BS_SOS: BlinkSeq   =   &[Bc::Pause,Bc::Short,Bc::Short,Bc::Short,Bc::Pause,Bc::Long,Bc::Long,Bc::Long,Bc::Pause,Bc::Short,Bc::Short,Bc::Short];
// Hi in Morsecode:  • • • •  • • 
pub const BS_HI: BlinkSeq    =   &[Bc::Short,Bc::Short,Bc::Short,Bc::Short,Bc::Pause,Bc::Short,Bc::Short];

#[inline(never)]
fn sleep(value: u32) {  
    for _ in 1..value {
        unsafe { asm!(" "); } 
    }
}

pub fn blink_once(s: BlinkSeq) {
    let led_on  = GPSET1;
    let led_off = GPCLR1; 

    for c in s {
        let sym: Bc = c.clone();
        match sym {
            Bc::Long =&gt; {
                unsafe { *(led_on) = 1 &lt;&lt; 15; }
                sleep(Bc::Long as u32);
                unsafe { *(led_off) = 1 &lt;&lt; 15; }
            },
            Bc::Short =&gt; {
                unsafe { *(led_on) = 1 &lt;&lt; 15; }
                sleep(Bc::Short as u32);
                unsafe { *(led_off) = 1 &lt;&lt; 15; }
            }
            Bc::Pause =&gt; {
                sleep(Bc::Pause as u32);
            }
        }
        sleep(Bc::Short as u32);
    }
}

pub fn blink(s: BlinkSeq) {

    loop {
        blink_once(s);
        sleep(200000);
    }
}


Man beachte, dass die auch hier benutzte Verzögerung durch &#8222;_busy idling_&#8220; so nicht funktioniert, wenn wir die Codeoptimierung einschalten: Dann wird die Schleife einfach wegoptimiert. Allerdings besteht bisher für eine Optimierung noch kein Grund &#8211; die Ausführungszeit ist derzeit noch egal, aber die Übersetzungszeit würde sich etwas verlängern.


  
  
  
  
    
      Die Darstellung des Boot-Prozesses ist etwas vereinfacht. &#8629;
    
    
      Diese Information ist eigentlich überflüssig, da sie sich später im Binärfile nicht mehr wiederfindet. Der Linker braucht sie aber, er würde sonst das Fehlen monieren. &#8629;
    
    
      Vielleicht werde ich in späteren Folgen nochmal cargo oder die Cross-Compile-Variante xargo einsetzen. &#8629;
    
    
      Vergleiche https://github.com/dwelch67/raspberrypi/tree/master/armjtag. &#8629;
    
    
      Schon Garn nicht in meiner Arbeitsumgebung auf dem Mac, da der Debugger hier auf MachO ausgelegt ist &#8629;
    
  


 [1]: http://sysop.matthias-werner.net/ein-betriebssystem-fuer-die-lehre-teil-2/
 [2]: https://www.raspberrypi.org/wp-content/uploads/2012/02/BCM2835-ARM-Peripherals.pdf
 [3]: https://docs.broadcom.com/docs/12358545
 [4]: https://sourceware.org/binutils/docs/ld/Scripts.html
 [5]: https://sourceware.org/binutils/docs/binutils/objcopy.html#objcopy
 [6]: http://sysop.matthias-werner.net/aihpos-ein-betriebssystem-fuer-die-lehre-teil-1/
 [7]: http://wiki.osdev.org/Raspberry_Pi_Bare_Bones
 [8]: https://pypi.python.org/pypi/gdbgui/
 [9]: http://sysop.matthias-werner.net/wp-content/uploads/2017/04/debug.png