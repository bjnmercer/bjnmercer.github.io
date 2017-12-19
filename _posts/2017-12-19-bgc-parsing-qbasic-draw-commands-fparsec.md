---
layout: post
title: "Parsing QBASIC DRAW commands with FParsec"
date: 2017-12-19
---

>(Pro-tip: this article is part of the [BGC Rewrite Series](/blog/2017/10/03/bgc-rewrite-series))

In the previous article, I discussed [reverse engineering the QBASIC DRAW statement](/blog/2017/12/09/bgc-qbasic-draw-command). In this article, I create a parser to interpret the DRAW commands directly.

There were a few reasons for doing this:
1. While conversion from QBASIC DRAW text version to a list of F# Discriminated Union cases would be fairly simple, I did not fancy manually parsing over 40 drawings; this would be even more work than simply capturing sprites.
2. I discovered that the DrawCommand DU was not able to properly express the domain. DUs can be very concise, but the new structure is a record -- suddenly not so simple, and a heck of a lot more tedius.
3. I liked the technical challenge of learning FParsec.

## FParsec

FParsec is a fantastic library consisting of many simple 'character-by-character' parsers. Each parser does one thing only. You use these simple parsers to build more complex parsers, and so on. Such a system is called a 'parser combinator' library. It is based on Haskell's Parsec library.

A quick demonstration is in order. Let's start with the absolute simplest example -- parsing an int:

```
> run pint32 "12";;
val it : ParserResult<int32,unit> = Success: 12
```

Note that a parser returns a DU consisting of two cases: Success and Failure. Success contains the parsed value, and Failure returns an error message.

For example:

```
> run pint32 "hi";;
val it : ParserResult<int32,unit> =
  Failure:
Error in Ln: 1 Col: 1
hi
^
Expecting: integer number (32-bit, signed)
```

This is perhaps the beauty of FParsec: detailed parser error messages!

