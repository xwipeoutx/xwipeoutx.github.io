---
layout: post
title: WebXR - the game changer
categories:
- Programming
- XR
tags:
- XR
- VR
- WebXR
- BabylonJS
date: 2020-08-08 21:30:00 +1000
---
Let's start with a message of mine on the [HoloDevelopers Slack Community](https://holodevelopersslack.azurewebsites.net/) I wrote yesterday:

> So I'm doing my first WebXR project now (on Babylon). I must say, I'm loving it. Having the power of every NPM module under the sun makes it extremely easy to integrate with things, and the async-by-default nature of the web just makes delivering content nice and simple. Deployment of new versions and things is also ridiculously simple, including things like multiple environments (dev/test/prod type thing) - it's basically using the maturity of the web for immersive experiences.
> 
> Plus it's easy to preview/emulate the base experience, and you can just use normal web stuff for debug/UI/options etc.  It's just an amazing fit.
> 
> I just felt like gushing. Go back to your normal lives 🙂

I think that captures it all quite nicely - really, you can stop reading here if you want - or the even more conscise version:

**WebXR is a game-changer in immersive technologies, especially in enterprise.**

Why? Read on - but be warned; this is developer centric. You won't find many art or design opinions here.

<!--break-->

## In the user's hands, faster

App deployment in enterprises right now is not a fun affair.  Dealing with MDM, provisioning profiles with user-specific lists, not having the same hardware as the users (e.g. I run Windows - sorry iOS users, I can't build your software) and/or manual installation steps - these should be familiar to anyone deploying in an enterprise.

It's especially problematic when you're taking your product to customers - saying "Please download and install my app (after I get through the submission process), so I can sell you my widgets better" goes down like a lead balloon.

The web is another ballgame entirely.  I can push to GitHub and my software is available in seconds (maybe minutes if it gets complex).

Getting a user to test it out is saying "press F5".  There you go, new version is just there.  No updating from the app store required.

Want to take it to the public? Well you already have a website. There you go! Or, send them a link.  Couldn't be easier.

It's also much simpler to have multiple streams - eg. "Dev", "Test", "MPE", and "Production".  We've been doing multi-environment web pushes for years now.  The tooling on your CI/CD platform of choice does this out of the box - simply see the "DevOps 101" part of the documentation.

In general, the move from App Dev to Web Dev means faster iterations, quicker feedback, and costs you less money. Why should immersive tech be any different?

## Easy to pick up, for experts and newbies alike

Existing non-game developers can jump into WebXR relatively simply - no learning complex editor UIs, just grab a starter, download some assets and render it.

The _Hello, World!_ of Augmented Reality - [show an astronaut in your living room](https://modelviewer.dev/examples/augmented-reality.html) - is, well, easy  - literally some script includes, and a single HTML tag

<script type="module" src="https://unpkg.com/@google/model-viewer/dist/model-viewer.min.js"></script>
<script nomodule src="https://unpkg.com/@google/model-viewer/dist/model-viewer-legacy.js"></script>
<model-viewer src="/assets/models/Astronaut.glb" ar ar-modes="webxr scene-viewer quick-look" ar-scale="auto" camera-controls alt="A 3D model of an astronaut" skybox-image="/assets/environments/aircraft_workshop_01_1k.hdr" ios-src="/assets/models/Astronaut.usdz" style="width: 640px; height: 400px;"></model-viewer>

<script src="https://gist.github.com/xwipeoutx/b2b2b189dd037409b41c7b4183613e81.js"></script>

Honestly, as proper app in an engine isn't much more complicated - [BabylonJS does it in 20 lines of code](https://www.babylonjs-playground.com/pg/F41V6N/revision/32).

## Better integrations

What's your average enterprise running their entire operation on right now? It's almost certain that most people spend their days with a browser window opened, or some internet-enabled technologies doing their day-to-day work.

I work with someone who bought and uses a Chromebook, because he "doesn't need anything but a browser".

If you're building out enterprise solutions, you will be integrating with these other services.  Make it easier on yourself, and use the well-known web standards for this - what is better equipped for calling a JSON service: Unity, or a web browser?

## NPM. Wow.

> _“Any application that can be written in JavaScript, will eventually be written in JavaScript.”_
>
> -[Jeff Atwood, Author, Entrepreneur, Cofounder of StackOverflow](https://blog.codinghorror.com/the-principle-of-least-power/)

Look, it's a reality the NPM has a [ridiculous number of packages](http://www.modulecounts.com/). There's an NPM package for everything. It has more than all the other languages combined. 

Need to consume/render GIS data? [TurfJS](https://turfjs.org/)

Need to do machine learning? [TensorFlow](https://www.tensorflow.org/js)

OCR? [Tesseract has been ported](https://tesseract.projectnaptha.com/)

[Authentication](https://github.com/IdentityModel/oidc-client-js), [Analytics](https://analytics.google.com/), [PDF Rendering](https://mozilla.github.io/pdf.js/), [DOOM](https://js-dos.com/DOOM/), [Real Time Communication](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API) - JavaScript, and the browser has it all.

And by using WebXR, you can have it in your project.

## Better suited UI

A lot of enterprise experiences have a decent amount of pre-work before starting your immersive experience. Take VR training - you need to authenticate, selecting your training modules, configure any scenarios (maybe even review past ones first) - before you even begin.  Once you're done, you might need to review/comment on it, do quizzes, submit it.

I don't want to do that in VR - noone likes typing on a VR keyboard, and any VR based menu system is going to be harder to learn than a web-based one.

So why not build that UI in HTML, and then pop the VR headset when you're good to go? It's all _just web stuff_

## Control your code

Using game engines (Unity, I'm looking at you) involves forsaking control.  Want some behaviour in Unity? Inherit from MonoBehaviour, use these ~~magically~~ specifically named methods.  Oh, and forget about dependency injection.

These things don't hold so much in WebXR lands - engines like [three.js](https://threejs.org/) and [babylon.js](https://www.babylonjs.com/) are built in the JavaScript ecosystem, which has long favoured modular systems that can be independenty controlled.

Don't like the physics engine? Switch it out. Don't like multiplayer system? I don't have to wait for [UNet to be deprecated](https://blogs.unity3d.com/2018/08/02/evolving-multiplayer-games-beyond-unet/?_ga=2.79025494.393690968.1596882967-720316448.1594185464) - I can switch it out.

## Async by default

Loading an asset over HTTPS? Guess what, it's asynchronous.  You can't avoid it.

Game engines generally do avoid it. You have one workflow for embedded content (drag the model into the scene), and one workflow for async content (cry and Google it yet again).

Why is this a good thing? Well, it's async by default - so you won't fall into that trap of "Oh wait, I want that asset to be externally, I have to rejig my whole scene".  You're already async.

## Summing up

I've painted a fairly rosey picture of WebXR - I honestly believe that this standard is the tipping point for enterprise immersive tech uptake.

Not everything is perfect - the standards aren't fully settled yet, JavaScript is slower* than compiled code (and lacks useful constructs like custom value types 😢), and browser support is OK at best.

_* Worth mentioning, I am nearly always GPU-bound on my projects, not CPU-bound. So JavaScript speed isn't really a big deal.  Enterprise apps don't necessarily have a lot of AI, physics, etc. going on for this to be a problem.  That said, lack of custom value-types (structs) are painful, especially for vectors. Gotta keep cloning them!_

Ultimately, you will still need to test cross-device. You will still have to keep one-eye-in-the-headset when debugging. And you will _also_ have to deal with browser quirks.

But you will be faster. You will get feedback quicker.  It will be safer. You will have more control, and more options. It will be easier to pivot. You will have simpler access to SDKs and integrations.  Playgrounding code will be trivial.

It's a game changer.