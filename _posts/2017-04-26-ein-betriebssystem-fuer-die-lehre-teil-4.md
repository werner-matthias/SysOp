---
layout: post
title: Please Mr. Postman
published: true
author: mwerner
comments: true
date: 2017-04-26 11:04:55
tags:
    - Rust
    - SOPHIA
categories:
    - aihpos
    - computer
permalink: /2017/04/26/ein-betriebssystem-fuer-die-lehre-teil-4
---
Bevor wir uns dem Design des Kernels widmen, wollen wir Informationen über das System einsammeln. Auch wenn mein Testboard ein Raspberry Pi 1B+ ist, sollte ein Betriebssystem dafür doch mindestens so portabel sein, dass es auf verschiedenen Raspberries laufen kann. Dazu wäre es wichtig zu erfahren, welchen Raspberry-Typ wir haben, wieviel Speicher zur Verfügung steht und wo die I/O-Hardware liegt.

# Mailboxen

Glücklicherweise kümmert sich um das alles die GPU. Diese hat auch eine Schnittstelle, über die es mit dem ARM reden kann, das sogenannte [Mailbox-System][1]. Mailboxes sind FIFO-Speicher mit einer Breite von 28 Bit. Jede Mailbox ist in Kanäle unterteilt, die für unterschiedliche Arten von Informationen vorgesehen ist. Es kann pro Mailbox maximal 16 Kanäle geben. Daten und Kanalinformationen werden mit einem Datenwort übergeben, wobei die Kanalnummer die untersten vier Bit einnimmt und das zu übertragende Datum die oberen 28. Auch wenn die &#8222;größeren&#8220; Raspberries eine ganze Menge von Mailboxen haben, interessiert uns im Moment nur die Mailbox 0 (bzw. Mailbox 1 für die Rückantwort). Mailbox 0 hat 9 Kanäle. Für die Datenübertragung muss jeweils auf den entsprechenden Status gewartet werden. Wir machen das im Moment mit einem Busy Wait. Allerdings kann man auch wesentlich eleganter einen Interrupt auslösen lassen, wenn Daten bereit stehen.

~~~ Rust
use core::intrinsics::volatile_load;
use core::mem::transmute;
    
const MAILBOX_BASE: u32 = 0x2000B880;

#[allow(dead_code)]
#[derive(Clone,Copy)]
#[repr(u32)]
pub enum Channel {
    PowerManagement = 0,
    Framebuffer,
    VirtualUart,
    Vchiq,
    Leds,
    Buttons,
    Touchscreen,
    Unused,
    ATags,
    IATags
}

const MAILBOX_FULL:  u32 = 1 &lt;&lt; 31;
const MAILBOX_EMPTY: u32 = 1 &lt;&lt; 30;

#[allow(dead_code)]
#[repr(C)]
pub struct Mailbox {
    pub read:    u32,      // 0x00
    _unused: [u32; 3],     // 0x04 0x08 0x0C
    pub poll:    u32,      // 0x10 
    pub sender:  u32,      // 0x14
    pub status:  u32,      // 0x18
    pub configuration: u32,// 0x1C
    pub write:   u32,      // 0x20 Mailbox 1!
}

impl Mailbox {
    pub fn write(&mut self, channel: Channel, addr: u32) {
        assert!(addr & 0x0Fu32 == 0);
        loop{ 
            if unsafe{volatile_load(&mut self.status)} & MAILBOX_FULL == 0 { break }; 
        }
        self.write =addr | channel as u32; 
    }

    pub fn read(&mut self, channel: Channel) -&gt; u32 {
        loop {
            if (unsafe{volatile_load(&mut self.status)} & MAILBOX_EMPTY == 0) && (self.read & 0xF == channel as u32)
                { break };
        }
        self.read &gt;&gt; 4 
    }
}

pub fn mailbox(nr: u8) -&gt; &'static mut Mailbox {
    match nr{
        0 =&gt; unsafe{transmute(MAILBOX_BASE)},
        _ =&gt; panic!()
    }
}
~~~


# Property Tags

Der für uns interessante ist Kanal 8. Über ihn werden Eigenschaften abgefragt und gesetzt, jeweils über einen _property tag_. Einige dieser Tags werden auch als [ATAG][2] dem Kernel beim Start zur Verfügung gestellt, aber der Kanal 8 bietet mehr. Diese Property Tag werden &#8211; sowohl bei der Abfrage als auch bei der Antwort über eine verkettete Liste übertragen, die folgende Struktur hat:[][3]

Die Tag-ID bestimmt, welche Art von Information übertragen wird. Und auch wenn im Moment noch nicht alle Informationen wichtig sind, definieren wir schon mal alle Tags &#8222;auf Vorrat&#8220; in einem enum:

~~~ Rust
#[derive(Clone,Copy)]
#[repr(u32)]
pub enum Tag {
    /// Ende der Liste
    None = 0,
    /// Firmware
    GetFirmwareVersion = 0x1,

