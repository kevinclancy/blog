---
layout: post_classic
title:  "Composing a game from lenses"
date:   2025-07-11 20:56:48 -0700
categories: posts
published: false
---

# Seeking a mathematical model of computer games

Almost two decades ago, I worked in the computer game industry as a gameplay programmer. My intuition was that the languages I used, C++ and Lua, were not suitable for gameplay programming.

There were a few reasons for this. First, not only did C++ and Lua lack in-built notions of space and time, but they also lacked general purpose feautures for reasoning about them. For example, I could represent a 3D vector as an object in C++, but the C++ type system had no means to specify which basis (a.k.a. "coordinate system") the vector belonged to, making it easy to perform the invalid operation of adding two vectors belonging to different coordinate systems. Second, despite game characters dynamically adjustinging their goals in response to stimulus from their environment, C++ and Lua had no in-built notion of things like imperfect perception and goal setting.

It's fallacious to claim that any of the above features are better off built in to a language instead of implemented using general purpose features. In most mainstream langauges, we can implement some sort of state machine system for controlling game characters, where each state represents a distinct goal that a character is persuing. We *could* tag each vector with a basis identifier, and tag each point with its affine frame, dynamically enforcing proper usage at runtime. Further, perhaps we could use some sort of indexed type system to enforce proper vector operations statically, though I'm not sure if any mainstream languages are capable of this.

Nonetheless, I think it's worth experimenting with the design of game specific languages. Even if it turns out that the abstractions of general purpose languages are sufficient for game specific needs, our game specific experimentation may clarify exactly what is needed and how it should be structured.

There are a few ways we could go about experimenting with game specific languages:

* Implement a real game specific language that compiles to executable games. This would require an enormous amount of work. We would evaluate our experiment by using our language to create games. This too would require an enormous amount of work. The iteration time for this approach is simply too lengthy to be an effective form of experimentation.

* Model computer games, or simplified approximations of computer games, as mathematical objects. Then design a language that compiles to such mathematical objects. We evaluate our language by designing a logical system to reason about the game models. If this logic is expressive, we evaluate the language positively, because it implies that humans can reason about it easily.

The first approach has the advantage of being real, while the second approach provides deeper understanding and faster iteration times. In this post, I'm going to persue the first step toward the second approach: **Model computer games, or simplified approximations of computer games, as mathematical objects.** I've never created mathematical models of computer games before, so I'm going to try modelling some of the simplest games I can find that still features space, time, and interacting agents. To this end, I've chosen to use a game creation system called *MegaZeux* as an inspiration for my model. I will present my model using elementary mathematics; my goal is that anyone who knows what sets and functions are can follow it.

# Understanding the problem

Before we get into modelling, I'm going describe Megazeux. Then I will identify some fundamental features of a gameplay programming by exploring a concrete scenario in a Megazeux game called *Weirdness*. Finally, I'll discuss what's at stake by highlighting a bug in *Weirdness*.

## MegaZeux

*MegaZeux* is a game creation system from the 90s, whose games take place on a grid of 8x14 pixel images. That's right: in a typical Megazeux game, every significant object, whether a goblin, a wall, or a tree, is depicted using an 8x14 image. This extreme constraint comes from DOS text-mode graphics, where textual documents were displayed in grids of 8x14 pixel characters. While 8x14 might be a reasonable size to depict a single letter of the English alphabet, depicting something more complex, like a human, is much more challenging. As a result, players do not expect the graphics of a Megazeux game to look good. For a game developer, depicting a game world using simple abstract art rather than poring over complex visual details greatly reduces the effort used to bring a game world to life.

In Megazeux, scriptable game characters are, idosyncratically, called *robots*. However, we will often refer to them as agents. Each robots/agent is controlled by a *robotic script*. Because robotic scripting is not the state of the art in game character scripting, I will intentionally avoid discussing the details of how it works.

Here are a few examples of Megazuex games:

![Image]({{ site.baserul }}/assets/images/gameloop/mzx-depot-dungeons.png)

