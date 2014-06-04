---
layout: post
title: Making WebStorm Friendlier
categories:
- Programming
tags:
- jetbrains
- webstorm
- configuration
date: 2014-06-04 22:00:00 +1000
---
There's a few settings I recently tweaked in [WebStorm](http://www.jetbrains.com/webstorm/) that vastly improved my development experience.  
Here's a quick rundown.
 
Ensure "Show Reformat Code" is disabled
---------------------------------------
![Show "Reformat Code" dialog](/images/2014-06-04-reformat-code.png)

Make sure these guys are off to get the popups out of your way.  I also remap the commands to `Ctrl+K, D` and `Ctrl+R, S` 
to match my Visual Studio setup.

Tweak code completion popups
----------------------------
This is the setup I use:

![Code Completion](/images/2014-06-04-code-completion.png)

The most important one is the Parameter Info popup - I really like having the method signatures there all the time,
so making this nice and small makes it much more useful.  I'd recommend trying "`Insert selected variant by typing dot, space, etc`"
as well, for more natural completions.  Can be annoying, though 

Key Bindings
------------
The default key bindings are a little all over the shop.  Here's some handy ones:

- `Ctrl+T`: Navigate to symbol
- `Ctrl+Shift+T`: Navigate to file
- `F2` and `Ctrl+R, R`: Rename (works for files and symbols)
- `Alt+Enter`: Autocomplete
- `Alt+Home`: Super types hierarchy
- `Alt+End`: Sub types hierarchy
- `Ctrl+R, S`: Optimise Imports
- `Ctrl+K, D`: Reformat Document

Some of these are muscle memory from my Visual Studio settings at work, but be sure to make them something you use, 
because they are all VERY useful commands. 

Plugins
-------
The best plugin I've seen is [Key Promoter](http://plugins.jetbrains.com/plugin/4455?pr=webStorm).  This handy extension shows a popup whenever
you do something inefficiently.  For example, if I go to Settings via the menu, instead of `Alt+F7`, it'll show a blocker for 1s that shows the shortcut
in an impossible to miss way.  Recently, it noticed that I use "Close all documents" a lot, and nagged me to make a shortcut for it. I can now use 
`Ctrl+Shift+W` for that purpose, and it will nag me if I don't.