---
layout: post
title: "Protocol-relative URLs"
categories: blog
---

Here's a neat trick I've learned today: protocol-relative URLs. How do they work? Well, consider [my current homepage](http://www.bartwolff.com/), which has a link on it to this blog. My homepage can be viewed both over HTTP and HTTPS. Previously, the link to my blog looked like this:

```html
<p>
    <img src="/Content/blog.png" alt="Blog" />
    <a href="http://blog.bartwolff.com/">read blog</a>
</p>
```

This meant that whether or not you were viewing the page over HTTPS, the link would always send you to my blog over HTTP. So can we do something about this? Preferably something that doesn't involve JavaScript? Well, if we change the snippet above to the following:

```html
<p>
    <img src="/Content/blog.png" alt="Blog" />
    <a href="//blog.bartwolff.com/">read blog</a>
</p>
```

...we get what is called a _protocol-relative URL_. If you're viewing the page through HTTP, it'll link you to my blog over HTTP, and if you're viewing the page over HTTPS you'll get a link using HTTPS. By leaving out the protocol from the URL, the currently used protocol is selected, in a way that's similar to how "regular" relative URLs work.
