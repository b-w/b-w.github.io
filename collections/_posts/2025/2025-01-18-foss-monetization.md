---
layout: post
title: "How not to make money from open-source software"
categories: blog
---

Building and maintaining FOSS software can be a thankless endeavour. Hundreds, thousands, or even millions of people could be using your software on a daily basis, including for commercial purposes, but without giving you anything in return except for complaints when something breaks.

It should be no surprise then that monetizing open-source projects is notoriously difficult. Today, I'll look at two recent examples from the .NET ecosystem (where I mostly dwell) of large and popular projects that shot themselves in the foot in spectacular fashion while trying to get that bag. These are their stories.

## Moq

Moq is a mocking library used in unit testing. It lets you create and configure mock objects, as well as perform assertions on them:

```csharp
// setup
var fooMock = new Mock<IFooService>();
fooMock.Setup(x => x.Bar("something"))
       .Returns(42);

IFooService fooService = fooMock.Object;

// test
var actual = fooService.Bar("something");

// assert
Assert.IsEqual(actual, 42);
fooMock.Verify(x => x.Bar("something"), Times.AtMostOnce());
```

It's one of several mocking libraries available within the .NET ecosystem, but with (at time of writing) [730+ million downloads](https://www.nuget.org/packages/Moq) on NuGet it is by far the most popular one. It is licensed under the [3-Clause BSD license](https://opensource.org/license/bsd-3-clause), which freely permits both non-commercial and commercial use.

Then, one day, an innocent minor version update of Moq was released, jumping from version 4.18.4 straight to 4.20.0 (and yes, using the funny weed number for this was apparently intentional). And this update brought with it the true villain of this story: **SponsorLink**.

### Enter SponsorLink (stage left)

