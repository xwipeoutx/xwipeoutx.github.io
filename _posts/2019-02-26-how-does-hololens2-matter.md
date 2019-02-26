---
layout: post
title: How does the Hololens 2 matter?
categories:
- Programming
- XR
tags:
- XR
- Hololens
- Unity
date: 2019-02-26 23:30:00 +1000
---
So that happened, [the Hololens 2 has been released](https://www.youtube.com/watch?v=c1CZsqwnWtM).  A few people have asked me what I think, so it's about time I got my thoughts down on ~~paper~~ the interwebs.  I've had a day or 2 to mull over it now - so here it is for you.

The main features touted are improved FOV, fully articulated hand tracking and increased engagement and collaboration - partnerships and technology enablements.

But how will this play out? How much is smoke and mirrors? What are the effects here?  Read on for my thoughts.

<!--break-->

## Field of view

The elephant in the room (that you can't see much of because it doesn't fit in viewport).

If you gave a Hololens 1 user/dev a handful of Microsoft dollars to improve the headset, chances are they'd spend it all improving the field of view.

Well, that's what they've done.  2x improvement is their numbers.  But let's look at the numbers

### How much better?

Fortunately [Alex Kipman tweeted](https://twitter.com/akipman/status/1100069645661495298) the FOV details for us.

Hololens 1 has a FOV of **30° horizontally**, and **17.5° vertically** (source: [Wikipedia](https://en.wikipedia.org/wiki/Microsoft_HoloLens)).  So, thanks to pythagoras, we get a **diagonal FOV of around 35°**.  Note the aspect ratio - approx 16:9

Hololens 2 now has a **52° diagonal FOV** and a 3:2 aspect ratio - so **43° horizontally** and **29° vertically**.  

Just for reference, the magic leap has **40° horizontally**, **30° vertically** - so a **diagonal of 50°**.

The image on [UploadVR](https://uploadvr.com/hololens-2-field-of-view/) is a good view comparing all 3, but I'm gonna do it here in CSS _because I can_ (and so you can view the source to check my measurements for yourself).

<div style="position:relative; width: 600px; height: 320px; text-align: center; background: #1e1e1e">
    <div style="position: absolute; border: 4px solid rgba(255, 255, 0, 0.5); width: 430px; height: 290px; top: 5px; left: 87.5px; color: rgb(255, 255, 0); text-align: center;"><br />Hololens 2</div>
    <div style="position: absolute; border: 4px solid rgba(0, 255, 255, 0.3); width: 400px; height: 300px; top: 0px; left: 100px; color: rgb(0, 255, 255);">Magic Leap</div>
    <div style="position: absolute; border: 4px solid rgba(255, 0, 0, 0.3); width: 300px; height: 175px; top: 62.5px; left: 150px; color: rgb(255, 0, 0);">Hololens 1</div>
</div>

The take home: It's pretty much the same as the Magic Leap, and calling it double is pretty misleading.  

It's not even double the perimeter (`95°` vs `144°`) but it **IS** more than 4x the area (`525°²` vs `2236°²`) - but these units don't even make sense* !  **Comparing diagonals gets about 1.5x improvement**, which I think it the more realistic number.

_(*ok, so a [square degree](https://en.wikipedia.org/wiki/Square_degree) is apparently a valid measurement, if not SI-compliant.  There are apparently 41,252.96 square degrees in a sphere, so if we stack 19 Hololens 2 units perfectly, we have ourselves a fully Holographic sphere)_

So don't get hyped on that!  It's a nice improvement, certainly, but it's not mind blowing as it sounds - and certainly not "double the FOV" unless you _really_ stretch the definition of FOV.

### What does it mean?

I'll throw this out there - a bigger FOV is not better because you can fit more on - it's better because you can _get **closer** to your holograms_.

Microsoft recommends the holograms do not show any nearer than 85cm for the original Hololens - any closer and your eyes go all funky.  Why? Because at that distance, some of the hologram is hanging off the side of one of your eyes, and not the other.

To show what I mean, I've done a quick Unity scene - a single sphere with a radius 15cm sitting 50cm* from the eyes.  This is the approximate view on each hololens for the sphere:

(*I chose these sizes because that makes the sphere take up a similar viewport % on the Hololens 2 as the same size sphere at 85cm on the original Hololens)

![Comparing Hololens and Hololens 2 FOV impact](/images/2019-02-26-fov-example.png)

**Figure**: Comparison of Hololens 1 and 2 rendering from the left and right eyes

The first view will make you feel cross-eyed, with about 20% of the object cut off on each eye.  Comparitively, on the Hololens2, most of the sphere is in view - it will appear close to you, but your eyes still agree on what they're seeting.

THIS is why the FOV is _so important_ - at 85cm, you can only _just_ reach the holograms.  at 50cm though? It's within reach.  This is a big deal. Arguably the high quality finger/hand recognition wouldn't have worked well without this proximity anyway.

## Gestures

You know what you get sick of when showing people the Hololens?  Teaching someone to Air-Tap.

The always want to be reaching forward and touching the Holograms - but it's been impossible.

I'm gonna say it - I think this high-accuracy hand and finger tracking is the killer feature for this device.  Assuming minimal smoke-and-mirrors, [Julia Schwarz played a freakin' holographic piano](https://youtu.be/c1CZsqwnWtM?t=2200).  

I think this is what the Holographics UIs have been waiting for - simple, intuitive interaction.  No longer do I need to make a `+` and `-` button for a number in put - sliders will work! No longer does someone have to be looking _directly_ at a button to tap it - the can just... _actually_ tap it.

This will greatly reduce the main issue with using the Hololens in educational environments - having a no-training-required experience to learn the app.  It's important!

Of course, there's an issue.  Notice Julia is always an arms length away from what she's interacting with?  My guess is the near-clip plane sits at about 40-50cm for the Hololens 2 by default, so we're not getting any small-and-up-close UI elements any time soon.  Expect to have to reach out to them.

Admittedly this is a very minor issue.

## Performance

We got ourselves a pretty big upgrade here!

Hololens 1 was pretty bad performance-wise.  You're lucky to push 400k triangles, even with the simplest of shaders.  Want reflections? Hope you're not using too many pixels! And do you have a lot of transparency? Are you overdrawing? Not on this device, my friend!

Enter Hololens 2 - it has a [Qualcomm Snapdragon 850 Compute Platform](https://mspoweruser.com/full-tech-specs-of-microsoft-hololens-2/) for a CPU/GPU - compared to the 1Ghz Atom processor in the Hololens 1.

We're talking nearly a tripling of maximum clock rate, and a huge jump in GPU power too.  If Hololens 1 was an iPhone 5 (a good comparison), then Hololens 2 is an iPhone X + 20% ish.

So we've got some actual (comparative) horsepower now.

The flipside? We're having to push many more pixels with the FOV increase.  With a viewing area jump of about 4x, we have about 4x the number of pixels - the equivalent to rendering 4k instead of 1080.

So it's gonna _need_ that extra GPU power.

And honestly? CPU power hasn't been a problem in any of the apps I've created.

We'll have to wait and see how this performance pans out.  Having recently done a fairly graphics intensive iPad-Pro ARKit application, I know first hand that graphics power unlocks a lot of that "wow" factor that's hard to get on the original Hololens.  I hope we can get that same "wow" on the Hololens 2.

## Partners and open tech

In case you missed it, [Readify](https://readify.net) made a front and center appearance on the stage with Alex Kipman!

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Oh hi there <a href="https://twitter.com/readify?ref_src=twsrc%5Etfw">@readify</a> logo 🧐<br><br>Recognition for some great work done, and being done, by our teams! 💪 Great to be part of this journey with <a href="https://twitter.com/HoloLens?ref_src=twsrc%5Etfw">@HoloLens</a>, and their HoloLens 2 launch this morning. <a href="https://t.co/FI5Cd8IpHf">pic.twitter.com/FI5Cd8IpHf</a></p>&mdash; Tatham Oddie (@tathamoddie) <a href="https://twitter.com/tathamoddie/status/1099797151943553024?ref_src=twsrc%5Etfw">February 24, 2019</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Ok, so more like... rear and slightly to the right.  You get the point.  I'm pretty sure he picked his stage position careuflly.

Microsoft have been putting in the hard yards attracting, partnering and equipping partners to deliver this stuff well - and I must say, it's been super useful to partner with them in this.

They really are thinking of developers for this journey - with the [MRTK](https://github.com/Microsoft/MixedRealityToolkit-Unity) and a [other tools](https://github.com/Microsoft/VisualProfiler-Unity) that I use on every gig (and [probably future gigs](https://github.com/Microsoft/MRLightingTools-Unity)), it makes everything easy.

Indeed, the next version of this tooling goes _beyond_ Mixed Reality devices - Microsoft seem to be bringing in other AR devices (Android, iOS) into the fold and enabling them to work.  My guess is this is part of the "use whatever OS you want, as long as it's in Azure" aspect - providing services like cloud anchors and speech understanding transcends devices, and brings in the bacon.

I've heard enterprises are squeamish about this openness - but I whole-heartedly embrace it.

_**Shameless plug**: Do you want the chance to work on this stuff? Reach out - Readify are always hiring.  Reach out on [twitter](https://twitter.com/xwipeoutx) or hit up [https://join.readify.net](https://join.readify.net)._

## Unfortunately...

Sadly, some things are missing here that I was really hoping for.

Firstly, there's no talk of GPS/Compass/4g/5g support.  Outdoor is not a thing.  I get it - this sort display does not lend itself to outdoor - but I was kinda hoping, y'know?

I haven't heard any word on the development story being improved - specifically, holographic remoting and build times (IL2CPP is a slow way to go about things).  Native code is a bit of a second class citizen here, so I don't have much hope for this improving.  But dang it would be nice to go from pressing "build" to on device in seconds rather than _several_ minutes.

## Final bits

I haven't touched on a lot of the cool things to watch out for - mostly because they were cool, but I didn't have any insight.

Windows Hello integration and eye tracking is super cool - finally some simple (automatic) IPD adjustment, which should sharpen things up a bit!  I'm pretty interested in how the eye tracking will work from an interaction perspective. I suspect _really annoying_ - we're not used to things moving with our eyes.

The comfort factor is a big one, and it does look much better in this regard.  I'm torn about the location of the computing device (keeping it on the unit instead of offloading it a.l.a. Magic Leap).  Center of Mass is important, and I'm glad it's to the geometric center now - but overall weight is, too.  We'll see.

Azure Kinect might be a bit of a game changer.  I see this becoming a 3d room scanner, a remote-assist companion, a game enabler and a cheap way to get a high quality spectator view for an application (c'mon first party support out of the box for this!).  When I watched the MWC announcement, _this_ was the most exciting to me (though mostly because I'd assumed the rest was happening already).

This release is _super_ exciting, I can't wait to get my hands on one and see what it can do.