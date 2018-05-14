---
layout: post
title: "Line Rendering Engine with SDL2"
date: 2018-05-14
---

>(Notice: this article is part of the [BGC Rewrite Series](/blog/2017/10/03/bgc-rewrite-series))

In the previous article, I discussed [handling colour commands with the FParsec based DRAW command parser](/blog/2018/04/03/bgc-parsing-qbasic-draw-commands-fparsec-part2), where I let slip that I actually had real output on the screen. In this article, I discuss how I translate a QBASIC DRAW command string directly into a vector graphic using SDL2.

In other words, the Line Rendering Engine.

## The Line Renderer

```fsharp
        lines         
        |> transformLines (rotatePoint (angle) >> (translatePoint (200.0, 200.0)))
        |> List.iter (fun (v1,v2) ->             
             renderer |> setDrawColor (v2.color |> drawColor)
             let x1,y1 = v1.points
             let x2,y2 = v2.points
             SDL.SDL_RenderDrawLine(renderer, int x1, int y1, int x2, int y2) |> ignore)
```

And there you have it:

![all lines are correctly rendered with their colors](/assets/bgc/line5.PNG)

## End of Article?

Note: I have a suspicion that my articles are generally considered too long, and thus are subject to the *TL;DR Effect*, so I thought I'd try things a little different and show the conclusion first. Those who are skimming will at least see the end result of the massive article below.

If you are now interested, you may continue your (ir)regularly scheduled programming.

## Previously...

I know I know, I have already referenced a previous article, and now you need to check out another one?

To be fair, this article really follows on from [Reverse Engineering the QBASIC DRAW Command](/blog/2017/12/09/bgc-qbasic-draw-command), but I have to put in at least one link to the previous article somewhere. *grin*

At this point I had a function for translating a list of DRAW commands represented as a simple Discriminated Union into a list of co-ordinates, which I could then plot onto an XY line graph using FSharp.Charting:

![Line chart depicting the eponymous tank rotated](/assets/bgc/draw2.PNG)

But this is not the same as using the real thing, SDL2. The focus of this article is on translating QBASIC draw commands into lines.

## Line Drawing

SDL1 does not have line drawing functions. You get a graphics buffer and the ability to set pixels.

Thankfully, I am not using SDL1; SDL2 does have a simple line drawing function.

```csharp
		/* renderer refers to an SDL_Renderer* */
		[DllImport(nativeLibName, CallingConvention = CallingConvention.Cdecl)]
		public static extern int SDL_RenderDrawLine(
			IntPtr renderer,
			int x1,
			int y1,
			int x2,
			int y2
		);
```

As you can see, the C#-SDL wrapper library is barely above the bare metal. However, with a few exceptions, F# generally makes this no big deal to work with.

The first issue is that lines need two sets of co-ordinates, usually represented as (x1,y1)-(x2,y2). The function used for translating DRAW commands into a list of co-ordinates is actually only giving back a list of points.

The charting software used in the above graphic handles the connection of lines automatically. But I do not have that luxury. I need some way to map from a list of points to a list of co-ordinate pairs.

Turns out there's a function for that: `Seq.pairwise`. This function takes a list and returns a 2 element tuple consisting of the 1st and 2nd elements, the 2nd and 3rd elements etc. Perhaps a demonstration is in order:

```
> let elements = ["A"; "B"; "C"; "D"; "E"];;

val elements : string list = ["A"; "B"; "C"; "D"; "E"]

> elements |> Seq.pairwise;;
val it : seq<string * string> =
  seq [("A", "B"); ("B", "C"); ("C", "D"); ("D", "E")]
```

That is pretty much exactly what I need. So feeding a punch of point coordinates into `Seq.pairwise`

