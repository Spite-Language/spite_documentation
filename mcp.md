# The language's MCP

An AI working on a Spite program wants what a person gets from the REPL ([repl.md](repl.md)): run a line of code
inside the running program and see what it answers. It does not want a console. It wants tools it can call, and it
wants to know that the file it just saved is the code that answers. The language's MCP gives it both: the compiler is
a [Model Context Protocol](https://modelcontextprotocol.io) server whose tools run Spite in a running program.

```text
evaluate  {"code": "monsters[0].health"}
→ 30  (Integer)

# the AI saves monster.spite, where roar() now says "{name} roars louder"
evaluate  {"code": "monsters[0].roar()"}
→ Goblin roars louder  (String)
```

No call asks whether the save is live: `evaluate` waits until every saved file is compiled and swapped in, then runs
the code. When a save cannot go live, `evaluate` says why instead of answering from the old code.

It runs any code, with no password, so it is for development only: a production build never carries it. Windows only
for now, like everything that swaps code into a running program
([spite_compiler#297](https://github.com/Spite-Language/spite_compiler/issues/297)).

## Starting it

Two halves. The program listens for the MCP:

```
spite --mcp-port=4000
```

and the AI's MCP client starts the server, which talks to that program:

```json
{
  "mcpServers": {
    "spite": { "command": "spite", "args": ["--mcp=4000"] }
  }
}
```

`--mcp-port` includes `--hot-reload` ([Live reload](repl.md#live-reload---hot-reload)): the program watches its files
and swaps every save in, since the AI's next call expects its edit to be live. The program listens on `127.0.0.1`
only, one connection at a time, and runs what it is sent the next time it waits or finishes a pass of a loop, so
code never sees it halfway through a step. When the constructor returns, the program stays alive until code it is
sent calls `program.exit(0)`. The server keeps running when no program is listening, and its tools say so, so
the AI can start the program and try again.

## The tools

**`evaluate`** runs Spite in the program, the way a REPL line runs ([Typing Spite into a running
program](repl.md#typing-spite-into-a-running-program)). An expression answers its value and its class; a statement
answers nothing. `program` is the entry instance, and its attributes and functions can be named bare. `var` and
`func` stay for later calls:

```text
evaluate  {"code": "var strongest = monsters[1]"}
→ (Nothing)
evaluate  {"code": "strongest.name"}
→ Orc  (String)
evaluate  {"code": "Monster.functions.has(\"growl\")"}
→ false  (Boolean)
```

Each of these is an error the AI reads instead of an answer, and nothing runs:

- a save that does not compile, with the compiler's error: the program still runs the code from before it;
- a save the reload holds, such as an attribute renamed (below);
- code that does not compile, with the compiler's error;
- saves that are still not live after ten minutes.

A crash in the code ends the program, as any crash does, and the answer says so with the crash line.

**`reload`** compiles the saved files now and answers what it rebuilt, moved or removed. It is how a rename keeps
its value: a save where one attribute is gone and another is new could be a rename or a removal and an addition, so
the reload holds it and `evaluate` answers what it holds:

```text
evaluate  {"code": "hero.rank"}
→ error: the reload of Hero is held, since it would lose data: level is gone and rank is new: reload with renames {Hero.attributes["level"]: "rank"} to keep what level holds, or with renames {} to let it go

reload  {"renames": "{Hero.attributes[\"level\"]: \"rank\"}"}
→ rebuilt Hero, ...
  moved every Hero to its new attributes: level is now rank

evaluate  {"code": "hero.rank"}
→ 7  (Integer)
```

The map is written as Spite writes one, the same map a person gives `load` at the REPL
([Changing a class's attributes](repl.md#changing-a-classs-attributes)).

## For an AI driving a program

- Start the program with `--mcp-port`, then call `evaluate`. Edit files as you normally would: every `evaluate`
  runs on what you saved.
- Ask reflection what is there before changing anything: `Spite.Class.instances`, `Monster.attributes`,
  `monsters[0].attributes`, `World().entity_count()`.
- To see a value at a moment, add a `console.print` to the code and save, then read the program's output where you started it; to change
  state once, `evaluate` an assignment.
- Read every error: a held reload or a save that does not compile means the program is not running your edit yet.

## Shipping an MCP of your own

The language's MCP is a debugging tool, and never part of what you ship. A program that should let an AI change it
in production, a game server whose tick rate or spawn rules an AI tunes, writes its own tools: an application's MCP,
where the program decides what an AI can touch. A library for writing one is proposed in
[spite_compiler#298](https://github.com/Spite-Language/spite_compiler/issues/298).

---

Next: [Testing](testing.md), a test is a package that crashes.