OK, that is the absolute bare minimum introduction. For more detailed information, I refer you to the [FParsec User Guide](http://www.quanttec.com/fparsec/users-guide/) and also the [FParsec Tutorial](http://www.quanttec.com/fparsec/tutorial.html).

## New Drawing structure

The first thing I started with was the `DrawCommand` DU structure. I had worked with FParsec before and what I learned from that experience is the FParsec char parsers only ever work with a single character at a time, unless you want to define your own parser and user state. I wanted to avoid the complexity for now, so I changed the design.

`DrawCommand` was changed from a Discriminated Union to a record type:

```fsharp
type DrawCommand = {
     moveType:DrawMoveType
     direction:DrawMove          
}
```

In a way, I was already doing this informally with the `bool * (int * int)` tuple originally returned by the internal `nextCoord` function ([refer to the previous article](/blog/2017/12/09/bgc-qbasic-draw-command)) -- in that the first `bool` element signified if this was a blind move or not. The second `(int * int)` tuple becomes `direction` in the `DrawCommand` record.

I then defined a `DrawMove` DU, essentially the original `DrawCommand` DU but only dealing with direction, and not any of the special modifiers:

```fsharp
type DrawMove =
| M of int * int //M[{+|-}]x%,y%    Moves cursor to point x%,y%. If x% is preceded, by + or -, moves relative to the current point.
| D of int //"D n" draws a line vertically DOWN n pixels.
| E of int //"E n" draws a diagonal / line going UP and RIGHT n pixels each direction.
| F of int //"F n" draws a diagonal \ line going DOWN and RIGHT n pixels each direction.
| G of int //"G n" draws a diagonal / LINE going DOWN and LEFT n pixels each direction.
| H of int //"H n" draws a diagonal \ LINE going UP and LEFT n pixels each direction.
| L of int //"L n" draws a line horizontally LEFT n pixels.
| R of int //"R n" draws a line horizontally RIGHT n pixels.
| U of int //"U n" draws a line vertically UP n pixels.
```

Note that the special modifiers such as Blind, Color etc are removed, and instead become a DrawMoveType:

```fsharp
type DrawMoveType =
| B
| N
| P
```

I _should_ be giving these more meaningful names (as pointed out by a friend), however I am lazy. Also, I can refactor later. The `DrawMoveType.P` case is there for the standard drawing; you can think of it as "Pen" mode.

Finally, I should mention that this still does not handle the `C` colour command, or  `TA` turn angle command. These are special cases that need user state, something I can't handle right now, but might try and tackle in a future article.

## Parsing

The simplest case is parsing a single character. FParsec offers the built in string parser `pstring`.

`pstring` takes a string, and attempts to consume said string, returning Success if it could, or failure if it could not:

```
> run (pstring "Duh") "Duh";;
val it : ParserResult<string,unit> = Success: "Duh"
> run (pstring "Duh") "fail";;
val it : ParserResult<string,unit> =
  Failure:
Error in Ln: 1 Col: 1
fail
^
Expecting: 'Duh'
```

I attempted to define a parser that would parse a single character, followed by parsing an int32.

```fsharp
let pDrawMove (ch:string) = (pstring ch) >>. pint32
```

The `>>.` is an infix operator function which will execute both `(pstring ch)` and `pint32` in order, discarding the value from `(pstring ch)` and returning the result of `pint32`

```
> run (pDrawMove "R") "R8";;
val it : ParserResult<int32,unit> = Success: 8
```

Next I defined a function that builds on `pDrawMove`, which returns the parsed value converted to a `DrawMove` DU

```fsharp
let pDrawMoveR:Parser<DrawMove,unit> = (pDrawMove "R") |>> fun dx -> R(dx)
```

This is accomplished using the infix operator function `|>>`. The purpose of this function is to run the parser on the left, and call a function supplied by you to perform a calculation or conversion -- in this case, an int to `DrawMove.R` DU case, and return it wrapped as a `ParserResult`. I use the anonymous function `fun dx -> R(dx)` to accomplish this.

```
> run pDrawMoveR "R8";;
val it : ParserResult<DrawMove,unit> = Success: R 8
```
Again, you still get a `ParserResult`. Now if you attempt to use this parser to do anything else, you will get an error:

```
> run pDrawMoveR "F8";;
val it : ParserResult<DrawMove,unit> =
  Failure:
Error in Ln: 1 Col: 1
F8
^
Expecting: 'R'
```

... And now I'll skip to the part where I discovered this was an awful idea. At least, it was the most awful in the sense that it would take a lot of extra code to achieve what can be done using the `anyOf` function.

`anyOf` is a parser which will parse a character and return success if that character matches an array of characters supplied as a string. For example:

```
> run (anyOf "DEFGHLRU") "D";;
val it : ParserResult<char,unit> = Success: 'D'

> run (anyOf "DEFGHLRU") "!";;
val it : ParserResult<char,unit> =
  Failure:
Error in Ln: 1 Col: 1
!
^
Expecting: any char in ‘DEFGHLRU’
```

This is handy because if this parser fails, we know we aren't dealing with a DrawMove. But if we are, we can get the extracted character and perform a match expression on it to convert to the appropriate `DrawMove` case.

There's just one problem:

```fsharp
let pDrawMove (ch:string) = (anyOf "DEFGHLRU") |>> fun dm -> 
    match dm with
    | 'D' -> D //how do we get int value?
```

This is where the `pipe2` function comes into play. `pipe2` takes two parsers, and runs each parser to get their parsed values, and "pipes" these two values into a function supplied by you so that you can perform a calculation or conversion. The typical use case is conversion to a DU.

In this case, we take the `anyOf` parser and the `pint32` parser and join them using the `pipe2` function. Note the last catch all case, this is included only as a formality in order to satisfy the F# compiler warning about incomplete pattern matches. That said, if I ever change the parser, I will at least get a failure.

```fsharp
let pDrawMove:Parser<DrawMove,unit> = pipe2 (anyOf "DEFGHLRU") (pint32) (fun dm dx -> 
    dx |>
    match dm with
    | 'D' -> D
    | 'E' -> E
    | 'F' -> F
    | 'G' -> G
    | 'H' -> H
    | 'L' -> L
    | 'R' -> R
    | 'U' -> U
    | _ -> failwith "pDrawMove")
```

One shiny new function, let's give it a run:

```
> run pDrawMove "D8";;
val it : ParserResult<DrawMove,unit> = Success: D 8

> run pDrawMove "F8";;
val it : ParserResult<DrawMove,unit> = Success: F 8

> run pDrawMove "X8";;
val it : ParserResult<DrawMove,unit> =
  Failure:
Error in Ln: 1 Col: 1
X8
^
Expecting: any char in ‘DEFGHLRU’
```

So far so good, but there is one problem with this design. Recall that QBASIC DRAW commands can omit the second integer value, which is assumed to be 1 pixel.

The parser in it's current form expects there to always be an integer present:

```
> run pDrawMove "F";;
val it : ParserResult<DrawMove,unit> =
  Failure:
Error in Ln: 1 Col: 2
F
 ^
Note: The error occurred at the end of the input stream.
Expecting: integer number (32-bit, signed)
```

Luckily, FParsec provides three more inbuilt functions that address this scenario: `attempt`, `preturn` and `<|>` (yes, the latter one is a function!)

We'll examine the effect of `<|>` first. This infix operator function attempts to run the parser on the left, and if that fails, it runs the parser on the right, but *only if the parser on the left fails without consuming any input*.

```
> run (pint32 <|> preturn 1) "123";;
val it : ParserResult<int32,unit> = Success: 123
> run (pint32 <|> preturn 1) "F123";;
val it : ParserResult<int32,unit> = Success: 1
```

If `pint32` is unable to parse the next character, `pint32` fails and thus `<|>` causes the right function to be evaluated -- in this case, `preturn`. `preturn` is a parser that returns a value and does no parsing.

Now recall earlier the emphasis on not consuming any input. The `pint32` function description:

```fsharp
/// Parses an integer in decimal, hexadecimal ("0x" prefix), octal ("0o") or binary ("0b") format.  The parser fails without consuming input, if not at least one digit (including the '0' in the format specifiers "0x" etc.) can be parsed, after consuming input, if no digit comes after an exponent marker or no hex digit comes after a format specifier, after consuming input, if the value represented by the input string is greater than `System.Int32.MaxValue` or less than `System.Int32.MinValue`.
val pint32 : Parser<int32,'u>
```

In other words, `pint32` will fail without consuming input, unless it thinks this is a hexadecimal number. This can be easily demonstrated:

```
> run (pint32 <|> preturn 1) "0x!@#";;
val it : ParserResult<int32,unit> =
  Failure:
Error in Ln: 1 Col: 3
0x!@#
  ^
Expecting: hexadecimal digit
```

In theory this is not an issue, as `0x` is not a valid QBASIC DRAW command anyway. Regardless, the `attempt` allows the character stream to "backtrack" to before the parser being attempted fails. It is almost like a database transaction.

Below is a demonstration of how that works:

```
> run (attempt pint32 <|> preturn 1) "0x!@#";;
val it : ParserResult<int32,unit> = Success: 1
```

So we now have our final function, which is able to handle a single Draw Move.

```fsharp
let pDrawMove:Parser<DrawMove,unit> = pipe2 (anyOf "DEFGHLRU") (attempt pint32 <|> preturn 1) (fun dm dx -> 
    dx |>
    match dm with
    | 'D' -> D
    | 'E' -> E
    | 'F' -> F
    | 'G' -> G
    | 'H' -> H
    | 'L' -> L
    | 'R' -> R
    | 'U' -> U
    | _ -> failwith "pDrawMove")
```

```
> run (pDrawMove) "F123";;
val it : ParserResult<DrawMove,unit> = Success: F 123
> run (pDrawMove) "F";;
val it : ParserResult<DrawMove,unit> = Success: F 1
```

The next function is for parsing a `DrawMoveType`, a DU that determines if it is a special `DrawMove` (i.e. Blind move).

It is similar to the `pDrawMove` function earlier, but only parses a single character, otherwise it assumes the default case of `DrawMoveType.P`

```fsharp
let pMoveType:Parser<DrawMoveType,unit> = 

    let convertToDrawMoveType c =
         match c with 
         | 'B' -> B 
         | 'N' -> N 
         | _ -> P

    attempt (anyOf "BN" |>> convertToDrawMoveType) <|> preturn P
    
```

The `anyOf` parser attempts to match `B` or `N`, and if it does match, the value is passed to the function `convertToDrawMoveType` to translate to a `DrawMoveType` DU using the `|>>` function. If `anyOf` fails, the `attempt` function rolls back to the start and the `<|>` function then chooses `preturn P`.

In action:

```
> run pMoveType "BF123";;
val it : ParserResult<DrawMoveType,unit> = Success: B

> run pMoveType "F123";;
val it : ParserResult<DrawMoveType,unit> = Success: P
```

And it's worthwhile re-iterating that a parser only parses exactly as much it was intended to parse and no more. Even though the value F123 is present in BF123, the pMoveType parser only cares about the first character.

OK, so we have a function for parsing a `DrawMoveType`, and a function for parsing a `DrawMove`. Next up is a parser function that essentially combines those togther to parse a DrawCommand record. And it's actually pretty simple:

```fsharp
let pDrawCommand:Parser<DrawCommand,unit> = pipe2 pMoveType pDrawMove (fun mt dm -> { moveType = mt; direction = dm })
```

I hope by this point is pretty self explanatory :)

