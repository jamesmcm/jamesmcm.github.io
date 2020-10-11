---
layout: post
title: "What should I program?"
titleen: "What should I program?"
titlesp: "What should I program?"
date: 2020-10-11 10:59:37 +0200
comments: true
categories: [General]
meta: false
---

Doing projects is often the best way to get more programming experience
and learn new concepts and problem domains. However, a common issue for
beginner and intermediate programmers is finding feasible projects of a
reasonable scope that can be completed in a few weekends.

In this post I will list some classic project ideas for beginner and
intermediate programmers (most of which should be achievable in 2-3
weekends), along with many other project ideas I've had but have never
had time to implement (and a few which I did!).

Within each section the projects are ordered in ascending difficulty /
time investment.

<!--more-->

Opinions expressed are solely my own and do not express the views or opinions of my employer.

# Classic projects

In this section I list some classic project ideas, in ascending order of
time investment / difficulty. I'd recommend these to all new
programmers.

## Sudoku solver

Since a [Sudoku](https://en.wikipedia.org/wiki/Sudoku) grid is only 9x9, it is feasible to just use Depth First
Search and check if the grid is still valid at each step (backtracking
when it isn't).

This approach means this is a very simple problem which can be implemented in
less than an hour, and could be good experience if you are new to graph
search algorithms.

Here is [my implementation in Scala](https://github.com/jamesmcm/scala-sudoku-solver/blob/master/src/main/scala/sudoku/Main.scala) (this was my first Scala
program).

To make it more interesting, you could solve it using [simulated
annealing](https://www.adrian.idv.hk/2019-01-30-simanneal/) or
[other methods](https://www.researchgate.net/publication/256087959_Finding_Solutions_to_Sudoku_Puzzles_Using_Human_Intuitive_Heuristics).

## TCP client/server or proxy

Writing a TCP client and server, or a TCP proxy server, will force you
to work with multithreading or asynchronous programming (or both - see
[my previous post on asynchronous programming](http://jamesmcm.github.io/blog/2020/05/06/a-practical-introduction-to-async-programming-in-rust/#en) 
if you are unsure of the difference).

I recently wrote a [simple synchronous TCP proxy server](https://github.com/jamesmcm/basic_tcp_proxy) in Rust for
use in [vopono](https://github.com/jamesmcm/vopono) without adding a 
lot of async dependencies. You could also write a
[simple file transfer program](https://github.com/dpohanlon/Transfer).

## CHIP-8 emulator

[CHIP-8](https://en.wikipedia.org/wiki/CHIP-8) is an interpreted
programming language, that was used on several microcomputers in the
late 1970s. 

The language is effectively a high-level instruction set,
where the opcodes correspond to different instructions from low-level
mathematical operations to high-level operations like drawing on the
screen.

I highly recommend reading 
[Cowgod's Chip-8 Technical Reference](http://devernay.free.fr/hacks/chip8/C8TECH10.HTM)
if you choose to do this project. I wrote [an implementation in Go](https://github.com/jamesmcm/chip8go)
which includes the documentation of the opcodes and some issues which I hit 
mentioned in the README.

If you enjoy building an emulator/Virtual Machine I'd also recommend the 
[Advent of Code 2019](https://adventofcode.com/2019/day/2) intcode problems
(starting on day 2) which involve building a VM to execute the intcode instructions.

Then if you really want a greater challenge you could try implementing a 
NES or Gameboy emulator (also check out [Reddit's /r/EmuDev](https://www.reddit.com/r/EmuDev/)). The [NesDev Wiki](http://wiki.nesdev.com/w/index.php/Nesdev_Wiki)
is a great resource for the former, combined with [OneLoneCoder's video
series](https://www.youtube.com/watch?v=nViZg02IMQo&list=PLrOv9FMX8xJHqMvSGB_9G9nZZ_4IgteYf). 
For the Gameboy the [gbdev community](https://github.com/gbdev/awesome-gbdev) 
has a lot of comprehensive documentation. Here is [a Gameboy emulator
compiled to WebAssembly](https://github.com/BlueBlazin/gbemu) for
example.

A great advanced project idea would be a SNES emulator compiled to WebAssembly
that could play peer-to-peer multiplayer over the internet by using
WebRTC and verifying that both players have the same ROM. The (possibly
copyrighted) ROM itself would not need to be sent over the network.

## Nand2Tetris

The [Nand2Tetris course](https://www.nand2tetris.org/) covers everything
from building an [Arithmetic Logic Unit](https://en.wikipedia.org/wiki/Arithmetic_logic_unit) using a simple Hardware
Description Language in the [first course](https://www.coursera.org/learn/build-a-computer), to building an assembler for a
simple instruction set and a compiler for a simple Java-inspired language
in the [second course](https://www.coursera.org/learn/nand2tetris2).

This is the best course I have ever taken, and I wholeheartedly
recommend it to anyone who hasn't completed it already. It is a very
hands-on and fun computer architecture course with the basics of
compilers and operating systems development included too. If you do try
this course, I've uploaded the [KeepUp game I made for Week 8 to Github](https://github.com/jamesmcm/HackKeepUp).

If you find the course interesting, for computer architecture projects
(in addition to the emulator ones mentioned above), you could try
[Project Oberon](http://www.projectoberon.com/), for a simpler project
there is also the [Write your own Virtual Machine](https://justinmeiners.github.io/lc3-vm/) tutorial.

For compilers and programming language development, you might be
interested in reading [Crafting Interpreters](https://craftinginterpreters.com/),
[Writing an interpreter in Go](https://interpreterbook.com/),
[Writing a compiler in Go](https://compilerbook.com/) and the 
[Create your own programming language with Rust](https://createlang.rs/)
online book.

For Operating Systems development, the [osdev Wiki](https://wiki.osdev.org/Expanded_Main_Page)
has a lot of comprehensive resources. There is also the 
[Writing an OS in Rust](https://os.phil-opp.com/) blog series, and the
rust-embedded group's [Rust RaspberryPi OS tutorials](https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials).

[Charles Petzold's Code](https://www.amazon.com/Code-Language-Computer-Hardware-Software/dp/0735611319)
is a lighter and slightly dated book covering similar areas in computer
architecture, which I'd still recommend reading as a companion book and
some appreciation of the history (just like [Linus Torvald's biography](https://www.amazon.com/Just-Fun-Story-Accidental-Revolutionary/dp/0066620732/) is also
good for a very light read about the history of Linux).

Finally, since this entry is actually a course, I also wanted to
recommend other excellent online courses I have completed:

* Andrew Ng's famous [Machine Learning course](https://www.coursera.org/learn/machine-learning) (even if it is slightly
  dated still using Octave now).
* Stanford's [Mining Massive Datasets](https://online.stanford.edu/courses/soe-ycs0007-mining-massive-data-sets)
   ([and the book here](http://www.mmds.org/)) covering different algorithms for processing large datasets.
* Dan Boneh's [Cryptography](https://www.coursera.org/learn/crypto) course
  ([and the book here](http://toc.cryptobook.us/)) - I'd also recommend completing
  the [cryptopals problems](https://cryptopals.com/) if you find this interesting.
* Stanford's [Algorithms](https://www.coursera.org/specializations/algorithms)
  ([and the book here](http://www.algorithmsilluminated.org/)) course covering
  popular algorithms and data structures.

## Ray Tracer

Implementing a [Ray Tracer](https://en.wikipedia.org/wiki/Ray_tracing_(graphics\)) is a great project for producing a visible
result that you can share with others. I highly recommend [The Ray
Tracer Challenge book](https://pragprog.com/titles/jbtracer/the-ray-tracer-challenge/)
(and don't forget [the bonus chapters](http://www.raytracerchallenge.com/#bonus) at the end!).
The book uses Test-Driven Development and does not prescribe a specific
programming language.

I implemented the [project in Scala](https://github.com/jamesmcm/raytracer_challenge_scala).
If I were to do it again, I'd recommend making sure your primitives are
as fast as possible (i.e. use the appropriate linear algebra libraries)
to ensure better performance later on. I'd also recommend implementing
it such that it can compile to WebAssembly or JavaScript - e.g. with [wgpu in
Rust](https://sotrh.github.io/learn-wgpu/) so that it is easy to show to
others whilst keeping good performance.

There is also the [Ray Tracing in One Weekend](https://raytracing.github.io/)
for an alternative resource aimed at C++.

Graphics programming is a very deep rabbit hole, if you find the Ray
Tracer interesting I'd definitely recommend Bisqwit's YouTube channel
with videos on [DOOM-style rendering](https://www.youtube.com/watch?v=HQYsFshbkYw),
[Polygon Rasterisation](https://www.youtube.com/watch?v=PahbNFypubE) and 
[Illumination](https://www.youtube.com/watch?v=Nwfm6cpskIM) mostly in C
and C++.

# My project ideas

The above project ideas are very popular and almost a rite of passage
for new programmers now. In this section I will present many different
project ideas I've had but have not had time to fully implement (or even
begin!).

As in the previous section, projects are sorted by approximate difficulty in
ascending order.

## Scrabble solver

[Scrabble](https://en.wikipedia.org/wiki/Scrabble) / Words With Friends 
is a popular word game where you must form the best words from your character
tiles to score the highest points against your opponent.

I wrote a simple [Scrabble solver in Scala](https://github.com/jamesmcm/scala-scrabble-solver)
which uses a more-or-less brute force approach of scoring every possible
word, using [regular expressions](https://en.wikipedia.org/wiki/Regular_expression)
to find possibilities from the wordlist for each
possible board position. The main issue is that it can be very slow for
complicated boards (i.e. near late-game) taking ~3-4 minutes to produce
the best solution.

I'd really like to re-implement this in Rust so it could be built for
WebAssembly. Note that in WebAssembly multiple threads are not yet
supported in most browsers, so the solution would need to be
single-threaded. This also makes the performance very important and it
would be necessary to either improve the algorithm if possible (maybe
some sort of [Trie structure](https://en.wikipedia.org/wiki/Trie) could be used instead of separate
regular expressions) or use heuristics to discard many positions/moves
so they don't all need to be evaluated.

But remember that the board bonuses can mean that score isn't directly
correlated with word length. Also you may often have the 3rd and 5th
letter of a word constrained but not the first letter for example, so
the data structure for the wordlist would need to account for that.

## Crusader Kings 3 save-game file analyzer

[Crusader Kings 3](https://store.steampowered.com/app/1158310/Crusader_Kings_III/) 
is an excellent Grand Strategy game available on GNU/Linux. Overall, the
game is a great improvement to its predecessor, however one feature that
it lacks is the "Chronicle" which, in Crusader Kings 2, documented the
history of your dynasty and their rise to power.

It'd be great to build a WebAssembly based tool that could parse CK3
save game files and show the history of the game (using the title owner
change dates and their respective dynasties), the trees of different
characters' dynasties, and graphs for the number of alive dynasty
members over time, and other interesting metrics.

Whilst running the game in debug
mode - without Ironman mode so achivements are disabled, it is possible
to export a save game in JSON format that is easy to parse. However,
for Ironman saves in the normal game (the most common scenario for
players unlocking achievements) the savegame is serialised in a
different, binary format.

The idea would be to first handle deserialising the relevant parts of
the JSON debug saves to useful structs, and then build this to
WebAssembly with a frontend framework (e.g. [Seed](https://seed-rs.org/))
and some JavaScript (e.g. [plotly.js](https://github.com/plotly/plotly.js)) 
to create the visualisations.

Once something is useful for the debug saves, then a deserialiser for
the Ironman saves could be created. From my brief analysis of the save
files, it seems the fields are in the same order as in the debug saves
and I was already able to extract character IDs, names and faiths for
example.

The end result could be something like [Rakaly](https://rakaly.com/eu4)
(note the [save game analysis crate is FOSS](https://github.com/rakaly/eu4save))
is for EU4 saves ([also written in Rust](https://nickb.dev/blog/my-bet-on-rust-has-been-vindicated))
but focussed on the Chronicle functionality. Rakaly has already dealt
with the Ironman issue for EU4 saves too, so it is not insurmountable.


## Using an Xbox One controller as a MIDI synthesizer

It would be cool to be able to use an Xbox One controller to create
music with a software [MIDI](https://en.wikipedia.org/wiki/MIDI) synthesizer.
I found two repositories on Github doing this already:
[midibox360](https://github.com/quotepilgrim/midibox360) and 
[xbox-midi-controller](https://github.com/maxmechanic/xbox-midi-controller).
I tried out midixbox360 but the latency was too high to be usable so I
tried my own test in Rust.

* Not Bluetooth audio
* JACK
* Realtime Linux kernel

## Alexa skill for Reddit usage
Able to read and post comments, navigate subreddits and comment trees
Authentication issues
No free narration - cannot post comments?
Can host state machine in AWS Lambda

## ALMA docker support + ALMA Hub

## Linux CLI Hex editor
neovim plugin?
Like ghex show highlighted text in both text and bytes
support UTF-8 conversion (i.e. valid UTF-8 can show in text window, need
to handle multiple bytes -> 1 UTF char mapping)
support little-endian and big-endian conversions
support search by decimal values (converting to big-endian or
little-endian)
Copy from bytes or text

## Linux packet editor
like Winsock Packet Editor (WPE)
need to MITM TLS
How to get application specific packets - namespace + nftables?
memory editor - GameConqueror

## Joust Battle Royale
Real-time web assembly game
Browsers TCP only
multiplayer puzzle game post

## Google Keep alternative
Support org-mode style markdown for notes
Markdown editor online when editing notes

## AI for Democracy 3
Equations relating variables in game files
Need to simulate outside of the game?
Use Q-learning and simulation to learn policy

## FOSS engine for Netstorm
Online play from Linux
Server improvements

## Writing your own database
Github repo

## Conversion Optimisation platform
Use multi-armed bandit models
Decide which content alternative to show in real-time by polling the
model
Click or conversion can be "success event"
Bayesian hackers book

# Completed projects

## s3rename
Needed to restructure s3 directories to use AWS Glue
Previously generated awscli commands with Python script - slow and
awkward

## vopono
Originally used bash scripts for OpenVPN only

# Summary

Find something creative and enjoyable which can be useful in a few
weekends

Beginner: Leetcode, Project Rosalind, Advent Of Code, Project Euler

IRC Client/Server - now Matrix?
Email client
Simple Web Browser

OneLoneCoder community

Free Software, Free Society
