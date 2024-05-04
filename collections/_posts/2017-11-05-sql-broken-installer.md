---
layout: post
title: "Debugging the broken SQL Server 2017 installer"
categories: blog
---

Today I tried installing the latest Express Edition of SQL Server: version 2017\. Sadly, Microsoft no longer seems to offer offline installers. If you look at their [download page](https://www.microsoft.com/en-in/sql-server/sql-server-editions-express), you'll only find the option for downloading a small, 5 MB bootstrapper that will then take care of downloading the actual installation files. Even Scott Hanselman's [famous page](http://www.hanselman.com/blog/DownloadSqlServerExpress.aspx) only links to it now, instead of any real installation media.

Why is this important? Well... here's why:

![Oops... We failed to download files we require. Please check your network connection and try again.](/assets/img/blog/2017/11/sql-2017-installer-1.png)

Every time I launch the installer, I simply get this message. No further explanation. It just doesn't work. And before you ask: yes, I'm connected to the internet.

After searching for this error (and finding nothing) and searching for an offline installer (and finding nothing), I was about to just give up when I decided to check on a whim if the installer was a .NET executable. And as it turns out, it is. So that means it's...

## Debugging time!

I decided to see if I could debug the installer, in order to see what goes wrong so that I might be able to fix it. Since I obviously don't have the installer's source code (or even a pdb), Visual Studio by itself would be pretty much useless. Fortunately we still have [dnSpy](https://github.com/0xd4d/dnSpy/), which is literally made for these kind of situations.

After loading the installer into dnSpy, we can see it's a WPF app:

![dnSpy](/assets/img/blog/2017/11/sql-2017-installer-2.png)

The only code we see here contains the UI layer; the actual logic is found in a separate assembly called the InstallerEngine. This assembly is contained as a resource within the installer executable and is loaded dynamically at runtime (a pretty common technique about which I've [written before]({% post_url 2016-09-14-dotnet-dependencies %})).

![dnSpy](/assets/img/blog/2017/11/sql-2017-installer-3.png)

We could extract the InstallerEngine dll to take a look at its code, but we don't really need to do this. When we start debugging, all loaded modules will be browsable through dnSpy anyway. Speaking of which, let's get started.

The first thing I want to try is to see if there's an exception that I can pinpoint as the cause of my problems. I'll just configure dnSpy to break on any CLR exception:

![dnSpy](/assets/img/blog/2017/11/sql-2017-installer-4.png)

Then I'll launch the installer:

![dnSpy](/assets/img/blog/2017/11/sql-2017-installer-5.png)

And what do you know, we've caught an exception:

![dnSpy](/assets/img/blog/2017/11/sql-2017-installer-6.png)

Ah, ye olde null pointer. It seems we're trying to load a resource with a name that's null. The exception is thrown in mscorlib, but we can trace it back up the call stack to the InstallerEngine:

![dnSpy](/assets/img/blog/2017/11/sql-2017-installer-7.png)

When we do this, we eventually end up in the StartPageViewModel, where the exception is caught and we are redirected to the error page:

![dnSpy](/assets/img/blog/2017/11/sql-2017-installer-8.png)

_(in the above screenshot it's actually the second catch block that handles our exception, but this block also ends by navigating us to the error page)_

So what about the actual cause? Well, if we look at the InitializeDownloadManifest function, it seems that the installer is trying to load a Dutch language resource file:

![dnSpy](/assets/img/blog/2017/11/sql-2017-installer-9.png)

Such a resource file doesn't exist, and the installer just crashes instead of falling back to some default. This does make me wonder what happens if we try a different language. We'll set a breakpoint in this function and re-launch the app. When it hits, we'll manually change the language to "en-US".

![dnSpy](/assets/img/blog/2017/11/sql-2017-installer-10.png)

Let's continue execution and see what that does...

![SQL Server 2017 installer](/assets/img/blog/2017/11/sql-2017-installer-11.png)

Well then. I wasn't expecting that to work, but it seems that got us past the first error screen. Sadly, if we try to press any of those buttons, the whole thing still blows up:

![dnSpy](/assets/img/blog/2017/11/sql-2017-installer-12.png)

We've only changed the language in one place, to get us past this initial error screen. I'm guessing we haven't actually found the underlying cause, and if we want the installer to run without error, we'll have to fix this problem at its root.

If we take a peek at the ResolveLanguage function that the InitializeDownloadManifest function calls, we see that it uses some kind of global property bag to fetch the language code:

![dnSpy](/assets/img/blog/2017/11/sql-2017-installer-13.png)

We'll go into this PropertyBag class and place a conditional breakpoint at the setter function, that hits when the SqlSetupLocale property is set:

![dnSpy](/assets/img/blog/2017/11/sql-2017-installer-14.png)

Checking the call stack leads us to the InitializeLanguage function:

![dnSpy](/assets/img/blog/2017/11/sql-2017-installer-15.png)

Not a very surprising function name. Poking around this method for a bit finally gets us to the root of our issues:

![dnSpy](/assets/img/blog/2017/11/sql-2017-installer-16.png)

The function contains some logic for determining the language to use for the installer. It does this by looking at the culture values of the current execution environment (i.e. my PC). Because I live in The Netherlands but use Windows 10 in English, my CurrentCulture and CurrentUICulture values are different. The setup code can't properly deal with this, tries to continue in Dutch anyway, and then fails to load the non-existent Dutch resource files.

Interestingly, if we check out the TrySetCulture function (where most of the logic is contained), we see that it has an option for forcing an English locale installation:

![dnSpy](/assets/img/blog/2017/11/sql-2017-installer-17.png)

That looks like it might be the solution to my troubles! Now, if only we can figure out how to set this value...

Some more digging around in the code for the ForceEnglishInstall flag quickly leads us to a command-line parameter called "ENU", that seems like should do the trick:

![dnSpy](/assets/img/blog/2017/11/sql-2017-installer-18.png)

So now I suppose we'll just have to run the installer using the command line, aaand...

![SQL Server 2017 installer](/assets/img/blog/2017/11/sql-2017-installer-19.png)

Finally! All that debugging, and in the end we didn't even have to modify any code. Just add an obscure command line flag.

## tl;dr

Just run:

```
.\SQLServer2017-SSEI-Expr.exe /ENU
```

## Lessons learned

*   I guess Microsoft never assumed people from different countries might try to install their software.
*   dnSpy is awesome.
*   Generic error messages are completely useless. The InstallerEngine at one point actually produces an error that basically tells you everything you need to know ("your language is not supported, continue in English?"), but it gets swallowed by the calling code.
*   Being able to look in the code and debug is just so very, very useful. Code is the most precise documentation you can write, and being able to read it really helps in these kinds of situations. If the installer hadn't been written in .NET I'd still be sitting here, SQL Server-less.
