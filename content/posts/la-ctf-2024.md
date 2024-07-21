---
title : "Flag Finder - LA CTF 2024"
date : 2024-02-18
slug : la-ctf-2024
aliases:
    - la-ctf-2024
description: The intended solution for my LA CTF 2024 challenge
summary: Quick writeup covering the details of my LA CTF 2024 challenge, "Flag Finder"
tags:
    - game-hacking
    - rev
    - la-ctf

---

Flag Finder is a challenge I wrote for LA CTF 2024! Despite being one of the main organizers for the event last year, I mostly stuck to logistics and design which is why I'm pretty excited to have written something this year.

### Me Rambling (Life Story in Front of a Recipe)
The idea for this challenge comes from me being very into the Undertale community back in 2016. Upon start-up of a new file of Undertale, the "fun" value in your Undertale save folder will be a random number from 1-100 - where certain "fun" values trigger certain events. Since certain cool events are locked behind very specific "fun" values, it wasn't uncommon for fans of the game to manipulate their savefiles to trigger these events.

Since Undertale is made in GameMaker Studio 2, I wanted to create a very basic "Save-Load" functionality where you can manipulate your own savefile to access an item that is normally inaccessible.

## Intended Solution
Determine this is a Gamemaker Studio 2 game and use the [Undertale Mod Tool](https://github.com/krzys-h/UndertaleModTool) to decompile the GML bytecode (located in either `data.win`, `game.ios`, or `game.unx` depending on your distro)

Utilize the "Save/Load" function inside of the `FlagFinder` game to generate the `mysave.txt` file inside the directory GameMaker has decided to generate files in (for Windows, it's `C:\Users\<user>\Appdata\Local\FlagFinder`).

This is a sample savefile (contains player location and the current item they are holding):
```
{ "player_coordinates": { "x": 1160.798095703125, "y": 621.602294921875 }, "item": 100062.0 }
```

Using Undertale Mod Tool, locate the main room of the game and extract the instance number of the "Key" object. Then, replace `"item": <id>` in `mysave.txt` with the instance number you found.
![image](https://hackmd.io/_uploads/SJ_YgbehT.png)
(if you double-click on the instance you'll see it corresponds to a "key" sprite)

Load your manipulated savefile and give the "Key" object to the "Flag" character
![image](https://hackmd.io/_uploads/S17NbZgnp.png)
![image](https://hackmd.io/_uploads/rJ1HbZenp.png)
and you're done!

## Some other Solutions
This challenge is fairly open-ended by nature so there are several different solutions besides my own. I'll briefly list some (with screenshots):

1. You can locate the code the "Flag" object calls to draw the flag on the screen within UndertaleModTools
![image](https://hackmd.io/_uploads/rJlNrWg3p.png)
You can then use something like Python turtle or p5js to draw these lines manually and extract the flag.
3. You can also manipulate the item accepted by the "Flag" object by locating its "PreCreate" script and changing the value of myItem to that of an item on-screen (normally belonging to another NPC)
![image](https://hackmd.io/_uploads/r1rQIWg26.png)


## Final Details + Source
You can find the source to the game [here](https://drive.google.com/drive/folders/1XHLMgQZbM0f3BUdG7ZjLrIx--OD1xyJZ?usp=drive_link) (too big for Github) if you want to take a look around.

TLDR: all roads lead to Undertale