```fsharp
let coords = [(0, 0); (-5, 0); (-5, -5); (-4, -5); (4, -5); (5, -4); (5, -2); (4, -1);
   (4, 1); (5, 2); (5, 4); (4, 5); (-4, 5); (-5, 4); (-5, 2); (-4, 1);
   (-4, -1); (-5, -2); (-5, -4); (-4, -5)]
```

results in 

```fsharp
> coords |> Seq.pairwise;;
val it : seq<(int * int) * (int * int)> =
  seq
    [((0, 0), (-5, 0)); ((-5, 0), (-5, -5)); ((-5, -5), (-4, -5));
     ((-4, -5), (4, -5)); ...]   
```

Which looks like a bunch of line coordinates too me!

From here, it's a relatively trivial exercise in getting the lines onto the screen:

```fsharp
    let coords = [(0, 0); (-5, 0); (-5, -5); (-4, -5); (4, -5); (5, -4); (5, -2); (4, -1);
                   (4, 1); (5, 2); (5, 4); (4, 5); (-4, 5); (-5, 4); (-5, 2); (-4, 1);
                   (-4, -1); (-5, -2); (-5, -4); (-4, -5)]

    coords
    |> Seq.map(fun (x,y) -> (x*10+200,y*10+200))
    |> Seq.pairwise
    |> Seq.iter (fun ((x1,y1), (x2,y2)) ->
        SDL.SDL_RenderDrawLine(renderer, x1, y1, x2, y2) |> ignore)
```

"Trivial" I hear you say. Don't worry, I'll go over each line

```fsharp
    coords
    |> Seq.map(fun (x,y) -> (x*10+200,y*10+200))
```

First off, I put in a quick Seq.map to do a couple of point translations: scale by 10, and translate both x and y by 200 pixels. If I didn't do this, the render would be tiny and partly off the screen.

If the `fun (x,y)` looks weird, this is a good time to mention deconstruction and pattern matching. The `(x,y)` is a two element tuple, and we have a list of two element tuples. So this anonymous function just deconstructs the tuple values into x and y.

```fsharp
    |> Seq.pairwise
```

This transforms the list of tuples into two tuple pairs, i.e. `((x1,y1),(x2,y2))`

```fsharp
    |> Seq.iter (fun ((x1,y1), (x2,y2)) ->
        SDL.SDL_RenderDrawLine(renderer, x1, y1, x2, y2) |> ignore)
```

Here the actual rendering takes place. The two tuple pair is deconstructed as above and the values are placed into the `SDL.SDL_RenderDrawLine` function.

And that's the line renderer in a nutshell.

## From QBASIC DRAW to Putting Lines on the Screen

At this point, we are ready to start turning this

```
"C2 BU5 BL5 R10 D10 L10 U10 BF2 BR2 C15 R2 D2 R2 D2 L2 D2 L2 U2 L2 U2 R2 U2"
```

into this

![Output from the line renderer after interpreting the QBASIC DRAW string directly](/assets/bgc/healthbox_sdl.PNG)

Using nothing but this