    /// Board
    GetBoardModel = 0x10001,
    GetBoardRevision,
    GetBoardMacAddress,
    GetBoardSerial,
    GetArmMemory,
    GetVcMemory,
    GetClocks,

    /// Befehlszeile
    GetCommandLine = 0x50001,

    /// DMA
    GetDmaChannels = 0x60001,

    /// Powermanagement
    GetPowerState = 0x20001,
    GetTiming = 0x20002,
    SetPowerState = 0x28001,

    /// Uhren
    GetClockState = 0x30001,
    SetClockState = 0x38001,
    GetClockRate = 0x30002,
    SetClockRate = 0x38002,
    GetMaxClockRate = 0x30004,
    GetMinClockRate = 0x30007,
    GetTurbo = 0x30009,
    SetTurbo = 0x38009,

    /// Spannungsregelung
    GetVoltage = 0x30003,
    SetVoltage = 0x38003,
    GetMaxVoltage = 0x30005,
    GetMinVoltage = 0x30008,
    GetTemperature = 0x30006,
    GetMaxTemperature = 0x3000A,
    AllocateMemory = 0x3000C,
    LockMemory = 0x3000D,
    UnlockMemory = 0x3000E,
    ReleaseMemory = 0x3000F,
    ExecuteCode = 0x30010,
    GetDispmanxResHandle = 0x30014,
    GetEDIDBlock = 0x30020,

    /// Framebuffer
    AllocateFrameBuffer = 0x40001,
    ReleaseFrameBuffer = 0x48001,
    BlankScreen = 0x40002,
    GetPhysicalDisplaySize = 0x40003,
    TestPhysicalDisplaySize = 0x44003,
    SetPhysicalDisplaySize = 0x48003,
    GetVirtualDisplaySize = 0x40004,
    TestVirtualDisplaySize = 0x44004,
    SetVirtualDisplaySize = 0x48004,
    GetDepth = 0x40005,
    TestDepth = 0x44005,
    SetDepth = 0x48005,
    GetPixelOrder = 0x40006,
    TestPixelOrder = 0x44006,
    SetPixelOrder = 0x48006,
    GetAlphaMode = 0x40007,
    TestAlphaMode = 0x44007,
    SetAlphaMode = 0x48007,
    GetPitch = 0x40008,
    GetVirtualOffset = 0x40009,
    TestVirtualOffset = 0x44009,
    SetVirtualOffset = 0x48009,
    GetOverscan = 0x4000A,
    TestOverscan = 0x4400A,
    SetOverscan = 0x4800A,
    GetPalette = 0x4000B,
    TestPalette = 0x4400B,
    SetPalette = 0x4800B,
    SetCursorInfo = 0x8011,
    SetCursorState = 0x8010
}
~~~

Bei der Antwort auf einen Request wird der Frage-Puffer überschrieben. Es muss also dafür gesorgt werden, dass genügend Platz für die Antwort zur Verfügung steht. Je nach [Tag-Typ][4] ist das unterschiedlich:
~~~ Rust
struct ReqProperty {
    pub tag:  Tag,
    pub buf_size: usize,
    pub param_size: usize,
}

impl ReqProperty {
    fn new(tag: Tag) -&gt; ReqProperty {
        let (buf_size, param_size) = // buf_size ist in Bytes, param_size in u32
            match tag {
                Tag::GetFirmwareVersion|
                Tag::GetBoardModel |
                Tag::GetBoardRevision |
                Tag::GetBoardMacAddress |
                Tag::GetBoardSerial |
                Tag::GetArmMemory |
                Tag::GetVcMemory |
                Tag::GetDmaChannels |
                Tag::GetPhysicalDisplaySize |
                Tag::GetVirtualDisplaySize |
                Tag::GetVirtualOffset
                =&gt; (8,0),
                Tag::GetClocks |
                Tag::GetCommandLine
                =&gt; (256,0),
                Tag::GetPowerState |
                Tag::GetTiming |
                Tag::GetClockState |
                Tag::GetClockRate |
                Tag::GetMaxClockRate |
                Tag::GetMinClockRate |
                Tag::GetTurbo |
                Tag::GetVoltage |
                Tag::GetMaxVoltage |
                Tag::GetMinVoltage |
                Tag::GetTemperature |
                Tag::GetMaxTemperature |
                Tag::GetDispmanxResHandle |
                Tag::AllocateFrameBuffer
                =&gt; (8,1),
                Tag::TestPhysicalDisplaySize |
                Tag::SetPhysicalDisplaySize |
                Tag::TestVirtualDisplaySize |
                Tag::SetVirtualDisplaySize |
                Tag::TestVirtualOffset |
                Tag::SetVirtualOffset |
                Tag::SetPowerState |
                Tag::SetClockState |
                Tag::SetTurbo |
                Tag::SetVoltage
                =&gt; (8,2),
                Tag::SetAlphaMode |
                Tag::SetPixelOrder |
                Tag::SetDepth |
                Tag::LockMemory |
                Tag::ReleaseMemory |
                Tag::UnlockMemory |
                Tag::BlankScreen |
                Tag::TestDepth |
                Tag::TestPixelOrder |
                Tag::TestAlphaMode
                =&gt; (4,1),
                Tag::GetAlphaMode |
                Tag::GetPixelOrder |
                Tag::GetPitch |
                Tag::GetDepth
                =&gt; (4,0),
                Tag::GetOverscan  
                =&gt; (16,0),
                Tag::SetOverscan |
                Tag::TestOverscan
                =&gt; (16,4),
                Tag::SetClockRate
                =&gt; (12,3),  // Antwortgröße: 8
                Tag::AllocateMemory
                =&gt; (12,3),  // Antwortgröße: 4
                Tag::ExecuteCode
                =&gt; (28,1),
                Tag::GetEDIDBlock
                =&gt; (136,1),
                Tag::ReleaseFrameBuffer
                =&gt; (0,0),
                Tag::GetPalette
                =&gt; (1024,0),
                Tag::TestPalette |
                Tag::SetPalette
                =&gt; (1032,258),  // maximale Größe
                Tag::SetCursorInfo
                =&gt; (24,6),
                Tag::SetCursorState
                =&gt; (16,4),
                Tag::None =&gt; (0,0)
        };
        ReqProperty{ tag: tag, buf_size: buf_size, param_size: param_size}
    }
}
~~~

