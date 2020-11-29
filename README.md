- [Voorbereiding](#voorbereiding)
- [GitHub classroom](#github-classroom)
- [Locks](#lock)
- [Multi-processing](#Multi-processing)
- [Physical memory allocator](#physical-memory-allocator)

# Voorbereiding

Ter voorbereiding van deze oefenzitting word je verwacht:
- De oefenzitting over [traps][oz traps] te hebben voltooid;
- Hoofdstukken 6 en 7 van het [xv6 boek][xv6 book] te hebben gelezen;
- Sectie 3.5 van het xv6 boek nog eens opgefrist te hebben.

# GitHub classroom

TODO

# Locks

TODO

# Multi-processing

TODO

# Physical memory allocator

In de oefenzitting over virtual memory hebben we gezien hoe de mappings van virtuele naar fysieke adressen opgesteld worden in xv6.
De implementatie van lazy allocation bouwde hier op verder en introduceerde de functie [`kalloc`][kalloc] om fysieke frames to alloceren.
We gaan nu de werking van `kalloc`, de _physical memory allocator_ van xv6, in detail bekijken en de performantie op multi-processor systemen proberen te verbeteren.

De taak van een physical memory allocator (vanaf nu kortweg `kalloc` genoemd) is, zoals de naam doet vermoeden, het beheer van het fysieke geheugen in het systeem.
Telkens de kernel meer geheugen nodig heeft, zal het `kalloc` eerst vragen om een nieuw fysiek frame, en dit frame dan mappen op een virtuele page.
Een aantal voorbeelden van wanneer dit gebeurt:
- Bij het inladen van een nieuwe executable door [`exec`][exec] wordt [`uvmalloc` gebruikt][exec uvmalloc] om user pages aan te maken.
  [`uvmalloc`][uvmalloc] [alloceert een frame via `kalloc`][uvmalloc kalloc] en [mapt deze via `mappages`][uvmalloc mappages].
- Gelijkaardig, [`sbrk`][sys_sbrk] roept [`growproc`][growproc] op om de heap van een proces te laten groeien.
  Deze functie roept op zijn beurt weer `uvmalloc` op.
- Wanneer er nieuwe page tables aangemaakt moeten worden, zal [`walk`][walk] weer [gebruik maken van `kalloc`][walk kalloc] om hier een frame voor te voorzien.

> :question: Vind andere plekken in de kernel waar `kalloc` gebruikt wordt en en probeer te begrijpen waarom.
> Je kan het volgende commando in een Linux terminal gebruiken om alle voorkomens van de string "kalloc" te vinden in alle `.c` files:
> ```bash
> grep -n kalloc kernel/*.c
> ```

Maar hoe beheert `kalloc` het fysieke geheugen precies?
Om deze vraag te beantwoorden, moeten we eerst bepalen _wat_ `kalloc` precies beheert.
Herinner je de volgende figuur uit het xv6 boek:

![Memory layout](img/mem.png)

De rechterhelft toont de _memory map_ van de _physical address space_ waar xv6 gebruik van kan maken.
Zoals je kan zien, wordt slechts een gedeelte van deze address space gebruikt voor physical memory (RAM).
De rest van de address space wordt ofwel niet gebruikt, ofwel in beslag genomen door _memory-mapped I/O_ (MMIO) devices (CLINT, PLIC, UART, VIRTIO).
MMIO is een techniek om te communiceren met devices: loads en stores naar MMIO regios zullen niet in het geheugen terechtkomen maar worden afgehandeld door devices.

> :bulb: Een memory map is zeer systeem-afhankelijk.
> De bovenstaande is de memory map zoals die door de qemu emulator voorzien wordt.
> Op andere systemen kan deze volledig anders zijn en zal xv6 dus niet meteen werken.
> Vaak voorzien systemen een manier om de memory map te achterhalen maar dit wordt niet gebruikt door xv6.
> De memory map die xv6 verwacht is _hardcoded_ in [`memlayout.h`][memlayout.h].

> :bulb: Er gebeuren dus twee verschillende adres mappings door de hardware:
> - Virtuele adressen worden eerst op fysieke adressen gemapt door de MMU;
> - Deze fysieke adressen worden dan door de memory controller op RAM of andere devices gemapt.
>
> De tweede mapping is meestal niet onder controle van de kernel en wordt als een gegeven beschouwd.

De figuur toont dat het fysieke geheugen gemapt is vanaf adres `0x80000000`.
Dit adres wordt [`KERNBASE`][KERNBASE] genoemd in xv6.
xv6 gaat er verder van uit dat er 128MiB RAM gemapt is vanaf dit adres.
Dit wilt zeggen dat het einde van het RAM geheugen valt op adres `KERNBASE + 128MiB`.
Dit adres, `0x88000000`, wordt [`PHYSTOP`][PHYSTOP] genoemd (merk op dat de figuur een fout adres aangeeft).

De kernel kan het fysieke geheugen tussen `[KERNBASE, PHYSTOP)` beheren zoals het zelf wil.
Een gedeelte wordt in beslag genomen door de code en statische data van de kernel en de rest wordt dynamisch beheerd door `kalloc`.
xv6 laadt de kernel in het geheugen vanaf `KERNBASE` en definieert een symbool genaamd [`end`][end] om het einde van de kernel aan te geven.
`kalloc` beheert dus het geheugen tussen `[end, PHYSTOP)`.

Nu we weten _wat_ er beheerd wordt, kunnen we bekijken _hoe_ het beheerd wordt.
Laten we eerst de interface die `kalloc` aanbiedt bespreken:
- [`void* kalloc()`][kalloc]: Alloceert één frame en returnt het fysieke adres.
- [`void kfree(void* pa)`][kfree]. Geeft het frame op fysiek adres `pa` vrij.
  Vanaf dit moment is dit frame weer beschikbaar voor `kalloc`.

> :question: Verklaar waarom `kalloc` niet _minder_ dan één frame kan alloceren.

`kalloc` moet dus op de één of andere manier bijhouden welke frames vrij zijn, en welke in gebruik.
De keuze om alle allocaties en vaste grootte van één frame te geven maakt dit een relatief eenvoudig probleem.
De standaard techniek voor het implementeren van dit type allocator (vaak een _memory pool_ genoemd) is een _free list_: een linked list van vrije frames.

Deze linked list wordt in xv6 voorgesteld door [`struct run`][struct run] en een pointer naar het eerste element van deze list wordt opgeslagen in de globale variabele [`kmem.freelist`][kmem.freelist].
Opvallend aan deze `struct run` is dat het _enkel_ een `next` pointer bevat, _geen_ data zoals je verwacht in een typische linked list implementatie.
De data in dit geval zijn de frames en om geen extra geheugen te moeten gebruiken voor de linked list, wordt de `struct run` _in_ de frames geplaatst.
De frames in de free list zijn immers niet in gebruik dus is het geen probleem om hier de nodige metadata (de `next` pointer) voor de linked list in op te slaan.

`kalloc` zal het begin van de frames gebruiken om een `struct run` in op te slaan.
Dit kan in C simpelweg door een adres te casten naar een `struct run`.
De volgende code voegt bijvoorbeeld een frame op adres `pa` toe aan de free list:
```c
struct run* r = (struct run*)pa; // Interpret the memory at pa as a struct run
r->next = kmem.freelist;
kmem.freelist = r;
```

De linked list die door `kalloc` gebruikt wordt, kan "grafisch" dus als volgt voorgesteld worden (zoals meestal bij linked lists wordt het einde aangeven door een `next` pointer die op 0 staat):

<pre>
kmem.freelist
      |
      v             |-----------v
+-   -+------+-   -+------+-   -+------+-   -+
|     |n     |     |n     |     |0     |     |
| ... |e ... | ... |e ... | ... |0 ... | ... |
|     |x     |     |x     |     |0     |     |
|     |t     |     |t     |     |0     |     |
+-    +------+-   -+------+-   -+------+-   -+
       |-----------^            \--v--/
                                 frame
</pre>

Aangezien `kalloc` door alle CPUs simultaan gebruikt wordt, wordt de free list enkel bewerkt in een kritische sectie beschermd door de spinlock [`kmem.lock`][kmem.lock].

> :question: Je hebt nu genoeg informatie om `kalloc` volledig te begrijpen.
> Bekijk en verklaar [`kalloc`][kalloc], [`kfree`][kfree] en [`kinit`][kinit] (de initialisatie van `kalloc` via [`freerange`][freerange]).

[oz traps]: https://github.com/besturingssystemen/traps
[xv6 book]: https://github.com/besturingssystemen/xv6-riscv
[kalloc]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/kalloc.c#L65
[kfree]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/kalloc.c#L42
[kinit]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/kalloc.c#L26
[freerange]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/kalloc.c#L33
[exec]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/exec.c#L12
[exec uvmalloc]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/exec.c#L52
[uvmalloc]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/vm.c#L218
[uvmalloc kalloc]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/vm.c#L231
[uvmalloc mappages]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/vm.c#L237
[sys_sbrk]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/sysproc.c#L41
[growproc]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/proc.c#L243
[walk]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/vm.c#L71
[walk kalloc]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/vm.c#L94
[memlayout.h]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/memlayout.h
[KERNBASE]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/memlayout.h#L55
[PHYSTOP]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/memlayout.h#L56
[end]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/kernel.ld#L43
[struct run]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/kalloc.c#L17
[kmem.freelist]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/kalloc.c#L23
[kmem.lock]: https://github.com/besturingssystemen/xv6-riscv/blob/85bfd9e71f6d0dc951ebd602e868880dedbe1688/kernel/kalloc.c#L22