```fsharp
let nextCoord (b:bool, ((px:int, py:int), c:int)) (command:DrawCommand) =
    let notImplemented () = failwith "not implemented"
    (match command.moveType with
    | B -> false
    | _ -> true),
    (match command.direction with
    | D (dx) -> (px, py + dx)
    | E (dx) -> (px + dx, py - dx)
    | F (dx) -> (px + dx, py + dx)
    | G (dx) -> (px - dx, py + dx)
    | H (dx) -> (px - dx, py - dx)
    | L (dx) -> (px - dx, py)
    | R (dx) -> (px + dx, py)
    | U (dx) -> (px, py - dx)            
    //unsupported yet
    | _ -> notImplemented()
    ,command.color)

let scalePoints (factor:float) (points:(float * float) list) =
    points |> List.map (fun (x,y) -> (x*factor,y*factor))

let scaleLines (factor:float) (lines:(((float*float)*int)*((float*float)*int)) list) =
    lines |> List.map (fun (((x1,y1),c1),((x2,y2),c2)) -> ((x1*factor, y1*factor),c1),((x2*factor, y2*factor),c2))

let scaleLinesXY (factorX:float, factorY:float) (lines:(((float*float)*int)*((float*float)*int)) list) =
    lines |> List.map (fun (((x1,y1),c1),((x2,y2),c2)) -> ((x1*factorX, y1*factorY),c1),((x2*factorX, y2*factorY),c2))


let translatePoints (x1,y1) (points:((float * float)*int) list) =
    points |> List.map (fun ((x,y),c) -> ((x+x1,y+y1),c))

let translateLines (dx,dy) (lines:(((float*float)*int)*((float*float)*int)) list) =
    lines |> List.map (fun (((x1,y1),c1),((x2,y2),c2)) -> ((x1+dx, y1+dy),c1),((x2+dx, y2+dy),c2))

let rotatePoint (angle:float) (x:float, y:float) =     
    //gosh, at no point did I think to store (-tankangle * PI.f) / 180 into a variable
    //rotatey.f = ((unit()\weapon_x[I]) * Cos(((-tankangle * PI.f) / 180))) - ((unit()\weapon_y[I]) * Sin(((-tankangle * PI.f) / 180)))
    //rotatex.f = ((unit()\weapon_y[I]) * Cos(((-tankangle * PI.f) / 180))) + ((unit()\weapon_x[I]) * Sin(((-tankangle * PI.f) / 180)))   
    let rad = ((angle)*Math.PI)/180.0
    let cos' = cos(rad)
    let sin' = sin(rad)    
    //you know, I can't remember I came up with this, but it works.
    (x * cos') - (y * sin'), (y * cos') + (x * sin')


let rotatePoints (angle:float) (points:((float * float)*int) list) =               
    points |> List.map (fun (xy,c) -> rotatePoint angle (xy),c)

let rotateLines (angle:float) (lines:(((float*float)*int)*((float*float)*int)) list) =
    lines |> List.map (fun (((p1),c1),((p2),c2)) ->
        ((rotatePoint (angle) (p1)), c1), (((rotatePoint(angle) (p2)), c2))
        )    
        

let sdlRenderClear = SDL.SDL_RenderClear >> ignore
let sdlInit = SDL.SDL_Init
let renderPresent = SDL.SDL_RenderPresent
let setDrawColor (r:byte, g:byte, b:byte) renderer = SDL.SDL_SetRenderDrawColor(renderer, r, g, b, 0uy) |> ignore

let lines =     
    healthbox
    |> List.scan nextCoord (false, ((0,0), 0))
    |> List.pairwise 
    |> List.where (fun ((b1, ((_,_),_)), (b2, ((_,_),_))) -> 
        match b1,b2 with
        | true,true -> true
        | true,false -> false
        | false,true -> true
        | false,false -> false
        ) 
    |> List.map (fun ((_, ((x1,y1), c1)), (_, ((x2,y2), c2))) -> ((float x1, float y1), c1),((float x2, float y2), c2))    
    |> scaleLines (10.0)

```

Yeah OK I admit that the code has gotten fairly messy! Part of the fun is starting with a design and taking it to it's horrible conclusion.

So we have a bunch of QBASIC DRAW commands which have been parsed into an F# record structure.

```fsharp
let healthbox = [{moveType = B;
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
... note: not all commands are shown
```

The process of converting them into a bunch of line co-ordinates is pretty much the same as detailed in [Reverse Engineering The QBASIC DRAW Command article](/blog/2017/12/09/bgc-qbasic-draw-command); the `nextCoord` function is changed to also handle the colour. At this point the design was starting to bug me. But I pressed on.

