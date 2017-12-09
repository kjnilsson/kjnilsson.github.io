---
title: "Happy Fezmas (or Getting started with Fez)"
date: 2017-12-09T13:11:51Z
draft: false
---

Fez has always had an affinity with Father Christmas (some call him Santa this is obviously not correct). Why I hear you ask? Well one of them wears a red hat and one of them _is_ a red hat. They are very close. Hence this blog post is coming incredibly easy for me (Karl) and not at all something I just I forced myself to write whilst jet-lagging.

You may by now assume that `fez` isn't just a hat from the Moroccan city of the same name. Not at all. Fez is also an F# to (core) erlang compiler! In short fez takes F# code (or rather a subset of F# code) and translates it to [core erlang](https://www.it.uu.se/research/group/hipe/cerl/doc/core_erlang-1.0.3.pdf) then compiles it to beam using the standard erlang (erlc) compiler. Some people may refer to this as "transpiling" - I am on the other hand pretty sure that is not a word or at least not a word with a clear definition so will never use it.

This blog will guide you through getting started with `fez` ( the version available at the time of writing is `0.1.0-alpha.2`).

### Fezzing up

To get started download the latest version available for your platform on the github [releases](RELEASES) page. Unzip the release and optionally add the directory to you PATH. All commands in this entry assumes the fez release was downloaded to the same directory as our project.

Create a directory called `fezmaz` and unzip the release into the same folder. Inside the release directory (which is `osx-x64` for me - replace as required)  there should be a `fez` or `fez.exe` binary. This is `fez`. Say hi `fez`. Run:


```
./osx-x64/fez help

```

To see the list of currently available commands:

```

USAGE:
    fez init
        Creates a dotnet core 2.0.0 fsharp project file in the current directory
        with a reference to the Fez.Core library. Requires dotnet to be installed
        and available in PATH.

    fez compile <options> <files>
        Compiles fsharp files to core erlang and beam.

        OPTIONS:
            -o | --output       set output directory
            --nobeam            do not compile core files to beam

    fez erl <args>
        Launches the erlang shell with the appropriate search paths set.

    fez help
        Shows this text.
```


