---
layout: post_classic
title:  "Composing a game from lenses"
date:   2025-07-11 20:56:48 -0700
categories: posts
published: false
---

# Seeking a mathematical model of computer games

Almost two decades ago, I worked in the computer game industry as a gameplay programmer. I felt that the languages I used, C++ and Lua, were not suitable for gameplay programming.

There were a few reasons for this. Not only did C++ and Lua lack in-built notions of space and time, but they also lacked general purpose feautures for reasoning about them. For example, I could represent a 3D vector as an object in C++, but the C++ type system had no means to specify which basis (a.k.a. "coordinate system") the vector belonged to, making it easy to perform the invalid operation of adding two vectors belonging to different coordinate systems.

Second, despite game characters dynamically adjustinging their goals in response to stimulus from their environment, C++ and Lua had no in-built notion of things like imperfect perception and goal setting.

It's fallacious to claim that any of the above issues require in-built language support. For example, in most main-stream langauges, we can implement some sort of state machine system for controlling characters, where each state represents a distinct goal that a character is persuing. In a general purpose language, we *could* tag each vector with a basis identifier, and tag each point with its affine frame, dynamically enforcing clean proper usage at runtime. Further, perhaps we could use some sort of indexed type system to enforce proper vector operations statically.

Nonetheless, I think it's worth experimenting with the design of game specific languages. Even if it turns out that the abstractions of general purpose languages are sufficient for game specific needs, our game specific experimentation may clarify exactly what is needed and how it should be structured.

There are a few ways we could go about experimenting with game specific languages:

* Implement a real game specific language that we can compile to executable games. This would require an enormous amount of work. We would evaluate our experiment by using our language to create games. This too would require an enormous amount of work. The iteration time for this approach is simply to lengthy to be an effective form of experimentation.

* Model computer games, or simplified approximations of computer games, as mathematical objects. Then design a language that compiles to such mathematical objects. We evaluate our language by designing a logical system to reason about the game models that it produces. If this logic is expressive, we evaluate the language positively, because it implies that humans can reason about it easily.

The first approach has the advantage of being real, while the second approach provides deeper understanding and faster iteration times. I'm going to persue the second approach.

This article isn't really about designing a language, though. It's about an important prerequisite to the second approach: **Model computer games, or simplified approximations of computer games, as mathematical objects.** I've never created a mathematical model of computer games before, so I'm going to try modelling some of the simplest games I can find that still features space, time, and interacting agents. To this end, I've chosen to model a game creation system called *Megazeux*. I'm going to present my model using elementary mathematics; my goal is that anyone who knows what sets and functions can follow it.

# Megazeux

Before we get into modelling, I'm going describe Megazeux. Then I'll discuss what's at stake by highlighting some issues in a famous Megazeux game called Weirdness.

Megazeux is a game creation system from the 90s whose games take place on a grid of 8x14 pixel images. That's right: in a typical Megazeux game, every significant object, whether a goblin, a wall, or a tree, is depicted using an 8x14 image. This extreme constraint comes from DOS text-mode graphics, where textual documents were displayed in grids of 8x14 pixel characters. While 8x14 might be a reasonable size to depict a single letter of the English alphabet, depicting something more complex, like a human, is much more challenging. As a result, players do not expect the graphics of a Megazeux game to look good. For a game developer, depicting a game world using simple abstract art rather than poring over complex visual details greatly reduces the effort used to bring a game world to life.

Here are a few examples of Megazuex games:

![Image]({{ site.baserul }}/assets/images/gameloop/mzx-depot-dungeons.png)

[Depot Dungeons](https://www.digitalmzx.com/show.php?id=2097) is a puzzle game where the player must traverse a dungeon while solving puzzles involving levers, crate pushing, and the like, while fighting off aggressive mutant cockroaches.

![Image]({{ site.baserul }}/assets/images/gameloop/mzx-kikan-intro.png)

[Kikan](https://www.digitalmzx.com/show.php?id=1539) is a turn based, story driven RPG similar to games in the Final Fantasy series. Unlike most RPGs, it features a real world setting.

## Roy: A Case Study in Megazeux Scripting

It's extraordinary that Megazeux was created by a high school kid, Alexis Janson. In addition to creating Megazeux, she used Megazeux to create a series of action adventure games called Zeux. Then, she created her final Megazeux game, Weirdness. It was a puzzle adventure that pushed the Megazeux scripting system to its limits, featuring complex character behaviors and even a first-person maze.

Creating complex character behaviors in Megazeux is fraught, because Megazeux's scripting language Robotic does not provide high level abstractions such as state machines and behavior trees. A robotic script is associated with a specific programmable game character, idosyncratically referred to as a "robot" in Megazeux, but I'll also refer to a programmable game characters as "agents". A robotic script consists of a set of labels, where each label is followed by a series of commands to execute. Commands can issue robot-relative actions, such as [go north](https://www.digitalmzx.com/mzx_help.html#COMMANDR.HLP___g4) and [wait for 2](https://www.digitalmzx.com/mzx_help.html#COMMANDR.HLP___w1). They can also print temporary text to the screen, as in [* "Hi! I'm a robot!"](https://www.digitalmzx.com/mzx_help.html#COMMANDR.HLP____3), modify global variables, as in [set "tostore" to 0](https://www.digitalmzx.com/mzx_help.html#COMMANDR.HLP___sF), and conditionally (or unconditionally) jump to another label, as in [if "darkness" = 1 then "fboff2"](IF "counter" !<>= # "label")



