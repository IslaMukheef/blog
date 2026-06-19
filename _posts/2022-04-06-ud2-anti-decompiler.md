---
layout: post
title:  "UD2 — Anti Decompiler"
date:   2022-04-06 
categories: Security
summary: How to bypass UD2 and what is it exacly!
---

This technique is really easy to bypass so I will talk a bit about it and I will show different ways to bypass it.

``` What is UD2 ? ```

UD2 is an assembly Instruction used to generate an invalid opcode exception. Basically programmers can use __asm__(“ud2”); to generate invalid opcode and stop decompliers from showing the rest of the instructions. For example

![](/assets/img/ud2/1.webp)
*Figure 1:*

If we try to decomplie this program

![](/assets/img/ud2/2.webp)
*Figure 2:*

IDA PRO stoped decompling after the __asm_ instruction(didn't decomplie the third printf function) which is what it should do. You gonna see this behavior with other decompliers such as Ghidra.

``` How to bypass UD2 ```

actually it’s pretty simple.

1. method; use different decompliers such as gdb and try to disassemble the function that has this instruction. That’s it you will be able to see the whole instructions.

2. method; try to change the instruction to NOP instruction and patch the program then try to decomplie it again and you will be able to read the wole instructions

![](/assets/img/ud2/3.webp)
*Figure 3:*

These are the most easy ways to bypass it.

