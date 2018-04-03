---
layout: post
title: "The 'Battle Ground Copyright' Rewrite Series"
date: 2017-10-03
---

This is a series of articles about the rewriting of _Battle Ground Copyright_: a two player, 2d vector tank combat game that I originally wrote in QBASIC. This is the main index for those articles, where you will find the table of contents below.

It is called "Battle Ground Copyright" because it shamelessly rips off several ideas from multiple games:
- The _Battle Force Tank_ from Recoil (as the tank)
- The _Plasma Battery_ from Total Annihilation (as a weapon)
- The [_Auto-Cannon_](http://kknd-the-krush-kill-n-destroy.wikia.com/wiki/Autocannon_Tank) from Krush, Kill n Destroy (as a weapon)
- The _GDI Ion Cannon_ from Command n Conquer (as a weapon)
- The [_Suprise Crate_](http://cnc.wikia.com/wiki/Crate_%28Red_Alert_1%29) from Command n Conquer
- Overall gameplay based on Liero ([You *should* know about this](http://www.liero.be/))

Here is a screenshot:

![Battle Ground Copyright, a 2d vector tank game that poorly approximates Liero while ripping off several ideas from other games](/assets/bgc/bgc_001.png)

*Above: Battle Ground Copyright, a 2d vector tank game that poorly approximates Liero while ripping off several ideas from other games*

You'll find the history of this game underneath the Table of Contents if you want to know more.

## Table of Contents

- [Choosing a Graphics Library](/blog/2017/11/28/bgc-graphics-library)
- [Reverse Engineering the QBASIC DRAW Command](/blog/2017/12/09/bgc-qbasic-draw-command)
- [Parsing QBASIC DRAW commands with FParsec](/blog/2017/12/19/bgc-parsing-qbasic-draw-commands-fparsec)
- [Parsing QBASIC DRAW commands with FParsec (Round II)](/blog/2108/04/03/bgc-parsing-qbasic-draw-commands-fparsec-part2)


## History

I recall Battle Ground Copyright started as a high school project for our Software Engineering class, to be shown at a sort of highschool open day. This game was basically the culmination of several years of stumbling in the dark learning QBASIC on my own; for a while the only source I had was inbuilt Help system (which by the way was amazing). Eventually I picked up a "_Beginning Programming for Dummies_" book which gave me a better understanding of programming, but it wasn't until I wrote a basic text editor that I really understood how arrays worked. You can laugh at my terrible beginnings with [this article on my programming history](/blog/2017/10/23/my-programming-history).

Once I got better with the concept of programming, the last piece of the puzzle was discovered at the end of the year 2000. I had an old Toshiba 386 laptop that happened to include QB version 4, which included a bunch programs someone else had written. One of these programs was called CUBE.BAS:

![A rotating RGB cube](/assets/learning/cube2.PNG)

I had run it before, but did not pay much attention to it. Silly me, as this single line pretty much made everything possible:

![Rotating RGB cube source code](/assets/learning/cube1.PNG)

From this I discovered that the QBASIC DRAW command accepted a command called "Turn Angle" (TA), which when applied to a list of draw commands, would rotate everything about the origin. I quickly confirmed this with a mock program called TANK2.BAS:

![Mock program to demonstrate TA command](/assets/learning/tank2a.PNG)

I remember being so happy at the time. I finally had the basis to make a tank game.

I created another mock program called "2DEXP1.BAS", which incorporated animation and movement about an origin -- that is, everything moved around the tank.

![A series of "RADAR dishes" all with different turning speeds](/assets/learning/2dexp1a.PNG)

*Above: This code was revolutionary: using arrays (somewhat) effectively AND rotated vector graphics. Also that tank looks remakably similar to the CORE Reaper from Total Annihilation...*

From this program, Battle Ground Copyright was finally born.