```fsharp
let nextCoord (b:bool, ((px:int, py:int), c:int)) (command:DrawCommand) =
    let notImplemented () = failwith "not implemented"
    (match command.moveType with
    | B -> false
    | _ -> true),
    (match command.direction with
    | D (dx) -> (px, py + dx)
    | E (dx) -> (px + dx, py - dx)
    | F (dx) -> (px + dx, py + dx)
    | G (dx) -> (px - dx, py + dx)
    | H (dx) -> (px - dx, py - dx)
    | L (dx) -> (px - dx, py)
    | R (dx) -> (px + dx, py)
    | U (dx) -> (px, py - dx)        
    | _ -> notImplemented()
    ,command.color)
```

```fsharp
let lines =     
    healthbox
    |> List.scan nextCoord (false, ((0,0), 0))
    |> List.pairwise 
    
val lines : ((bool * ((int * int) * int)) * (bool * ((int * int) * int))) list
= [((false, ((0, 0), 0)), (false, ((0, -5), 2)));
   ((false, ((0, -5), 2)), (false, ((-5, -5), 2)));
   ((false, ((-5, -5), 2)), (true, ((5, -5), 2)));
   ((true, ((5, -5), 2)), (true, ((5, 5), 2)));
   ((true, ((5, 5), 2)), (true, ((-5, 5), 2)));
   ((true, ((-5, 5), 2)), (true, ((-5, -5), 2)));
   ((true, ((-5, -5), 2)), (false, ((-3, -3), 2)));
   ((false, ((-3, -3), 2)), (false, ((-1, -3), 2)));
   ((false, ((-1, -3), 2)), (true, ((1, -3), 15)));
   ((true, ((1, -3), 15)), (true, ((1, -1), 15)));
   ((true, ((1, -1), 15)), (true, ((3, -1), 15)));
   ((true, ((3, -1), 15)), (true, ((3, 1), 15)));
   ((true, ((3, 1), 15)), (true, ((1, 1), 15)));
   ((true, ((1, 1), 15)), (true, ((1, 3), 15)));
   ((true, ((1, 3), 15)), (true, ((-1, 3), 15)));
   ((true, ((-1, 3), 15)), (true, ((-1, 1), 15)));
   ((true, ((-1, 1), 15)), (true, ((-3, 1), 15)));
   ((true, ((-3, 1), 15)), (true, ((-3, -1), 15)));
   ((true, ((-3, -1), 15)), (true, ((-1, -1), 15)));
   ((true, ((-1, -1), 15)), (true, ((-1, -3), 15)))]    
```

Technically this can be plotted right now, but all the lines including that which are supposed to be *Blind* moves will be rendered, which looks wrong:

![All lines are being rendered](/assets/bgc/line1.PNG)

*Above: all lines are being rendered*

At first I tried just filtering out any point that was the result of a blind move.

```fsharp
let lines =     
    tank
    |> List.scan nextCoord (false, ((0,0), 0))
    |> List.where (fun (b, ((_,_),_)) -> b)
    |> List.pairwise 
```

Seemed simple enough, but simple is sometimes not correct:

![Not enough lines are being rendered](/assets/bgc/line2.PNG)

*Above: too many lines are eliminated*

The problem is that a point which is the result of a blind move still forms part of line which should be visible. So I figured if either point in a line is not the result of a blind move, then the line should be visible.

```fsharp
let lines =     
    healthbox
    |> List.scan nextCoord (false, ((0,0), 0))    
    |> List.pairwise 
    |> List.where (fun ((b1, ((_,_),_)), (b2, ((_,_),_))) -> b1 || b2 )
```

Of course, just as I finished writing this code, I received a message from myself ... from the future, saying that this would not work, sending me this picture:

![The healthbox is rendered wrong](/assets/bgc/line3.PNG)

Damn! I had not even got to this part of the line renderer yet, and I am already wrong.

Unfortunately my future self was not able to tell me the solution, so I of course stumbled through a few more iterations until I finally came up with a state machine, where I explicitly spelled out every possible scenario:

