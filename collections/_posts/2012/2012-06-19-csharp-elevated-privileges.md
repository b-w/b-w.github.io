---
layout: post
title: "Running with elevated privileges from C#"
categories: blog
---

Sometimes, for some reason, you may require your C# program to run with elevated (Administrator) privileges, or you may need to run some other program with elevated privileges. So how do you do it? Well, there are multiple ways. Here, I will focus on elevating privileges from code. There is also the possibility of starting your app with elevated privileges by [configuring the application manifest](http://msdn.microsoft.com/en-us/library/bb756929.aspx), but I won't go into that now.

So, without further adieu, here's all the code you will need. It's not very complicated.

```csharp
var proc = new ProcessStartInfo();
proc.UseShellExecute = true;
proc.WorkingDirectory = @"C:\Windows\System32";
proc.FileName = @"C:\Windows\System32\cmd.exe";
proc.Verb = "runas";

try
{
    Process.Start(proc);
    Console.WriteLine("Successfully elevated!");
}
catch (Exception)
{
    Console.WriteLine("Failed to elevate.");
}
```

So what does it do? Well, right now this code attempts to start the command prompt with elevated privileges. It's comparable to going into the start menu, finding Command Prompt, right-clicking it, and saying "Run as Administrator". In fact, the inclusion of the "runas" verb in the process start info does exactly that. Also note the try-catch block. This is necessary for when the user declines the request from UAC for elevated privileges. The exception caught will be a Win32Exception stating "The operation was canceled by the user". If the user allows the elevation, you'll see something like the following:

![Elevated process](/assets/img/blog/2012/06/elevated-cmd.png)

Of course, you can elevate any process you like, including your own app:

```csharp
var proc = new ProcessStartInfo(); 
proc.UseShellExecute = true; 
proc.WorkingDirectory = Environment.CurrentDirectory; 
proc.FileName = Application.ExecutablePath; 
proc.Verb = "runas";

try
{
    Process.Start(proc);
}
catch (Exception)
{
    Console.WriteLine("Failed to elevate.");
    return;
}

Application.Exit();
```

Yes, you will need to restart your app to gain elevated privileges; I'm not aware of any method for elevating the privileges of a currently running process. If you occasionally need elevation and you don't want to restart your app then it may be a good idea to separate the logic requiring elevation into a different executable.
