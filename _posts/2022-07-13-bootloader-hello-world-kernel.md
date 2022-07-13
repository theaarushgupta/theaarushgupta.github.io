---
layout: post
title: "Bootloader and Hello World Kernel"
description: "Sanity Writeup #0"
---

Time spent: ~2 hours

This portion of development mainly involved researching and understanding 
booting and basic theory. The portion of coding was done in quite a short 
period, so the time spent is not representative of the time of 
development.

The intent of this section was to set up a basic kernel, bootable a multiboot
compatible bootloader, and to bring linking with C, where I could then 
implement basic functions such as `print` and util functions such as `itoa` 
and `strlen`.

Modern machines launch with UEFI, however, this being a hello world kernel
will consider the BIOS as the first code that runs. The BIOS does some 
configuring to the system, finds a bootable medium, loads it, and then jumps 
to it, effectively handing execution to it. The loaded code is the kernel and 
can be found in `boot/head.asm`.

The kernel starts by declaring a multiboot header, which allows the 
bootloader to identify it as a bootable source. Since the multiboot standard 
does not define a value for `esp`, the stack pointer register, it is up to us 
to provide one, giving us room for a small stack of 16KiB. The kernel is 
properly aligned to match the System V ABI standard, making sure that there is 
no undefined behavior. Following that, we have the `_start` section, which is 
defined in the linker script, `link.ld` as the entry point; this is where the 
bootloader will jump to after the kernel is loaded.

The bootloader loads the kernel into a 32-bit protected mode; interrupts and 
paging is disabled, and the kernel is given full and absolute power over the 
machine.

The first thing the kernel does is set up the aforementioned stack, pointing 
the top of it to the `esp` register. This makes sure C can function, for it 
cannot without a stack. Between this point and the next, we can later 
initialize processor state; features such as the GDT (Global Descriptor Table) 
and paging can be enabled here.

After this, we call the C kernel entry function `main` and pass off execution 
to another portion of the code. Following that, we put the computer into an 
infinite loop, force disabling interrupts with `cli`, locking the process with 
`hlt`, and jumping back the `hlt` if a non-maskable interrupt occurs.

The last line of `boot/head.asm` is simply put to be used when debugging or 
for call tracing.

The C portion of the code simply includes the `terminal.h` header and prints 
a hello world string to the VGA text mode buffer. Despite the fact that VGA is
deprecated on newer machines, this implementation is fine for a hello world 
kernel. The more advised GOP (Graphics Output Protocol) on UEFI systems is 
more complex to implement, and QEMU, the VM used to test this kernel, works 
with VGA just fine.

The simple VGA driver implements a variety of basic operations. The `clear`
function clears the screen by resetting all the values in the buffer to a blank
space. Scrolling works by moving every character in the buffer by the width of
the screen, effectively clearing the last row for more text. The main `print`
function uses the formula `row * width + column` to find the index where to
insert the character, which is formatted for the driver by the `applyColors`
function. Applying colors works by formatting the values for the foreground and
background, as well as the character the color is being applied to, into the
proper format. 

This setup allows a simple build toolchain, which cross-compiles for i686, to
link and create an ISO file to boot with QEMU.