* if both points are visible, the line is visible
* if the first point is visible, but the second is not, then not visible
* if the first point is invisible, but second is visible, then visible
* if both points are invisible, then not visible

Represented as a match statement, it was clear I was an idiot:

```fsharp
let lines =     
    healthbox
    |> List.scan nextCoord (false, ((0,0), 0))
    |> List.pairwise 
    |> List.where (fun ((b1, ((_,_),_)), (b2, ((_,_),_))) -> 
        match b1,b2 with
        | true,true -> true
        | true,false -> false
        | false,true -> true
        | false,false -> false
        ) 
```

The match statement showed a pattern: the line is visible if the second point is visible. And the above reduces to:

```fsharp
let lines =     
    healthbox
    |> List.scan nextCoord (false, ((0,0), 0))    
    |> List.pairwise 
    |> List.where (fun ((b1, ((_,_),_)), (b2, ((_,_),_))) -> b2 )       
```

Also, the translate and rotate functions require floating point numbers, so a final map to convert from integer to float was needed.

```fsharp
let lines =     
    healthbox
    |> List.scan nextCoord (false, ((0,0), 0))    
    |> List.pairwise 
    |> List.where (fun ((b1, ((_,_),_)), (b2, ((_,_),_))) -> b2 )
    |> List.map (fun ((_, ((x1,y1), c1)), (_, ((x2,y2), c2))) -> ((float x1, float y1), c1),((float x2, float y2), c2))
```

At this point I was beginning to feel the pain of using tuples. But I will get to that later.

So I have a bunch of line coordinates stored in the `lines` variable.

The final step is a function for translating a QBASIC colour into an RGB colour. This is fairly trivial to accomplish using a match statement:

```fsharp
let drawColor c =
    match c with
    | 0 ->  (0uy,   0uy,    0uy)
    | 1 ->  (0uy,   0uy,    170uy)
    | 2 ->  (0uy,   170uy,  0uy)
    | 3 ->  (0uy,   170uy,  170uy)
    | 4 ->  (170uy, 0uy,    0uy)
    | 5 ->  (170uy, 0uy,    170uy)
    | 6 ->  (170uy, 85uy,   0uy)
    | 7 ->  (170uy, 170uy,  170uy)
    | 8 ->  (85uy,  85uy,   85uy)
    | 9 ->  (85uy,  85uy,   255uy)
    | 10 -> (85uy,  255uy,  85uy)
    | 11 -> (85uy,  255uy,  255uy)
    | 12 -> (255uy, 85uy,   85uy)
    | 13 -> (255uy, 85uy,   255uy)
    | 14 -> (255uy, 255uy,  85uy)
    | 15 -> (255uy, 255uy,  255uy)
    | _ -> failwith "unexpected color code"
```

I recall stating in the [last article](/blog/2018/04/03/bgc-parsing-qbasic-draw-commands-fparsec-part2):

> I can't say that any thought was given to the order of colours.

And of course that statement comes back to haunt me. It's reasonably clear from the match statement that they are gradually increasing the RGB values from black to white, and that the colours are grouped by darker and lighter. So there you have it.

We now have everything needed to render the lines to the screen

```fsharp
        lines                     
        |> rotateLines (angle) 
        |> translateLines (200.0,200.0)
        |> List.iter (fun (((x1,y1),c1),((x2,y2),c2)) ->             
             renderer |> setDrawColor (c1 |> drawColor)
             SDL.SDL_RenderDrawLine(renderer, int x1, int y1, int x2, int y2) |> ignore)
```

![Healthbox line rendering - close but no cigar: the top line of the 'plus' shape is the wrong colour](/assets/bgc/line4.PNG)

*Above: close but no cigar: the top line of the 'plus' shape is the wrong colour*

... well, almost. Learning from my mistake earlier about blind moves, it follows the the line should probably be the colour of the second point.

