---
layout: post
title: "Parsing QBASIC DRAW commands with FParsec (Round II)"
date: 2018-04-03
---

>(Pro-tip: this article is part of the [BGC Rewrite Series](/blog/2017/10/03/bgc-rewrite-series))

In the previous article, I discussed [using FParsec to directly parse the QBASIC Draw Command string](/blog/2017/12/19/bgc-parsing-qbasic-draw-commands-fparsec). In this article, I build upon that parser to handle line colours.

This should hopefully be a quick article. If you have no idea what is going on, I encourage you to read the [previous article](/blog/2017/12/19/bgc-parsing-qbasic-draw-commands-fparsec). Otherwise, sit back, relax, and be completely baffled.

## Where we left off

Ok ok, I'll throw you a bone.

A QBASIC *Draw Command* is a series of small instructions that, for the most part, resemble a cut-down version of the LOGO programming language: the cursor position is moved based on the *Draw Move Type*, be that Up, Down, Left, Right, Up and Left, etc. with a number specifying how far to move.

Here's the QBASIC Draw Command text for the a "health box" in the BGC game:

```qbasic
healthbox$ = "C2 BU5 BL5 R10 D10 L10 U10 BF2 BR2 C15 R2 D2 R2 D2 L2 D2 L2 U2 L2 U2 R2 U2"
```

![BGC health box](/assets/bgc/draw4.PNG)

As stated in the last article, the parser was basically complete except for one caviat: it could not handle *C commands*. C commands are a special attribute that effect the colour of all proceeding lines created. The numbers 0 to 15 correspond to a colour, with 0 being black and 15 being white. I can't say that any thought was given to the order of colours. Here is a demo showing the number and colour:

![BGC health box](/assets/bgc/qbasic-colours.PNG)

So, for the above health box, the effect of the C attribute is shown

C2 <span style="background-color: black;color:green">BU5 BL5 R10 D10 L10 U10 BF2 BR2</span> C15 <span style="background-color: black;color:white">R2 D2 R2 D2 L2 D2 L2 U2 L2 U2 R2 U2</span>

The colour will remain in effect until it is changed.

How to handle this special command will be the focus of this article.

## FParsec User State

There are a few approaches to solving this problem:
1. Make C commands part of the parsed stream, i.e. just another Draw Command
2. Make use of FParsec's user state to keep track of the current colour

