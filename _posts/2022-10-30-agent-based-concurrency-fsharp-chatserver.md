---
layout: post
title: "Concurrent Programming - A Chat Server in F#"
date: 2022-10-30
---

A long time ago (but still in this galaxy), a junior programmer created a chat server, as a means of learning multithreaded programming, in the Delphi language.

Then several years later, he decided to rewrite it in F#.

This is that article.

## Another Chat Server? Really?

At my place of work, the preferred chat client of choice was MSN Messenger. I was not very fond of MSN Messenger; it used a lot of memory (at least 30 megabytes), it was slow, and it had annoying animated "emoticons".

I realise that these are not good reasons to roll your own chat program, but I was young and I was also keen to learn multithreaded programming, so this seemed like a good exercise in <s>re-inventing the wheel</s> learning.

Note: this chat program was named after my place of work but since I don't want to link them to this crime scene, this chat system is now known as _God Dammit Another Messaging Protocol What was Wrong With MSN Messenger/AIM/IRC/Jabber_ Chat or, for obvious reasons, abbreviated to INTERCHAT<sup>[1](#note1)</sup>.

## The Delphi version

I won't say too much about this, as it is not the focus of this article... and also technically the code is proprietary. :)

At the heart of the chat system was the Internet Direct aka "Indy" Components, which made TCP programming incredibly easy -- almost drag and drop depending on what was being done.

The server used the TIdTcpServer component. The TIdTcpServer class had several methods which would be invoked under certain conditions (which you can probably infer from the names)

```delphi  
  TIdTcpServer = class(TIdCustomTcpServer)
    public procedure OnConnect(AThread:TIdThread);
    public procedure OnDisconnect(AThread:TIdThread);
    public procedure OnExecute(AThread:TIdThread);
  end
```

Each TCP connection was encapsulated in a `TIdThread`, which you could then refer read and write to the thread's TCP stream, with something like

```delphi
  AThread.Connection.WriteLn('blah')
```

The magic happens in `OnExecute()`: every time a connection is established, the `OnExecute()` is invoked with the thread. With this framework, writing a simple TCP socket server with your own custom protocol is easily accomplished.

```delphi

public FTcpServer.OnExecute(AThread:TIdThread);
var
  data: string;
  packet: TStringList;
begin
  
	data = AThread.Connection.ReadLn();        
	packet := TStringList.Create();
	try            
	  //a TStringList by default will parse a string of tab delimited data into a list of strings
	  packet.Delimited = data;
	  if (packet[0] = 'SOME_COMMAND') then
	  begin
      //do something
      AThread.Connection.WriteLn("OK#9Command Successful")
	  end
	  else
	  begin
      AThread.Connection.WriteLn('ERROR#9Invalid command!');
	  end
	finally
	  packet.Free;
	end;
   
end
```

And as long as you avoided shared state, multithreaded programming was almost the same as linear programming.

In this `OnExecute` function, I had my own Command Handling system which mapped the `SOME_COMMAND` seen above to a static handler function, which I thought was quite clever. Years later, I would discover that I had re-invented that which was already done by the TIdTcpServer's built in Command Handler system. D'oh.

# Agent Based Concurrency

The main concern was that of concurrent access to global state - the User list and the TCP Streams. Concurrent access to the User List was needed to prevent the list being modified by one thread while another was busy iterating the collection, potentially causing the wrong data to be retrieved or List Index out of Bounds exception being thrown.

More serious was that of the TCP Streams - I could not imagine the outcome of two threads simultaneously writing data to a single TCP stream, but it is probably something to avoid.

In the Delphi version, I protected concurrent access to the User List and TCP Streams by way of Critical Sections. This meant that any Command Handler which had to interact with the User list had to use a critical section.

At first I was going do the same - define a bunch of functions for getting, adding and deleting users that implement locking. I would need to do the same for the sessions list. But then I learned about the F# `MailboxProcessor<T>`:

> One of the keys to successful agent programming in F# is isolation. Isolation means that resources exist which "belong" to a specific agent, and are not accessed except by that agent. This means isolated state is protected from concurrent access and data races.

I had an "A-ha" moment. What I needed was a Mailbox that listened to messages to add sessions, remove sessions or send commands to sessions. The mailbox nature means all access to the global user list and TCP streams are serialized.