Da für die über den Kanal ausgetauschte Information wenige Bytes mitunter nicht ausreichen, werden über die Mailbox Adressen eine Puffers kommuniziert, die die eigentliche Information enthalten. Selbst eine solche Adresse kann mit 28 Bit nicht vollständig dargestellt werden: Es werden nur die obersten 28 Bit der Adresse werden übertragen, der Rest als 0 angenommen. Damit muss ein solcher Puffer ein entsprechendes Alignment haben, d.h., seine Adresse muss mit 0000 enden.

## Alignment

In Standard-Rust ein bestimmtes Alignment zu erreichen, ist ohne externe Hilfe unmöglich.1 Natürlich können wir im Linkerfile einen statischen Speicherbereich mit entsprechenden Alignment anlegen und von Rust aus nutzen, aber das ist erstens unsicher und zweitens ist dieser Speicher dann für diesen Zweck reserviert. Schöner wäre es, wenn wir den Puffer dynamisch anlegen können, und -- da wir noch keine Heap-Verwaltung haben -- natürlich auf dem Stack.

Glücklicherweise bietet nightly Rust hier Möglichkeiten. Bis vor kurzem musste man tricksen und mit Hilfe von `#[repr(simd)]`  behaupten, dass eine Vektorverarbeitung vorgenommen wird. Seit neuesten[^2] steht ein eigenes Alignment-Attribut zur Verfügung. Bei seiner Nutzung muss man beachten, dass die Syntax aus dem entsprechenden [RFC 1358][5] falsch ist, statt z.B. #[repr(align=&#8220;16&#8243;)]  wird das Alignment als (ebenfalls relativ neues) &#8222;Attribut-Literal&#8220; angegeben:  #[repr(align(16))]. Es müssen also gleich zwei Featuregates freigeschaltet werden, `#![feature(repr_align)]` und `#![feature(attr_literals)]`.

## Tag-Puffer

