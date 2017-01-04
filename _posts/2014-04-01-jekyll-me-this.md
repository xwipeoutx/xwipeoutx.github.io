---
layout: post
title: Jekyll Me This
categories:
tags:
date: 2014-04-01 20:30:00 +1000
---
So, I've moved the blog over to [Jekyll](http://jekyllrb.com/), a static site generator that's pretty well suited to doing blogs,
if you're a tech person, like myself.  No, this is not an April fool's joke!

<!--break-->

Moving over was a little arduous, not least of all because all the Jekyll tooling (including Ruby) really targets Linux or Mac users more than Windows users.

So, if you're thinking of doing the same, here's a few tips:

1.  Use the guide by juthilo on GitHub on how to [run Jekyll on Windows](https://github.com/juthilo/run-jekyll-on-windows/)
2.  Pay attention to the version numbers in the article above, as they are written for a reason.
When I set mine up, it was Jekyll v1.4.2, but it looks like he's now recommending 1.5.1.
3.  Consider using [Javascript code prettifier](http://google-code-prettify.googlecode.com/svn/trunk/README.html) instead of Pygments
if you don't want to install Python.
4.  GitHub pages does NOT support most plugins, only the [ones found in the master repository](https://github.com/jekyll/jekyll/issues/325).  So either
avoid using them, or generate the HTML locally and commit that
5.  While there are a tonne of themes at [Jekyll Themes](http://jekyllthemes.org/), it's very hard to find a good one - with no search or rating system,
it's really a list of the latest submitted ones.  A lot of them have hardcoded site names and categories.  Avoid the hassle and style it yourself.

I have [a GitHub repository](https://github.com/xwipeoutx/xwipeoutx.github.io) for this blog if you want a fairly simple setup or template to base it off.  I'm not
using [Jekyll Bootstrap](http://jekyllbootstrap.com/), so it shows how easy it is to make a very functional blog with vanilla Jekyll.

I still don't know what I want to do with comments, I may simply avoid using them, as I'd have to hand them over
to a 3rd party (like [Disqus](https://disqus.com/)), something I'd rather not do.

If you have any helpful comments or find any problems, let me know by [raising an issue](https://github.com/xwipeoutx/xwipeoutx.github.io/issues/new)
or submitting a pull request.