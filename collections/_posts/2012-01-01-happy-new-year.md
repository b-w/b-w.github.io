---
layout: post
title: "Happy new year!"
categories: blog
---

A very happy new year to all 9 unique visitors to my blog!

Before we dive into the year that will disappoint conspiracy theorists around the globe, let's go a bit deeper into those numbers. First, demographics. Out of those 9 visitors, 3 were search engine bots, 2 were Dutch, 2 were Italian, 1 was French, and 1 was American. Interesting. Now, what were they all looking at? Well, the most popular article was my [unboxing of the Asus Eee Note]({% post_url 2011-11-02-azure-eee-note-impressions %}), which drew my Italian and French visitors. Both the search engines and the rest of the visitors preferred the homepage. Rough estimates put total bandwidth usage for my blog this year on about 40 MB.

Not included in those 9 but deserving special mention nonetheless is the fellow with the Chinese IP-address who on three separate occasions tried retrieving non-existent PHP configuration files from my server. I don't really count you as a visitor as you never looked at an actual page, but I hope to see you again in the future also.

Next year, double digits! Hopefully. Or not, since my server cannot take the strain.

For now, happy 2012! Or, uhm...

```fsharp
open System;

let NumToRoman(n : int) =
    let rec step(x : int, l : (int * string) list) =
        if x = 0 then ""
        elif fst(List.head(l)) > x then step(x, List.tail(l))
        else snd(List.head(l)) + step(x - fst(List.head(l)), l)
    step(n, [ (1000, "M"); (900, "CM"); (500, "D"); (400, "CD"); (100, "C"); (90, "XC");
            (50, "L"); (40, "XL"); (10, "X"); (9, "IX"); (5, "V"); (4, "IV"); (1, "I") ])

Console.WriteLine(NumToRoman(2012))
```

...happy MMXII!