Die Implementation des Tag-Puffers ist unkompliziert. Man muss lediglich darauf achten, dass die Daten u32-Wörter sind, aber die Größen stets in Byte gemessen werden. Entsprechend ist ab und zu eine Multiplikation oder Division mit 4 notwendig, die durch die Shift-Operatoren (&#8222;&#8222;) realisiert werden.

enum TagReqResp {
    Request = 0,
    Response = 1 &lt;&lt; 31
}

enum PbOffset {
    Size = 0,
    Code = 1
}

enum TagOffset {
    Id = 0,
    Size = 1,
    ReqResp = 2,
    StartVal = 3,
}

#[repr(C)]
#[repr(align(16))]
pub struct PropertyTagBuffer {
    pub data: [u32; BUFFER_SIZE],
    pub index:    usize,
}

impl PropertyTagBuffer {

    pub fn new() -&gt; PropertyTagBuffer {
        PropertyTagBuffer{
            data: [0; BUFFER_SIZE],
            index: 2,
        }
    }

    pub fn init(&mut self)  {
        self.index = 2;
        self.data[PbOffset::Size as usize] = 12; // Size + Code + Endtag
        self.data[PbOffset::Code as usize] = ReqResCode::Request as u32;
        self.data[self.index] = Tag::None as u32;
    }

    fn write(&mut self, data: u32){
        self.data[self.index] = data;
        self.data[PbOffset::Size as usize] += mem::size_of::&lt;u32&gt; as u32;
        self.index += 1;
    }

    pub fn add_tag_with_param(&mut self, tag: Tag,  params: Option&lt;&[u32]&gt;) {
        let old_index = self.index;
        let prop = ReqProperty::new(tag);
        self.write(prop.tag as u32);
        self.write(prop.buf_size as u32);
        self.write(TagReqResp::Request as u32);
        match params {
            Some(array) =&gt; {
                assert!(array.len() == prop.param_size); // ToDo: Überprüfung zur Compilezeit wäre besser
                for p in array.into_iter() {
                    self.write(*p);
                }
            },
            None =&gt; {}
        }
        self.index = old_index + 3 + (prop.buf_size &gt;&gt; 2) ;
        self.data[self.index] = Tag::None as u32; 
        self.data[PbOffset::Size as usize] = ((self.index +1) &lt;&lt; 2) as u32; 
    }

    fn read(&mut self) -&gt; u32 {
        self.index + 1;
        self.data[self.index - 1]
    }

    pub fn get_answer(&self, tag: Tag) -&gt; Option&lt;&[u32]&gt; {
        if self.data[PbOffset::Code as usize] != ReqResCode::Success as u32 {
            return None
        }
        // Es wird ein lokaler Index benutzt, so dass der PropertyTagBuffer nicht geändert wird
        let mut index: usize = 2;

        while (index as u32) &lt; (self.data[PbOffset::Size as usize]) {
            if (self.data[index] == tag as u32) &&
                (self.data[index + TagOffset::ReqResp as usize] & TagReqResp::Response as u32 == TagReqResp::Response as u32) {
                let to = index + TagOffset::StartVal as usize +
                    ((self.data[index + TagOffset::Size as usize] & !(TagReqResp::Response as u32)) &gt;&gt; 2) as usize;
                let ret = self.data.get(index + TagOffset::StartVal as usize .. to);
                return ret;
            }
            index += (self.data[index+TagOffset::Size as usize] &gt;&gt; 2) as usize + 3;
        }
        None
    }
}


## Zuwenig Speicher?

Mit Hilfe der Property-Tags können jetzt gewünschte Informationen erlangt werden. Um nicht mit den &#8222;rohen&#8220; Puffer-Daten umgehen zu müssen, habe ich Schnittstellenfunktionen geschrieben:

pub enum BoardReport {
    FirmwareVersion,
    BoardModel,
    BoardRevision,
    SerialNumber
}

pub fn report_board_info(kind: BoardReport) -&gt; u32 {  
    let mut prob_tag_buf: PropertyTagBuffer = PropertyTagBuffer::new();
    prob_tag_buf.init();
    let tag = match kind {
        BoardReport::FirmwareVersion =&gt; Tag::GetFirmwareVersion,
        BoardReport::BoardModel      =&gt; Tag::GetBoardModel,
        BoardReport::BoardRevision   =&gt; Tag::GetBoardRevision,
        BoardReport::SerialNumber    =&gt; Tag::GetBoardSerial
    };
    prob_tag_buf.add_tag_with_param(tag,None);
    let mb = mailbox(0);
    mb.write(Channel::ATags, &prob_tag_buf.data as *const [u32; self::propertytags::BUFFER_SIZE] as u32);
    mb.read(Channel::ATags);
    match prob_tag_buf.get_answer(tag) {
        Some(n) =&gt; n[0],
        _       =&gt; 0
    }
}

#[allow(dead_code)]
pub enum MemReport {
    ArmStart,
    ArmSize,
    VcStart,
    VcSize,
}

pub fn report_memory(kind: MemReport) -&gt; u32 {
    let mut prob_tag_buf = PropertyTagBuffer::new();
    prob_tag_buf.init();
    let tag = match kind {
        MemReport::ArmStart | MemReport::ArmSize =&gt; Tag::GetArmMemory,
        MemReport::VcStart  | MemReport::VcSize  =&gt; Tag::GetVcMemory
    };    
    prob_tag_buf.add_tag_with_param(tag,None);
    let mb = mailbox(0);
    mb.write(Channel::ATags, &prob_tag_buf.data as *const [u32; self::propertytags::BUFFER_SIZE] as u32);
    mb.read(Channel::ATags);
    let array = prob_tag_buf.get_answer(tag);
    match array {
        Some(a) =&gt; match kind {
            MemReport::ArmStart | MemReport::VcStart =&gt; a[0],
            MemReport::ArmSize  | MemReport::VcSize  =&gt; a[1]
        },
        None =&gt; 0
    }
}

Die Nutzung dieser Funktionen gab für mich zwei Überraschungen:

  1. Bei der Revisionsnummer des Boards hatte ich bei einem Raspberry Pi 1B+ den Wert 0x10 erwartet. Tatsächlich erhielt ich 0x13, der in einigen Verzeichnissen nicht gelistet ist. [Hier][6] ist eine vermutlich vollständige Liste, demnach handelt es sich tatsächlich um einen 1B+, aber mit geänderten Leiterplattenlayout.
  2. Es wurden 256 MByte Speicher berichtet, obwohl der 1B+ doch 512 MByte haben sollte, von denen die GPU standardmäßig lediglich 64 MByte &#8222;abzweigt&#8220;. Eine kurze Web-Recherche ergab, dass es sich dabei um einen bekannten Bug der Firmware handelt, für den auch ein entsprechender Bugfix existiert. Wenn man in das Boot-Verzeichnis die Datei fixup.dat aus dem originalen Boot-Verzeichnis kopiert, werden korrekt 448 MByte verfügbarer Speicher gemeldet.

# Kprint

Der Property-Tags-Kanal der Mailbox hat noch viel mehr parat: Mit Hilfe der Property Tags kann nämlich der Framebuffer konfiguriert werden. [Blinksignale][7] zum Debuggen sind ja gut und schön, aber ein richtiger Text ist doch bequemer.  Zwar hat die Mailbox dafür auch einen eigenen Kanal, aber der existierte bevor die Property Tags hinzu kamen und kann nicht ganz soviel wie diese.3

Der Framebuffer ist ein Bereich im Speicher, in dem die Bildschirmdaten pixelweise gespeichert sind.4 Er kann für verschiedene Bildschirmauflösungen und Farbtiefen konfiguriert werden. Der Raspberry-Framebuffer unterscheidet zwischen einer virtuellen und einer physischen Bildschirmauflösung. Erstere muss immer größer oder gleich der physischen Bildschirmauflösung sein. Die physische Bildschirmauflösung ist das sichtbare Fenster, da über die Parameter `x_offset` und `y_offset` über dem virtuellen positioniert werden kann.

Zunächst definieren wir eine Struktur:

#[allow(dead_code)]
pub struct Framebuffer&lt;'a&gt; {
    screen: &'a mut[u32],
    width: u32,
    height: u32,
    depth: u32,
    row: u32,
    col: u32,
    pitch: u32,
    fg_color: u32,
    bg_color: u32,
    virtual_width:  u32,
    virtual_height: u32,
    x_offset: u32,
    y_offset: u32,
    size: u32,
}

