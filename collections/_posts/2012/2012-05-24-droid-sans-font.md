---
layout: post
title: "A new look using the Droid Sans font"
categories: blog
---

If my blog looks slightly different today, that's because I've changed the main font to Google's [Droid Sans](https://www.google.com/webfonts/specimen/Droid+Sans). The font has actually been there all the time as part of the theme I'm using; it's just never gotten a chance to show itself due to a funny bug in BlogEngine.NET. Sounds weird? Well... here's what happened:

It turns out that BlogEngine.NET passes all JavaScript and CSS files through its own custom file handlers, which perform a server-side compression step before sending the files to a client. This is in principle a good thing as it uses a little CPU power of the server to save some bandwidth. Of course, when your compression algorithm is flawed...

The theme I'm using has the following line in its CSS file:

```css
/* Google Fonts Import
--------------------------------------------- */
@import url("https://fonts.googleapis.com/css?family=Droid+Sans:regular,bold|Droid+Serif:regular,italic,bold,bolditalic&subset=latin");
```

This imports the CSS needed for the font using Google's public API for its Web Fonts. There is nothing wrong with this line, but here's how it looks after being served by BlogEngine.NET to a client browser:

```css
@import url("https://fonts.googleapis.com/cssfamily=Droid+Sans:regular,bold|Droid+Serif:regular,italic,bold,bolditalic&subset=latin")
```

Notice the change? Aside from the comment lines being gone, that is? It's actually very subtle: the "?" between _css_ and _family_ is gone in the compressed CSS. The result: a big fat 404 on the resulting URL, and no Web Font.

And the fix? Well, I've considered modifying the BlogEngine.NET source code to fix it, but that would be a lot of work as I first have to fetch it and it would mean compiling and reinstalling the entire blog. If possible, I'd like to avoid that. Eventually, I decided on simply adding the following CSS directly to the site master for my current theme:

```html
<style type="text/css">
    @import url("https://fonts.googleapis.com/css?family=Droid+Sans:regular,bold|Droid+Serif:regular,italic,bold,bolditalic&subset=latin");
</style>
```

This seems to avoid BlogEngine.NET's compression algorithms and successfully imports the Web Font. So that's why my blog looks different now.

## Update

Well... when looking further into the problem, I found out a new version of BlogEngine.NET had just been released a few days ago. Now, after upgrading, it turns out CSS compression is no longer enabled. It would seem the problem has been fixed. Again.