Before I continue, it may help to see what a typical `MailboxProcessor<T>` looks like.

```fsharp
let agent = MailboxProcessor.Start(fun inbox->

    // the message processing function
    let rec messageLoop() = async{

        // read a message
        let! msg = inbox.Receive()

        // process a message
        printfn "message is: %s" msg

        // loop to top
        return! messageLoop() 
		
        }

    // start the loop
    messageLoop()

    )
```

This version here is using recursion but you can also do standard while (true) loops.

So, I defined a `SessionManager` mailbox, which will be the central point of all async processes

![A diagram showing how the TCP Client instances interact with the SessionManager mailbox](/assets/interchat/fschat2.png) 

F# Discriminated Unions work really well with the `MailboxProcessor<T>` class. I define all the Commands the `SessionManager` will handle:


```fsharp
namespace FSChatServer.Library

open System

module Interface =   

    type SessionCommand =
    | AddSession of sessionId:Guid * mailbox:MailboxProcessor<ClientCommand>   
    | RemoveSession of sessionId:Guid
    | SendCommand of sessionId:Guid * command:Result<ServerCommand>

``` 

SendCommand is another (poorly named) discriminated union representing all the ServerCommands that can be sent to the server

```fsharp
    type ServerCommand =
    | ThisGuy  
    | Login of userId:string
    | Logout
    | SendMessage of userId:string * message:string
    | GetUserList
```
 

An async function which runs a TcpListener waits for connections, and posts an `AddSession` message to the `SessionManager`, with the session's ID and client `MailboxProcessor`

```fsharp
    let startServer (ip, port) =       
        let listener = TcpListener(ip, port)
        listener.Start()

        async {

            while true do
                let! client = listener.AcceptTcpClientAsync() |> Async.AwaitTask

                let sessionId = Guid.NewGuid()

                let clientMailbox = createClientMailbox (new StreamWriter(client.GetStream()))            

                sessionManager.Post(AddSession(sessionId, clientMailbox))               
				
                acceptClient sessionId client |> Async.Start

        }
        |> Async.Start
```

At the same time, another async function is started: acceptClient, which reads the incoming TCP stream and converts the raw TCP protocol into ServerCommands, via the function `getServerCommand`. When the loop terminates, it posts a final RemoveSession command to the SessionManager
 
```fsharp
    let acceptClient sessionId (client:TcpClient) = async {

        use stream = client.GetStream()       
        use reader = new StreamReader(stream)
        use writer = new StreamWriter(stream)       

        try

            writer.WriteLine("Welcome to INTERCHAT!");
            writer.WriteLine("Your session id: " + sessionId.ToString());
            writer.Flush()

            let rec loop header =           

                let command = OldProtocol.getServerCommand header           

                sessionManager.Post(SendCommand(sessionId, command))

                match command with
                | Success ServerCommand.Logout -> ()
                | _ -> loop (reader.ReadLine())

            loop (reader.ReadLine())

            sessionManager.Post(RemoveSession(sessionId))

            client.Close()

        with ex ->

            printfn "ACCEPT ERROR: %s" (ex.ToString())
    }
```

The function for translating the TCP stream into a `ServerCommand` is also nothing special:

```fsharp

module OldProtocol =       

    let getServerCommand (data:string): Result<ServerCommand> =                
        if (isNull data) then            
            Success(Logout) //assume disconnected
        else
            try                
                let (command, text) = parseData data
                let items = text.Split([|'\t'|])
                match command.ToLower() with
                | "this_guy" -> Success(ServerCommand.ThisGuy)
                | "login" -> Success(Login(text))
                | "disconnect" -> Success(Logout)
                | "send" -> Success(SendMessage(items.[0], items.[1]))
                | "get_user_list" -> Success(GetUserList)
                | _ -> Failed("Command not recognised")
            with ex ->
                Failed ("Invalid command structure " + data)

```

A quick note about the "OldProtocol" module name: originally, INTERCHAT was tested using plain old TELNET. If you like causing pain for yourself, you can still use TELNET

You can see these commands handled here in `SessionManager` (code is a bit messy)

