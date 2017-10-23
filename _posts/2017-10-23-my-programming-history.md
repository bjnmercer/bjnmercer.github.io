---
layout: post
title: My Programming History
date: 2017-10-23
---

I started out programming in QBASIC for DOS in Highschool. And according to [Edsger Dijkstra](https://en.wikiquote.org/wiki/Edsger_W._Dijkstra#How_do_we_tell_truths_that_might_hurt.3F_.281975.29), apparently this means I am "mentally mutilated beyond hope of regeneration". After reviewing some of my early code, I can see why he would come to this conclusion.

But it's OK! With enough practice, you can eventually get better! The fact that I can hold down a programming job -- and my co-workers do not want to slap me up the side of the head -- is testament to that. ;)

So, this article is about how I got started in programming. I will talk about how I got into game programming in another article, but I can offer a small bit of history: *Battle Ground Copyright* started as a high school project for our Software Engineering class, to be shown at a sort of highschool open day. This game was basically the culmination of several years of stumbling in the dark learning QBASIC on my own; for a while the only source I had was inbuilt Help system (which by the way was amazing). Eventually I picked up a "_Beginning Programming for Dummies_" book which gave me a better understanding of programming, but it wasn't until I wrote a basic text editor that I really understood how arrays worked.

If you think that's bad, I recall being completely amazed by one of my friends who showed me a program they wrote, which was able to take my name as input and display it as output. Literally:

```basic
INPUT "Enter your name", name$
PRINT "Hello " + name$ + ", you are an idiot!"
```

I mean, I _knew about variables_ but had no idea you could do that!

## The Beginning

The story behind how I got started in programming is perhaps even stranger. I started soldering kits from Jaycar and eventually I came across an 8-relay control card which could be controlled via Parallel Port. This relay card was going to be used to control an air-capable "All Terrain Vehicle", essentially a station-wagon with wings and 2 plasma(?) jet engines. Yes, I had great imagination (and ambition) as a kid.

The demo program, included in the printed manual, used QBASIC to send the commands via Parallel Port. This meant I had to learn how to program.

<s>I recall doing something like this</s> I actually found the very first "program", apparently on the 7th of September, 1999:

![My very first program](/assets/learning/learning1.png)

*Above: hilariously named INTERFAC.E due to MS-DOS 8.3 file format. No wonder I never found this file again.*

I was baffled by this. Initially I just assumed that like Word Perfect (For DOS), I just write text into the program and the text appears on the screen; not realising I needed to use PRINT in order to put text on the screen.

I cannot recall exactly how I got past that stumbling block. But I do still have the original BAS files from my early development in 1999. Sorted in order of modification date, some of the very first files consist of a bunch of code copied directly from the brilliant QBASIC Help Index. So I probably eventually just tried things until I got the result I wanted.

There was also a heap of random text files with the contents: "This is saved to the file." After some confusion, I realised it was this example:

![File IO example](/assets/learning/learning2.png)

And I obviously had no idea what was going on...

Interestingly, I had learned how to do graphics first, with this circuit diagram:

![Circuit Drawing](/assets/learning/learning3.png)

This was before I learned how to put text on the screen with PROGRAM.BAS. It seems to mimic the QBASIC startup dialog.

![Putting text on the screen](/assets/learning/learning4.png)

About 3 months after the very first program (with lots of stuffing around), I had come up with a program called CONTROL.BAS, which was meant to be the flight control software. Recall that I was learning QBASIC in order to build a program that would control the 8-relay card from Jaycar Electronics.

![ATV Controller Program: startup screen](/assets/learning/control1.png)

For some reason I felt it necessary to emphasise that this software was for _FLIGHT CONTROL --ONLY--_; like as if someone might try to use it to control a Nuclear Reactor?

Here's some more screenshots from that program: 

![ATV Controller Program: intro screen](/assets/learning/control2.png)

I do like how I've stated the environment used to build this program.

![ATV Controller Program: main menu](/assets/learning/control3.png)

I liked green on dark green.

![ATV Controller Program: engine setup screen](/assets/learning/control4.png)

![ATV Controller Program: engine setup screen](/assets/learning/control5.png)

This was supposed to be a scanner, just like Microsoft ScanDisk.

Anyway, so I've blathered on for a fair bit and the original point may have been lost. To refresh your memory, the above CONTROL program was created to control a relay card, which was going to control a flight capable vehicle, powered by some kind of turbine engines. And this is why I learned how to program. _Yes_.

As time went on I realised I wanted to make games (as it was also probably becoming apparent that building a flight capable car was going to be impossible). But considering I didn't have a handle on arrays, this was going to be rather tough!

Around the year 2000 I picked up a book on programming in QBASIC, which had a bunch of simple examples (mostly calculations). Also at some point I had downloaded a game called Super Galactic Wars ([source code here](http://nordman.tripod.com/galactic.bas)):

![Galactic](/assets/learning/galactic.png)

I remember being confounded by the code for generating the scrolling star pattern. I just could not fathom it.

Attempts at games quickly went nowhere.

![A fantasic example of me not quite getting arrays](/assets/learning/learning7.png)

*Above: A fantasic example of me not quite getting arrays*

You should know that for variables Sdat$, A1$, A2$ and A3$, I duplicated the loops, sound generation and the effect of writing the text to the screen one character at a time over and over again. I seemed to be quite averse to creating procedures and functions.

The exact date I finally figured out arrays is sketchy. Unfortunately I did not make a habit of putting the creation date in a comment and all I have is the modified date of each file. But if the modified date is to be believed, the first time I used arrays was about September 2000.

It's worth mentioning that I knew arrays existed, I just did not get the point of arrays. Why write

```basic
  A$(1) = "Foo"
  A$(2) = "Bar"
```

When I could just write

```basic
  A1$ = "Foo"
  A2$ = "Bar"
```

It was shorter! Using arrays as just variables, I can see why I thought like this. What I did not understand was the abstraction that arrays could provide in the form of _iteration_, and being able to perform generic operations on a collection of values.

For whatever reason, arrays just did not click until I wrote my first text editor.

## Enter 'FileSAVER'

![Startup screen for FileSAVER](/assets/learning/wordeng1.PNG)

*Above: Nice little "Y2K bug" humour for the creation date*

My memory would have me believe that I truely did not understand the point of arrays until I wrote my first text editor. I think that is still true, given the "RTS Game" I created in September 2000 did not go anywhere, and a cursory examination of the code suggests I really did not know what I was doing.

I note that I had downloaded a game called STOCK00.BAS, a Stock Exchange Simulator. I think it was this game that gave me an idea of how arrays work.

What I do remember was creating the screen and menu system first. This is confirmed by the fact I had created seperate "mock" programs that focused on the screen, open and "save as" dialogs, etc.

The menu system was special: I put each menu item into an array.

![BASIC source for a rudimentary drop-down menu system](/assets/learning/menu1.png)

*Above: Probably the very first instance of me understanding the point of arrays*

This allowed me to easily do the menu highlight and move the selection up or down using the arrow keys.

![Output from the test menu code in the previous image](/assets/learning/menu2.png)

*Above: note how I start the array at 2 to coincide with the row position on the screen*

Such code _not_ using arrays would require a massive case statement and a lot of code duplication.

In FileSAVER, I used global arrays to define the menus like so:

![Arrays that defined the menues](/assets/learning/menu5.png)

Now of course, I didn't yet understand that _arrays were generic_ themselves, so I ended up _duplicating the code for every blasted menu_. Yes. To my early mind, an array named file$() was not the same as an array called options$(). But, at least this was progress.

![Code too horrible to comment on](/assets/learning/menu3.png) ![Code too horrible to comment on](/assets/learning/menu4.png)

*Above: sigh*

I also used an array to hold the contents of the loaded file. Here is where I got somewhat clever. The code was able to simulate scrolling just by incrementing the start index (i.e. the scroll offset) to iterate through the text array. The cursor position corresponded directly to the position in the array, plus the scroll offset.

Despite the terrible code, FileSAVER was reasonably advanced. It supported word wrap, and in later versions, was able to automatically wrap words to the next line if you inserted text.

Here's a quick tour of this program:

![Cool splash screen](/assets/learning/wordeng2.PNG)

*Above: Cool splash screen :)*

You can probably guess which program I modelled the interface off.

![A clear ripoff of QBASIC](/assets/learning/wordeng3.PNG)

Nice menu system. Later versions would even appear to cast the text underneath in shadow.

![A clear ripoff of QBASIC](/assets/learning/wordeng4.PNG)

The open dialog didn't actually do anything. You had to know the exact path of the file you wanted to edit.

![A clear ripoff of QBASIC](/assets/learning/wordeng5.PNG)

And of course, the obligatory _text editor loading it's own source code_ screenshot :)

![A clear ripoff of QBASIC](/assets/learning/wordeng6.PNG)

Of course, trying to load too big a text file into memory results in a runtime error:

![A clear ripoff of QBASIC](/assets/learning/wordeng7.PNG)

Most hilarious are my passive aggressive comments to my highschool friends

![A clear ripoff of QBASIC](/assets/learning/wordeng9.PNG)

But most of all, my ability to write more advanced programs -- such as computer games -- came very soon after.

## General idiocy

Perhaps the worst mistake I ever committed was when I **printed out the source code** of a game called _Planet Wars_ ([source code here](http://www.o-bizz.de/qbdown/qbcom/files/plntwars.bas)), and proceeded to type it in. Do you know why? Because I didn't realise a .BAS file was just a .TXT file. It must have taken me a week to type the code in! The date modified attribute shows I last saved the file at 4am, which sounds about right. I remember being up _very_ late when I finally finished the game.

Planet Wars was actually a pretty cool artillery style game, except with star ships and planets with simulated gravity on the trajectory of shots.

![Planet Wars](/assets/learning/planetwars4.PNG)

## Final thoughts

I'm going to stop here, as I think that the history of my game development is perhaps seperate to my journey to understanding programming. Also, this article is quite long.

With that, I leave you with this: while looking through my old computer I actually found the source code for controlling the 8 relay card -- that I never ended up using:

```basic
10 CLS : KEY OFF: DEFINT A-C    ' define A, B, C as intergers
20 PORTA = &H378: REM &H378 is for LPT1. use &H278 for LPT2
30 PORTB = PORTA + 1: PORTC = PORTB + 1 'define port addresses
40 OUT PORTA, 0: OUT PORTC, 10'turn all relays off, take CO high
50 OUT PORTC, 11: OUT PORTC, 10 'select card one, CO low then high
60 FOR A = 0 TO 7 'relays are coded 1, 2, 4, 8, 16, 32, 64, 128
70 OUT PORTA, 2 ^ A 'select relay 1 to 8 in turn
80 OUT PORTC, 11: OUT PORTC, 10 'select card one, strobe high then low
90 B$ = RIGHT$(TIME$, 2): WHILE RIGHT$(TIME$, 2) = B$: WEND'wait one second
100 NEXT A

110 OUT PORTA, 0: OUT PORTC, 11'turn all relays off
120 OUT PORTB, 120: OUT PORTC, 5'take input lines high
130 FOR A = 1 TO 200: NEXT 'delay for IC4c increase value if necessary
140 LIN = 0: B = INP(PORTB): C = INP(PORTC) 'read port values
150 IF (B AND 128) THEN BIN = B - 135 ELSE BIN = B + 121 'complement bit 8
160 CIN = C AND 14 ' mask high bits and CO
170 IF (C AND 2) = 0 THEN CIN = CIN + 2 ELSE CIN = CIN - 2 'complement bit 2
180 IF (C AND 8) = 0 THEN CIN = CIN + 8 ELSE CIN = CIN - 8 'complement bit 8
190 CIN = INT(CIN / 2): TIN = 255 - (BIN + CIN)
200 FOR A = 0 TO 7: IF TIN / 2 ^ A = 1 THEN LIN = A + 1'find low line
210 NEXT
220 LOCATE 24, 20: PRINT "line"; LIN; 'print it
230 GOTO 140 'loop
```

There was absolutely no way I was ever going to understand that, at the level of knowledge I had in 1999.

## End of article

Usually marks the end of an article, the "End of article" marker indicates that this is the end of this article.