The second option is what I tried. It has the advantage of not requiring any additional domain complexity; no extra case to handle it, and no additional post-processing (I'll get to that in a minute). I should still just get a list of DrawCommands, with the colour already assigned. I just need to add a new field to the DrawCommand structure, called color:

```fsharp
type DrawCommand = {
     moveType:DrawMoveType
     direction:DrawMove          
     color:int
}
```

The first option is definitely easier from a parser implementation perspective -- to this point I have never had the need for User State. On top of the DrawCommand struct, I'd need a Discriminated Union to be able to handle the two cases: Colour commands and Move Commands.

Something like this:

```fsharp

type MoveCommand = {
     moveType:DrawMoveType
     direction:DrawMove     
}

type DrawCommand =
| Color of int
| Move of MoveCommand

```

`DrawCommand` becomes a Discriminated Union, and the original `DrawCommand` record structure becomes `MoveCommand`.

A common Discriminated Union is needed to allow the parser for C commands and the parser for everything else to match, i.e.

```fsharp
let pDrawColor:Parser<DrawCommand,unit> =
    pstring "C" >>. pint32 |>> (fun c -> Color(c))

let pDrawCommand:Parser<DrawCommand,unit> = pipe2 pMoveType pDrawMove (fun mt dm ->         
    Move({ moveType = mt; direction = dm })
    )

```

And then you get a list of DrawCommands as usual. The code for translating DrawCommands into a list of X,Y coordinates will need to keep track of the current colour. Something like

```fsharp
  let mutable currentColour = 0;
  
  let mapCommand command =
    match command with
    | Color(c) -> 
      currentColour <- c
      None
    | Move (m) -> Some { Coordinates = (nextCoord m); color = currentColour }

  commands 
  |> Seq.choose (mapCommand)
  |> Seq.map //do something with coordinates

```

But to me, I already had the structure I wanted, I just needed the color

```fsharp
type DrawCommand = {
     moveType:DrawMoveType
     direction:DrawMove
     color: int
}
```

So I went with the user state option... And proceeded to tear my hair out! [There is an example](http://www.quanttec.com/fparsec/users-guide/parsing-with-user-state.html), but by golly it took a number of goes to understand it. But I got there in the end and I hope this article will give you another take on the User State part of FParsec.

## Implementing the new Parser

Now first up, I cannot say if this was the best approach. Maybe the other approach using the Discriminated Union is better. More idiomatically functional?

I will also admit I was curious to how User State worked.

So here we go.

The first thing I needed was the User State. I defined a record as recommended by [the user guide](http://www.quanttec.com/fparsec/users-guide/parsing-with-user-state.html).

```fsharp
type DrawState = {
    currentColor: int
}
```
*Above: I am not imaginative when it comes to naming.*

The reason for using a Record instead of an object with mutable state, is that you are always creating a new copy of a record when you update user state -- therefore if your parser has to backtrack, it is able to easily restore the previous user state. This is akin to [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) by the way.

The next change was that the existing parser functions I had defined needed to now specify the User State type.

```fsharp
pDrawMove:Parser<DrawMove,DrawState>
pMoveType:Parser<DrawMoveType,DrawState>
pDrawCommand:Parser<DrawCommand,DrawState>
pDrawCommands:Parser<DrawCommand list, DrawState>
```

Kind of annoying that I had to go and specify the `DrawState` in every function, but there you go. Probably a hint I should be using type inference.

Now, I needed some way to set and get the user state.

FParsec provides functions if you do not want to work with the lower level CharStream API directly. In my case I wanted
* `getUserState`
* `setUserState`

There is also `updateUserState` although that needs a function that returns a new copy of the user state. I'll explain the difference in a moment.

```fsharp
let setDrawColor (c:int) =    
    setUserState { currentColor = c }
    
let pDrawColor:Parser<unit, DrawState> =    
    (pstring "C" >>. pint32) >>= setDrawColor
```

As mentioned earlier, I could have used `updateUserState`, this is what it would look like

```fsharp
let setDrawColor (c:int) =   
    updateUserState (fun us -> { us with currentColor = c })
```

Although I can't say why you would use one over the other.

Now I could have done without the `setDrawColor` function, but in this case it just a little neater than using an anonymous function.

So now was the time to test. And it is at this point that I should mention that because I am using User State, I can no longer use the convenience function `run` for my parser. Instead I now have to use `runParserOnString`, because we have to specify the initial User State.

```
> runParserOnString pDrawColor { currentColor = 0} "Test" "C1";;
val it : ParserResult<unit,DrawState> = Success: ()
```

Hmm. So how do I get that at that state?

```
> getUserState;;
val it : Parser<'a,'a> = <fun:getUserState@111>
```

OK, not much help.

```
run getUserState "blah";;
val it : ParserResult<unit,unit> = Success: ()
```

At this point I was a little baffled, but I carried on.

The next step was to modify the function for parsing a `DrawCommand`. Looking at this example, I attempted to do something like 

```fsharp
let pDrawCommand:Parser<DrawCommand,DrawState> = pipe2 pMoveType pDrawMove (fun mt dm ->     
    let us = getUserState
    { moveType = mt; direction = dm; color = us.currentColor }
    )
```

Except that didn't work. As before, I end up with `us` being a `Parser<'a,'a>`, which is of zero help.

[Another person had the same question](http://fpish.net/topic/None/60071), and the answer given unfortunately still didn't show how to use the `getUserState` function, instead using the lower level `CharStream<'u>` API directly.

I scratched my head until I remembered the `pipe3` function. The `pipe*` functions execute parsers in sequence, and return the results of each parser to a function supplied, for the purpose of calculation.

The `pDrawCommand` parser used a pipe2 function, so.....

```fsharp
let pDrawCommand:Parser<DrawCommand,DrawState> = pipe3 pMoveType pDrawMove getUserState (fun mt dm us ->         
    { moveType = mt; direction = dm; color = us.currentColor }
    )
```

The parameter `us` was indeed a `DrawState`. This works because getUserState is also a "parser" which happens to return the value "parsed", in this case, the User State.

Again, I can't say this is the most elegant solution, but it works. With that particular issue out of the way, the last step was to combine the `pDrawColor` and `pDrawCommand` functions.

```fsharp
let pDrawCommands:Parser<DrawCommand list, DrawState> = sepBy (pDrawColor <|> pDrawCommand) (pstring " ")
```

In my mind, it would try `pDrawColor` first, and if that failed, then try `pDrawCommand`. Instead:

```
error FS0001: Type mismatch. Expecting a
    Parser<DrawCommand,DrawState>    
but given a
    Parser<unit,DrawState>    
The type 'DrawCommand' does not match the type 'unit'
```

Of course. All functions combined with `<|>` have to be the same type.

This particular issue vexxed me for a few weeks.

At first I gave in and decided that I'd just make the `pDrawColor` function return an invalid value which I would just ignore (a worse strategy than the Discriminated Union version, I will admit)

```fsharp
let setDrawColor (c:int) =    
    setUserState { currentColor = c }
    { moveType = P; direction = D(0); color = 0 }
    
let pDrawColor:Parser<DrawCommand, DrawState> =
    (pstring "C" >>. pint32) |>> setDrawColor
    
```

Instead I was greated with a warning:

```
warning FS0193: This expression is a function value, i.e. is missing arguments. Its type is Parser<unit,DrawState>.
```

I was getting a warning for `setUserState { currentColor = c }` which I did not understand, but I tried to test the function anyway


```
> runParserOnString pDrawColor { currentColor = 0 } "Test" "C2";;
val it : ParserResult<DrawCommand,DrawState> =
  Success: {moveType = P;
 direction = D 0;
 color = 0;}
> runParserOnString pDrawCommand { currentColor = 0 } "Test" "D1";;
val it : ParserResult<DrawCommand,DrawState> =
  Success: {moveType = P;
 direction = D 1;
 color = 0;}
```

The user state, as I feared, was not getting updated.

Back to the drawing board.

The problem, it seemed is that the setUserState function was returning a parser of type `Parser<unit, DrawState>`, when I needed a `Parser<DrawCommand, DrawState>`. Eventually I gave in and learned about the so called low-level `CharStream` API. After a bit of stumbling around, I came up with this bit of madness:

```fsharp
let pDrawColor':Parser<DrawCommand, DrawState> =    
    pstring "C" >>. (fun stream ->                
        let mutable digits:string = String.Empty
        while (isDigit (stream.Peek())) do
            digits <- digits + (stream.Read() |> string)

        printfn "digits %s" digits
        let color = digits |> int        
        stream.UserState <- { stream.UserState with currentColor = color }
        Reply({ moveType = DrawMoveType.P; direction = DrawMove.D(0); color = color })        
        )
```
*Above: so much 'fun' in F#, according to a friend of mine*

What in the universe is going on here?

We'll start with `fun stream`: this is effectively an anonymous parser being declared on the spot (I could have defined a specific function, but what do I call it?) `stream` is a `CharStream<DrawState>`. I make use of the "low level" functions such as `Peek()` and `Read()` to keep reading a character at a time until there are no more digits.

These digits are converted to an int. I then update the UserState with the following

```fsharp
stream.UserState <- { stream.UserState with currentColor = color }
```

And reply with a useless DrawCommand, which I could filter out later

```fsharp
Reply({ moveType = DrawMoveType.P; direction = DrawMove.D(0); color = color })
```

And this has the intended effect.

```
runParserOnString pDrawCommands { currentColor = 0 } "Test" "C2 D1"
val it : ParserResult<DrawCommand list,DrawState> =
  Success: [{moveType = P;
  direction = D 0;
  color = 2;}; {moveType = P;
                direction = D 1;
                color = 2;};]
```

Now I could have left it at that... But that extra filtering step needed to remove the "false `DrawCommand`" was bothering me. I researched how to make a parser that would skip over the parsed characters instead of producing a return value, but I could not figure it out.

And then, I realised I had forgotten a few fundamental combiner functions provided by FParsec: `.>>` and `>>.`

I can't remember how I came to that realisation, but I tried the following:

```fsharp
let pDrawCommands:Parser<DrawCommand list, DrawState> = sepBy ((pDrawColor >>. pstring " ") >>. pDrawCommand) (pstring " ")
```

```
> runParserOnString pDrawCommands { currentColor = 0 } "Test" "C1 D1";;
digits 1
val it : ParserResult<DrawCommand list,DrawState> =
  Success: [{moveType = P;
  direction = D 1;
  color = 1;}]
```

Exactly what I was looking for! The false `DrawCommand` created by the C command was not appearing in the list. If you do not recall, `p1 >>. p2` will execute each parsers p1 and p2 in sequence, but discard the result of p1, and `p1 .>> p2` will dicard the output of p2.

I hit one more snag when I tried the ultimate example, the health box:

```
val it : ParserResult<DrawCommand list,DrawState> =
  Failure:
Error in Test: Ln: 1 Col: 8
C2 BU5 BL5 R10 D10 L10 U10 BF2 BR2 C15 R2 D2 R2 D2 L2 D2 L2 U2 L2 U2 R2 U2
       ^
Expecting: 'C'
```

Ah right, it now always expected a colour command to proceed a move command. I remembered there was an `optional` function, so tried that out:

```fsharp
let pDrawCommands:Parser<DrawCommand list, DrawState> = sepBy ((optional (pDrawColor .>> pstring " ")) >>. pDrawCommand) (pstring " ")
```

```
> runParserOnString pDrawCommands { currentColor = 0 } "Test" "C2 BU5 BL5 R10 D10 L10 U10 BF2 BR2 C15 R2 D2 R2 D2 L2 D2 L2 U2 L2 U2 R2 U2";;
digits 2
digits 15
val it : ParserResult<DrawCommand list,DrawState> =
  Success: [{moveType = B;
  direction = U 5;
  color = 2;}; {moveType = B;
                direction = L 5;
                color = 2;}; {moveType = P;
                              direction = R 10;
                              color = 2;}; {moveType = P;
                                            direction = D 10;
                                            color = 2;}; {moveType = P;
                                                          direction = L 10;
                                                          color = 2;};
  ... you get the idea ...
```

This makes sense because a colour command can optionally preceed every single move command.

I could now simplify the `pDrawColor` parser like so

```fsharp
let setDrawColor (c:int) =   
    setUserState { currentColor = c }

let pDrawColor:Parser<unit, DrawState> =    
    (pstring "C" >>. pint32) >>= setDrawColor
```

Overall, I'm pretty happy with this solution.

I was so excited I finally had this working, that I quickly implemented coloured lines in my test line renderer.

...Oh right, I have not even talked about the line renderer!

Anyway, here is output. I have multiplied the size by 10 to make it more visible:

![Output from the line renderer after interpreting the QBASIC DRAW string directly](/assets/bgc/healthbox_sdl.PNG)

In the next article, I discuss the line renderer. Thanks for reading!