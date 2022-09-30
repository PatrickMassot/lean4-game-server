# Lean Game Server

## Scope

This is a prototype Lean game framework for Lean 4. It serves three purposes:

* Teaching myself some Lean 4 meta-programming;
* Powering a Lean 4 version of the [Natural Number Game (NNG)](https://www.ma.imperial.ac.uk/~buzzard/xena/natural_number_game/) of Kevin Buzzard and Mohammad Pedramfar, as well other games and teaching tools hopefully;
* Being a technological demo of what can be done with Lean 4 meta-programming. Beware that it doesn't mean it is a demo of how things should be done (see the first item above).

Before giving more details, it is useful to understand a bit more what the Lean 3 version of NNG really is. 

First there is a non-technical aspect. Way back in 2018 and early 2019, Kevin started giving lots of talks about formalized mathematics to very diverse audiences. He noticed that one topic that really worked well was explaining how to build natural numbers and prove fundamental properties of their algebraic operations. So he decided to help people working on this themselves instead of simply watching him during talks. But installing Lean was a huge barrier at that time (even today it is not yet as easy as we would like it to be). The goal wasn't to get more Lean users. Choices were made to favor learning mathematics over Lean expertise. But the project worked way beyond expectations and many people started to see this game as an entry gate to actual use of Lean. This discrepancy led to some issues down the road where people got confused by ways tactic have subtly different behaviors in NNG and regular Lean. On the other hand the originally intended users were not shielded from the complexity of Lean. They were basically editing a Lean file. They saw errors as they typed even if they were half-way through typing a line. They were also allowed to continue typing new lines after errors, which can only lead to chaos. This Lean 4 game framework is even more opinionated in not pretending to be regular Lean. It does not let users edit Lean file, only execute one tactic at a time.

Another motivation for this project is the [dEAduction software](https://github.com/dEAduction/dEAduction/) by Frédéric Le Roux, Florian Dupeyron, Antoine Leudiere and Marguerite Bin. 
This experimental teaching tool uses Lean 3 but has a completely different interface. It communicates with Lean by hacking the lean server protocol (this was the original motivation for my [lean-client-python](https://github.com/leanprover-community/lean-client-python/) project). This does hide the complexity of Lean to users but is awful to developers. The Lean server protocol is simply not meant for this. It has constraints coming from the worlds of real-time editing that are completely orthogonal to what would be needed here. We will come back to this below.


## Defining game levels and in-game documentation

The Lean 3 NNG is based on two technological pieces, besides Lean itself which is of course by far the main piece. The first one is the [Lean 3 web editor](https://github.com/leanprover-community/lean-web-editor) which leverages the WebAssembly build of Lean to provide a full Lean+editor pair in any web browser. The second one is Mohammad's [Lean game maker](https://github.com/mpedramfar/Lean-game-maker), based on my [Lean Formatter](https://github.com/leanprover-community/format_lean). That tool is a python library which read Lean file containing structures comments containing commands and markdown text zones. See for instance the [first level](https://github.com/ImperialCollegeLondon/natural_number_game/blob/master/src/game/world1/level1.lean). The comment 

```lean
/- Tactic : refl
## Summary
`refl` proves goals of the form `X = X`.
## Details
The `refl` tactic will close any goal of the form `A = B`
where `A` and `B` are *exactly the same thing*.
### Example:
If it looks like this in the top right hand box:
```
a b c d : mynat
⊢ (a + b) * (c + d) = (a + b) * (c + d)
```
then
`refl,`
will close the goal and solve the level. Don't forget the comma.
-/
```

is of course ignored by Lean but the game-maker collects it as the documentation of the `refl` tactic that will be available on the left-hand side of the game interface. Some meta-data is also provided in config files such as [game_config.toml](https://github.com/ImperialCollegeLondon/natural_number_game/blob/master/game_config.toml).

In the current Lean 4 prototype, we leverage a tiny part of the syntax extensibility of Lean 4 to produce a very simple domain-specific language. For instance, a simple NNG level can look like:
```lean
import NNG.Metadata

Level 1

Title "The reflexivity tactic"

Introduction "Don't worry, this will be easy"

Statement (x y z : ℕ) :  x * y + z = x * y + z := by
rfl

Conclusion "Congratulations for completing your first level!"

Tactics rfl
```

The above snippet is a fully valid Lean file. The proof can be worked on interactively. The last line indicates the list of tactics that are allowed in that level. One could also indicate lemmas using a similar command. Tactic documentations are also declared using similar commands, such as:

```lean
TacticDoc rfl "`rfl` proves goals of the form `X = X`."
```

We also have a messaging mechanism which allows to display messages when certain tactic states are reached. For instance we could have:

```lean
Statement (x y : ℕ) (h : y = x + 7) : 2 * y = 2 * (x + 7) := by 
  rewrite [h]
  rfl

Message (x : ℕ) (y : ℕ) (h : y = x + 7) : 2*(x + 7) = 2*(x + 7) => 
"Well rewritten! Now the goal should be easy to reach using the `rfl` tactic."
```

This avoids moving back and forth between the explanation and the proof as in the original NNG. 

All these game building commands are accessible using the [`GameServer.Commands`](./GameServer/Commands.lean) module.

## Running games

Once a game is ready, it can be run using the `runGame` function from the [`GameServer.Commands`](./GameServer/Server.lean) module. This will *not* use the `lean` command line of the language server protocol. The game is not pretending to be a text editor editing a file. The program will wait for Json-formatted instruction on its standard input stream and answer each command with exactly one Json answer on its standard output stream. That answer will come when it's ready and there won't be any half answer. The main input commands are `"info"` which asks for game metadata, `{"loadLevel": N}` where `N` is a natural number, `{"runTactic": tac}` where `tac` is a string, and `"undo"`. See the `Action` structure in the `Server` module for an exhaustive list of commands.

That part of the framework is based on Daniel Selsam's [lean-gym prototype](https://github.com/dselsam/lean-gym/), heavily modified.

On top of this Json-based communication, you can build any kind of interface. Since I also needed to learn [React](https://reactjs.org/) for [widget-writing](https://leanprover.github.io/lean4/doc/examples/widgets.lean.html) purposes, I built a sample react-based web interface (here also don't use this as an example of how to write code, this is my very first react project!). 

One could easily put such an interface inside [Electron](https://www.electronjs.org/) or [Neutralino](https://neutralino.js.org/) for a purely local experience. However the interface uses web sockets to talk to a remote game server. It means we need a tiny piece of glue to send websocket requests to the game server stdin and send back responses taken from stdout. Lean 4 does not have a websocket lib yet, so this is a remaining piece of python in this whole story (about 50 lines).

The interface is still lacking many features. For instance one can not explore a proof, you can only see the current tactic state.