`screen` ist dabei die Referenz auf den eigentlichen Framebuffer-Speicherbereich, die anderen Felder sollten selbsterklärend sein. `screen` und die resultierende Speicherzeilenlänge (`pitch`) werden bei der Initialisierung abgefragt, nachdem die Grundparameter vorgegeben werden:

pub fn new() -&gt;  Framebuffer&lt;'a&gt; {
        /* Die Voreinstellung des Bootloaders wird ignoriert.
           Gegebenenfalls sollten die Einstellungen abgefragt und verwendet werden. Allerdings erfordert dies mindestens
           bei der Farbtiefe größere Änderungen. */
        let mut prob_tag_buf: PropertyTagBuffer = PropertyTagBuffer::new();
        prob_tag_buf.init();
        prob_tag_buf.add_tag_with_param(Tag::SetPhysicalDisplaySize,Some(&[FB_WIDTH,FB_HEIGHT]));
        prob_tag_buf.add_tag_with_param(Tag::SetVirtualDisplaySize,Some(&[FB_WIDTH,2*FB_HEIGHT]));
        prob_tag_buf.add_tag_with_param(Tag::SetDepth,Some(&[FB_COLOR_DEP]));
        prob_tag_buf.add_tag_with_param(Tag::AllocateFrameBuffer,Some(&[16]));
        prob_tag_buf.add_tag_with_param(Tag::GetPitch,None);
        let mb = mailbox(0);
        mb.write(Channel::ATags, &prob_tag_buf.data as *const [u32; BUFFER_SIZE] as u32);
        mb.read(Channel::ATags);
        let ret = prob_tag_buf.get_answer(Tag::AllocateFrameBuffer);
        let mut adr: &'a mut[u32];
        let size: usize;
        match ret {
            Some(a) =&gt; {
                size = a[1] as usize;
                adr  = unsafe{ slice::from_raw_parts_mut(a[0] as *mut u32, size)};
            }
            _   =&gt; {
                blink::blink(blink::BS_SOS);
                unreachable!();
                
            }
        };
        
        let pitch: u32;
        let ret = prob_tag_buf.get_answer(Tag::GetPitch);
        match ret {
            Some(a) =&gt; {
                pitch = a[0];
            }
            _ =&gt; {
                blink::blink(blink::BS_SOS);
                unreachable!();
            }
        }        
        let fb = Framebuffer {
            screen: adr,
            width: FB_WIDTH,
            height: FB_HEIGHT,
            depth: FB_COLOR_DEP,
            row: 0, col: 0,
            pitch: pitch,
            fg_color: DEF_COLOR, bg_color: DEF_BG_COLOR,
            virtual_width: FB_WIDTH,
            virtual_height: 2*FB_HEIGHT,
            x_offset: 0,
            y_offset: 0,
            size: size as u32,
        };
        fb
    }

