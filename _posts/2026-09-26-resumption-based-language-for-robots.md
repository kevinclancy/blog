---
layout: post_classic
title: "A Resumption-Based Language for Robots"
date: 2026-09-26 09:00 -0700
categories: [modelling-computer-games]
subseries: language
published: true
---

$$\newcommand{vrt}[2]{\left ( \begin{array}{l} #1 \\ #2 \end{array} \right )} $$
$$\newcommand{defeq}{\overset{\mathit{def}}{=}}$$

# Robot scripting

Recall that in our MegaZeux model, a [robot is modelled]({% post_url 2025-11-04-modelling-megazeux %}#the-robots) as a partial possibilistic lens of the following form:

$$\mathit{Robot}_i : \vrt{\mathit{State}_{\mathit{Robot}_i}}{\mathit{State}_{\mathit{Robot}_i}} \leftrightarrows \vrt{\mathit{In}_{\mathit{Robot}_i}}{\mathit{Out}_{\mathit{Robot}_i}} \defeq \vrt{nextState_{\mathit{Robot}_i}}{\mathit{output}_{\mathit{Robot}_i}}$$

and its state representation $$\mathit{State}_i$$ is decomposed as

$$\mathit{State}_{\mathit{Robot}_i} \defeq S \times \Sigma_i$$

Above, $$S$$ is an *administrative state* component common across all robots. It keeps track of the robot's interaction with the game stepper: whether it's currently in a receive or send phase, and in the later the output data that has been staged for sending to the environment. We define $$S \defeq 1 + \mathit{Act}$$, where $$\mathit{Act}$$ is the set of actions the robot may submit to the environment; the left injection represents a robot that is currently in its receiving phase, and the right injection represents a robot that is currently in its sending phase.

In contrast, $$\Sigma_i$$ is *mental state* of robot $$i$$, which tracks information related to the robot's knowledge and goals. It may store a program counter and private data fields, for example. This component of the state varies across distinct robots.

The $$\mathit{Robot}_i$$ lens's output function $$\mathit{Out}_i$$ is trivial: it just sends its administrative state as output:

$$\mathit{Out}_{\mathit{Robot}_i}(s, \sigma) \defeq s$$

The $$\mathit{Robot}_i$$ lens's update function is more interesting.

$$\mathit{nextState}_{\mathit{Robot}_i} : S \times \Sigma_i \times (1 + \mathit{BoardObservation} \times \mathbf{2}) \to 1 + P_+ (S \times \Sigma_i)$$

On the robot's send phase, it deterministically selects $$((0, \ast), \sigma) \in S \times \Sigma_i$$ as its successor state, i.e. the ``receive phase'' administrative state $$(0, \ast)$$ paired with its current mental state $$\sigma$$. The behavior of the robot's update function on its receive phase is central to the present post:

$$\mathit{nextState}_{\mathit{Robot}_i}((0, \ast), \sigma, (1, (b,s))) \defeq (1, \{ ((1,a), \sigma') \})$$

Above, the staged output action $$a$$ and the successor mental state $$\sigma'$$ are computed by a function $$\phi_i : \Sigma_i \times \mathit{BoardObservation} \times \mathbf{2} \to \mathit{Act} \times \Sigma_i$$; i.e., we have:

$$(a, \sigma') \defeq \phi_i(\sigma, b, s)$$

In words, given the present mental state $$\sigma$$, the present board observation $$b$$, and a boolean ``success'' flag $$s$$ indicating whether the environment accepted or rejected the last action submitted by robot $$i$$, it produces the next action to submit $$a$$ and the next mental state $$\sigma'$$.

In this post, I will explore what a programming language designed for defining the functions $$\phi_i$$ might look like.


# Roy Revisited

Recall [the *Roy* robot from Weirdness]({% post_url 2025-07-11-towards-mathematical-model %}#roy-a-typical-gameplay-scenario). Upon receiving the "fuse box off" message, it performs the following scripts, from top to bottom:

* Curse in anger upon watching his computer shut down.
* Walk to the fuse box and fix it.
* Walk back to the computer.

Each of the above scripts is a complex sequence of simpler scripts. For example, the first "curse" script might actually involve three sub-scripts:

* Spawn a text box saying "$%#&&*! I haven't saved my game yet!"
* Wait for one second
* Spawn a text box saying "I'd better go check the fuse box..."

The second script "walk to fuse box" might consist of the following sub-scripts:

* Move to the fuse box
* Span a text box saying "Hmmm... nothing wrong here."

Because our game stepper only allows a robot to move a distance of 1 cell every turn, we know that the first sub-script above must remain "active" across many game steps.

From the above discussion we draw the following two criteria. First, the "scripts" in a robot's code should be nestable, just like function calls. Second, each script should have the ability to span multiple game steps. To satisfy the first criterion, we will include functions in our language: to execute an script, we simply call a function. To satisfy the second criterion, we must provide a construct that submits an output action to the environment, and yields control to the environment until it provides a new observation of the game board and a response boolean indicating whether the action succeeded; this may look like a function call, e.g. `let (obs, succ) = yield WalkEast`. The important point here is that the expression `yield WalkEast` does not immediately produce the results `(obs, succ)` but instead waits until the results are available (at the next game step) to resume control.

# Resumptions for Game Scripting

## Asymmetric Coroutines in Lua

Variations of the `yield` construct described above have, of course, appeared in many languages and have frequently been used for game scripting just as I've described. For example, Lua's *asymmetric coroutines* feature allows the programmer to create a *thread* from a function *f*. Subsequently, *resuming* the thread causes the thread to execute to *f*'s next *yield*. For example, in Lua, an implementation of Roy might look like this:

```
-- wait(s) waits for s seconds before returning
local function wait(seconds)
  local start_time = time.get()
  while time.get() - start_time < seconds do
    coroutine.yield("idle")
  end
end

-- notice the computer shut down, and complain
local function curse()
  say("$%#&&*! I haven't saved my game yet!")
  wait(1)
  say("I'd better go check the fuse box...")
end

-- walk to the fuse box and notice nothing's wrong
local function check_fuse_box()
  walk_to(locations.fuse_box)
  say("Hmmm... nothing wrong here.")
end

-- walk back to the computer
local function return_to_computer()
  walk_to(locations.computer)
end

function on_fuse_box_off()
  curse()
  check_fuse_box()
  return_to_computer()
end
```

The above code assumes that some parent thread repeatedly resumes a thread constructed from the `on_fuse_box_off` function. The initial resume invokes the `curse` function, but rather than returning from `curse`, the `on_fuse_box_off` thread yields to its parent thread. It yields to wait for the player acknowledge the speech by pressing a key. The `on_fuse_box_off` thread may get resumed several times before it detects a keypress and returns from `say`. Then, `wait` is called and the thread may get resumed several additional times while waiting for a second to elapse.

In Lua, some "parent thread" is responsible for creating the `on_fuse_box_off` thread and resuming it. The parent thread might execute the following code:

```
local fuse_box_off = coroutine.create(on_fuse_box_off)

-- Called by the game stepper once per step.
function step(board_obs, succ)
  if coroutine.status(fuse_box_off) == "dead" then
    return { move = "idle" }
  end
  local _, action = coroutine.resume(fuse_box_off, board_obs, succ)
  return action
end
```

Above, the call to `coroutine.create` creates the thread. Then, at each game step, the `step` function is called in the parent thread, which resumes the `fuse_box_off` thread by passing it (along with the board state and success flag) to `coroutine.resume`. Note that `step` also checks if the `fuse_box_off` thread's status is equal to `"dead"`; this happens when the outer function `on_fuse_box_off` returns. Having a special unique control flow location that causes the thread to "die", preventing further resumptions, seems unnatural. On the other hand, the ability to define zero or more explicit "exit" control locations may be useful. I'll present a better approach later in this article.

## The Importance of Productivity

In the above example, imagine that we had placed an infinite loop at the top of the `on_fuse_box_off` function:

```
function on_fuse_box_off()
  while true do
    -- this block is intentionally empty
  end
  curse()
  check_fuse_box()
  return_to_computer()
end
```

If a parent thread were to construct a thread from `on_fuse_box_off` and then resume it, the parent thread would be forced to wait indefinitely while `on_fuse_box_off` spins in a loop. This is not desirable behavior. There could exist computer programs where an infinite, non-yielding loop is desirable, where each iteration of the loop performs some form of communication. However, in a game, the player expects constant progression. The game world, like the real world, cannot stop advancing simply because a robot has fallen asleep. Therefore, any infinite loops in our language must be *productive*. A loop is productive if at every point in time, there exists a later point in time at which it yields.

Here is an example of a productive infinite loop that is used to control a robot:
```
while true do
  walk_east(2)
  walk_south(2)
  walk_west(2)
  walk_north(2)
end
```
The robot is a "patroller" that walks around in circles indefinitely. The loop is productive because each iteration yields at least once. In fact, each iteration yields exactly 8 times: first submitting two `Walk East` actions, then submitting two `Walk South` actions, etc.

From this discussion, we conclude that our robot scripting language should not feature arbitrary loops, but instead restrict the loops occurring in the program to two forms:

* Terminating loops.
* Productive infinite loops, which are guaranteed to yield every time they are resumed.

# A Syntax For Robotic
