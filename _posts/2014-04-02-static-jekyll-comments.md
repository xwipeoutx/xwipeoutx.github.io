---
layout: post
title: Static Jekyll Comments
categories:
tags:
date: 2014-04-02 20:45:00 +1000
---
Those of you paying very close attention will have noticed that posts now have a comment section at the bottom.
It's currently reading the comments reading in from specially named YAML files in the _data directory of the site,
which I have to put in manually.

<!--break-->
You can find my comment [HTML](https://github.com/xwipeoutx/xwipeoutx.github.io/blob/master/_includes/comments.html)
and [YAML](https://github.com/xwipeoutx/xwipeoutx.github.io/blob/master/_data) on GitHub, hopefully it's pretty straight self-explanatory.

To do it, I had to get a filename-friendly page identifier from the, which I did by replacing slashes with dashes and capturing it in a comment id.

    {% raw %}{% capture commentid %}comments{{ page.id | replace: '/','-'}}{% endcapture %}{% endraw %}

From here, I read the data from `site.data[commentid]` and display it on the page.  I wanted to allow some formatting in comments, so I pass the content
through the `| markdownify` filter.  The comment contents in the YAML have to be multi line as well, ending up with a YAML file with a bunch of entries like so:

    - author: Steve
      date: 2014-04-02 10:00 +1000
      contents: |
        I've added comment support to the site now! See [my post](/2014/04/static-jekyll-comments) to see how I went about it.

        You'll note that it has *markdown* support too! I'll moderating it though, so don't try any dirty hacks...

The plan now is to make use of the GitHub API to allow comment submission by creating issues.  I think this means that the commenter requires
GitHub account, but that's ok.  I don't think I'll get enough comments to need more automation than that - I will want to moderate them to ensure
there's nothing malicious in the comment anyway, so I don't mind having to copy-paste the markdown from the GitHub issue into a YAML file.

For now, you'll have to create the GitHub issue yourself!