Wir wählen einen virtuellen Puffer, der doppelt so hoch ist wie die vertikale Bildschirmauflösung. Damit können wir ein kontinuierliches Scrollen erreichen,5 indem wir einen virtuellen Ringpuffer aufbauen, siehe Abschnitt &#8222;[Scrollen][8]&#8222;.[][9]

## Vom Pixel zum String

Wenn jetzt ein Pixel in einer bestimmten Farbe dargestellt werden soll, muss entsprechender Wert in den Puffer geschrieben werden. Der Farbwert (im eingestellten Modus) setzt sich folgendermaßen zusammen:

[table id=1 /]

fn draw_pixel(&mut self, color: u32, x: u32, y: u32) {
        self.screen[(y*self.width + x) as usize] =  color;
    }


Jetzt zeigt es sich von Vorteil, dass wir eine Farbtiefe gewählt haben, die der Wortbreite entspricht, dadurch wird die Indexierung sehr einfach. Bei 24 Bit Farbtiefe, die z.B. auch möglich gewesen wäre, wäre es etwas aufwendiger gewesen.6 Damit wir auch Text ausgeben können, brauchen wir einen Font. Freundlicherweise hat auf [Stackoverflow][10] jemand einen kompletten ASCII-Font für C bereit gestellt, der für SOPHIA einfach an Rust angepasst und um die deutschen Umlaute erweitert wurde:

// Die Daten für diesen Font stammen von http://stackoverflow.com/a/23130671, mit kleinen Modifikationen
pub trait Font {
    fn glyph_width()  -&gt; u32;
    fn glyph_height() -&gt; u32;
    fn glyph_pixel(char: u8, row: u32, col: u32) -&gt; Option&lt;bool&gt;;
}

pub struct SystemFont {}

impl Font for SystemFont {
    fn glyph_width() -&gt; u32 {8}
    fn glyph_height() -&gt; u32 {13}
    fn glyph_pixel(char: u8, row: u32, col: u32) -&gt; Option&lt;bool&gt;  {
        if (row &gt;= Self::glyph_height()) || (col &gt;= Self::glyph_width()) { return None };
        let glyph =
        match char {
            32  =&gt;[0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8],// space
            33 =&gt; [0x00u8, 0x00u8, 0x18u8, 0x18u8, 0x00u8, 0x00u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8],// ! 
            34 =&gt; [0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x36u8, 0x36u8, 0x36u8, 0x36u8],// "
            35 =&gt; [0x00u8, 0x00u8, 0x00u8, 0x66u8, 0x66u8, 0xffu8, 0x66u8, 0x66u8, 0xffu8, 0x66u8, 0x66u8, 0x00u8, 0x00u8],// #
            36 =&gt; [0x00u8, 0x00u8, 0x18u8, 0x7eu8, 0xffu8, 0x1bu8, 0x1fu8, 0x7eu8, 0xf8u8, 0xd8u8, 0xffu8, 0x7eu8, 0x18u8],



  &#8230;


124 =&gt; [0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8, 0x18u8],// |
            125 =&gt; [0x00u8, 0x00u8, 0xf0u8, 0x18u8, 0x18u8, 0x18u8, 0x1cu8, 0x0fu8, 0x1cu8, 0x18u8, 0x18u8, 0x18u8, 0xf0u8],// }
            // Umlaute
            228 =&gt; [0x00u8, 0x00u8, 0x7fu8, 0xc3u8, 0xc3u8, 0x7fu8, 0x03u8, 0xc3u8, 0x7eu8, 0x00u8, 0x00u8, 0xc6u8, 0x00u8],// ä
            252 =&gt; [0x00u8, 0x00u8, 0x7eu8, 0xc6u8, 0xc6u8, 0xc6u8, 0xc6u8, 0xc6u8, 0xc6u8, 0x00u8, 0x00u8, 0xc6u8, 0x00u8],// ü
            246 =&gt; [0x00u8, 0x00u8, 0x7cu8, 0xc6u8, 0xc6u8, 0xc6u8, 0xc6u8, 0xc6u8, 0x7cu8, 0x00u8, 0x00u8, 0xc6u8, 0x00u8],// ö
            196 =&gt; [0x00u8, 0x00u8, 0xc3u8, 0xc3u8, 0xc3u8, 0xc3u8, 0xffu8, 0xc3u8, 0xc3u8, 0xc3u8, 0x66u8, 0x3cu8, 0xc3u8],// Ä
            220 =&gt; [0x00u8, 0x00u8, 0x7eu8, 0xe7u8, 0xc3u8, 0xc3u8, 0xc3u8, 0xc3u8, 0xc3u8, 0xc3u8, 0xc3u8, 0x00u8, 0xc3u8],// Ü
            214 =&gt; [0x00u8, 0x00u8, 0x7eu8, 0xe7u8, 0xc3u8, 0xc3u8, 0xc3u8, 0xc3u8, 0xc3u8, 0xe7u8, 0xfeu8, 0x3cu8, 0xc3u8],// Ö
            223 =&gt; [0xc0u8, 0xc0u8, 0xceu8, 0xc7u8, 0xc3u8, 0xc3u8, 0xc7u8, 0xceu8, 0xc7u8, 0xc3u8, 0xc3u8, 0x67u8, 0x3eu8],// ß
            _ =&gt; [0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x00u8, 0x06u8, 0x8fu8, 0xf1u8, 0x60u8, 0x00u8, 0x00u8, 0x00u8],
            
        };
        let glyph_row = glyph[row as usize];
        let val: bool = (glyph_row &gt;&gt; col) & 0x1 != 0;
        Some(val)
    }
}