```fsharp
let sessionManager =

        MailboxProcessor.Start(fun inbox ->           

            let sessions = Dictionary<Guid, MailboxProcessor<ClientCommand>>()
            let database = inMemoryUserService(new ResizeArray<User>())
            let rec messageLoop () = async {

                let! command = inbox.Receive()
				
                printf "%A\n" command

                match command with
                | AddSession (sessionId, stream) -> sessions.Add(sessionId, stream)
                | RemoveSession (sessionId) -> sessions.Remove(sessionId) |> ignore               
                | SendCommand (sessionId, serverCommand) ->
                    let res =                                                 
                        match serverCommand with
                        | Success sc ->
                            let authenticationRequired =            
                                if (commandRequiresAuthentication sc) then
                                    let user = database.getUserById(sessionId)
                                    user.IsNone
                                else
                                    false

                            if authenticationRequired then
                                Unauthenticated
                            else
                                match sc with
                                | Login (userId) ->                                
                                    match (validateUser database.getUser sessionId userId) with
                                    | Success (user) ->
                                        match database.addUser(user) with
                                        | Success (user) ->
                                            sendBroadcast sessionId (AddUser(user))
                                            OK
                                        | Failed (msg) -> Error(msg)
                                    | Failed (msg) -> Error(msg)                           
                                | ServerCommand.ThisGuy -> Data ServerDataResult.ThisGuy
                                | GetUserList -> getCurrentUsers database.getUserList  
```

Finally, all the Commands that `SessionManager` can send to a Client Session are represented as a discriminated Union as well.

```fsharp

    type ClientCommand =
    | OK
    | ReceivedMessage of fromUserId:string * toUserId:string * message:string
    | Data of data:ServerDataResult
    | Error of message:string
    | Notice of message:string
    | Unauthenticated
    | AddUser of user:User
    | DeleteUser of userId:string
    | Stop
```fsharp
 
 
And this is how those Client commands are translated into TCP streams:

 
```fsharp
    let createClientMailbox (stream:StreamWriter) =

        let mb = MailboxProcessor.Start(fun inbox ->

            let rec messageLoop () = async {

                let! command = inbox.Receive()
				
                if command <> Stop then

                    let response =
                        match command with
                        | OK -> "OK"
                        | Data data ->
                            match data with           
                            | ThisGuy -> "THIS_GUY"
                            | UserList users -> getUserList users
                        | Error msg -> "ERROR\t" + msg
                        | Notice msg -> "NOTICE\t" + msg
                        | Unauthenticated -> "ERROR\tYou must LOGIN first. Usage: LOGIN[tab]login_name"
                        | AddUser user -> "ADD_USER\t"+user.handle
                        | DeleteUser handle -> "DELETE_USER\t"+handle               
                        | ReceivedMessage (fromHandle, toHandle, msg) -> (sprintf "MSG\t%s\t%s" fromHandle msg)
                        | Stop -> String.Empty
						
                    stream.WriteLine(response)
                    stream.Flush()

                    return! messageLoop()
            }

            messageLoop()

        )

        mb.Error.Add(fun ex -> printfn "EXCEPTION: %s" ex.Message)

        mb
```
 

I also use a Mailbox to control access to the Client's TCP output stream. It may seem unnecessary, since the SessionManager means that it is impossible for two sessions to simultaneously write to another session, but posting a message asynchronously instead of writing directly to the TCP stream has the benefit of shielding the `SessionManager` from any network failures or slow connections. In the original Delphi Server design, a slow connection would have stopped the entire server. Also, in the situation of a slow client connection, the `SessionManager` is still able to post commands to that client's queue, and the client mailbox is able to process those items in the queue as fast as it can.

Overall I'm pretty happy with the design.

I will admit the design is a bit messy, perhaps I will dedicate an article on cleaning it up. I should mention that prior to using the `MailboxProcessor<T>` design, I was getting very bogged down in trying to create a `clean architecture` with everything properly separated, all dependencies passed via parameters etc. I think I got to the point where I needed to be able to actually start sending data, and realised my entire architecture was either wrong or would make it incredibly difficult to actually be the thing it was meant to be - a multi-user chat server. Not much fun being only able to talk to yourself!

## Bolting on Web Sockets

