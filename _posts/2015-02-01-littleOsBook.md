---
layout: post
title: "Working Through the Little OS Book: Chapter 2"
---

This is a quick overview of my journey through the second chapter of the <u>The
little book about OS development</u>, found
[here](https://littleosbook.github.io). The book was missing a few details, and
contained a few code errors (potentially due to tooling changes over time).
I'll make a note of the necessary changes where appropriate.

## Setup ##

The install list is pretty simple, from the book:

{% highlight bash %}
sudo apt-get install build-essential nasm genisoimage bochs bochs-x bochs-term
{%  endhighlight %}

The change from the book is the addition of the `bochs-term` package, which
lets us run bochs when you do not have an X window server available.

## Writing the OS ##

The loader code is quite interesting - mostly in its simplicity. Most of the
magic for creating a properly aligned OS is actually in the linker scripts, to
follow.

The only magic bit here is ironically the Magic Number, which our grub
bootloader requires to ensure that its loading an OS and not random data. There
are also flags which are required, though we won't set any in this case (this
was missing in the book; and it took quite some time and research to figure it
out). Throw in a checksum to prove its not random data, and we're ready to
compile.

{% highlight nasm %}
global loader               ; the entry symbol for ELF

MAGIC_NUMBER equ 0x1BADB002 ; define the magic number constant
FLAGS        equ 0x00000000
CHECKSUM     equ -(MAGIC_NUMBER+FLAGS) 
                            ; Calculate the checksum (checksum + magic number +
			    ;   flags = 0)

section .__mbHeader
align 4                     ; The code must be 4 byte aligned
        dd MAGIC_NUMBER     ; Write the magic number to machine code
        dd FLAGS
        dd CHECKSUM         ; and add the checksum

section .text:              ; Start of the text segement (the program)
loader:                     ; the loader (defined as the entry point in the
                            ;   linker script)
        mov eax, 0xDEADBEEF ; Our indicator that the program actually ran
.loop:
        jmp .loop           ; Loop forever
{% endhighlight %}

Note that this loader does not actually do anything too interesting other than
set the contents of EAX to 0xDEADBEEF, which gives us a value to look for in
the VM logs. If we see DEADBEEF, and we know that the OS was loaded and
executed.

If you're following along in the book, you will also notice an extra section
header. I tracked this down in the [OS Dev
Wiki](http://wiki.osdev.org/Expanded_Main_Page) while trying to figure out why
Grub wouldn't recognize my code as a valid OS. The actual cause of the failure
to load was the missing FLAGS, but this makes for a good habit to ensure that
the magic number is in the location where Grub expects it.

## Configuring the linker ##

Another requirement for a OS executable is its alignment in memory - it needs
to start at the 1MB location, so we don't try and write data to locations in
memory which are reserved for interacting with the hardware. To achieve this,
we create a linker configuration which puts the OS code in right locations in
memory, and orders/aligns the remaining program sections correctly.

{% highlight text %}
ENTRY(loader)

. = 0x00100000;

SECTIONS {
        .__mbHeader :{ *(.__mbHeader) }
        .text ALIGN(0x1000) :{ *(.text) }
        .rodata ALIGN(0x1000) :{ *(.rodata*) }
        .data ALIGN(0x1000) :{ *(.data) }
        .bss ALIGN(0x1000) :{ *(COMMON) *(.bss) }
}
{% endhighlight %}

The `. = 0x00100000;` (note that the book is missing the `;`) ensures that the
code is loaded at a 1MB offset in memory . The individual program sections are
loaded at aligned 1k intervals, as indicated by the section alignments (note
that the book was missing `:` here as well).

## Compiling ##

My makefile:

{% highlight makefile %}
os.iso: iso/boot/grub/stage2_eltorito iso/boot/grub/menu.lst iso/boot/kernel.elf
        genisoimage -R                          \
                -b boot/grub/stage2_eltorito    \
                -no-emul-boot                   \
                -boot-load-size 4               \
                -A os                           \
                -input-charset utf8             \
                -quiet                          \
                -boot-info-table                \
                -o os.iso                       \
                iso


iso/boot/grub/stage2_eltorito: stage2_eltorito
        cp stage2_eltorito iso/boot/grub/stage2_eltorito

iso/boot/grub/menu.lst: menu.lst
        cp menu.lst iso/boot/grub/menu.lst

iso/boot/kernel.elf: kernel.elf
        cp kernel.elf iso/boot/kernel.elf

kernel.elf: loader.o
        ld -T linker.ld -melf_i386 loader.o -o kernel.elf

loader.o: loader.s
        nasm -f elf32 loader.s

clean:
        $(RM) loader.o kernel.elf os.iso

.PHONY: clean
{% endhighlight %}

This does a few things - it assembles the loader (which was written in the NASM
dialect), links it into an elf executable, then puts together a bootable CD ISO
image. We use that ISO to do our tests.

# Testing #

A few pieces of note here - we are using bosch as our VM to run the commands,
and grub as the bootloader. The menu.lst configures grub to look for our newly
compiled kernel, and we use bochsrc.txt to configure the VM.

bochsrc.txt
{% highlight text %}
megs:                   32
display_library:        term
romimage:               file=/usr/share/bochs/BIOS-bochs-latest
ata0-master:            type=cdrom, path=os.iso, status=inserted
boot:                   cdrom
log:                    bochslog.txt
clock:                  sync=realtime, time0=local
cpu:                    count=1, ips=1000000
magic_break:            enabled=1
{% endhighlight %}

menu.lst
{% highlight text %}
default=0
timeout=0

title os
kernel /boot/kernel.elf
{% endhighlight %}

Kick off the bochs VM against our newly created OS, interrupt our infinite loop
with a `killall -SIGHUP bochs`. This will cause it to end while writing the
state to the log file, so we look for this:

{% highlight text %}
01273065000i[CPU0 ] EFER   = 0x00000000
01273065000i[CPU0 ] | RAX=00000000deadbeef  RBX=000000000002cd80
01273065000i[CPU0 ] | RCX=0000000000000001  RDX=0000000000000000
01273065000i[CPU0 ] | RSP=0000000000067ed0  RBP=0000000000067ee0
01273065000i[CPU0 ] | RSI=000000000002cef0  RDI=000000000002cef1
01273065000i[CPU0 ] |  R8=0000000000000000   R9=0000000000000000
01273065000i[CPU0 ] | R10=0000000000000000  R11=0000000000000000
01273065000i[CPU0 ] | R12=0000000000000000  R13=0000000000000000
01273065000i[CPU0 ] | R14=0000000000000000  R15=0000000000000000
{% endhighlight %}

And it works!

