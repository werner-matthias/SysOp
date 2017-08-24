---
title: Reihenfolge
layout: page-fullwidth
author: mwerner
show_meta: true
categories: small hacks
meta_description: Moderne Compiler sind gut in der Lage, Code zu optimieren, so dass
  viele Performance-Tricks auf der Hochsprachenebene hinfällig geworden sind. Aber
  manchmal muss man den Compiler auch überzeugen, nicht zuviel des Guten zu tun.
---

Bei [SOPHIA](/blog/ahipos) hatte ich diesr Tage das Phänomen, dass Code, der in der Debugging-Version tadellos lief, in der Release-Version abstürzte. Der Unterschied zwischen beiden Versionen ist -- da ich die Debugsymbole nicht ins Binärfile übernommen hatte[^1]-- lediglich die Optimierung. 

Nach längerer Fehlersuche stellte sich folgende Funktion als Verursacherin heraus:
~~~ rust
#[inline(never)]
fn init_stacks() {
    let adr = determine_irq_stack();
    Cpu::set_mode(ProcessorMode::Irq);
    Cpu::set_stack(adr);
    Cpu::set_mode(ProcessorMode::Fiq);
    Cpu::set_stack(adr);
    Cpu::set_mode(ProcessorMode::Abort);
    Cpu::set_stack(adr);
    Cpu::set_mode(ProcessorMode::Undef);
    Cpu::set_stack(adr);
    // ...und zurück in den Svc-Mode
    Cpu::set_mode(ProcessorMode::Svc);
}
~~~
Das war etwas verwunderlich, da sowohl `Cpu::set_mode()` als auch `Cpu::set_stack()` jeweils nur einen Assembler-Befehl erzeugen sollten:
~~~ rust 
    /// Setzt Prozessormodus
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

    /// Setzt das Stack-Register 
    #[inline(always)]
    pub fn set_stack(adr: Address) {
        unsafe{
            asm!("mov sp, $0"::"r"(adr)::"volatile");
        }
    }
~~~
Der Blick auf den generierten Assembler-Code (via <kbd>objdump</kbd>) zeigt das Problem. Während die Debug-Version so aussah ...
~~~ slim
 000006b4 <_ZN13aihpos_kernel11init_stacks17hbe91fb6176eaf21dE>:
     6b4:	e92d4800 	push	{fp, lr}
     6b8:	e1a0b00d 	mov	fp, sp
     6bc:	ebffffba 	bl	5ac <_ZN13aihpos_kernel19determine_irq_stack17h107193fbe9f3e16eE>
     6c0:	f1020012 	cps	#18
     6c4:	e1a0d000 	mov	sp, r0
     6c8:	f1020011 	cps	#17
     6cc:	e1a0d000 	mov	sp, r0
     6d0:	f1020017 	cps	#23
     6d4:	e1a0d000 	mov	sp, r0
     6d8:	f102001b 	cps	#27
     6dc:	e1a0d000 	mov	sp, r0
     6e0:	f1020013 	cps	#19
     6e4:	e8bd8800 	pop	{fp, pc}
~~~
... lautete die optimierte Version so;
~~~ slim
000010bc <_ZN13aihpos_kernel11init_stacks17h0c7ec04f5f8147a7E>:
    10bc:	e92d4800 	push	{fp, lr}
    10c0:	ebfffe4d 	bl	9fc <_ZN13aihpos_kernel19determine_irq_stack17hc16efbf461b228f8E>
    10c4:	e1a0d000 	mov	sp, r0
    10c8:	f1020012 	cps	#18
    10cc:	f1020011 	cps	#17
    10d0:	f1020017 	cps	#23
    10d4:	f102001b 	cps	#27
    10d8:	f1020013 	cps	#19
    10dc:	e1a0d000 	mov	sp, r0
    10e0:	e1a0d000 	mov	sp, r0
    10e4:	e1a0d000 	mov	sp, r0
    10e8:	e8bd8800 	pop	{fp, pc}
~~~

Der Optimierer hatte also die Befehle umgeordnet. Warum er das für *besser* hielt, ist mir nicht ganz klar. Dass er es allerdings glaubt zu *dürfen*, liegt daran, dass `cps` für ihn kein Lese- oder Schreibebefehl ist, so dass er keine Datenabhängigkeit sah.

Zur Behebung dieses Problems genügt der alte Trick, eine Abhängigkeit zu behaupten. Jetzt sieht der `set_mode()`-Code so aus:

~~~ rust
    #[inline(always)]
    pub fn set_mode(mode: ProcessorMode) {
        unsafe{
            match mode {
                ProcessorMode::User =>   asm!("cps 0x10":::"memory":),
                ProcessorMode::Fiq =>    asm!("cps 0x11":::"memory":),
                ProcessorMode::Irq =>    asm!("cps 0x12":::"memory":),
                ProcessorMode::Svc =>    asm!("cps 0x13":::"memory":),
                ProcessorMode::Abort =>  asm!("cps 0x17":::"memory":),
                ProcessorMode::Undef =>  asm!("cps 0x1B":::"memory":),
                ProcessorMode::System => asm!("cps 0x1F":::"memory":),
            };
        }
    }
~~~
Der vierte Parameter im `asm!()`-Macro sagt normalerweise, welche Register "verschmutzt" werden, so dass der Compiler sich nicht darauf verlassen darf, dass deren Inhalte gleich bleiben. Man kann dort aber auch "memory" hinschreiben, was behauptet, dass der Assember-Code Nebeneffekte hat. Damit darf der Compiler die Reihenfolge nicht mehr ändern, und SOPHIA funktioniert auch mit `--opt-level=3`. 

[^1]: Die Symbole nützten mir auf dem Raspberry Pi nichts, da das Debugging über den Entwicklungsrechner läuft.
