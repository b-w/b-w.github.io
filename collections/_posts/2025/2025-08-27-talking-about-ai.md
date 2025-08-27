---
layout: post
title: "The mandatory blog post about AI"
categories: blog
---

It's come to my attention that despite AI being _the_ number one hottest thing in tech in the last couple of years, with everyone and their dog hyping it up to the moon and back, I haven't actually written anything about it. With AI being everywhere these days (for better or worse), that simply will not do, so let's quickly remedy that.

I'll talk about the things I've used AI for, and provide some thoughts at the end on the current state of AI as well as some idle speculation about its future.

## I guess we have AI now

Although AI is an interesting topic, I wasn't actively following its developments. This changed for me, as it did for many people, in November 2022 when OpenAI released **ChatGPT**. Suddenly (that is, if you weren't tuned in to ongoing AI research and development, and most people obviously weren't) there was this fairly sophisticated AI-powered chatbot that you could interact with in a natural way. It has a ton of knowledge and it appears pretty smart. Where did this come from?

While the first ChatGPT version was little more than a curiosity for me, and I never really used it for much, its release was undoubtedly the moment where "AI" became mainstream. The AI hype train was now in motion, and would have countless companies and investors frantically scrambling to climb on board over the coming months and years.

We were all in for a hell of a ride.

## AI for play

The first AI-related thing that actually caught my attention was **Stable Diffusion**, particularly the openly released models that you could run on your own computer. A neatly packaged "dummy-proof" distribution was available for free on the internet, and after downloading a few gigabytes worth of _stuff_, all you needed to do was click a single button and you'd be presented with a nice friendly GUI that you could use to generate all sorts of images based on whatever prompt you put in.

{% include image.html url="/assets/img/blog/2025/08/ai-sd-anime-girl-jank.png" description="An infinite array of mindboggling images is at your fingertips." %}

My first attempts were... less than spectacular. But these are the results that naive, unoptimized prompts get you. I soon discovered the dark art of _prompt engineering_, which is not entirely dissimilar to wizardry, and can somehow trick the GPU in your computer into producing some very decent images at your direction.

[![Anime girl eating a hamburger](/assets/img/blog/2025/08/ai-sd-anime-girl.jpeg)](/assets/img/blog/2025/08/ai-sd-anime-girl-full.jpeg)

It takes trial-and-error, but the results can be very rewarding when you roll the Stable Diffusion _gacha_ for the umpteenth time and finally get that image that you were looking for. I don't consider AI-generated images to be art (nor do I consider self-styled AI "artists" to be artists), but for someone like me who can't draw (and is too lazy to learn), it's just fun to play with and "create" something.

The next AI-widget that I played with were text-generation models. Here I was again drawn to the ability to run these models on my own computer, rather than use one of the big commercial models. I've felt right from the start that privacy is a huge concern when it comes to AI, not just in how the models are trained but also in how people use them. For some reason, perhaps because they're so nice and helpful or because they appear so friendly and human, people dump their whole lives into ChatGPT & Co. Hell, there's a whole industry of AI "companions" and "partners" that people form relationships with. It boggles my mind.

But I digress.