Ein Textstring kann jetzt im Framebuffer Zeichen für Zeichen, ein Zeichen ggf. interpretiert und dann pixelweise ausgegeben werden. Dabei wird immer Buch über die aktuelle Zeichenposition &#8211; d.h. Spalte und Zeile &#8211; geführt und entsprechend angepasst. Bei der Interpretation beschränken wir uns erst einmal auf die Steuerzeichen für &#8222;neue Zeile&#8220; und &#8222;Tabulator&#8220;.

pub fn print(&mut self, s: &str) {
        for c in s.chars() {
                self.putchar(c as u8);
        }
    }

    pub fn putchar(&mut self, c: u8) {
        if (self.row+1) * SystemFont::glyph_height() - self.y_offset &gt; self.height {
            self.scroll();
        }

        match c as char {
            '\n' =&gt; {
                self.row += 1;
                self.col = 0;
            },
            '\t'  =&gt; {
                for _ in 0..4 {
                    self.putchar(' ' as u8);
                }
            },
            _ =&gt; {
                let (icol,irow) = (self.col, self.row); // Copy 
                self.draw_glyph(c, icol * (SystemFont::glyph_width()+1), irow * SystemFont::glyph_height());
                
                self.col += 1;
                if self.col * (SystemFont::glyph_width()+1) &gt;= self.width {
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
                    Some(true) =&gt; self.fg_color,
                    _ =&gt; self.bg_color
                };
                self.draw_pixel(color, x + SystemFont::glyph_width() - 1 - col, y + SystemFont::glyph_height() - 1 - row)
            }  
        }
    }


Für die formatierte Ausgabe von Werten kann bequemerweise die Rust-Core-Bibliothek genutzt werden. Dazu muss unser Framebuffer den Trait `fmt::Write` implementieren:

impl&lt;'a&gt; fmt::Write for Framebuffer&lt;'a&gt; { 
    fn write_str(&mut self, s: &str) -&gt; fmt::Result {
        self.print(s);
         Ok(())
    }
}

## Scrollen {#scrollen}

Nun muss noch die Scroll-Logik implementiert werden. Die Idee ist sehr einfach: Die eigentliche Verschiebung erfolgt auch die Änderung von `y_offset`, wofür wieder die Mailbox benutzt wird. Sobald das sichtbare Fenster nach unten geschoben wurde, wird eine Bildschirmzeile nach oben in den jetzt verstecken Bereich kopiert. Der noch nicht sichtbare Bereich unten wird von evtl. vorhandenen alten Daten bereinigt. Wenn die maximale Verschiebung erreicht ist, springt der sichtbare Ausschnitt wieder nach oben, was durch das vorherige Kopieren aber wie ein einfaches Scrollen aussieht.