SponsorLink is developed by the same author as Moq, Daniel Cazzulino. Its purpose is to integrate with [GitHub Sponsors](https://github.com/sponsors), an (unaffiliated) initiative by GitHub that allows individuals and companies to more directly sponsor open source projects as well as individual developers.

However, unlike the open-source projects it aims to support, SponsorLink itself is actually not only **closed**-source, but distributed only as an obfuscated binary. An obfuscated binary that is now suddenly attached to every installation of the Moq library. This by itself is already a huge red fag.

But what does SponsorLink actually do? Well, it includes an *analyzer* that runs every time you build your code. This analyzer looks at the configured email from your Git config (by running `git config --get user.email`) and then calls an Azure blob storage account somewhere to see if this particular email address is a registered sponsor of whatever organization or project the current library belongs to. Which effectively exfiltrates PII from every computer it runs on. So now we're looking at a GDPR incident, lovely. That is red flag number two.

Finally, if your are **not** detected as a sponsor, SponsorLink:

1. Delays your build by some random amount of time, which could be several seconds.
2. Generates a build warning that nags you about sponsorships.

The first one is merely an insult, but the second point actually had significant consequences where it started breaking builds (including CI builds) in projects that treat warnings as errors. Big, big red flag number three. None of this was communicated or announced, by the way. It was just suddenly there for users to find. In a minor version update, no less.

{% include image.html url="/assets/img/blog/2025/01/sponsorlink.png" description="This is a real screenshot from real software that someone shipped." %}

All these red flags together resulted in a nice little [security advisory](https://github.com/advisories/GHSA-6r78-m64m-qwcf). That's typically when you know you really done fucked up.

### The community reacts

Predictably, the community responded with significant backlash, pointing out all of the red flags mentioned above, venting their frustration over broken builds and now having to remove Moq from countless projects over GDPR concerns, and trying to make Daniel see reason.

{% include image.html url="/assets/img/blog/2025/01/gh-sponsorlink-discussion.png" description="The community was angry that day, my friends..." %}

That last part proved difficult, as Daniel seems to have... *interesting* ideas about open-source monetization and sustainability. In his introductory [blog post](https://www.cazzulino.com/sponsorlink.html) about SponsorLink, he writes:

> And I'm a firm believer that supporting your fellow developers is something best done personally. Having your company pay for software surely doesn't feel quite as rewarding as paying from your own pocket, and it surely feels different for me too. We really don't need to expense our employers for a couple bucks a month, right??

It almost reads like the infamous "pride and accomplishment" comment.

Furthermore, in his various discussions in the GitHub issues for Moq and SponsorLink, he also doesn't seem to feel there is any issue interfering with a developer's IDE to nag them about sponsorships. And even though he did eventually remove SponsorLink from Moq, it doesn't seem like he sees any real issues with the core ideas behind it. He also doesn't really seem capable of looking further than his own projects. Let's imagine for a second that every library on NuGet started doing this, and now the average project is riddled with a dozen or more SponsorLink-integrated dependencies that turn developers' IDEs into an early-2000s shareware hellscapes... did this thought never occur to him?

Now, more than a year since *the incident*, Moq is still free from SponsorLink (which has since been open-sourced itself), but a lot of community trust has been burned overnight that will likely take much longer than this to rebuild, if it can be rebuilt at all. And with SponsorLink still looming over the horizon, it's easy to see why many developers simply switched to another mocking library.

## FluentAssertions

FluentAssertions is another helper library for unit tests (why is it always unit tests?), focused on (as the name implies) letting you write your *assertions* in a more *fluent* way.

So, instead of something like this:

```csharp
Assert.IsEqual(actual, 42);
```

...you can do this:

```csharp
actual.Should().Be(42);
```

Whether you like it is a matter of taste, but I personally think it offers some nice syntactic sugar and useful deep-object comparison with good error messages. Regardless, it is one of the most popular libraries on NuGet with (at time of writing) [450+ million downloads](https://www.nuget.org/packages/FluentAssertions).

The project has always been licensed under the [Apache 2.0 license](https://opensource.org/license/apache-2-0), which (among other things) allows it to be freely used in commercial applications. But then, on Jaruary 14th, with the release of major version 8, FluentAssertions suddenly moved to the "Xceed Software Community License", which still allows for free non-commercial use but requires a paid license otherwise.

There are two major problems with this move and how it was handled, which lead to substantial community backlash.

### Pricing

The first and most obvious problem is the pricing itself. Now, pricing is a difficult topic about which entire books have been written (some of which I've actually read, believe it or not). But as a general rule of thumb, the price of something should roughly align with the true value of that thing.

And don't get me wrong, I'm not trying to belittle the value that FluentAssertions offers, nor the time its contributors have invested in creating it. But let's also not have any illusions of grandeur and just call it for what it is: a nice little focused helper library that makes writing unit tests a bit nicer.

And the price its creators put on this is:

> $130 per seat, per year

...yes, really.

My brother in Christ, that is more than we pay for [JetBrains Rider](https://www.jetbrains.com/rider/), a fully featured IDE. *What the hell are you smoking?*

### Timing and Communication

The second issue is the way in which the change was made and communicated.

As I mentioned, the project was under the Apache 2.0 license for the entire time up until version 8. And by "up until version 8", I actually mean "mere hours before the release of version 8". Because yes, the actual change to the Xceed license was made in [a commit](https://github.com/fluentassertions/fluentassertions/pull/2943) by project owner Dennis Doomen on the same day that v8 went live. It even came as a surprise to the other contributors.

This has some interesting implications. A bunch of volunteers have contributed code to version 8 while the project was still under the Apache 2.0 license, only to have that code now suddenly ship in a differently licensed product. There is ongoing community discussion about whether this is even allowed under the terms of the Apache 2.0 license, as it seems like re-licensing of (parts of) code like this is simply not allowed without explicit permission from each contributor.

But even aside from any legal issues, it is most definitely a spit in the face of those contributors, as several of them have come out and affirm [[1]](https://github.com/fluentassertions/fluentassertions/pull/2943#issuecomment-2590630206) [[2]](https://github.com/fluentassertions/fluentassertions/pull/2943#issuecomment-2591040698) [[3]](https://github.com/fluentassertions/fluentassertions/pull/2943#issuecomment-2591099420) [[4]](https://github.com/fluentassertions/fluentassertions/pull/2943#issuecomment-2594782282) [[5]](https://github.com/fluentassertions/fluentassertions/pull/2943#issuecomment-2598413310). Those people who have helped make the project a success and have freely invested their time now feel betrayed by a decision they (apparently) had no input in or even notice of. A lot of goodwill has been burned here, so I'm hoping for Dennis' sake that the money is worth it at least.

On a side note, the communication from Xceed around the whole thing feels incredibly disingenuous, with the [FAQ page](https://xceed.com/fluent-assertions-faq/) regarding FluentAssertions feeling like it was just straight-up generated by an AI. It includes wonderful sections such as:

> 4\. How will the free version differ from the commercial version? 
> 
> The free version of Fluent Assertions will continue to offer the core functionalities that the community has come to rely on. The commercial version, on the other hand, will include additional features such as **enhanced scalability, advanced security options**, and priority support, which are tailored for enterprise needs.

Yes. Enhanced scalability and security. For a unit test assertions helper. That's enterprise software, baby!

But then again, this is the same company that includes the following in their "community license":

> Xceed does not allow Community Licensees to publish results from benchmarks or performance comparison tests (with other products) without advance written permission by Xceed.

I really don't know what to even think about that one.

## Summing up

The interesting thing about these two cases is that even though each project took a completely different approach at attempting some form of monetization, they both equally qualify for the summarizing statement of "...and they probably couldn't have handled it in a worse way if they tried." I do admire the creativity.

But if you're expecting me to wrap up with some kind of insightful analysis of how to address the issue of open-source software monetization and sustainability, don't hold your breath. I don't have any smart ideas on this one, unfortunately. Of course, turning a FOSS project commercial isn't impossible to do. To take another example from the .NET sphere, [ImageSharp](https://sixlabors.com/products/imagesharp/) actually had a fairly well-managed pivot from fully FOSS to split-licensing. But that's a story for another day. For now, I still have about 30 projects that need their FluentAssertions references pinned to v7...
