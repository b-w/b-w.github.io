---
layout: post
title: "Some quines in C#"
categories: blog
---

In computer programming, a [quine](http://en.wikipedia.org/wiki/Quine_%28computing%29) is a program which produces a copy of its own source code as its output. They serve absolutely no purpose, but it is sometimes considered a challenge to see which is the shortest or the most complicated quine someone can produce for a given language.

Here's an example of a simple quine in C#:

{% raw  %}
```csharp
using System;

namespace Quine
{
    class Program
    {
        static void Main(string[] args)
        {
            var p = "using System; namespace Quine {{ class Program {{ static void Main(string[] args) {{ var p = {1}{0}{1}; Console.WriteLine(p, p, '{1}'); Console.ReadKey(); }} }} }}";
            Console.WriteLine(p, p, '"');
            Console.ReadKey();
        }
    }
}
```

For readability here, I have placed the program string across multiple lines. This program produces the following output, all of which is on a single line:

```
using System; namespace Quine { class Program { static void Main(string[] args)
{ var p = "using System; namespace Quine {{ class Program {{ static void Main(string[] args)
{{ var p = {1}{0}{1}; Console.WriteLine(p, p, '{1}'); Console.ReadKey(); }} }} }}";
Console.WriteLine(p, p, '"'); Console.ReadKey(); } } }
```

Yes, this _is_ the full source code. However, even though it will compile just fine, it's not very nicely formatted. Of course, you can fix this by putting the source code in an array and printing in a loop with newlines in between (as shown in the Wikipedia article), but I was wondering if it could also be done using just a single string and a few formatting operations. After a bit of tinkering, I arrived at the following:

```csharp
using System;

namespace Quine
{
    class Program
    {
        static string prg = "using System;\r\n" +
                            "\r\n" +
                            "namespace Quine\r\n" +
                            "{{\r\n" +
                            "\tclass Program\r\n" +
                            "\t{{\r\n" +
                            "\t\tstatic string prg = \"{0}\";\r\n" +
                            "\r\n" +
                            "\t\tstatic void Main(string[] args)\r\n" +
                            "\t\t{{\r\n" +
                            "\t\t\tConsole.WriteLine(prg, prg
                                .Replace(\"\\\\\", \"\\\\\\\\\")
                                .Replace(\"\\t\", \"\\\\t\")
                                .Replace(\"\\\"\", \"\\\\\\\"\")
                                .Replace(\"\\r\\n\", \"\\\\r\\\\n\\\"
                                +\\r\\n\\t\\t\\t\\\"\"));\r\n" +
                            "\t\t\tConsole.ReadKey();\r\n" +
                            "\t\t}}\r\n" +
                            "\t}}\r\n" +
                            "}}";

        static void Main(string[] args)
        {
            Console.WriteLine(prg, prg.Replace("\\", "\\\\").Replace("\t", "\\t")
                .Replace("\"", "\\\"")
                .Replace("\r\n", "\\r\\n\" +\r\n\t\t\t\""));
            Console.ReadKey();
        }
    }
}
```

It's a pretty nasty mess, but it does work. Here's the output:

```csharp
using System;

namespace Quine
{
    class Program
    {
        static string prg = "using System;\r\n" +
                            "\r\n" +
                            "namespace Quine\r\n" +
                            "{{\r\n" +
                            "\tclass Program\r\n" +
                            "\t{{\r\n" +
                            "\t\tstatic string prg = \"{0}\";\r\n" +
                            "\r\n" +
                            "\t\tstatic void Main(string[] args)\r\n" +
                            "\t\t{{\r\n" +
                            "\t\t\tConsole.WriteLine(prg, prg
                                .Replace(\"\\\\\", \"\\\\\\\\\")
                                .Replace(\"\\t\", \"\\\\t\")
                                .Replace(\"\\\"\", \"\\\\\\\"\")
                                .Replace(\"\\r\\n\", \"\\\\r\\\\n\\\"
                                +\\r\\n\\t\\t\\t\\\"\"));\r\n" +
                            "\t\t\tConsole.ReadKey();\r\n" +
                            "\t\t}}\r\n" +
                            "\t}}\r\n" +
                            "}}";

        static void Main(string[] args)
        {
            Console.WriteLine(prg, prg.Replace("\\", "\\\\").Replace("\t", "\\t")
                .Replace("\"", "\\\"")
                .Replace("\r\n", "\\r\\n\" +\r\n\t\t\t\""));
            Console.ReadKey();
        }
    }
}
```
{% endraw  %}

Yes, they're identical. What did you expect?
