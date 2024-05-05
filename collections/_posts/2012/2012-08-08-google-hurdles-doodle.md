---
layout: post
title: "The horrible implications of cheating at the Google Hurdles doodle"
categories: blog
---

So Google has a [hurdles doodle](https://www.google.com/doodles/hurdles-2012) up where you play as an athlete competing in the 100m hurdles. You run by alternating between the left- and right arrow keys, and jump with space. Alternate left and right faster and you run faster. Obviously being a programmer this means I will run like the fastest bastard that ever lived:

```csharp
[DllImport("user32.dll")]
static extern bool PostMessage(IntPtr hWnd, UInt32 Msg, int wParam, int lParam);

public const int WM_KEYDOWN = 0x0100;               // Key down flag
public const int WM_KEYUP = 0x0101;                 // Key up flag
public const int VK_LEFTARROW = 0x0025;             // Left arrow key
public const int VK_RIGHTARROW = 0x0027;            // Right arrow key

static void Main(string[] args)
{
    var processes = Process.GetProcessesByName("firefox");
    var target = processes[0].MainWindowHandle;

    while (true)
    {
        PostMessage(target, WM_KEYDOWN, VK_LEFTARROW, 0);
        PostMessage(target, WM_KEYUP, VK_LEFTARROW, 0);

        System.Threading.Thread.Sleep(1);

        PostMessage(target, WM_KEYDOWN, VK_RIGHTARROW, 0);
        PostMessage(target, WM_KEYUP, VK_RIGHTARROW, 0);

        System.Threading.Thread.Sleep(1);
    }
}
```

We'll be sending alternating left- and right arrow key presses to the browser window with only a single millisecond of delay in between them. I didn't even bother with jumping. The current world record stands at 12.29 seconds [[1]](https://en.wikipedia.org/wiki/100_metres_hurdles#Milestones), so let's see how we did in comparison:

![How did this not get me 3 stars?](/assets/img/blog/2012/08/google-hurdles.png)

Less than a single second. Not bad. Although... now that I think about it...

Traveling 100 meters in 0.8 seconds means the hurdler has to have had an average speed of 125 m/s. That's 450 km/h. And considering we observe him reaching this top speed almost instantaneously, say in 100 ms, this means he is hit with a 1250 m/s<sup>2</sup> acceleration force. That's about 127 g. This cannot be healthy for a human being to experience, but our intrepid hurdler doesn't have time to think about such things seeing as in less than a second he's smashed straight through 10 standard track hurdles at 450 km/h before coming to a full stop almost directly after crossing the finish line, resulting in another 127 g of deceleration.

I fear I may have killed someone today.
