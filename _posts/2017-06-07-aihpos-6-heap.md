---
layout: page-fullwidth
title: Wir machen einen Haufen
subheadline: aihPOS - Ein Betriebsystem für die Lehre
meta_description: 
published: true
author: mwerner
comments: true
date: 2017-06-07
tags:
    - Rust
    - aihpos
    - Microkernel
categories:
    - aihpos
    - computer
show_meta: true	
permalink: 2017/06/07/aihpos6-haufen
---
**Inhalt**
- TOC
{:toc}
{% include next-previous-post-in-category %}
SOPHIA soll ein dynamisches Betriebssystem werden, also eines, bei denen die Anzahl der Prozesse sich zur Laufzeit ändern kann. Unser Mikrokernel hat eine klassische Architektur, die aus vier Schichten besteht:

- Schicht 4: der Kernoperation, der wie Prozessverwaltung, Prozessinteraktion oder Prozessspeichermanagement
- Schicht 3: Prozesswechseloperationen; hier wird die Abstraktion Prozess gewährleistet
- Schicht 2: Datenstrukturen
-  Schicht 1: Kernel-Speicherverwaltung

![]({{site.urlimg}}/mk-architektur.png){:class="img-responsive"}

Damit wäre eine der ersten Aufgaben, sich um den Speicher kümmern.

## Speicher in verschiedenen Geschmacksrichtungen

Die Speicherverwaltung in Betriebssystemen hat verschiedene Aspekte:
- Prozesse und der Kernel brauchen eine Buchführung über bei Operationen wie `malloc` oder `new` allozierten[^1]  Speicher. Das Modell des Speichers ist hier ein linearer
   Adressraum. Die Herausforderung besteht, möglichst wenig Overhead  und eine möglichst kleinen Verschnitt zu produzieren. 
- Der insgesamt zur Verfügung stehende Speicher muss zwischen den einzelnen Prozessen verteilt werden. Hierbei geht es nicht nur um die Reservierung von
   Speicherbereichen, sondern um die Abbildung der Prozesssicht (logische Adressen) auf die Prozessorsicht (physische Adresse). Verwaltungseinheiten sind hier in der Regel
   physische Speicherseiten (Pages) oder Segmente. 
- Die Prozesse und der Kernel müssen vor ungewollten Speichereingriffen anderer Prozessen geschützt werden.
- Häufig  sollen die vorhandenen Speicherreserven durch Nutzung einer Speicherhierachie virtuell vergrößert werden.

SOPHIA soll (zunächst) die ersten drei Aspekte unterstützen; eine virtuelle[^2]  Speicherverwaltung kann erst in Angriff genommen werden, wenn es Treiber für
Speichermedien wie die SD-Karte oder externe USB-Laufwerke gibt.

Dabei muss erst einmal der Kernel selbst seinen Speicher verwalten. Dabei stellt sich natürlich die Frage: wie hat er das in unseren bisherigen Programmen gemacht? Da
haben wir doch auch schon Speicher (in Form von Variablen) benutzt. Das waren einmal statische (aus Programmsicht: globale) Variablen, wie der Framebuffer. Diese haben
eine Lebenzeit, die die Lebenzeit des ganzen Programmes umfasst. Variablen mit beschränkter Lebenzeit muss zur Laufzeit Speicher alloziert werden.
Dies wird auf zwei verschiedene Arten und an zwei verschiedenen Orten realisert[^3] : 
- Auf dem Stack. Dort werden nicht nur Rücksprungadressen gespeichert, sondern auch _lokale_ Variablen einer Funktion. Beim Verlassen der Funktion werden diese
   automatisch aufgeräumt.
- Auf dem Heap. Dort werden Variablen angelegt, die die Laufzeit einer Funktion überdauern. Diese müssen dann entweder manuell weggeräumt werden, oder es existiert eine
   Speicherberäumung (_garbage collection_)

## Stack
### Prozessor-Modi
Bisher haben wir ausschließlich den Stack benutzt, und zwar den Stack im SVC-Mode. Im Gegensatz zu beispielsweise dem x86-Prozessoren kennt ARM nämlich mehrere Stacks.
Diese sind an Prozessor-Modi gebunden. Im einzelnen es beim ARM1176JZF-S folgende Modi:
-	Supervisor-Mode (SVC): Mode, in dem die CPU startet und mit dem wir es bisher zu tun hatten. Er ist privilegiert, hat also die volle Kontrolle über die CPU.
-	User-Mode: unprivilegierter Mode, der für normale Programme genutzt wird. 
-	System-Mode: privilegierter Mode mit der gleichen "Sicht" wie der User-Mode.
-	Interrupt-Mode (IRQ): Mode in dem zur Behandlung eines  Interrupts gewechselt wird.
-	Fast-Interrupt-Mode (FIQ): Mode für die besonders schnelle Behandlung eines  Interrupts.
-	Memory-Abort (ABT): Mode, in dem zur Behandlung eines Traps (also internen Interrupts) gewechselt wird, der durch einen ungültigen Speicherzugriff ausgelöst wird. Ursache können z.B. fehlende Zugriffsrechte sein.
-	Undefined-Instruction-Exception (UND): Ein weiterer Trap-Behandlungsmode, in den gewechselt wird, wenn ein unbekannter Op-Code gelesen wird. Damit kann ein erweiterter Befehlssatz emuliert werden.
-	Secure-Monitor-Mode (MON): Der ARM1176JZF-S hat eine Erweiterung (TrustZone), bei der zwischen einer vertrauenswürdigen und einer unsicheren "Welt" unterschieden wird. Der Secure-Monitor-Mode dient der Interaktion dieser Welten.

