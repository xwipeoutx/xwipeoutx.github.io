---
layout: post
title: 3d icons for your Mixed Reality application
categories:
- Programming
- XR
tags:
- XR
- Unity
- HoloLens2
- UWP
date: 2020-09-01 10:30:00 +1000
---
The default app launchers for Mixed Reality applications built in Unity is a flat image - essentially a splash screen on a floating rectangle.

![2D app launchers on a couch](/images/2020-09-01-mixed-reality-3d-app-launcher/hl2-2d-launchers.jpg)

I'm sure we can have better immersion than a bunch of 2d panels around the place.

The good news is, Windows Mixed Reality supports this, and have a [guide on how to do it](https://docs.microsoft.com/en-us/windows/mixed-reality/implementing-3d-app-launchers) - so you can easily turn it into this:

![3D app launchers on a couch](/images/2020-09-01-mixed-reality-3d-app-launcher/hl2-3d-launchers.jpg)

Isn't that much nicer?  [Grab the gist](https://gist.github.com/xwipeoutx/913449ad97a719e434f2803d8476183f) and use it in Unity straight away, or read on for usage.

<!--break-->

## Status quo

To add a 3D app launcher to your Unity normally requires a few steps.

1. Create your 3d model, export as GLB
2. Build your UWP project
3. Copy it into the `Assets/` folder
4. Update your project file and Package.appxmanifest to reference your new model.

I didn't think this was very repeatable - especially since the guidance is to _not check in your build folder_ (good advice, follow it).

## Usage

I've automated the steps above into a little script, so now it's simply

1. Copy the `MixedReality3dAppLauncher.cs` script into an `Editor/` folder of your scripts
2. Save your app launcher to `Assets/app-icon.glb`
3. Build your project

That's it! Your app launcher will be included in the project, appxmanifest, and copied across when you build.  Or you can manually patch it by going to `UWP Tools/Patch manifest and project`.

<script src="https://gist.github.com/xwipeoutx/913449ad97a719e434f2803d8476183f.js"></script>

Enjoy!