First we are going to initialize the current directory with `fez init`. This step requires [dotnet core](https://www.microsoft.com/net/learn/get-started) to be installed and available.

```
./osx-x64/fez init
```

This step is optional as `fez` does not actually require .NET core to compile code. This step is only so that existing editor tooling (vim, emacs, ionide) works better with the `fez` dependencies.

Now we should have an `fezmaz.fsproj` and `code.fs` in our current directory. Open up the `code.fs` file in your [favourite editor](http://www.vim.org/).


Type the following code in:


```
module xmas
open Fez.Core


let greet who =
    printf "Hi %s" who
```

Compile it using:

```
./osx-x64/fez compile code.fs

```

Then run

```
./osx-x64/fez erl

```

To start an erlang shell with the appropriate fez search paths in place. You should be greeted wilike the following:


```
Erlang/OTP 20 [erts-9.1.5] [source] [64-bit] [smp:8:8] [ds:8:8:10] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Eshell V9.1.5  (abort with ^G)
2>
```

We can now call our `greet` function using erlang syntax:

```
1> xmas:greet("Fez").
Hi Fez
ok
2>
```

And there that's it. A simple, sequential Hello World done effectively. Which isn't very interesting.

You may have heard that the erlang programming model is very much about message passing between processes and that is true and `fez` support these patterns and we should make a simple program that highlight how these work.


So we will write a program that spawns a Father Christmas process that has got an [ETS](http://erlang.org/doc/man/ets.html) lookup table of people and whether they have been good or bad this year. We can then send a wishlist to Father Christmas and possibly receive a present back.

To achieve this we first have to interop with native erlang modules. This is done using a dummy function ( of any name) decorated with the `ModCall(module, function)` attribute.
The first thing to do is to create the new table using `ets:new/2`. Looking at the erlang documentation we can see that `ets:new/2` takes a Name that needs to be an atom and a list of options, many of which are atoms as well. F# doesn't have atoms (Can we have atoms Don? For Christmas?). In erlang atoms are either used as stand alone values or often as the tag in a tagged tuple. in short they're used somewhat similarly to discriminated unions and as it happens that is exactly what is used in `fez` to represent atoms.

The erlang code we'd like to be able to call would be something like:

```
ets:new(good_bad, [named_table, protected])

```

This means we need a DU to represent name of the table like this:

```
[<ErlangTerm>]
type EtsName = | Good_bad
```

Decorating the type with the `ErlangTerm` turns a DU case without any data into an atom. It will lower-case the first character as all atoms in erlang need to begin with a lower case and all cases in F# DUs need to begin with an upper case.

We also need to represent the two option atoms:

```
[<ErlangTerm>]
type EtsOption =
    | Named_table
    | Protected
```

Good. Looking at the docs we see that `ets:new/2` returns a `tid()`. A tid is a table identifier and is effectively opaque to the user so we can simply create a single case DU to represent it:

```
type Tid = Tid
```



Jolly good. That is quite a few types for just calling one function! Ideally at some later release of `fez` we'd ship a shim module inside `Fez.Core` for common erlang modules like `ets` but for now this is the interop story. Once you've got the types right it should be straight-forward to program accurately against it.

Ok so we also need to be able to insert into the table as well as lookup. Here is the types and functions for that.

Flicking back to the docs we see that the return type of `ets:insert/2` is simply the
atom `true`. So again we have to create a DU just for this value. We could have used a bool but there are no cases where the function would return `false` so it is better to use a more accurate representation.

We also create another erlang term to represent the tuple that is used for the key/value data. We have to wrap the tuple in a DU and annotate it with `ErlangTerm`. This is due toa bug in the fez compiler that hopefully should be fixed in the next version allowing us to simply use a tuple instead.

```
[<ErlangTerm>]
type EtsInsertRrt = | True

[<ErlangTerm>]
type GoodBadRecord = | T of string * bool

[<ModCall("ets", "insert")>]
let etsInsert (tab: Tid) (o: GoodBadRecord) = True

[<ModCall("ets", "lookup")>]
let etsLookup (tab: Tid) (key: string) : GoodBadRecord list  = []
```

Next up we need some code to seed the table with some interesting data. We also need to interop with the `random` module to randomly select who has been good and who has been bad:

```
[<ModCall("random", "uniform")>]
let randomUniform (n: int) : int = 1

let randBool () =
    match randomUniform 2 with
    | 1 -> true
    | _ -> false

let seed () =
    let t = etsNew Good_bad [Named_table; Protected]
    let insert name good =
        let True = etsInsert t (T(name, good))
        ()
    randBool() |> insert "Karl"
    randBool() |> insert "Michael"
    randBool() |> insert "Diana"
    randBool() |> insert "Gerhard"
    randBool() |> insert "Jean-Sebastién"
    randBool() |> insert "Luke"
    randBool() |> insert "Daniil"
    randBool() |> insert "Dan"
    randBool() |> insert "Arnaud"
    t
```

The names are arbitrary. Any similarity with my team-members at work is simply accidental. ;)

Now. Now we need to create a long living process representing Father Christmas. The simpest way is to create a recursive function that waits for incoming messages, replies and recurses.

In erlang receive statements are always selective. Only messages that match a pattern match are picked out of the mailbox. In `fez` these selective receive matches are represented by...

You guessed it. Discriminated Unions. Here is the full FC code:

```
type WishlistData =
    { Name : string
      Pid : Pid
      Stuff : string list }

type Wishlist = Wishlist of WishlistData

type MaybePresent =
    | Present of string
    | Nothing


let lookup t who =
    match etsLookup t who with
    | [x] -> Some x
    | _ -> None

let rec fcLoop t =
    match receive<Wishlist>() with
    | Wishlist { Stuff = [] } ->
        printfn "an empty wishlist - how strange..."
        fcLoop t
    | Wishlist { Name = key
                 Pid = pid
                 Stuff = stuff } ->
        match lookup t key with
        | None ->
            printfn "curious - I have no record of this person..."
            fcLoop t
        | Some (T(_, true)) ->
            printfn "%s has been good and will get a present" key
            pid <! (Present <| List.head stuff)
            fcLoop t
        | Some (T(_, false)) ->
            printfn "%s has NOT been good and will get nothing" key
            pid <! Nothing
            fcLoop t

let spawnFc () =
     fun () -> seed() |> fcLoop
     |> spawn
```

The `spawnFc` function first seeds the data table then spawns a process that calls the recursive `fcLoop` function.


All that is left are some functions for sending a wishlist and waiting for replies:

```
let waitForPresent () =
    match receive<MaybePresent>() with
    | Present thing ->
        printfn "yay I got a %s" thing
    | Nothing ->
        printfn ":((((("

let sendWishList fc who stuff =
    let w = Wishlist { Name = who
                       Pid = self()
                       Stuff = stuff}
    fc <! w
    waitForPresent ()
```

With all pieces in place we can compile this again using `fez compile` the run the shell using `fez erl` and have a play.

```
Eshell V9.1.5  (abort with ^G)
1> Fc = xmas:spawnFc().
<0.62.0>
2> xmas:sendWishList(Fc, "Karl", ["synth module", "synth", "book about synths"]).
Karl has been good and will get a present
yay I got a synth module
ok
3> xmas:sendWishList(Fc, "Diana", ["climbing stuff", "bat stuff"]).
Diana has NOT been good and will get nothing
:(((((ok
```

So that's (t)hat.

Laters!









































Happy Fezmas!
