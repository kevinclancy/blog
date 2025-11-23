---
layout: post_classic
title: "Loose Ends and Future Directions"
date: 2025-11-18 09:00 -0700
published: true
---

$$\newcommand{vrt}[2]{\left ( \begin{array}{l} #1 \\ #2 \end{array} \right )} $$
$$\newcommand{defeq}{\overset{\mathit{def}}{=}}$$

# Introduction

In this series, I've explored how to model computer games using dynamical systems and lenses: starting with [motivational foundations]({% post_url 2025-07-11-towards-mathematical-model %}), then [formalizing tic-tac-toe]({% post_url 2025-10-18-modelling-tic-tac-toe %}), and most recently [modelling MegaZeux-style games]({% post_url 2025-11-04-modelling-megazeux %}) with robots moving on grids.

# Loose Ends

I've had a few small ideas about these game models that I've haven't gotten around to describing. They're relevant to the self-critique below, so I'll describe them here.

## Player as Robot

In my first post, I mentioned that I wanted to analyze games as *closed* dynamical systems, i.e. systems which are not influenced by input and which do not produce any output. These systems are more simulations than games. Since closed systems do not take input, does that mean they cannot model the notion of a *player*, a robot modelling a sentient being whose actions are driven by external input?

If we're using possibilistic dynamic systems, then the answer is no. In possibilistic systems, it is extremely simple to model a player using a robot. A player can be modeled as a robot whose $$\mathit{nextState}$$ function produces the set of all possible successor states paired with all possible action submissions.

## Robot Input as Observations

In [Modelling MegaZeux]({% post_url 2025-11-04-modelling-megazeux %}), every robot receives the full board state as input at every frame. This means that every robot is omniscient. For example, it somehow knows where every other robot is, even if they are separated from it by a wall. More generally, we could make robot perception realistic by modifying the environment to only send a robot the positions of those robots and walls within its line of sight.

This isn't to say that giving robots omniscient perception is undesirable, just that in the most general case, robot perception is limited, and each individual robot's perception may only contain strict subset of the complete game state.

# Fine State vs Coarse State

In [my first post]({% post_url 2025-07-11-towards-mathematical-model %}), I awkwardly noted that a robotic script performs the role of a robot's brain. I pointed out that while the fuse box is implemented as a robot, the real world object that it simulates (a fuse box) is not sentient and therefore does not have a brain. But the fuse box does have physical characteristics, such as a boolean flag indicating whether it is on or off, that is both beyond the scope of the MegaZeux game engine's capacity to understand and relevant to how it interacts with the other objects in the game world.

We can think of the physical properties within the scope of the game engine as *coarse grained*. A robot's cell position in the game world's grid is the only coarse grained physical property that we have discussed so far. Physical properties that fall outside the scope of the game engine, such as the on/off position of the fuse box, can only be read and written by custom scripts. We can think of these properties as *fine grained*.

Game logic is consists of *scripts* acting on *data*. All scripts are inherently fine grained, because they are written by a game developer for a specific game, and do not describe behavior that is generic across all games. Data can be fine grained or coarse grained; the mental states described in [modeling MegaZeux-style games]({% post_url 2025-11-04-modelling-megazeux %}) are fine grained, while robot positions are coarse grained.

A robot that simulates a sentient being could have additional fine grained state beyond its mental state. For example, the fine grained state of a swordsman may include a boolean flag indicating whether or not his sword is drawn. In the game Weirdness, described in [my first post]({% post_url 2025-07-11-towards-mathematical-model %}), the player maintains a large inventory of items. This too could be considered fine grained state.

From the above discussion, I conclude that "pseudo brain" and "mental state" are confused terminology that must be retired immediately. From now on, the mental states $$\Sigma_i$$ discussed in [modeling MegaZeux-style games]({% post_url 2025-11-04-modelling-megazeux %}) shall be referred to as *fine states*. Physical properties such as robot positions, which fall within the scope of the game engine (and were represented by elements of $$\mathit{BoardState}$$ in modelling MegaZeux), shall be referred to as *coarse state*.

The following table summarizes these concepts:

| Concept | Granularity | Managed By | Examples |
|---------|-------------|------------|----------|
| **Coarse State** | Coarse-grained | Game engine | Robot positions, wall locations, tile types |
| **Fine State** | Fine-grained | Game-specific scripts | Fuse box on/off flag, player inventory, NPC dialog state |
| **Scripts** | Fine-grained | Game developer | Robot behavior code, custom game logic |

## Defining the Fine/Coarse Boundary

Placing the boundary between fine and coarse is a pragmatic decision made by the game engine developer; there are no formal restrictions on how it is done. In fact, the table in the previous section is just an example. The designer of MegaZeux (Alexis Janson) could have decided to have included an inventory system directly into MegaZeux. Each MegaZeux game would declare a set of items, and during the execution of the game, MegaZeux would associate each robot with the subset of the declared items that it currently posesses. This inventory system would not be pragmatic for most games, because players do not tend to care about what items non-player NPCs possess.

As a side note, MegaZeux does provide a limited coarse inventory system, but just for the player instead of arbitrary robots. The game engine tracks the number of gems (which can be used as money) the player posesses, as well as which keys (which can be used to open doors) they possess. This quirky system makes it easy for newcomers to make simple games.

# Robots, Encapsulation, Interactions

There's something else that bothers me about using the term "pseudo brain" in my initial post. The player and the fuse box are completely different types of things. The player simulates a sentient being that can establish goals and affect the world around it. The fuse box is just an object with no cognitive power, which contains fine state but remains static unless acted upon. Why does the fuse box have a script associated with it? Scripts are actions that change the world state, wheras inanimate objects like the fuse box cannot change the world state. When the player touches the fuse box, the fuse box does not decide to change its position and a send a message along the "fuse box off" channel; that action is caused by the player.

Perhaps MegaZeux is under-taxonomized. Instead of robots, we could use constructs of a different type to represent inanimate objects with fine state but no behavior; let's call this type of construct a *widget*. When a robot attempts to move into a widget's position, a function which performs their interaction should execute. This function should take as parameters both the fine state of the widget and the fine state of the robot, as well as the coarse state that the robot observes on the particular frame that the two objects collide. The function should produce as output a pair of two new states for the widget and robot.

Also consider what happens when one robot attempts to move into a cell that is already occupied by another robot. The environment could send both of the robots a "touch" message, but both robots would need to access the fine state of the other in order to respond properly. One robot may even need to change the fine state of the other, say by knocking a sword out of the other robot's hands. Should one robot A's touch handler drop its sword in response to a collision, or should robot B's handler knock robot A's sword from its hands? This discussion suggests that handling interactions from a first-person perspective is a flawed idea. What we really need is a single interaction handler. This handler could take the fine states and observations of both robots as arguments, and its results should include updated fine states of both robots.

Interactions generally take multiple frames to complete. In the first-person model that I began with, a robot is notified of an interaction by receiving a message. The message handler may then switch the robot to a state intended solely for responding to the message; this state may perform actions over the course of many frames responding to the message. With an interaction handler, we could change states of the involved robots so that they spend several frames responding to the interaction event. But naively, the would do so independently, controlled by separate scripts and without each other's fine state. Instead, what if a single piece of code, or a single "interaction" entity, to control the interaction for several frames? This entity would have access to both robots' fine states and possibly posess some fine state of its own. It would be similar to a robot but without an associated grid location; let's call such entities "ambient robots".

Here is an example of a multi-frame interaction: a dialog between the player and an NPC (non-player character). This dialog is initiated by the player's attempt to move into the cell of the NPC. The dialog can't be resolved in a single frame, as it requires the player to answer a sequence of questions posed by the NPC. So the interaction handler mutates the player and NPC into "acquired" states in which they stay idle for the duration of the dialog. The ambient robot managing the interaction maintains its own fine state, which simulataneously simulates the mental states of both the player and the NPC. Here is an example of how such an interaction handler might be implemented.

```
enum {
  Buy,
  Sell
}

ambient robot player_npc_dialog {

  // reference that provides access to player coarse state, such as portrait and position
  robot player;

  // reference that provides access to npc coarse state, such as portrait and position
  robot npc;

  // a coroutine that runs the dialog between the two characters
  script run_dialog() {
    let choice = npc.ask("Would you like to buy or sell items?", [("Buy", Buy), ("Sell", Sell)]);
    if choice == 0 {
      player.say("I'd like buy something.");
      // code for buy user interface here
    } else if choice == 1 {
      player.say("I'd like to sell something.");
      // code for sell sell user interface
    } else {
      assert false;
    }

    // send the player and npc back into the states they were in before they were acquired
    player.release();
    npc.release();
  }
}
```

This ambient robot does not contain fine state that replicates the fine state of the `player` and `npc` robots. Instead, it temporarily manages new fine state that describes additional fine-grained information about the `player` and `npc` robots. Its two fields `player` and `npc` simulate the shared understanding between the player and npc that they are engaged in an interaction with each other. It has additional fine state: namely, the program counter and the callstack of the `run_dialog` coroutine. These pieces of information simulate the two character's shared understanding of specifics of the dialog. For example, `run_dialog`'s callstack may contain the `choice` variable, which stores the player's choice of whether to buy or sell items.

# Message Channels and Location

[Your analysis of actual MegaZeux channel usage here]

# Why I'm Pausing This Work

[Your explanation about job search priorities here]

# Conclusion

[Your wrap-up here]