pub fn scroll(&mut self) {
        // kopiere letzte Zeile
        if self.y_offset &gt; SystemFont::glyph_height() {
            for y in self.y_offset -2*SystemFont::glyph_height()..self.y_offset - SystemFont::glyph_height() {
                for x in 0..self.width {
                    self.screen[(y*self.width +x) as usize] = self.screen[((y+self.height+SystemFont::glyph_height()-2)*self.width +x) as usize];
                }
            }
        }
        if self.y_offset + SystemFont::glyph_height() &lt; self.height { // solange das Fenster noch nicht das Ende des Puffers erreicht hat...
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


## (No) Race Codition

Um den Framebuffer zum Debuggen zu nutzen, muss er initialisiert werden. Wo und wann soll das geschehen? Offensichtlich soll er überall zur Verfügung stehen. Da es nicht sinnvoll ist, eine entsprechende Variable als Funktionsparameter an sämtliche Funktionen weiterzureichen, muss dies Variable global sein, in Rust seit das statisch (Schlüsselwort `static`). Da aber nicht nur lesend auf den Framebuffer zugegriffen wird, muss sie auch veränderbar (_mutable_) sein. In Rust gilt aber jeder Zugriff auf eine veränderbare statische Variable als unsicher! Durch einen nebenläufigen Zugriff kann es bei einer solchen Variable nämlich zu Wettlaufsituationen (_race conditions_) kommen, die dann zu Inkonsistenzen führen.

Nebenläufiger Zugriff? Denn haben wir doch gar nicht, solange wir keine Interrupts einschalten. Das ist zwar richtig, aber das weiß Rust doch nicht. Es kommt also darauf an, Rust davon zu überzeugen. Natürlich könnten wir auch einfach alle unsicheren Zugriffe in einem Modul kapseln, aber da das Problem ein grundsätzliches ist, was vielleicht noch häufiger auftritt, ist eine generische Lösung vorzuziehen.

Im Gegensatz zu veränderbaren statischen Variablen sind unverändertere (_immutable_) überhaupt kein Problem. Leider ist eine unverändertere Frambuffer-Struktur nutzlos. Rust kennt aber das Konzept der internen Veränderbarkeit (_Interior mutability_). Die Idee ist, eine unveränderbare Referenz auf einen veränderbaren Wert zu haben.

Um mit Nebenläufigkeit umzugehen, kenn Rust zwei Traits: Send und `Sync`. Es sind sogenannte Marker-Traits, die selbst keine Methoden verlangen. `Send` sagt aus, dass es bei nebenläufiger Nutzung eines Wertes keine _race condition_ gibt, während `Sync` dies für die nebenläufige Nutzung seiner Referenz zusichert. Beide sind unsicher, was heißt, dass der Compiler die Einhaltung der Garantien nicht überprüfen kann.

use core::cell::UnsafeCell;

pub struct NoConcurrency&lt;T: Sized&gt; {
    data: UnsafeCell&lt;T&gt;,
}

unsafe impl&lt;T: Sized + Send&gt; Sync for NoConcurrency&lt;T&gt; {}
unsafe impl&lt;T: Sized + Send&gt; Send for NoConcurrency&lt;T&gt; {}

impl&lt;'b, T&gt; NoConcurrency&lt;T&gt;{
    pub const fn new(data: T) -&gt; NoConcurrency&lt;T&gt; {
        NoConcurrency{
            data: UnsafeCell::new(data)
        }
    }

    pub fn set(&self, data: T) {
        let r = self.data.get();
        unsafe { *r = data;}
    }

    pub fn get(&self) -&gt; &mut T {
        let r =self.data.get();
        unsafe { &mut (*r)}
    }
}

# Der Linker zickt mal wieder

Bei der Übersetzung des Codes entsteht ein Linkerfehler:


  
    error:&nbsp;linking&nbsp;with&nbsp;`arm-none-eabi-gcc`&nbsp;failed:&nbsp;exit&nbsp;code:&nbsp;1
  
  
  
    /Users/mwerner/Development/aihPOS/xargo/aih_pos/src/hal/board/framebuffer.rs:198:&nbsp;undefined&nbsp;reference&nbsp;to&nbsp;`__aeabi_uidiv&#8216;
  
  
  
    /Users/mwerner/Development/aihPOS/xargo/aih_pos/src/hal/board/framebuffer.rs:201:&nbsp;undefined&nbsp;reference&nbsp;to&nbsp;`__aeabi_uidivmod&#8216;
  


Eine kurze Recherche zeigt, wo er herkommt: Unser Prozessor kann keine Ganzzahldivision (aber Gleitkommadivision!) und der Compiler verlässt sich darauf, dass die Plattform entsprechenden Code zur Verfügung stellt, was wir aber bisher nicht machen. Da es einige dieser Befehle gibt, verzichte ich darauf, sie alle per Hand zu definieren und greife auf die [Compiler-Buildins-Bibliothek][11] zurück, die in der &#8222;Rust Language Nursery&#8220; zur Verfügung gestellt wird.

Nun habe ich schon zwei Rust-Bibliotheken (core und compiler-buildins), die händisch eingepflegt werden, wenn es zu Updates bei ihnen oder dem Compiler kommt. Höchste Zeit, das Projekt auf Rust-Tools umzustellen, die dies automatisch erledigen. Dies wird das Thema des nächsten Beitrags sein.

&nbsp;


  
  
  
  
    
      Das ist nicht ganz präzise: Rust hält das &#8222;natürliche&#8220; Alignment ein, das ggf. durch repr(packed) unterlaufen werden kann. &#8629;
    
    
      Version 6d841da vom 22. April 2017 &#8629;
    
    
      Eine weitere Alternative für das Debuggen wäre auch noch die Nutzung der UART, Kanal 2. &#8629;
    
    
      Diese Darstellung ist etwas vereinfacht. &#8629;
    
    
      Genau genommen bräuchte man dafür nur ein oder zwei &#8222;Reservezeilen&#8220;, aber so sind wir etwas flexibler. &#8629;
    
    
      Es wäre dann vermutlich besser, wenn ein u8-Array genutzt würde. &#8629;
    
  


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
