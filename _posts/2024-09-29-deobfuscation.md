---
layout: post
title: Deobfuscation of Indirect Jump Obfuscation (Chapter 1)
description: In this article, we are diving into the details of indirect jump obfuscation. We are going to demonstrate how to revert it, and what are the challenges that we are going to mess with.
author: Unex
tags: x86 Miasm Obfuscation
---

<br>
Welcome to my article which demonstrates indirect jump (de)obfuscation.
For this article, we are handling a module of a known online game. - Sorry, I will not name it for now. -


Before writing this article, I've just obtained some information about module. I didn't want to keep those steps in this article, to avoid getting away from our subject.


For the rest of article, I call this module module.dll, so don't feel strange when you see that name. It stands for our target module.


There is a picture below which shows the beginning of the thread which involves all functionalities about module.dll. Even not just about module.dll, also all the security operations about process.

<br><br>
<img src="/assets/images/article-1/pseudo-1.png"/>
<br><br>

As you can see from pseudo-code, because of 'too many' branches on the rest of flow; our decompiler (GHidra) fails to disassemble continuous jump addresses. That is not a good new because we generally want to see full CFG (control flow graph) to have some ideas about concepts. Moreover, it's really unbearable to compute each path for every branch we encounter. More and moreover, nobody wants to see a pseudo - or assembly - code which translates branches like that. In this article, we are going to get rid of this problem.

<span class="danger">Note: In this article, I'm not going to post a full source code which takes your module as input and gives deobfuscated module as output. This way of learn is not such a thing that I support, so I am not going to be a part of this fault. I just want you to figure out things like how deobfuscation process is done, what are the challenges on this road and what you can do for getting rid of them.</span>

We can't really have much idea from this pseudo-code, so let's dive into its assembly code.

<br><br>
<img src="/assets/images/article-1/assembly-1.png"/>
<br><br>

From this perspective, we might figure out a few things about the way of obfuscation.

First, we need to re-define indirect jump: Indirect jumps means that `JMP` instruction takes an operand which is not a memory address. In the end, it can be a memory location but it doesn't change that it's not a memory address. On the other hand, `JMP` takes an operand which is a 'register'.

Example:
```
; indirect
JMP RCX
; indirect
JMP RDX
; indirect
JMP RAX
; indirect
JMP R8
.
..
...
```
All of these instructions are indirect jumps.

By the way, to avoid misunderstandings, existence of an indirect jump **doesn't mean that function is obfuscated**, unless almost all the blocks are terminated by an indirect jump.

Normally, indirect jumps are used to implement switch statements most of time. When you see an indirect jump on an unsecured code, that would probably be a switch statement. But all the control flow can't be managed with switch statements, right? It should sound really weird. Why all the branches be consist of indirect jumps or switch statements? Nobody is going to believe that. There's probably a big deal in there, in other words; you are trying to hide something from us.

Well, let's take a look at the jump register, because if there's a memory location that this block is going to follow; then it should be stored in jump register. I mean, where else it can be stored?

In the previous image, you can see that our jump register for this block is `R11`. To find it yourself, just look up where the block ends (which means 'where is the terminator') and check the operand of terminator instruction. On our block, terminator instruction is `JMP R11` so our jump register is `R11`.

Now we need to look up the read/write attempts to this register in reverse order. We don't need to definition or irrelevant uses of `R11`, we are just trying to find its value statically. That's why we should do that in reverse order.

<br><br>
<img src="/assets/images/article-1/assembly-2.png" />
<br><br>

I want you to take your attention of `CMOV (conditional MOV)` instruction. It's the first instruction that `R11` is used before jump. Also, I want you to notify that it's a conditional write operation. So basically, there's a condition there; and the jump address is being changed based on whether condition is satisfied or not.

Let's talk more specificly, `R11` is probably our default value. If condition is satisfied, value of `R10` is written into `R11`. That means we need to look up both `R11` and `R10` registers in reverse order to find continuous paths. In C code, we can represent it as:

```c
if (condition == true) R11 = R10
// else R11 = R11
jmp(R11)
```

Now, let's scan our assembly code more. We need to find the static values of both `R11` and `R10`.

When I look up the instructions more, I've found something useful:

<img src="/assets/images/article-1/assembly-3.png" />

When we try to figure out this code 'nestedly', we can easily see that there's a def-use chain for both `R11` and `R10` registers. For better understanding, I want to split these lines for both `R11` and `R10`. First instruction list I'm going to type below will include all the instructions which are a part of `R11`'s def-use chain. Then we are going to discuss `R10`'s one later.

```
LEA         R11, [LAB_1824b3d43]
MOV         qword ptr [RBP + local_50], R11
MOV         R11, 0xFFFD5FAB
ADD         R11, qword ptr [RBP + local_50]
CMOVLE      R11, R10
JMP         R11
```

Now, I want you to take a look at these instructions for 2 minutes before following rest of this article. You can figure out something, maybe? If you can't, there's not a problem. I'm already posting this thread to make you figure them out.

Alright, your time is over. Now let's check it together. First thing that I'm going to do about these lines is to convert them into mathematical equations. After the convertion, results will look like below:

```c++
R11 = 0x1824b3d43 // just considering it as address
stack_local_50 = 0x1824b3d43 // just replaced R11 with constant, we already knew what R11 was equal to, from previous instruction
R11 = 0xFFFD5FAB
R11 = R11 + 0x1824b3d43 // just replaced stack_local_50 with constant, we already knew what it was equal to

// I am ignoring CMOV and JMP instructions, because they will not give us an idea about what are stored in R11 and R10
```

And after simplifying these equations, we get:

```c++
R11 = -0x2a055 + 0x1824b3d43 // -0x2a055 is signed form of 0xFFFD5FAB
// which computes to
// R11 = 0x182489CEE
```

`0x182489CEE` is our constant value for `R11`.

So since the `R10` is written into `R11` if condition is satisfied and `0x182489CEE` is the default value for `R11`, we can say that `0x182489CEE` is address of next path **if condition is not met**.

Not to take your time reading this article for long, I will apply the same steps to find the static value of `R10` myself.

According to my calculations, constant value for `R10` is equal to `0x182489F9D`.

<span class="warning">Note: Don't be confused, I use both 'constant' and 'static' terms for those values at the same time because these values are both constant (which means **not mutable**) and static (which means **independent from runtime values, resulting same for all given program inputs**). They both have almost same meaning in this article.</span>

<br>
Well, after all these calculations and stuff; we finally found the successors (it means continuous blocks). They are `0x182489CEE (on condition is not met)` and `0x182489F9D (on condition is met)`.

<br>
*I'm going to end this chapter here. I don't want you to be confused, so I am not going to tell all the details about this deobfuscation process in this chapter. For a better understanding, you should have some time to implement it in your mind. Also, don't forget to make some practices in this period.*

*We just talked about deobfuscation of one of the heavily obfuscated binaries with indirect jump obfuscation in this chapter. I hope it helps you to figure out some concepts to beat these security walls for good purposes. I tried my best to make you figure out how it's done and I'm expecting that the informations I shared here will help you.*

**Have a good day, see you in chapter 2 :)**

<br>