I might be a decade too late to be getting in on the Web Socket action, but non-the-less I wanted to learn about it. Luckily for me someone on GitHub had already done the hard work of creating a basic F# based Web Socket Server.

I know, I know -- the .NET ecosystem already has an off the shelf Web Socket library. But it seems to be heavily tied into ASP.NET, which I found a bit heavy handed for my needs (read: couldn't be bothered learning), and this example socket server was less than 200 lines of code. *Cue common complaints about rolling your own <insert thing here>*

_* cough *_ so anyway you can find the [original Gist here](https://gist.github.com/jonschoning/14f3a087b2bb79236131). The implementation was basic and required me to extend so that more than 127 bytes could be sent. That's glossing over a lot of detail and I could probably dedicate an entire article on what it took to send and receive more than 127 bytes, so I will not talk about it here. Suffice to say, by the time I mostly implemented RFC6455, it probably would have been quicker to use ASP.NET's inbuilt socket system. But where's the fun in that?

The main concern is retrofitting the web socket system into the existing INTERCHAT server. This turned out to be easier than expected (read:copy pasta).

First, I needed a function for running the Web Socket server:

```fsharp
    let startServerWS (ip, port) =        
        let listener = TcpListener(ip, port)
        listener.Start() 

        let handshake (tcp:TcpClient) = async {                          
            let ns = tcp.GetStream()
            let bytes = Array.create tcp.ReceiveBufferSize (byte 0)
            let bytesReadCount = ns.Read (bytes, 0, bytes.Length)
            
            if bytesReadCount > 8 then                
                let lines = bytes.[..(bytesReadCount)]
                            |> System.Text.UTF8Encoding.UTF8.GetString 
                            |> fun hs->hs.Split([|"\r\n"|], StringSplitOptions.RemoveEmptyEntries) 
                match WebSocketProtocol.isWebSocketsUpgrade lines with
                | true ->
                    let acceptStr =  
                                    (WebSocketProtocol.getKey "Sec-WebSocket-Key:" lines                 
                                    ).Substring(1)                                // skip space
                                    |> WebSocketProtocol.calcWSAccept6455 
                                    |> WebSocketProtocol.createAcceptString6455
                    Console.WriteLine acceptStr                                   // debug
                    do! ns.AsyncWrite <| Encoding.ASCII.GetBytes acceptStr
                    return true
                | _ ->                      
                    tcp.Close()  
                    return false                                                    // validation failed - close connection
            else
                tcp.Close()
                return false   
            
                                                                // validation failed - close connection
            }

                       
        async { 
            while true do 
                let! client = listener.AcceptTcpClientAsync() |> Async.AwaitTask
                let sessionId = Guid.NewGuid()

                let! isWSConnection = handshake client

                if isWSConnection then
                    let clientMailbox = createClientMailboxWS (sessionId) (client.GetStream())                             
                
                    sessionManager.Post(AddSession(sessionId, clientMailbox))                

                    acceptClientWS sessionId client |> Async.Start
        }
        |> Async.Start

```

It's pretty much the same as the original `startServer` function. The main difference is this one is called `startServerWS`... That, and after accepting a TCP connection, it checks to see if this is a WebSocket connection by performing a handshake. Such similarities between `startServer` and `startServerWS` suggest a common function could be extracted, but I did not bother for this first round.

The handshake code is more or less lifted straight from the [gist](https://gist.github.com/jonschoning/14f3a087b2bb79236131) mentioned earlier, as can be inferred from the comments.

Next up, I need a function for handling TCP Clients:

```fsharp
    let acceptClientWS sessionId (client:TcpClient) = async {        
        try

            let loop () = async {
                use stream = client.GetStream()
                let mutable keepRunning = true
                while keepRunning do
                    let bytes = Array.create client.ReceiveBufferSize (byte 0)
                    
                    let! bytesReadCount = stream.ReadAsync (bytes, 0, bytes.Length) |> Async.AwaitTask

                    if bytesReadCount > 0 then                    
                        let frame = WebSocketProtocol.readFrame bytes

                        match frame.opCode with
                        | 8 -> keepRunning <- false
                        | 1 ->
                            //printfn "%A" frame
                            let decodedData = (WebSocketProtocol.decodeMaskedData frame.maskingKey frame.payload) |> System.Text.UTF8Encoding.UTF8.GetString
                            //printfn "Decoded message: %s" (decodedData)

                            let command = WebSocketProtocol.getServerCommand decodedData
            
                            sessionManager.Post(SendCommand(sessionId, command))
                        
                            if command = (Success ServerCommand.Logout) then
                                keepRunning <- false
                        | _ -> ()
                    
                    //do! Async.Sleep 10 |> ignore

                sessionManager.Post(RemoveSession(sessionId))
                client.Close()
            }                                          
            loop () |> Async.Start
            

        with ex -> 
            printfn "ACCEPT ERROR: %s" (ex.ToString())
            sessionManager.Post(RemoveSession(sessionId))
       
    }
```

This one is a bit different to `acceptClient`. It works directly with byte streams compared to `acceptClient`, which makes use of regular `ReadLine` and `WriteLine` functions. But otherwise it's the same: get data from the stream, decode into a `ServerCommand` and post it to the Session Manager mailbox, just with an extra step to translate the byte stream into a Web Socket Frame.

Finally, the client mailbox to handle data written to the TCP stream:

```fsharp
    let createClientMailboxWS (sessionId:Guid) (ns:NetworkStream) =
        let mb = MailboxProcessor.Start(fun inbox ->
            let rec messageLoop () = async {
			
                let! command = inbox.Receive()
                if command <> Stop then
                    let frame =
                        command
                        |> WebSocketProtocol.getClientCommand 
                        |> Encoding.UTF8.GetBytes
                        |> WebSocketProtocol.makeFrame                    

                    let response = 
                        frame
                        |> WebSocketProtocol.writeFrameNew                                           

                    do! ns.AsyncWrite(response, 0, response.Length)
                
                    return! messageLoop()
            }
        
            messageLoop()
        )
        mb.Error.Add(fun ex -> 
            printfn "EXCEPTION: %s" ex.Message
            sessionManager.Post(RemoveSession(sessionId))
            )        
        mb
```

If it isn't clear by now, I just duplicated these three functions and modified them until they worked for Web Sockets, which to be honest, are just glorified TCP streams with an extra packet frame on top of TCP, which is also packet based... Why they couldn't just use TCP itself, is a mystery. I mean, straight from the Mozilla Developer Network:

> The WebSocket API is an advanced technology that makes it possible to open a two-way interactive communication session between the user's browser and a server. With this API, you can send messages to a server and receive event-driven responses without having to poll the server for a reply.

I mean, if that's not what TCP already does... I dunno. Maybe I'm missing something here.

Ahem. Anyway, perhaps in the future I might attempt to consolidate these three functions and make use of HOF to cater for the differences. The hardest part of this was implementing the Web Socket layer itself.

One final difference is the chat protocol between client and server. The original Delphi based client was designed to work with raw TELNET if you really <s>wanted to make your life difficult</s> wanted a challenge. It also made testing the TCP Server easier since I did not need to first make an application, but I digress.

The WebSocket version works differently: I just serialize the `ClientCommand` and `ServerCommand` discriminated unions directly to and from JSON.

```fsharp
module WebSocketProtocol =

    let getClientCommand (command:ClientCommand) = 
        command |> JsonConvert.SerializeObject

    let getServerCommand (data:string): Result<ServerCommand> =
        try
            JsonConvert.DeserializeObject<ServerCommand>(data) 
            |> Success
        with ex -> Failed(ex.Message)
```

Considering that I'd need a WebSocket client aka a JavaScript based application, it didn't make sense to focus on making the protocol easily interacted with by a human, and of course JSON works seamlessly with JavaScript. You might however call it being lazy.

Anyway, I quickly knocked up a Web socket client which you can see below. Here is a TELNET client talking to the web socket client.
![Interchat On the Web](/assets/interchat/fschat3.jpg)

## Source Code

I have actually created a github repository! After a bit of effort this was ported to .NET core so it could be run on anything.

[INTERCHAT F#](https://github.com/bjnmercer/interchat-fsharp)



______________________________________________
Footnotes

<a name="note1"></a>Note #1: Yes, I am ripping off [INTERCAL](https://en.wikipedia.org/wiki/INTERCAL). But for the actual namespace, I just generically named it "FSChat".
