---
layout: post
title: "Unlocking SQL Server Management Studio's hidden dark theme"
categories: blog
---

If there's one minor annoyance I have with SQL Server Management Studio, it's how it looks. Specifically how _bright_ it looks. This becomes obvious when using SSMS side-by-side with Visual Studio. Of course, Visual Studio has come with a dark theme since version 2012, which is how it should be for all IDEs. I mean, look at it:

![](/assets/img/blog/2018/11/vs.png)

So dark. So soothing. Like a warm bath for your eyeballs.

Then you alt-tab over to SSMS, and you instantly get your retinas singed by this horrible default-WinForms-theme looking abomination.

![](/assets/img/blog/2018/11/ssms-light.png)

Seriously, it's like staring into the sun.

Fortunately, there is a cure. Well, a partial cure at least. You see, it turns out that SSMS _does_ actually have a dark theme. It's just been disabled all this time. But it's there. And it's been waiting to be discovered, hungering for freedom like some terrible ancient horror.

It's actually quite silly. There's a file called _ssms.pkgundef_ in the SSMS installation directory.

![](/assets/img/blog/2018/11/pkgundef.png)

And in this file is a line of configuration code with a comment that literally states its purpose is to disable the dark theme.

![](/assets/img/blog/2018/11/pkgundef-dark-theme-remove.png)

All you have to do to enable the dark theme is comment out this single line, turning

```
...

// Remove Dark theme
[$RootKey$\Themes\{1ded0138-47ce-435e-84ef-9ec1f439b749}]

...
```

into

```
...

// Remove Dark theme
// [$RootKey$\Themes\{1ded0138-47ce-435e-84ef-9ec1f439b749}]

...
```

Then, when you restart SSMS, you can select the dark theme in its settings dialog:

![](/assets/img/blog/2018/11/ssms-options.png)

It really is that simple.

But, like I said, the cure is not perfect:

![](/assets/img/blog/2018/11/ssms-dark.png)

Not all elements of the UI support the dark theme, which is probably why it's disabled out-of-the-box. Still, it's an improvement, and one I wish I knew of earlier.