[Depot Dungeons](https://www.digitalmzx.com/show.php?id=2097) is a puzzle game where the player must traverse a dungeon while solving puzzles involving levers, crate pushing, and the like, while fighting off aggressive mutant cockroaches.

![Image]({{ site.baserul }}/assets/images/gameloop/mzx-kikan-intro.png)

[Kikan](https://www.digitalmzx.com/show.php?id=1539) is a turn based, story driven RPG similar to games in the Final Fantasy series. Unlike most RPGs, it features a real world setting.

## Roy: A Typical Gameplay Scenario

It's extraordinary that Megazeux was created by a high school kid, Alexis Janson. In addition to creating *MegaZeux*, she used *MegaZeux* to create the *Zeux* series of action adventure games. Then, she created her final *MegaZeux* game, *Weirdness*. *Weirdness* is a puzzle adventure that pushes *MegaZeux* scripting system to its limits, featuring complex character behaviors and even a first-person maze.

A typical game scripting scenario can be found at the beginning of *Weirdness*. A character named Roy sits at his computer in his house. If the player touches him, he says "Not now, Jace, I'm busy making UltraZeux games".

![Image]({{ site.baserul }}/assets/images/gameloop/mzx-weirdness-roy-computer.png)

If the player goes into the basement, he can turn off the fuse box, causing the lights in the house to shut off and destorying Roy's UltraZeux work.

![Image]({{ site.baserul }}/assets/images/gameloop/mzx-weirdness-fusebox-off.png)

Roy then walks into the basement and turns the fuse box back on.

![Image]({{ site.baserul }}/assets/images/gameloop/mzx-weirdness-fusebox-on.png)

In this scenario, the fuse box itself is a robot. When the player attempts to move into the grid cell occupied the fuse box, the game triggers the fuse box script's "touch" handler, which spawns a dialog asking the player if they want to turn the fuse box off. If they say "yes" then another handler inside the fuse box's script is executed. This handler sends a "fuse box off" message to Roy. Upon receiving the "fuse box off" message, Roy's robotic script causes him to expresses his frustration and walk into the basement to turn the fuse box back on.

Let's take a step back and think about what's going on here. *MegaZeux* is simulating the physical world in a rough manner. The robots in this scenario represent physical things in the world. Robotic scripts are an abstraction of the physical things' "brains": both actual brains, as in the case of Roy, and pseudo brains, as in the case of the fuse box. The pseudo brain of an inanimate object is its physical structure, which dictates how it responds various stimuli.

A robotic script issues commands instructing its robot to interact with the world in various ways. For example, `go SOUTH` instructs the robot Roy to attempt walking to the grid cell whose $$y$$-coordinate is one greater than Roy's current $$y$$-coordinate. This will only happen if the cell to the south of Roy is currently unoccupied. For this reason, we can view commands such as `go SOUTH` as a message sent from Roy's brain to the physical world, which we will call the *environment*. Upon receiving the message, the environment may or may not choose to move Roy, depending on whether such movement is compatible with the laws of physics, which are *dictated* by the environment.

From this discussion, we extract some key features we would like to model.
* Agents (in *MegaZeux* terminology, robotic scripts) that affect the physical world by making requests to an environment
* One type of request an agent can make is sending a message to another agent
* Agents receive messages from the environment and other agents; these message may affect their behavior

# Computer games as dynamical systems

## Basic structure

Now that we've highlighted the features that we wish to focus on modelling by taking inspiration from *MegaZeux* and *Weirdness*, let's consider the structure of a computer game.

![Image]({{ site.baserul }}/assets/images/gameloop/game-system.drawio.png)

We can view the game's state as encapsulated in the game stepper; nothing outside of the game stepper may modify it. The game evolves over a sequence of discrete time steps. At each time step, the game stepper must compute two quantities. From the game's current state, the game stepper must compute its output, a matrix of color values to display to the screen. The game stepper must also compute the game's next state from its input and its current state.  Its input might be a mapping from key identifiers to booleans indicating whether each key is pressed. The game's state might contain a matrix of physical locations, where each location contains an identifier denoting either a game character, a wall, or an empty space.

Computing the quantity to send out of the Output channel is often called computer graphics. From the discussion above, you may have inferred that graphics is not what I'm interested in, nor am I interested in processing the input. So the perspective I'd like to take on computer games actually looks more like this:

![Image]({{ site.baserul }}/assets/images/gameloop/game-closed-system.drawio.png)

Without input or output, what we have is more of a closed simulation than a game. Now, at each time step, the game stepper no longer computes an output. It still must compute a next state. But because it no longer receives input, only its current state is used to compute its next state. So this diagram depicts a function from game states to game states.

A box doesn't seem very interesting as a diagram. It becomes interesting when we compose it from stateful subcomponents. Each subcomponent is a dynamical system that transforms zero or more inputs and an internal state into zero or more outputs over a sequence a time steps.

TODO: maybe create a final diagram here, with components for Roy, the Environment, etc. inside of a container called "Game Stepper".

## Dynamical systems

TODO: the stuff below is bad because it gets into too much detail without being mathematical

We will be using *wiring diagrams* such as the one above for the rest of this article. Wiring diagrams
represent dynamical systems. A dynamical system consists of stateful components, which we depict as boxes, and transmission channels, which we depict as lines.

Each transmission channel and stateful component has an associated datatype. For channels, it's the type of data transmitted along the channel. For the stateful component, it's the type data stored as state. Textual labels may explicitly specify these datatypes, or they may hint at them as in the diagram above. Transmission channels connected to the left side of a stateful component are considered its inputs, while transmission channels connected to the right side are considered its outputs.

A dynamical system evolves over a sequence of discrete time steps. At each time step, a component's input channels carry data, and the component itself carries state data from the previous timestep. The component uses its current state to compute data to send along its output channels, and it uses both its current state and its input data to compute its next state.

In the diagram above, the datatype corresponding to the ``Input'' channel may consists of a mapping from key identifiers to booleans indicating whether or not each key is pressed. It may also contain a floating point number representing the amount of passed since the last time step. We could even transmit these two values along two different input channels. The output channel typically transmits a matrix of color values to display to a computer monitor.


<details>
<Summary>Roy's Robotic Script (Click to expand)</Summary>
<p style="white-space:pre-line;background-color:lightgray;">: "do"
set "loopcount" to random 140 to 142
char "loopcount"
wait for 2
goto "do"
: "touch"
* "~5Roy: Not now, Jace, I'm busy making UltraZeux games."
send "msg" to "fad"
goto "do"
: "fboff"
lockself
char 'ç'
wait for 25
* "~5Roy: $%#&&*! I haven't saved my game yet!"
send "msg" to "fad"
: "fboff2"
wait for 45
char 'á'
cycle 2
* "~5Roy: I'd better go check the fuse box..."
send "msg" to "fad"
go SOUTH for 3
: "mv1r"
if touching SOUTH then "mv1"
go SOUTH for 1
go WEST for 2
: "mv2r"
if touching WEST then "mv2"
go WEST for 1
: "dr1r"
if c?? Space p?? at WEST then "dr1"
if touching WEST then "mvx5"
open at WEST
wait for 1
color c8e
go WEST for 4
go SOUTH for 8
rel to self
: "mv4r"
if player at 222 -12 "mv4"
rel to self
gotoxy 222 -12
go SOUTH for 7
go WEST for 12
: "dr2r"
if c?? Space p?? at NORTH then "dr2"
if touching NORTH then "mvx6"
open at NORTH
wait for 1
go NORTH for 4
go EAST for 6
: "mv5r"
if touching EAST then "mv5"
go EAST for 1
rel to self
: "mv6r"
if player at 74 0 "mv6"
rel to self
gotoxy 74 0
go WEST for 11
: "mv7r"
if touching WEST then "mv7"
go WEST for 1
wait for 10
if "darkness" = 0 then "noprob"
if "wire" = 1 then "nowire"
* "~5Roy: Ahh, here's the problem."
send "msg" to "fad"
wait for 10
send at WEST to "fix"
wait for 5
: "noprobr"
go EAST for 11
: "mv8r"
if touching EAST then "mv8"
go EAST for 1
rel to self
: "mv9r"
if player at -74 0 "mv9"
rel to self
gotoxy -74 0
go WEST for 7
: "mvx1r"
if touching SOUTH then "mvx1"
go SOUTH for 1
: "mvx2r"
if touching SOUTH then "mvx2"
: "dr4r"
if c?? OpenDoor p?? at SOUTH then "dr3"
rel to self
if c?? OpenDoor p?? at -1 1 then "dr4"
go SOUTH for 1
put c0f Door p04 to SOUTH
open at SOUTH
wait for 1
go SOUTH for 2
if "tostore" = 1 then "toss"
go EAST for 12
go NORTH for 7
rel to self
: "mvBr"
if player at -222 12 "mvB"
rel to self
gotoxy -222 12
go NORTH for 7
: "mvCr"
if touching NORTH then "mvC"
go NORTH for 1
: "mvx3r"
if touching EAST then "mvx3"
go EAST for 1
: "mvx4r"
if touching EAST then "mvx4"
: "dr6r"
if c?? OpenDoor p?? at EAST then "dr5"
rel to self
if c?? OpenDoor p?? at 1 1 then "dr6"
go EAST for 1
put c0f Door p03 to EAST
open at EAST
color c1e
wait for 1
go EAST for 5
go NORTH for 4
cycle 1
send "dthing" to "check"
unlockself
if "darkness" = 1 then "fboff2"
goto "do"
: "mvx1"
cycle 1
move player to EAST
cycle 2
goto "mvx1r"
: "mvx2"
cycle 1
move player to EAST
cycle 2
goto "mvx2r"
: "mvx3"
cycle 1
move player to SOUTH
cycle 2
goto "mvx3r"
: "mvx4"
cycle 1
put player SOUTH
cycle 2
goto "mvx4r"
: "mvx5"
cycle 1
put player EAST
cycle 2
goto "dr1"
: "mvx6"
cycle 1
put player SOUTH
cycle 2
goto "dr2"
: "dr1"
rel to self
put c01 Floor p?? at -2 0
rel to self
put c01 Floor p?? at -2 1
put c0f Door p05 to WEST
goto "dr1r"
: "dr2"
rel to self
put c04 Floor p?? at 0 -2
rel to self
put c04 Floor p?? at -1 -2
put c0f Door p00 to NORTH
goto "dr2r"
: "dr3"
: "dr4"
rel to self
put c04 Floor p?? at -1 1
put c04 Floor p?? to SOUTH
rel to self
put c0f Door p04 at 0 2
goto "dr4r"
: "dr5"
: "dr6"
rel to self
put c01 Floor p?? at 1 1
put c01 Floor p?? to EAST
rel to self
put c0f Door p03 at 2 0
goto "dr6r"
: "mv1"
cycle 1
move player to EAST
cycle 2
goto "mv1r"
: "mv2"
cycle 1
move player to NORTH
cycle 2
goto "mv2r"
: "mv4"
cycle 1
move player to WEST
cycle 2
goto "mv4r"
: "mv5"
cycle 1
move player to SOUTH
cycle 2
goto "mv5r"
: "mv6"
cycle 1
move player to NORTH
cycle 2
goto "mv6r"
: "mv7"
cycle 1
move player to NORTH
cycle 2
goto "mv7r"
: "mv8"
cycle 1
move player to NORTH
cycle 2
goto "mv8r"
: "mv9"
cycle 1
move player to SOUTH
cycle 2
goto "mv9r"
: "mvB"
cycle 1
move player to WEST
cycle 2
goto "mvBr"
: "mvC"
cycle 1
move player to WEST
cycle 2
goto "mvCr"
: "noprob"
* "~5Roy: Hmmm... nothing wrong here."
send "msg" to "fad"
wait for 15
goto "noprobr"
: "nowire"
* "~5Roy: Is this somebody's idea of a joke!?"
send "msg" to "fad"
wait for 15
* "~5Roy: I guess I'll go buy a new wire..."
send "msg" to "fad"
wait for 5
set "tostore" to 1
goto "noprobr"
: "toss"
go SOUTH for 3
: "ts1r"
if touching SOUTH then "ts1"
go SOUTH for 1
* "~5Roy: I'll be back later, Jace, DON'T touch my computer!"
send "msg" to "fad"
send "dthing" to "check"
set "tostore" to 0
die
: "ts1"
cycle 1
move player to EAST
cycle 2
goto "ts1r"
</p>
</details>


