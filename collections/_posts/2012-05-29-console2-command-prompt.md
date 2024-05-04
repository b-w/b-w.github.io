---
layout: post
title: "Console2 does command prompt right on Windows"
categories: blog
---

I'm not a huge fan of the command prompt style of working on a PC. It just feels clumsy and unintuitive. That said, I have been warming up to PowerShell recently. This post is not about that, though. It's about [Console2](http://sourceforge.net/projects/console/).

![Console2](/assets/img/blog/2012/05/console2.png)

Console2 is a host for various different consoles you may have on your system, like cmd.exe, PowerShell, F# Interactive, etcetera. It allows you to unify all these different consoles (or shells) in a single user interface. I even have Git Bash running in it without problems. The main advantages of Console2 are that it is very lightweight, its looks are highly configurable (I have it set up with white-on-black text and 40% alpha transparency), the UI is freely resizable (not always easy or possible with console windows), and it offers tabs.

The setup is pretty painless as well. For example, to configure Console2 for use with PowerShell, all I needed to do was add a new tab in the settings menu and point its shell to `C:\Windows\SysWOW64\WindowsPowerShell\v1.0\powershell.exe`. Things like the F# Interactive or the Git Bash require no more work than that. After that, you can run everything side by side in a single window.

In conclusion, _if_ you have to do command prompt work on Windows, at least do it right.