Lets give this new function a whirl:

```
> run pDrawCommand "F123";;
val it : ParserResult<DrawCommand,unit> =
  Success: {moveType = P;
 direction = F 123;}
> run pDrawCommand "B123";;
val it : ParserResult<DrawCommand,unit> =
  Failure:
Error in Ln: 1 Col: 2
B123
 ^
Expecting: any char in ‘DEFGHLRU’

> run pDrawCommand "BF123";;
val it : ParserResult<DrawCommand,unit> =
  Success: {moveType = B;
 direction = F 123;}
```

I accidently made a typo, but I left it there because I thought it good to show how it fails as well. This is much more graceful than QBASIC, that's for sure:

![QBASIC don't take no invalid input](/assets/bgc/draw3.PNG)

*Above: _Illegal Function Call_ is the only error you get if anything is wrong.*

The final piece of the puzzle: parsing multiple `DrawCommand` into a list. Again, this is pretty simple:

```fsharp
let pDrawCommands:Parser<DrawCommand list, unit> = sepBy pDrawCommand (pstring " ")
```

That is it! FParsec includes a function called `sepBy`, which attempts to run a parser as many times as it can, where the values to be parsed are seperated by a delimiter -- in this case a space.

Here it is in action, parsing the eponymous 'BGC' tank:

```
> run pDrawCommands "BL5 BU5 BR1 R8 F D2 G D2 F D2 G L8 H U2 E U2 H U2 E";;
val it : ParserResult<DrawCommand list,unit> =
  Success: [{moveType = B;
  direction = L 5;}; {moveType = B;
                      direction = U 5;}; {moveType = B;
                                          direction = R 1;}; {moveType = P;
                                                              direction = R 8;};
 {moveType = P;
  direction = F 1;}; {moveType = P;
                      direction = D 2;}; {moveType = P;
                                          direction = G 1;}; {moveType = P;
                                                              direction = D 2;};
 {moveType = P;
  direction = F 1;}; {moveType = P;
                      direction = D 2;}; {moveType = P;
                                          direction = G 1;}; {moveType = P;
                                                              direction = L 8;};
 {moveType = P;
  direction = H 1;}; {moveType = P;
                      direction = U 2;}; {moveType = P;
                                          direction = E 1;}; {moveType = P;
                                                              direction = U 2;};
 {moveType = P;
  direction = H 1;}; {moveType = P;
                      direction = U 2;}; {moveType = P;
                                          direction = E 1;}]
```

## Bonus round

One of the draw moves not supported is the `DrawMove.M` case. Partly because it would make parsing more complex, but also because I never really used it (even though `BM-5,-5` would technically be more efficient than `BL5 BU5`). After writing this article, I got curious as to how the parser `pMoveType` might handle an M command.

The basic idea is, after parsing any of the draw moves "MDEFGHLRU", they can either have 1 or 2 values. So I defined a new function `pMoveDist` which takes the place of the original `(attempt pint32 <|> preturn 1)` which assumed there was only a single `int32` value.

```fsharp
let pMoveDist:Parser<(int * int), unit> =    
    let parse2 = pipe2 pint32 ((pstring ",") >>. pint32) (fun dx dy -> (dx, dy))
    let parse1 = pint32 |>> fun dx -> (dx, 0)
        
    attempt parse2 <|> parse1
```

So what is this code doing? First it `attempt`s to parse two `int32` values seperated by a comma, and if that fails, retry parsing only 1 `int32`.

If you're wondering why I don't simply run the `pint32` parser directly, it because the `pMoveDist` parser returns a two element `(int * int)` tuple. The internal `parse1` function parses a `pint32` and then pipes the parsed value into an anonymous function which converts to a two element tuple, assuming the Y value is 0.

The new version of pMoveDraw (called pMoveDraw' in this example) is thus:

```fsharp
let pDrawMove':Parser<DrawMove,unit> = pipe2 (anyOf "MDEFGHLRU") (attempt pMoveDist <|> preturn (1,0)) (fun dm dx ->     
    if dm = 'M' then
        M(dx)
    else
        (fst dx) |>
        match dm with    
        | 'D' -> D
        | 'E' -> E
        | 'F' -> F
        | 'G' -> G
        | 'H' -> H
        | 'L' -> L
        | 'R' -> R
        | 'U' -> U
        | _ -> failwith "pDrawMove")

```

It's worth mentioning that dx has become an `(int * int)` tuple, but otherwise there is still only two parameters.

And now this new function in action:

```
> run pDrawMove' "M1,2";;
val it : ParserResult<DrawMove,unit> = Success: M (1,2)
> run pDrawMove' "D1";;
val it : ParserResult<DrawMove,unit> = Success: D 1
```

This is by no means complete. The M command is also able to handle relative versus absolute X and Y co-ordinates, something this parser cannot handle. However, this implementation demonstrates how a parser might handle multiple scenarios.

After thinking about this some more, ironically it appears I do need at least two seperate parsers: one for M commands, and another to handle everything else. This is somewhat similar to my original plan to have a `pMoveTypeR` and a `pMoveTypeD` etc seperate parsers, which I deemed too much work.

In the interests of keeping this article short, I'll save that for another future article.

## Caveats

This parser does not handle the `C` draw command. The `C` command gives different colours to the lines as they are drawn. For example, the "health box" is drawn with a white plus symbol:

```qbasic
healthbox$ = "C2 BU5 BL5 R10 D10 L10 U10 BF2 BR2 C15 R2 D2 R2 D2 L2 D2 L2 U2 L2 U2 R2 U2"
```

![BGC health box](/assets/bgc/draw4.PNG)

Something as simple and innocuous like this can actually be kind of complicated. This is because it requires _user state_. Effectively, the parser needs to keep track of the currently set colour. There are other ways of handling it, such as creating a "DrawMoveType.C" case, and making direction an Option<T> type, but this seems kind of hacky. It also forces the line drawer to keep track of state, and potentially re-calculate the state over and over. It really should just be given a list of draw commands.

Expect a future article where I detail how I handle this scenario.

## Conclusion

The beauty of FParsec is the emphasis of building complex parsers from simpler parsers and combining them together -- a so called parser combinator library. I might mention the word _monad_ but I don't want to scare people away :)

This parser does not handle every QBASIC DRAW command, but for my purposes it handles enough. And I have also learned a lot more about FParsec as a result of creating this library.

### Unit Tests?

If you had not noticed, this sort of system is very amenable to unit testing, as each parser can be easily tested in isolation. At the same time, it also plays well with the Interactive style of development. Once again, I don't really use unit tests here because the output (a vector image) will be very obvious if it is wrong. Besides, I can always come back to this article for an explanation, and runnable code, if I forget how this system works.

## Full sauce code

> Note: Make sure you include a reference to the FParsec dlls before attempting to run this code.

```fsharp
open FParsec
open System

let tank = "BL5 BU5 BR1 R8 F D2 G D2 F D2 G L8 H U2 E U2 H U2 E"

type DrawMove =
| M of int * int //M[{+|-}]x%,y%    Moves cursor to point x%,y%. If x% is preceded, by + or -, moves relative to the current point.
| D of int //"D n" draws a line vertically DOWN n pixels.
| E of int //"E n" draws a diagonal / line going UP and RIGHT n pixels each direction.
| F of int //"F n" draws a diagonal \ line going DOWN and RIGHT n pixels each direction.
| G of int //"G n" draws a diagonal / LINE going DOWN and LEFT n pixels each direction.
| H of int //"H n" draws a diagonal \ LINE going UP and LEFT n pixels each direction.
| L of int //"L n" draws a line horizontally LEFT n pixels.
| R of int //"R n" draws a line horizontally RIGHT n pixels.
| U of int //"U n" draws a line vertically UP n pixels.

type DrawMoveType =
| B
| N
| P

type DrawCommand = {
     moveType:DrawMoveType
     direction:DrawMove          
}

let pDrawMove:Parser<DrawMove,unit> = pipe2 (anyOf "DEFGHLRU") (attempt pint32 <|> preturn 1) (fun dm dx -> 
    dx |>
    match dm with
    | 'D' -> D
    | 'E' -> E
    | 'F' -> F
    | 'G' -> G
    | 'H' -> H
    | 'L' -> L
    | 'R' -> R
    | 'U' -> U
    | _ -> failwith "pDrawMove")


let pMoveType:Parser<DrawMoveType,unit> = 
    let convertToDrawMoveType c =
         match c with 
         | 'B' -> B 
         | 'N' -> N 
         | _ -> P

    attempt (anyOf "BN" |>> convertToDrawMoveType) <|> preturn P

let pDrawCommand:Parser<DrawCommand,unit> = pipe2 pMoveType pDrawMove (fun mt dm -> { moveType = mt; direction = dm })

let pDrawCommands:Parser<DrawCommand list, unit> = sepBy pDrawCommand (pstring " ")

let parse() =
    run pDrawCommands "BL5 BU5 BR1 R8 F D2 G D2 F D2 G L8 H U2 E U2 H U2 E"
```