```fsharp
        lines         
        |> transformLines (rotatePoint (angle) >> (translatePoint (200.0, 200.0)))
        |> List.iter (fun (v1,v2) ->             
             renderer |> setDrawColor (v2.color |> drawColor)
             let x1,y1 = v1.points
             let x2,y2 = v2.points
             SDL.SDL_RenderDrawLine(renderer, int x1, int y1, int x2, int y2) |> ignore)
```

And there you have it:

![all lines are correctly rendered with their colors](/assets/bgc/line5.PNG)

## Refactoring

The article could end now, but I thought I would take some time to address the attrocious code seen above.

You can see now that writing lambdas was now starting to get frustrating. I reckon it was about the time I wrote these three functions...

```fsharp
let scaleLines (factor:float) (lines:(((float*float)*int)*((float*float)*int)) list) =
    lines |> List.map (fun (((x1,y1),c1),((x2,y2),c2)) -> ((x1*factor, y1*factor),c1),((x2*factor, y2*factor),c2))

let translateLines (dx,dy) (lines:(((float*float)*int)*((float*float)*int)) list) =
    lines |> List.map (fun (((x1,y1),c1),((x2,y2),c2)) -> ((x1+dx, y1+dy),c1),((x2+dx, y2+dy),c2))
    
let rotateLines (angle:float) (lines:(((float*float)*int)*((float*float)*int)) list) =
    lines |> List.map (fun (((p1),c1),((p2),c2)) ->
        ((rotatePoint (angle) (p1)), c1), (((rotatePoint(angle) (p2)), c2))
        )    

```

*Above: probably some of the worst F# I have written*

... that I realised that I really should ditch the tuple and use a proper structure. Especially when I realised I needed to pass colours along with each line (hence the `c1` and `c2`), and had to modify EVERY function. Information needed for the line renderer, but completely irrelevent for point calculations.

I know, I know, you are probably yelling "This is why you use a game engine!" and you would be 100% correct. But then you would not be reading this article if you were interested in using a game engine *grin*

So I _finally_ decided to introduce a Vertex structure. I figure I just pinch the term from OpenGL.

```fsharp
type Vertex = {
    blind: bool
    color: int
    points: (float*float)
}
```

which was nothing more than the original tuple, just now with structure. The functions now look less terrible (but are probably terrible for other reasons **cough* re-inventing matrix math* )

```fsharp
let scaleLines (factor:float) (lines:(Vertex*Vertex) list) =
    let scale = scalePoint factor
    lines |> List.map (fun (v1,v2) -> ({ v1 with points = scale v1.points }, { v2 with points = scale v2.points }))

let translateLines (dx,dy) (lines:(Vertex * Vertex) list) =
    let t = translatePoint (dx,dy)
    lines |> List.map (fun (v1,v2) -> ({ v1 with points = t v1.points }, { v2 with points = t v2.points }))
    
let rotateLines (angle:float) (lines:(Vertex * Vertex) list) =
    let t = rotatePoint (angle)
    lines |> List.map (fun (v1,v2) ->  { v1 with points = t v1.points }, { v2 with points = t v2.points } )                

```

The functions `rotatePoint`, `scalePoint` and `translatePoint` are mini functions that make working with the (x,y) tuple easier.

The code for rendering also becomes a bit nicer

```fsharp
        lines
        |> rotateLines (angle) 
        |> translateLines (200.0,200.0)
        |> List.iter (fun (v1,v2) ->             
             renderer |> setDrawColor (v2.color |> drawColor)
             let x1,y1 = v1.points
             let x2,y2 = v2.points
             SDL.SDL_RenderDrawLine(renderer, int x1, int y1, int x2, int y2) |> ignore)
```

Actually, another thing that was bothering me was the fact that the rotation and translation were two steps, both requiring iteration of the lines list twice.

And it occurred to me here when I wrote these mini functions 

```fsharp
let translatePoint (dx,dy) (x,y) = (x+dx,y+dy)

let scalePoint (factor:float) (x,y) = (x*factor, y*factor)

```

