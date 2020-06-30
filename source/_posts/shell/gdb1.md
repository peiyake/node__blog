---
title: GDB常用命令
type: shell
description: 这里罗列一下GBD调试使用的常用命令
date: 2020-06-22 19:10:42
---

GDB是程序开发调试的利器，学会GDB的基本使用方法对程序问题处理有很大的帮助。本文主要罗列总结一下一些GDB常用命令。

## GDB常用命令


b main - Puts a breakpoint at the beginning of the program
b - Puts a breakpoint at the current line
b N - Puts a breakpoint at line N
b +N - Puts a breakpoint N lines down from the current line
b fn - Puts a breakpoint at the beginning of function "fn"
d N - Deletes breakpoint number N
info break - list breakpoints
r - Runs the program until a breakpoint or error
c - Continues running the program until the next breakpoint or error
f - Runs until the current function is finished
s - Runs the next line of the program
s N - Runs the next N lines of the program
n - Like s, but it does not step into functions
u N - Runs until you get N lines in front of the current line
p var - Prints the current value of the variable "var"
bt - Prints a stack trace
u - Goes up a level in the stack
d - Goes down a level in the stack
q - Quits gdb
