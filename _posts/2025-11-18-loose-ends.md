---
layout: post_classic
title: "Loose Ends and Future Directions"
date: 2025-11-18 09:00 -0700
categories: [modelling-computer-games]
published: true
---

$$\newcommand{vrt}[2]{\left ( \begin{array}{l} #1 \\ #2 \end{array} \right )} $$
$$\newcommand{defeq}{\overset{\mathit{def}}{=}}$$

# Brain Dump

In this series, I've explored how to model computer games using dynamical systems and lenses: starting with [motivational foundations]({% post_url 2025-07-11-towards-mathematical-model %}), then [formalizing tic-tac-toe]({% post_url 2025-10-18-modelling-tic-tac-toe %}), and most recently [modelling MegaZeux-style games]({% post_url 2025-11-04-modelling-megazeux %}) with robots moving on grids.

I've become exhausted working on this project for now. It's not the sort of thing that will help me get a job, so I'm going to try to record all of the unsolidified ideas I've been thinking about in this post. I'm going to try my best to keep it scrutable, but because it is a brain dump, expect it to be somewhat disconnected.

# Loose Ends

I've had a few small ideas about these game models that I haven't gotten around to describing. They're relevant to the self-critique below, so I'll describe them here.

## Player as Robot

In my first post, I mentioned that I wanted to analyze games as *closed* dynamical systems, i.e. systems which are not influenced by input and which do not produce any output. These systems are more simulations than games. Since closed systems do not take input, does that mean they cannot model the notion of a *player*, a robot modelling a sentient being whose actions are driven by external input?

If we're using possibilistic dynamic systems, then the answer is no. In possibilistic systems, it is extremely simple to model a player using a robot. A player can be modeled as a robot whose $$\mathit{nextState}$$ function produces the set of all possible successor states paired with all possible action submissions.

## Robot Input as Observations

In [Modelling MegaZeux]({% post_url 2025-11-04-modelling-megazeux %}), every robot receives the full board state as input at every frame. This means that every robot is omniscient. Hence, a robot somehow knows where every other robot is, even if they are separated from it by a wall. To gain some realism, we could modify the environment to only send a robot the positions of those robots and walls within its line of sight.

This isn't to say that giving robots omniscient perception is always undesirable. But just that in many games, players expect robots to
perceive realistically, and so in the general case each individual robot should perceive only a strict subset of the complete game state.

# Fine State vs Coarse State

In [my first post]({% post_url 2025-07-11-towards-mathematical-model %}), I awkwardly noted that a robotic script performs the role of a robot's brain. I pointed out that while the fuse box is implemented as a robot, the real world object that it simulates (a fuse box) is not an animal and therefore does not have a brain. But the fuse box does have physical characteristics, such as a boolean flag indicating whether it is on or off, that is both beyond the scope of the MegaZeux game engine's capacity to understand and relevant to how it interacts with the other objects in the game world.

We can think of the physical properties within the scope of the game engine as *coarse grained*. A robot's cell position in the game world's grid is the only coarse grained physical property that we have discussed so far. Physical properties that fall outside the scope of the game engine, such as the on/off position of the fuse box, can only be read and written by custom scripts. We can think of these properties as *fine grained*.

Game logic consists of *scripts* acting on *data*. All scripts are inherently fine grained, because they are written by a game developer for a specific game; they are not part of the engine and do not describe behavior that is generic across all games. Data can be fine grained or coarse grained; the mental states described in [modeling MegaZeux-style games]({% post_url 2025-11-04-modelling-megazeux %}) are fine grained, while robot positions are coarse grained.

A robot that simulates a sentient being could have additional fine grained state beyond its mental state. For example, the fine grained state of a swordsman may include a boolean flag indicating whether or not his sword is drawn. In the game Weirdness, described in [my first post]({% post_url 2025-07-11-towards-mathematical-model %}), the player maintains a large inventory of items. This too could be considered fine grained state.

From the above discussion, I conclude that "pseudo brain" and "mental state" are confused terminology that must be retired immediately. From now on, the mental states $$\Sigma_i$$ discussed in [modeling MegaZeux-style games]({% post_url 2025-11-04-modelling-megazeux %}) shall be referred to as *fine states*. Physical properties such as robot positions, which fall within the scope of the game engine (and were conveyed by elements of $$\mathit{BoardState}$$ in modelling MegaZeux), shall be referred to as *coarse state*.

The following table summarizes these concepts:

| Concept | Granularity | Managed By | Examples |
|---------|-------------|------------|----------|
| **Coarse State** | Coarse-grained | Game engine | Robot positions, wall locations, tile types |
| **Fine State** | Fine-grained | Game-specific scripts | Fuse box on/off flag, player inventory, NPC dialog state |
| **Scripts** | Fine-grained | Game developer | Robot behavior code, custom game logic |

## Defining the Fine/Coarse Boundary

Placing the boundary between fine and coarse is a pragmatic decision made by the game engine developer; there are no formal restrictions on how it is done. In fact, the table in the previous section is just an example. The designer of MegaZeux (Alexis Janson) could have decided to include an inventory system directly into MegaZeux. Each MegaZeux game would declare a set of items, and during the execution of the game, MegaZeux would associate each robot with the subset of the declared items that it currently possesses. This inventory system would not be pragmatic for most games, because players do not tend to care about what items non-player NPCs possess.

As a side note, MegaZeux does provide a limited coarse inventory system, but just for the player rather than all robots. The game engine tracks the number of gems (which can be used as money) the player possesses, as well as which keys (which can be used to open doors) they possess. This quirky system makes it easy for newcomers to make simple games.

# Robots, Encapsulation, Interactions

There's something else that bothers me about using the term "pseudo brain" in my initial post. The player and the fuse box are completely different types of things. The player simulates a sentient being that can establish goals and affect the world around it. The fuse box is just an object with no cognitive power, which contains fine state but remains static unless acted upon. Why does the fuse box have a script associated with it? Scripts are actions that change the world state, whereas inanimate objects like the fuse box cannot change the world state. When the player touches the fuse box, the fuse box does not decide to change its position and send a message along the "fuse box off" channel; that action is caused by the player.

Perhaps MegaZeux is under-taxonomized. Instead of robots, we could use constructs of a different type to represent inanimate objects with fine state but no behavior; let's call this type of construct a *widget*. When a robot attempts to move into a widget's position, a function which performs their interaction should execute. This function should take as parameters both the fine state of the widget and the fine state of the robot, as well as the coarse state that the robot observes on the particular frame that the two objects collide. The function should produce as output a pair of two new states for the widget and robot.

Also consider what happens when one robot attempts to move into a cell that is already occupied by another robot. The environment could send both of the robots a "touch" message, but both robots would need to access the fine state of the other in order to respond properly. One robot may even need to change the fine state of the other, say by knocking a sword out of the other robot's hands. Should one robot A's touch handler drop its sword in response to a collision, or should robot B's handler knock robot A's sword from its hands?

This discussion suggests that handling interactions from a first-person perspective is a flawed idea. What we really need is a single interaction handler. This handler could take the fine states and observations of both robots as arguments, and its results should include updated fine states of both robots. Be warned, this line of reasoning is leading to a radical departure from the models I've presented so far, as a dynamical system representing a robot can no longer have exclusive access to the robot's fine state.

```
function handle_robot_collision(swordsman_state a, swordsman_state b) : swordsman_state * swordsman_state = {
  match (a,b) with
  | (Drawn, Drawn) ->
     // play clanking sound
    (Drawn, Swordless)
  | (Drawn, Swordless) ->
    (Drawn, Dead)
  | (Drawn, Dead) ->
    (Drawn, Dead)
  | (Swordless, Drawn) ->
    (Dead, Drawn)
  | (Dead, Drawn) ->
    (Dead, Drawn)
}
```

Interactions generally take multiple frames to complete. In the first-person model that I began with, a robot is notified of an interaction by receiving a message. The message handler may then switch the robot to a state intended solely for responding to the message; this state may perform actions over the course of many frames responding to the message. With an interaction handler, we could change states of the involved robots so that they spend several frames responding to the interaction event. But naively, they would do so independently, controlled by separate scripts and without access to each other's fine state. Instead, what if a single piece of code, or a single "interaction" entity, controlled the interaction for several frames? This entity would have access to both robots' fine states and possibly possess some fine state of its own. It would be similar to a robot but without an associated grid location; let's call such entities "ambient robots".

Here is an example of a multi-frame interaction: a dialog between the player and an NPC (non-player character). This dialog is initiated by the player's attempt to move into the cell of the NPC. The dialog can't be resolved in a single frame, as it requires the player to answer a sequence of questions posed by the NPC. So the interaction handler mutates the player and NPC into "acquired" states in which they stay idle for the duration of the dialog. The ambient robot managing the interaction maintains its own fine state, which simultaneously simulates the mental states of both the player and the NPC. Here is an example of how such an interaction handler might be implemented.

```
enum {
  Buy,
  Sell
}

ambient robot player_npc {

  // reference that provides access to player coarse state, such as portrait image and position
  // shared across all player_npc states
  robot player;

  // reference that provides access to npc coarse state, such as portrait image and position
  // shared across all player_npc states
  robot npc;

  state Inactive {
    // a function to handle the collision immediately
    function player_npc_collide(player_state p, npc_state n) : player_state * npc_state * player_npc_state {
      // First component is player's new state - Acquired means that the player only responds to commands from player_npc robot
      // Second component is npc's new state - Acquired means that it only responds to commands
      // Dialog is the new state for player_npc - its definition is given below
      (Acquired, Acquired, Dialog)
    }
  }

  state Dialog
  {
    // a coroutine that runs the dialog between the two characters
    script run() {
      let choice = npc.ask("Would you like to buy or sell items?", [("Buy", Buy), ("Sell", Sell)]);
      match choice with
      | Buy ->
        player.say("I'd like to buy items.");
        // code for buy user interface here
      | Sell ->
        player.say("I'd like to sell items.");
        // code for sell user interface

      // send the player and npc back into the states they were in before they were acquired
      player.release();
      npc.release();
      set_state(Inactive);
    }
  }
}
```

This ambient robot does not contain fine state that replicates the fine state of the `player` and `npc` robots. Instead, it describes additional fine state associated with the `player` and `npc` robots. Its two fields `player` and `npc` simulate the shared awareness of the two robots' identities; though `player_npc` is a state machine, these fields are shared across both of its states `Inactive` and `Dialog`. In the `Dialog` state, it manages additional fine state: namely, the program counter and the callstack of the `run` coroutine. These pieces of information simulate the two characters' shared understanding of specifics of the dialog. For example, `run`'s callstack may contain the `choice` variable, which stores the player's choice of whether to buy or sell items.

## Doubts

The above section has some workable ideas: for example, ambient robots handling interactions over multiple frames would be useful for quickly creating short cinematic sequences. However, giving a single event handler access to multiple robots' fine states seems too large of a departure from both the framework I've designed so far and standard game scripting practice. If I come back to this project, I am going to abandon that particular idea. I'm still on the fence about distinguishing between widgets and robots.

✅ Ambient robots for multi-frame interactions
✅ Interaction handlers as separate entities
❌ Single function accessing multiple robots' fine states
🤔 Distinguishing widgets from robots

# Globals vs messages

In MegaZeux, fine state is stored in what are called *counters*. Counters are global integer-valued variables that can be read and written arbitrarily by robotic scripts. Here are some examples of counters used by Weirdness:

* The game starts in a house with three sinks. Each time the player turns on a sink, the `sinks` counter, which starts equal to 0, is incremented. Below the house is an area with a water stream blocking the path. Upon entering this area, a robot checks if `sinks` is equal to three and clears the stream if so.
* The player can interact with the world by applying objects stored in their inventory (a custom fine-state inventory rather than the built-in MegaZeux inventory). Corresponding to each inventory item is a counter that is equal to `1` if the player possesses the item and `0` otherwise.

A central guiding principle for my models has been *realism*. Because a game is a simulation of the real world, developers expect it to behave like the real world. For example, having all NPC's ``act simultaneously'' each frame, all responding to stimuli derived from a common world state at a moment in time, is a decision made in appeal to realism.

Global counters are not realistic. In the real world, state is located in space. By associating a piece of fine state with a robot that either has custody of it or shares proximity with it, we make it clear to the developer what state they are likely to manipulate when writing the robot's code. Quick interactions between spatially separated objects can be triggered by sending a message along a channel. We therefore do away with global counters, as they are subsumed by the more realistic combination of fine state and message channels. Weirdness would be rewritten in the following way:

* Turning a sink on or off sends a boolean message along the `water_pipe` channel. When the message arrives at the `stream` robot beneath the house, the `stream` robot either increments or decrements a fine state integer variable. The waterway "dries up" when the integer variable is incremented to 3.
* The player stores its inventory in its fine state as an element of a set datatype.

# Dynamic Behavior

So far, we've made references to the state of our models advancing over a discrete sequence of time steps. But we've done so informally; we still don't have a formal definition of our system's behavior. The behavior of a deterministic system could be represented as a sequence of states, i.e. a function of type $$\mathbb N \to \matit{State}$$. The way such a behavior is derived from a dynamical system is described in chapter 3 of *Categorical Systems Theory*. The behavior of a non-deterministic system would probably be the set of all possible state sequences. In Categorical Systems Theory, Myers doesn't explore the behavior of non-deterministic systems very far, acknowledging it as a blind spot.

# Partial Possibilistic Systems

In [modelling MegaZeux-style games]({% post_url 2025-11-04-modelling-megazeux %}), we used possibilistic systems to represent non-deterministic state update. The key feature was that the passback function produced a set of successor states instead of a single successor state

$$\mathit{nextState} : \mathit{State} \times \mathit{In} \to P(\mathit{State})$$

We handled precondition violations by mapping them to the empty set. This is a hack that is bound to cause problems when we try to define dynamic behavior. A more robust and principled approach would be to define a new type of dynamical system that can express both non-determinism and precondition violations. Let $$P_+$$ be the *non-empty powerset operator* which maps a set $$X$$ to the set of all non-empty subsets of $$X$$. Then our new type of dynamical system, which we could call a *partial possibilistic dynamical system* would have a passback function of the following form

$$\mathit{nextState} : \mathit{State} \times \mathit{In} \to 1 + P_+(\mathit{State})$$

A state/input pair that violates our system's precondition would map to the left component $$(0, \ast)$$, whereas any state/input pair that satisfies our system's precondition would map to a non-empty set of successor states $$X$$ in the right component $$(1, X)$$.

# Why I'm Pausing This Work

Over a year ago, I fell while going for a run, breaking my jaw and wrist. Immediately afterward, I didn't feel capable of doing much other than sitting around and playing video games all day. But I feel terrible if I spend too much time without learning or making things. So as a compromise, I started thinking about the present topic. Now I've mostly recovered, and I need to find a new job. Therefore, instead of continuing with this project, I'm going to spend time learning things that employers are interested in, like web development.

Though it's woefully incomplete, I'm glad that I wrote this series. Rather than just reading Categorical Systems Theory and solving the exercises, I was able to synthesize it into something new, which motivated me to dig in more deeply than I otherwise would have. Who knows? Some day I may return to this project and extend it into an actual game engine.