Are performing some sort of translation of a point. And in the revised functions

```fsharp
let scaleLines (factor:float) (lines:(Vertex*Vertex) list) =
    let scale = scalePoint factor
    lines |> List.map (fun (v1,v2) -> ({ v1 with points = scale v1.points }, { v2 with points = scale v2.points }))
    
let translateLines (dx,dy) (lines:(Vertex * Vertex) list) =
    let t = translatePoint (dx,dy)
    lines |> List.map (fun (v1,v2) -> ({ v1 with points = t v1.points }, { v2 with points = t v2.points }))    
    
let rotateLines (angle:float) (lines:(Vertex * Vertex) list) =
    let t = rotatePoint (angle)
    lines |> List.map (fun (v1,v2) ->  { v1 with points = t v1.points }, { v2 with points = t v2.points } )                
    
```

I am essentially hard-coding the point translation, but the code is otherwise identical. This suggests I could use a *Higher Ordered Function* to create a general `transformLine` function like so:

```fsharp
let transformLines transform (lines:(Vertex * Vertex) list) =    
    lines |> List.map (fun (v1,v2) -> ({ v1 with points = transform v1.points }, { v2 with points = transform v2.points }))    
    
```

From which I could dervice specific functions

```fsharp
let scaleLines (factor:float) = transformLine (scalePoint factor)
let translateLine (dx,dy) = transformLine (translatePoint (dx,dy))
let rotateLines (angle:float) = transformLines (rotatePoint (angle))    
```

But mathematically there's no reason why scaling, rotatating and translating have to be calculated as descrete steps. In that case, I could make use of function composition:

```fsharp
        lines         
        |> transformLines (rotatePoint (angle) >> (translatePoint (200.0, 200.0)))
        |> List.iter (fun (v1,v2) ->             
             renderer |> setDrawColor (v2.color |> drawColor)
             let x1,y1 = v1.points
             let x2,y2 = v2.points
             SDL.SDL_RenderDrawLine(renderer, int x1, int y1, int x2, int y2) |> ignore)
```

How about that! And that also saves multiple list iterations. I think sometimes I think too much in terms of SQL WHERE clause filtering.

## Aside: Function Composition

If you are scratching your head over this line

```fsharp
|> transformLines (rotatePoint (angle) >> (translatePoint (200.0, 200.0)))
```

you might not be aware of functional composition. the `>>` operator __creates a new function__ by taking two functions and essentially gluing them together -- taking the output from one function and feeding it directly into the input of another.

So if you have a hypothetical function that takes an A and return a B, and another function that takes a B and returns a C, you can glue them together to get a new function that takes an A and returns a C.

This sounds very similar to the Pipe operator `|>` and you may wonder why you cannot use the pipe operator instead:

```fsharp
|> transformLines (rotatePoint (angle) |> (translatePoint (200.0, 200.0)))
```

This does not work. `transformLines` expects a function, but instead it has been given a value.

The main difference is the `|>` operator will give you a value, whereas `>>` will give you a function.

```fsharp
> let transform = (rotatePoint (angle)) |> (translatePoint (200.0, 200.0));;

>   let transform = (rotatePoint (angle)) |> (translatePoint (200.0, 200.0));;
  ------------------------------------------^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

stdin(16,43): error FS0001: Type mismatch. Expecting a
    (float * float -> float * float) -> 'a    
but given a
    float * float -> float * float    
The type 'float * float -> float * float' does not match the type 'float * float'
```

Instead you have to make it a function

```fsharp
> let transform v = v |> (rotatePoint (angle)) |> (translatePoint (200.0, 200.0));;

val transform : float * float -> float * float
```

So in a way, using `>>` is shorthand for the code above:

```fsharp
> let transform = (rotatePoint (angle)) >> (translatePoint (200.0, 200.0));;

val transform : (float * float -> float * float)
```