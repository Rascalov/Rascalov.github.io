---
layout: post
title:  "Game Hacking in Rome Total War"
date:   2020-10-29 09:59:21 +0200
toc: true
tags:
 - Cheat-Engine
 - Rome-Total-War
 - Gaming
---

# Introduction
Fucking around with Cheat Engine back in the day was fun. Find a value, change it,
enjoy some easy cheating. I never saw cheat engine as more than that.

But then I decided to follow the cheat engine tutorial. There's a fuck ton of stuff
I didn't know about. So as I am writing this, I will use this post to document my 
progress in creating cheating software for Rome: Total War 1. 

# Wtf is Rome total war?
Old ass strategy game that I used to play, play as a faction in the time of Rome 
and conquer the world (europe, middle east + northern Africa).

Rome total war 1 (RTW1) had a cheating command line called "Romeshell". In it you
could give yourself money, auto-complete the building of structures in your settlements, 
give your generals perfect stats, and a lot more. 

## So, it's old and already has cheats? Why bother making more? 

Should be noted: old != useless endeavor. 

As for the existing cheats, there are some issues:
* Very few commands actually work
* You write them line by line, which makes cheating very slow
* Some cheats require names of generals, which often don't work because multiple generals can have the same name.

For the 2nd issue, I created a [program that makes cheating easier typing wise](https://github.com/Rascalov/RomeTotalWarRomeShellLazyMode).
But it still is tiresome to switch between programs and does not solve issue 1 and 3.

To fix these issues, I need to create a program that overlays itself on top of the game.
It needs to access the memory and alter it on that level.


# What should the program do?
I have a list, some are dreams, some are doable.

* Change stats of target unit/general/settlement.
* Allow control of other factions during the same campaign.
* Create units anywhere on the map and as a macro (create, move away and repeat).

## Change stats of target unit/general/settlement.
The target can be of any faction. 

# Research time
Worst part, I have less than zero experience with memory, pointer and all the other shit. 
Let alone do I have the capability to produce an overlay to compliment the backend. 

From what I gather/imagine, RTW1 is written in C++. So an overlay program should match that at least. 

## Cheat engine
I will post the results I get from fucking around on RTW1 in Cheat Engine.

### Stat stuff
Movement points are stored in a `float` for each `Unit`. Units in a `squad` have collective movement points
stored in a different `float`. 

`Command`, `Management`, `subterfuge`, and `influence` are all stored in a 4 byte int value.
`Management` had one other address with a corresponding value, 
but since the other 3 do not have such values, it could just be a fluke. 

Usually, generals and other unit leaders get these values increased with `traits`. 
A `trait` sometimes offers more than just those 4 stats. 

## C and shit