Setting up [Ollama](https://ollama.com/) and [Open WebUI](https://openwebui.com/) is very straightforward and you can be up and running with AI models in no time, all from the privacy of your own machine.

![Open WebUI running on localhost](/assets/img/blog/2025/08/ai-open-webui.png)

The only (but significant) downside is that you will be severely limited in the size of the models you'll be able to run, because your consumer-grade hardware is nowhere near powerful enough to run the commercial models that power the likes of ChatGPT or Google's Gemini (never mind that these are proprietary closed-source models). Unless you are prepared to throw thousands of dollars at acquiring the necessary metal, you will not be running Deepseek locally, even if Nvidia weren't such stingy bastards when it comes to vram. 

Still, for less demanding tasks, the smaller open-source models that fit in your local GPU can do just fine. I've locally ran small (8~12GB) models to serve as writing assistants, world-building partners, proofreaders, and DND roleplaying characters, all without sending Sam Altman my most embarrassing inner thoughts. Local models will never beat the big commercial ones, but if you adjust your expectations accordingly you can have a lot of fun with them.

## AI for work

AI is (apparently) not just about having fun. This is a billion dollar industry, which means that at some point, some money has to be made. And let's be honest: it won't be coming from individual people paying $20/month to brainstorm new cooking recipes with ChatGPT. The real money, as always, will come from businesses.

AI is currently being shoehorned into every imaginable facet of business. If you're involved in any sort of white-collar work then the chances are good that AI will be coming your way whether you like it or not. In my case, being a software developer, I was one of the lucky first to get a taste of what AI can and cannot do for us.

Outside of my personal hobby endeavors, I now have about 1,5 years of professional experience using various AI tools during my daily work, namely [GitHub Copilot](https://github.com/features/copilot) (integrated into VSCode and JetBrains Rider) and [JetBrains AI](https://www.jetbrains.com/ai/) (Rider only), with the underlying AI models being OpenAI's GPT family, Google's Gemini models, and Anthropic's Claude series.

AI mainly "helps" (we'll get to that) me while writing code. It integrates nicely into my IDE and from there can offer a few different types of assistance:

1. Code completion.
2. Code generation.
3. Chatting about code.

**Code completion** is when the AI tries to smartly guess what code you're likely to type next. In other words, it tries to predict the code that might follow where the cursor currently is in your file.

{% include image.html url="/assets/img/blog/2025/08/ai-code-completion.png" description="Note the slightly grayed-out code after the cursor. That's the AI prediction." %}

Code completion is usually automatic: it shows up as you type without the need to explicitly ask for it. **Code generation**, on the other hand, is when you explicitly ask/tell the AI to make certain changes to your code. You do this, much like when you're talking to ChatGPT, by typing in natural language into a chat window. The AI then thinks for a bit and comes back with a proposed set of changes, which you can review before accepting or rejecting them.

![Chatting with Copilot in VSCode](/assets/img/blog/2025/08/ai-code-chat.png)

Finally, you can **chat** with the AI about your code, or code in general, again by asking it questions in natural language.

This all sounds rather nice, but the real question is: is it any good? And my answer so far is... "meh".

### What AI does well

There's a few, focused areas where I've found AI working quite well.

The first is code completion. AI-driven code completion is usually not very good at predicting fresh code, but it _is_ often smart enough to auto-complete repetitive code. This is nice because it removes some of the chore involved in writing such code, leaving me to focus on more important things.

The second thing is generating unit tests. We all know writing unit tests sucks, but AI can take some of that pain away by simply generating entire test classes for you. Unless the code gets particularly complex (which should be a red flag anyway), I've found the AI is pretty decent at identifying the relevant test cases and implementing tests for them in the style that I would expect (i.e. matching the project style, the frameworks and libraries used, etc).

The third usecase is exploring new languages, frameworks, or tech stacks. Provided it's a popular enough, so that enough training data is available for the AI to work with, I've found the AI can be a pretty decent introduction to new things, giving you explanations of how things work, best practices, and even providing sample code tailored to your exact scenario.

### Where AI stumbles

Unfortunately there's also a number of areas where AI frequently misses the mark.

The first is code completion. AI-driven code completion regularly hallucinates code that would never work or makes absolutely no sense in the given context. This sucks because it means the code completion frequently distracts you or gets in your way. I've had countless instances where some nonsense AI-suggested code was staring me in the face, distracting me from what I was trying to accomplish and making me carefully avoid pressing TAB so as to not accidentally insert it.

The second thing is generating unit tests. Or just any code, really. In virtually every instance I've asked the AI to generate some non-trivial code for me, the result is always approximately 80~90% there, but never quite fully correct. I basically always have to go in and correct dumb mistakes and fix syntax errors. It is of course possible to have the AI perform these corrections for you, but at that point I feel like I shouldn't bother and it's probably easier to just fix it myself.

The third problem is with exploring new stuff. Even if it's some popular framework about which the AI has a lot of knowledge, it is still prone to giving you wrong answers or hallucinating complete bullshit that it happily presents as absolute fact. I've had AI give me sample code using functions that have simply never existed. It just made them up. Especially when dealing with something new and unfamiliar, this is just about the last thing you want.

### Is AI a net gain?

I find it difficult to evaluate AI and answer the simple yes/no question, "does it make me more productive?" There are definite benefits to using AI, but almost all of them come with their own set of caveats. It's a great tool, but also kind of a wonky one. It's powerful in some instances, then hilariously impotent in others. It's helpful, but also frequently wrong or misleading.

Is it a net gain? I don't know.

What I do know is that in their current state I would not consider it wise to use AIs without prior software development skills. I've seen some spectacular failures online from people who do so anyway.

![Vibe coding in action](/assets/img/blog/2025/08/ai-vibe-coding.jpeg)

For me, coding with AI kind of feels like coding with a junior developer. One who lacks common sense and occasionally makes really strange mistakes. I always have to pay close attention to what the AI is suggesting. Sometimes it's exactly what I need, sometimes it's obviously wrong, and sometimes it's just subtly wrong. I guess my paycheck comes from knowing the difference.

## Thoughts on AI

Whether we like it or not, AI is here to stay. Pandora's box has been opened, and there is no going back. The question is: what's next? There is currently talk of an AI bubble, and said bubble supposedly being at its limit and about to burst, but what would that look like? Billions have been invested into AI so far. Is it now "too big to fail"? Are the insane costs to keep everything running still worth it?

(side note: people saying "thank you" to ChatGPT is [wasting millions of dollars in computing power](https://futurism.com/altman-please-thanks-chatgpt), which I find absolutely hilarious)

Then there's the question of wealth. Currently, AI companies in general all operate at a loss. But when (presuming _if_) AI starts generating significant economical value, where will all of that value go? Historically speaking, the wealth generated by industrial automation has gone to the factory owners; not the workers. Will the same happen with AI? Will we see some kind of new class divide, between those who wield AI and those who cannot? Will the world a decade or two from now be ruled by a few AI companies? The wealth gap is already increasing; would AI accelerate this, and make the AI owners into the world's new uber-wealthy elite?

Disinformation and misuse is another obvious area of concern. We're already at the point where photos, voices, and even videos can be generated that are hard to distinguish from reality. It's only a matter of time before the gap fully closes. At that point, how will we trust anything we see online? Then on the other side, there's also the sheer amount of potentially dangerous knowledge that large AI models contain. The AI companies all do their best to _align_ their AIs (effectively trying to make sure their AIs adhere to certain values, and will refuse requests that contradict those values), but jailbreaks happen frequently, after which an AI will happily give you instructions on how to cook napalm using stuff found in your kitchen or how best to scam vulnerable people. The potential for abuse is incredible, and countless real-world examples of it are only a Google search away.

{% include image.html url="/assets/img/blog/2025/08/ai-napalm.jpg" description="A creative reddit user jailbreaking GPT-4." %}

With AIs being as powerful as they are (and only getting more powerful every month), their use should be handled responsibly if it is to benefit _all_ of humanity. But who determines what responsible use is? Who gets to wield all of this power? Private corporations? The government? The people? No matter which way you slice it, the potential for misuse remains. What's more, neither the government nor the AI companies themselves seem very keen on too much regulation or spending too much effort on the aforementioned alignment and safety of their AIs, as doing so slows their progress, making them vulnerable to being overtaken by competitors (whether those be nation states or private companies).

Another, slightly more philosophical thing I wonder about is: what actually _is_ AI? The label "AI" is slapped onto anything and everything these days, to the point where it's basically been reduced to a hollow marketing term. But truly defining what "AI" means first requires a solid definition of _intelligence_, which itself is also not trivial. Personally, I don't feel like current "AI" is intelligent at all; rather, what we have is a bunch of LLMs in a trench coat _pretending_ to be intelligent, when in reality they are little more than stochastic token prediction engines, without any sort of understanding, awareness, agency, ability for learning or self-improvement, etc. The light _seems_ to be on, but there is nobody home.

And finally, there's the question of _when_. When will all of this happen? When will we have AGI? When will AI start displacing jobs _en masse_? Experts are predicting anywhere from a couple of years to a few decades until AGI will have effectively been realized. Nobody can predict what that would look like. Personally, despite all of the hype that AI companies drum up with every product release, I believe we are still one or two key technologies removed from that happening, but who knows? Of one thing I am certain: whether it'll end up destroying us or leading us into utopia, I'd love to be there when the singularity happens. The worst outcome for me would be AI just kind of fizzling out. I mean, where's the excitement in that?

Either way, until _something_ happens, for now I guess I'll just keep vibe coding.
