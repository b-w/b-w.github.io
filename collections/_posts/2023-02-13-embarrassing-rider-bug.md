---
layout: post
title: "The most embarrassing Rider bug has finally been fixed after 3+ years"
categories: blog
---

_EDIT 2023-03-06: never mind, the fix was already reverted due to "possible performance degradations"_

[JetBrains Rider](https://www.jetbrains.com/rider/) is an amazing IDE for .NET development. I strongly prefer it to Visual Studio. However, for many years now Rider has had one deeply embarrassing flaw: the inability to properly debug async code.

Consider a very simple example program:

```csharp
// there should not be anything running on this address
var req = new HttpRequestMessage(HttpMethod.Post, "http://localhost:1337/");

var client = new HttpClient();

// this will throw
await client.SendAsync(req);
```

Provided you don't have a local webserver listening on port 1337, running this program will throw an HttpRequestException on line 7\. If you're debugging it, you'd expect to see something like the following example provided by VS Code:

![](/assets/img/blog/2023/02/vscode-async-debug.png)

In Rider, however, you'd get this:

![](/assets/img/blog/2023/02/rider-async-debug-fail.png)

Rather than showing the actual user code that caused the exception, Rider instead breaks somewhere deep in the bowels of .NET base library code. And to top it off, while it shows you the call stack, it will not even allow you to jump back to your own code.

The result: async code is effectively un-debuggable in Rider. And with async code being everywhere in modern .NET applications, this makes Rider's debugging experience a plainly embarrassing one, especially when considering that even free tools like VS Code can do so much better.

Not only that, but this bug has been in Rider for _years_ [[1]](https://youtrack.jetbrains.com/issue/RIDER-41241). At least 3 that I know of myself, but it looks like it's been there for a longer time than that, perhaps being fixed and then regressed again along the way [[2]](https://youtrack.jetbrains.com/issue/RIDER-13596). I wouldn't be surprised if this has never properly worked at all. Either way, it's pretty sad that such a fundamental bug can exist for so long.

Fortunately, here's how the most recent Rider build, 2023.1 EAP4, handles this apparently extremely difficult scenario:

![](/assets/img/blog/2023/02/rider-async-debug-fix.png)

Rejoice, oh weary user! No longer shalt thou debug thine code using Console.WriteLine statements! No longer shalt thou be forced to place breakpoints in places where thou thinketh thine code might throw that exception! For the Lord hath at last provided thou with an acceptable debugging experience! Rejoice! Rejoice in the Light of His Glory!
