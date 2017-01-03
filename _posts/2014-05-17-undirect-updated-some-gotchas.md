---
layout: post
title: Undirect updated to 1.1.1, and some "gotchas"
categories:
- Programming
tags:
- undirect
- chrome
- extensions
- javascript
date: 2014-05-18 00:00:00 +1000
---
I've updated [Undirect](https://chrome.google.com/webstore/detail/dohbiijnjeiejifbgfdhfknogknkglio),
a Chrome plugin that prevents annoying redirects on the google search page, because it had stopped working.
I ran into a few surprises along the way, and some old frustrations cropped up again, here's a rundown to help the
next person who tries.

<!--break-->
Chrome store feedback
---------------------
Since I'd released this in 2011, with a small update in 2012 to add RegEx to the pattern, I'd basically stopped checking the store's feedback.
I couldn't find a way to get notifications of comments (apparently [I'm not the only one](https://productforums.google.com/forum/#!topic/chrome/JidW4GlebXE)) -
and checking every few months is something that I probably could have done, but it never crossed my mind.

Now I wish I had, because apparently the plugin hasn't been working since about August 2013 - and I hadn't even noticed.
Which is especially concerning, because I use the plugin on all my machines, and so do a lot of friends of mine.  Being able to
get notifications of reviews would have served me well here, since the average Joe won't exactly raise an issue on GitHub

Manifest JSON
-------------
The `manifest.json` file has been updated to v2, and v1 can no longer be updated on the Web Store.  Not a big deal, but there were
a few new things about the configuration file which were non-obvious.

- `default_locale` is noted as "recommended" field, but without also having a `_locales`
   directory with the localisation, you get a confusing error when trying to load it.
- The [documentation page](https://developer.chrome.com/extensions/manifest#overview) for it looks good for a sample, but it took me
  a while to realise that the field names are clickable.  If you're looking for documentation, click those links!
- The sample uses comments to mark off sections - but keep in mind this isn't valid JSON! You'll need to take them out before uploading to the store

Isolated world - why it broke
-----------------------------
The reason for the extension breaking was a "security" feature they've added to the extension.  I put security in quotes, because it's easy to circumvent.

Content scripts now run in an [isolated world](https://developer.chrome.com/extensions/content_scripts#execution-environment) - which means that while it has access
to the page and the DOM and all the jazz, it's a different view than the page.  Any scripts or variables that are added by the page are
inaccessible from the script - in my case, the `onmousedown` attribute of the link tags didn't exist from the content script's perspective.

I thought all hope was lost, then came across a [neat little workaround](http://stackoverflow.com/a/9777536/185422), which is basically to add a `<script>`
tag to the head element, causing it to execute in page context.  Circumvented!

In my case, I wrote the function I wanted to execute as usually, then added the script to the
page by simple calling `.toString()` on the function.  Simple, and I get the full power of my IDE while I'm at it! Here's an example

{% highlight js %}
var fnToRunOnPage = function() {
  // code goes here
};

var fnContents = '(' + fnToRunOnPage.toString() + ')();';

var scriptTag = document.createElement('script');
scriptTag.textContent = executeFnScript;
(document.head || document.documentElement).appendChild(scriptTag);
scriptTag.parentNode.removeChild(scriptTag);
{% endhighlight %}

It's the same thing that jQuery does when you add an element with a script tag in it.  Works a treat!

The new solution
----------------
[Previously](https://github.com/xwipeoutx/undirect/blob/d6b613afa1201fa4fbd378c017cdc73dbfc74494/undirect.js), my plugin would search all the anchor tags, look for ones that had an `onmousedown` script with `return rwt` in it, and remove the value.
Every time a DOM element was added or removed, I would run it again.

[That's updated now](https://github.com/xwipeoutx/undirect/blob/544aefe555181fa823804a25fa30f6e0dfa2515e/undirect.js) to simply overwrite the `rwt` function on `window` to one that does nothing.
I (perhaps unnecessarily) future-proofed it against overwrites by making it a property with `writeable: false` as well.
Futile, probably - I'm sure they could get around that simple enough, but I have nothing to lose by doing that.