Da wir die TrustZone-Erweiterung nicht nutzen wollen, ist der letzte Mode für uns nicht interessant. Um die andern müssen wir uns kümmern. Dabei hat jeder Mode seinen
eigenen Stackpointer und sein eigenes Link-Register. Eine Ausnahme ist der System-Mode, der sich alle Register mit dem User-Mode teilt. Wir werden aber nicht für jeden
Mode einen eigenen Stack anlegen: Da die meisten Modi entweder ihren Stack aufräumen, oder nicht zurückkehren, können sie sich (erst mal) einen Stack teilen. Dieser Stack
soll im weiteren Interrupt-Stack heißen.

### Ja, wo liegen sie nur?
Wo sollen nun die Stacks liegen? Da Stacks traditionell[^4] nach "untern" wachsen, legen wir sie möglichst "hoch".
Da der Interrupt-Stack vorhersagbarer als der SVC-Stack ist, setzen wir ihn ganz an das Ende des verfügbaren Speichers und den SVC-Stack ein Stück weit darunter.
Allerdings kann das Ende des Speichers variieren, da es zur Bootzeit die Aufteilung zwischen ARM und GPU geändert werden kann.
Glücklicherweise haben wir in der [vorletzten Folge](/2017/04/26/aihpos-4-postman) das
Mailbox-System implementiert, mit dem wir ja das Ende des Speichers herausfinden können. 

Allerdings geschieht dies zur Laufzeit -- d.h., wir brauchen bereits einen Stack. Wir müssen also mit einem provisorischen Stack beginnen um später zu wechseln.
Damit sieht unsere Einsprungssequenz in den Kernel jetzt so aus:
~~~ rust
{%  github_sample werner-matthias/aihPOS/blob/9d8377774e8f3a7f9e67e8a8997fbf6eca5d5b96/kernel/src/main.rs 63 122 %}
~~~


### HAL
`Cpu::set_mode()` und `Cpu::set_stack()` sind natürlich keine Funktionen, die Rust bekannt sind. Der Übersichtlichkeit halber habe ich allen Assembler-Code in ein eigenes
Modul gepackt, das "HAL" heißt. HAL steht klassischerweise für "_Hardware Abstraction Layer_". Allerdings ist die Abstraktion in SOPHIA sehr dünn, ein Port über die
ARM-Architektur bedürfte etwas mehr Aufwand. Der Code für `Cpu::set_mode()` und `Cpu::set_stack()` sieht derzeit so aus:
~~~ rust
// Siehe ARM Architectur Reference Manual A2-3
pub enum ProcessorMode {
    User   = 0x10,
    Fiq    = 0x11,
    Irq    = 0x12,
    Svc    = 0x13, 
    Abort  = 0x17,
    Undef  = 0x1B,
    System = 0x1F,
}

pub struct Cpu {}

impl Cpu {
    #[inline(always)]
    pub fn set_mode(mode: ProcessorMode) {
        unsafe{
            match mode {
                ProcessorMode::User =>   asm!("cps 0x10"),
                ProcessorMode::Fiq =>    asm!("cps 0x11"),
                ProcessorMode::Irq =>    asm!("cps 0x12"),
                ProcessorMode::Svc =>    asm!("cps 0x13"),
                ProcessorMode::Abort =>  asm!("cps 0x17"),
                ProcessorMode::Undef =>  asm!("cps 0x1B"),
                ProcessorMode::System => asm!("cps 0x1F"),
            };
        }
    }

    #[inline(always)]
    pub fn set_stack(adr: u32) {
        unsafe{
            asm!("mov sp, $0"::"r"(adr)::"volatile");
        }
    }
}    
~~~
## Heap

{% include alert text=' Wird noch aktualisiert...' %}

{% include next-previous-post-in-category %}
[^1]: Das Verb heißt tatsächlich "allozieren", siehe auch [hier](http://faql.de/fremdwort.html#allocate).

[^2]: Achtung: Mitunter (auch in der ARM-Dokumentation) wird schon von "virtueller Speicherverwaltung" gesprochen, wenn eine streuende Adressabbildung erfolgt. Hier wird
       "Virtualisierung" aber im engeren Sinne verstanden, also so, dass mehr Ressourcen simuliert werden als vorhanden sind. Damit ist die Unterstützung der streuenden
       Adressierung nur ein Teil der Speichervirtualisierung.
	   
[^3]: Es gibt noch mehr Möglichkeiten; hier werden nur die beiden wichtigsten Arten besprochen.

[^4]: Für den ARM ist das lediglich eine Konvention, die Hardware unterstützt auch Stacks, die nach oben wachsen.
