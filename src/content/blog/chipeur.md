---
author: Gabin Le Saout
pubDatetime: 2025-02-19T21:29:00Z
title: Chipeur (Swiper)
featured: false
draft: false
tags: 
  - school_projects
  - C
  - Malware development
description: Swiper, no swiping. Swiper, no swiping. Swiper, no swiping
---
Here is my last project in school, a sad and happy feeling at the same time. It is a malware development one, done by a group of 4 in which we had several steps.

## Table of contents
# Introduction

Our requirements for chipeur (the French version of Swiper in Dora, you will get the joke later on) were to do write it in C with Windows as a target. C because some of us (not me, I swear) was too afraid to do it in Rust. Windows because most of our experience were on Linux, we needed some XP.

The first one was a bibliographic part, studying and understanding various aspects and mechanisms of malware. Since we were 4 we have divided the work, and my focus was on anti-static analysis :
- Stack strings
- Anti-debug techniques
- Dynamic API resolution

# Stack strings

As you may know, C does funny stuff regarding memory management. As an example :
```C
char *my_str1 = "hello, world!";
char my_str2[] = "hello, world!";
```
Those 2 lines are doing the same thing (I am lying), we have 2 variables referring to strings. But there is one funny thing, their location in memory is different.

The first version initializes a string in memory, and a pointer called my_str1 referring to the first character. The initialized string is in .rdata section, which stands for "read-only data". 
The second one creates an array in the stack, in which it writes the given string. An array is also a pointer in practice, but there is memory allocation in the process.

At the end we will have in one case, a string in the .rdata section, possible to retrieve by checking the binary. In the other one, instructions will place the string in the stack (so a runtime allocated memory place).

Great, something allocated at runtime, so it is harder to read. In practice, you will still get it back in seconds using Ghidra. That is why we have added obfuscation on characters and stored them as an XORed version.

# Anti-debug techniques
Great, we have interesting strings now ! Another thing we could do to make analysts work harder is to detect debuggers and react accordingly. 
Based on [this Lockbits ransomware analysis](https://chuongdong.com/reverse%20engineering/2022/03/19/LockbitRansomware/#anti-analysis-anti-debug-check) I have seen that we need to check at the Process Environment Block (PEB). What is it ?

The PEB is a data structure mostly used by the operating system that contains information such as global context, startup parameters or data structures. One of the fields documented by Microsoft is "BeingDebugged", there we go. Another mentionned thing is "Microsoft recommends not using this field but using the official Win32 CheckRemoteDebuggerPresent() library function instead."

Now that we know how to detect debuggers (better things could be done such as detecting breakpoints but this is an easy solution) let's react to it. The logical decision would be to stop our program but it is not funny, it is funnier to enter an infinite loop : much more diabolical. 

# Dynamic API resolution
TODO()
