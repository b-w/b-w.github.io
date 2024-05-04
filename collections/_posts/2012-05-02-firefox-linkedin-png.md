---
layout: post
title: "Bug: Firefox refuses to load linkedin.png"
categories: blog
---

A few days ago, I've changed the default [bartwolff.com](http://www.bartwolff.com/) homepage from a redirect to my blog to a very basic business card type deal. It seemed nicer. Anyway, that's not the point. The point is that while developing this simple little webpage, I've encountered a strange bug in Firefox: it simply would not load an image when that image is called _linkedin.png_.

Here's what the relevant part of my site looks like viewed in Internet Explorer 9:

![](/assets/img/blog/2012/05/load-linkedin-ie.png)

And here's what the exact same part looks like in Firefox 12.0:

![](/assets/img/blog/2012/05/load-linkedin-firefox.png)

As you can see, Firefox plainly refuses to either show or load this image. When viewing the source, both browsers display exactly the same:

```html
<div class="right">
    <p>
        <img src="/Content/blog.png" alt="Blog" />
        <a href="http://blog.bartwolff.com/">read blog</a>
    </p>
    <p>
        <img src="/Content/linkedin.png" alt="LinkedIn" />
        <a href="http://nl.linkedin.com/in/bgjwolff/">connect</a>
    </p>
</div>
```

Except for the file name, these is no notable difference between the two images (I've already determined the alt text makes no difference in behavior). They're both PNG image files, both in the same directory, even both of the same size. So why can't Firefox display the one when it can show the other? When I opened up Firefox's web console and reloaded the page, here's what I saw:

    GET http://www.bartwolff.com/ [HTTP/1.1 200 OK 24ms]
    GET http://www.bartwolff.com/Content/Site.css [HTTP/1.1 200 OK 282ms]
    GET http://www.bartwolff.com/Content/blog.png [HTTP/1.1 200 OK 347ms]

It didn't even try to fetch the image! Why not?! From manually browsing to the image's URL I know Firefox can at least display the image, so why doesn't it even try to load it when rendering the page? Well, I honestly don't know, but I do know that when renaming the image to _connect.png_ this is what Firefox shows me:

![](/assets/img/blog/2012/05/load-linkedin-firefox-success.png)

Here's the source:

```html
<div class="right">
    <p>
        <img src="/Content/blog.png" alt="Blog" />
        <a href="http://blog.bartwolff.com/">read blog</a>
    </p>
    <p>
        <img src="/Content/connect.png" alt="LinkedIn" />
        <a href="http://nl.linkedin.com/in/bgjwolff/">connect</a>
    </p>
</div>
```

As you can see, they're exactly identical up to renaming. Here's the web console output from Firefox:

    GET http://www.bartwolff.com/ [HTTP/1.1 200 OK 16ms]
    GET http://www.bartwolff.com/Content/Site.css [HTTP/1.1 200 OK 8ms]
    GET http://www.bartwolff.com/Content/blog.png [HTTP/1.1 200 OK 82ms]
    GET http://www.bartwolff.com/Content/connect.png [HTTP/1.1 200 OK 23ms]

This is what I was expecting in the first place. So what went wrong? What does Firefox have against LinkedIn? Are there any more randomly named resources that refuse to load? It's questions like these that keep me awake at night.
