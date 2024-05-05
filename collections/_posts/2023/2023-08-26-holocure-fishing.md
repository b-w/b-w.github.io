---
layout: post
title: "Automating HoloCure fishing with OpenCV for fun and profit"
categories: blog
---

The recent 0.6 update of [HoloCure](https://store.steampowered.com/app/2420510/) included a small minigame where you can fish (because every game must include a fishing mechanic). It's implemented as a very simple rhythm game with just 5 possible inputs.

![](/assets/img/blog/2023/08/holocure-fishing-explained.png)

The input sequence flows from left to right towards the target, and the goal is to hit the correct input when its corresponding symbol is over the target. There is some leeway in how precise you have to be, but generally with rhythm games the closer you are to a perfect input, the better. In any case, hit a correct (i.e. good enough) input and you'll gain progress towards catching the fish, miss an input and you lose some. When the progress bar is either completely full or completely empty, the round ends and you either have caught the fish or let it slip away.

Simple enough, right? With the preliminaries out the way, let's set about automating this whole thing, and become someone fish truly fear. We'll be using Python 3 today and the heavy lifting will be done by [OpenCV](https://opencv.org/).

## Fish fear me

At a high level, our script will do the following:

1.  Wait for the HoloCure game window to be the active window.
2.  Check if we're currently fishing.
3.  If not, spam the Z key until we are. This is the "confirm" key for HoloCure, which both dismisses dialogs and is used to start fishing when near the pond.
4.  If we are fishing, check if one of the input symbols is in the target area.
5.  If it is, hit the corresponding input key.
6.  Rinse and repeat.

We'll tackle finding the active window first. For this we'll do something that feels a bit un-Python-like, and use [ctypes](https://docs.python.org/3/library/ctypes.html) to call the Win32 api. It seems to be the simplest way to do this reliably.

```python
import ctypes
from typing import Optional, Tuple
from ctypes import wintypes, windll, create_unicode_buffer

def get_active_window_title() -> Optional[str]:
    h = windll.user32.GetForegroundWindow()
    l = windll.user32.GetWindowTextLengthW(h)
    b = create_unicode_buffer(l + 1)
    windll.user32.GetWindowTextW(h, b, l + 1)

    return b.value if b.value else None

def get_window_size(title: str) -> Optional[Tuple[int, int, int, int]]:
    h = windll.user32.FindWindowW(None, title)
    if not h:
        return None
    r = wintypes.RECT()
    windll.user32.GetWindowRect(h, ctypes.pointer(r))
    return r.left, r.top, r.right, r.bottom
```

Easy. I've also included a function to get the size of the window, which we'll also need in order to know where to look. Yes, we will be "looking" at this window.

With these two functions we can start the main loop of our script.

```python
# [...]

try:
    while True:
        # wait for HoloCure to be the foreground window
        print('waiting for focus of HoloCure window...')
        while True:
            if get_active_window_title() == 'HoloCure':
                break
            time.sleep(1)

        print('have focus, fishing time!')

        # get HoloCure window dimensions
        (wLeft, wTop, wRight, wBottom) = get_window_size('HoloCure')

        # continue here...

        time.sleep(1)
except KeyboardInterrupt:
    print('interrupted')
```

We just keep polling the active window every second until we find what we need. That was the easy part. Now we'll need to detect if we're currently fishing. This is where OpenCV comes in. We're going to be "looking" at a specific part of the screen, and checking if this fishing hook image is present:

![](/assets/img/blog/2023/08/holocure-fishing-hook.png)

This hook is always present in the same spot when the fishing game is active, and there's no other UI elements interfering with it, so it's a good indicator.

The first thing we'll need is a template image, i.e. what OpenCV is going to be looking for. For this I've taken a screenshot of the hook image and removed the background, resulting in a transparent PNG:

![](/assets/img/blog/2023/08/holocure-hook-template.png)

The transparency is needed because the background behind the image in the game can vary. We then load this image with OpenCV and create a mask from the transparent pixels:

    t_fishing = cv2.imread('templ_fish_hook.png', cv2.IMREAD_GRAYSCALE)
    t_fishing_unchanged = cv2.imread('templ_fish_hook.png', cv2.IMREAD_UNCHANGED)
    _, t_fishing_mask = cv2.threshold(t_fishing_unchanged[:, :, 3], 0, 255, cv2.THRESH_BINARY)

We'll be comparing this template to a very small screenshot of the part of the HoloCure game window where we expect the hook to appear. Taking these screenshots will be done using [MSS](https://pypi.org/project/mss/), which is quite performant and can take screenshots at very high fps. We'll set up our "camera" and start snapping.

```python
# [...]

def press_key(name: str):
    pyautogui.keyDown(name)
    time.sleep(0.1)
    pyautogui.keyUp(name)

try:
    while True:
        # [...]

        with mss.mss() as ss:
            mon_fish = {"left": wLeft + 1222, "top": wTop + 300, "width": 64, "height": 90}

            while True:
                if get_active_window_title() != 'HoloCure':
                    break

                time_st = time.time()

                img = np.array(ss.grab(mon_fish))
                img = cv2.cvtColor(img, cv2.COLOR_RGB2GRAY)
                m = cv2.matchTemplate(img, t_fishing, cv2.TM_CCOEFF_NORMED, mask=t_fishing_mask)
                (_, maxV, _, _) = cv2.minMaxLoc(m)

                if maxV > 0.9:
                    # we seem to be fishing

                    # more here...
                else:
                    press_key('z')

                time.sleep(max(1\. / 60 - (time.time() - time_st), 0))   # try to run at ~60fps max

        time.sleep(1)
except KeyboardInterrupt:
    print('interrupted')
```

The inner loop of our script now monitors a very small section of the game window and uses OpenCV's template matching to see if the fishing hook image appears in it. If not, we use [PyAutoGUI](https://pypi.org/project/PyAutoGUI/) to spam the Z button. I've also included a rough way of limiting the inner loop to somewhere around a reasonable 60 fps.

As for what to do when we are fishing, it's actually just more of the same.

I've created template images for each of the 5 inputs, and now it's just a matter of trying to see if one appears in the target area. We'll set up a second monitor for MSS and then use some nice nested if statements to match our little screenshot against the 5 inputs:

```python
# [...]

try:
    while True:
        # [...]

        with mss.mss() as ss:
            # [...]
            mon_target = {"left": wLeft + 1132, "top": wTop + 712, "width": 110, "height": 130}

            while True:
                # [...]

                if maxV > 0.9:
                    # we seem to be fishing

                    img = np.array(ss.grab(mon_target))
                    img = cv2.cvtColor(img, cv2.COLOR_RGB2GRAY)

                    m = cv2.matchTemplate(img, t_arrow_left, cv2.TM_CCOEFF_NORMED, mask=t_arrow_left_mask)
                    (_, maxV, _, _) = cv2.minMaxLoc(m)
                    if maxV > 0.9:
                        press_key('left')
                    else:
                        m = cv2.matchTemplate(img, t_arrow_right, cv2.TM_CCOEFF_NORMED, mask=t_arrow_right_mask)
                        (_, maxV, _, _) = cv2.minMaxLoc(m)
                        if maxV > 0.9:
                            press_key('right')
                        else:
                            m = cv2.matchTemplate(img, t_arrow_up, cv2.TM_CCOEFF_NORMED, mask=t_arrow_up_mask)
                            (_, maxV, _, _) = cv2.minMaxLoc(m)
                            if maxV > 0.9:
                                press_key('up')
                            else:
                                m = cv2.matchTemplate(img, t_arrow_down, cv2.TM_CCOEFF_NORMED, mask=t_arrow_down_mask)
                                (_, maxV, _, _) = cv2.minMaxLoc(m)
                                if maxV > 0.9:
                                    press_key('down')
                                else:
                                    m = cv2.matchTemplate(img, t_button, cv2.TM_CCOEFF_NORMED, mask=t_button_mask)
                                    (_, maxV, _, _) = cv2.minMaxLoc(m)
                                    if maxV > 0.9:
                                        press_key('z')
                else:
                    press_key('z')

                time.sleep(max(1\. / 60 - (time.time() - time_st), 0))   # try to run at ~60fps max

        time.sleep(1)
except KeyboardInterrupt:
    print('interrupted')
```

I'm well aware this code is ugly as hell; I just can't be bothered to prettify it right now.

Anyway, this was the last part. We can now let slip this ungodly machine onto our game. It works surprisingly well, both in terms of performance (the script hits 1~2% CPU max while fishing) as well as gameplay (I haven't seen it miss a single input, let alone fail to catch a fish). Here's it reaching a chain of 500:

![](/assets/img/blog/2023/08/holocure-fishing-chain.png)

_Wishin' I was fishin'..._
