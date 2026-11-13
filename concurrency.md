# Concurrency: waiting without colouring

There is no `async` and no `await` in Spite, and there never will be. In JavaScript or C# a function that waits
declares itself `async`, which changes its return type and forces every caller to `await` it, and every caller's
caller, all the way up: the waiting leaks into every signature above it. In Spite a function never says whether it
waits, and a program never says what runs at once: the compiler decides.

## Why one model, and no runtime to bring

Three things went wrong in other languages, and Spite avoids all three by where it puts the decision.

The first is the colour of a function. When the callee declares itself `async`, every caller has to change with it,
transitively, and every return type changes too. In Spite a call is a call: it waits where it is written, or the
compiler lets other work run while it waits, and no signature changes either way.

The second is the runtime you bring. When the language only describes a future and leaves the thing that drives it
to the community, several runtimes appear, and every library has to say which one it was written for. Spite has one
model, in the compiler, and no protocol for an outside driver to implement. A library cannot ask for something
else to run its work, because nothing else exists to ask.

The third is the person choosing. Which calls are independent is a fact about what each one reads and writes, and
the compiler reads every function a call reaches; a person guesses, and a guess that is wrong is a race. So there
is no form to ask for threads or for waiting work: `Parallel(...)` and `Concurrent(...)` are not part of Spite, and
writing either is a compile error that says to call the function itself.

Whatever the compiler runs at once, the program prints what running it in order prints. Where it cannot yet prove
work independent, the work simply runs in order.

## What runs at once

### Reads in a row overlap

When two or more declarations in a row each read a `File` (`read_as_string()`) or a `Tcp.Connection`
(`read_line()`), the compiler starts every read but the last, runs the last one, and waits for all of them before
the next statement. The program waits for the slowest file rather than for each in turn, and nothing it does
afterwards can tell: the values are the same, and a file written by the next statement is written after the reads.

```gdscript title=reads_in_a_row/reads_in_a_row.spite entry
var console = Console()

func ReadsInARow() {
    load "../spite_standard_library/windows"
    var settings_file = File(".spite/documentation_settings.txt")
    var scores_file = File(".spite/documentation_scores.txt")
    crash settings_file.write_string("volume=7")
    crash scores_file.write_string("ada=12")
    var settings = settings_file.read_as_string()
    var scores = scores_file.read_as_string()
    crash settings_file.write_string("volume=8")
    crash settings
    crash scores
    console.print(settings, scores)
}
```
```output
volume=7 ada=12
```

The two reads above ran at once. The rule is deliberately narrow: only reads, only side by side, and only when
the name read into is not assigned again later, so starting them early cannot change what the program sees. A
read followed by other work waits where it is written, and so does a name that is assigned again later, so starting
a read early never changes what the program sees.

### Calls in a row run at once

When two or more declarations in a row each keep the number a call answers, and the calls share nothing one of
them writes, the compiler runs them on threads at once:

```gdscript
func tick() {
    var physics_steps = physics.simulate(frame)
    var animation_steps = animation.simulate(frame)
    var audio_steps = audio.simulate(frame)
    console.print(physics_steps, animation_steps, audio_steps)
}
```

If each `simulate` only computes a number from its arguments, the first two run on threads of the program's pool
and the third on the calling thread, and each is waited for right before the first statement that reads its
name (or that holds a `return` or a `crash`), here the `console.print`. A call whose argument reads the name an
earlier one of the row declared stays in order. The receiver is `this` or an attribute of it, and the shape is
exactly this one: a `var` per call, with no written type. The program reads as one step after another and runs as
fast as the machine allows. The compiler decides it all while compiling, from what each call
reads and writes through every function it reaches, so a program with nothing to overlap carries none of it.

Calls whose order can be seen stay in order: anything that prints, touches a file or a socket, or reads and writes
the same state as another call. A call that does little (a few additions, no loop) is never handed to another
thread, since starting it there would cost more than running it. Two objects of one class count as one, unless
each belongs to its owner: two `Counter`s passed around and counted through any name share the attribute `total`
as far as this rule sees, so they stay in order, while a `Physics` and an `Audio` that each keep their own
`var meter = Meter()`, never assigned anything else, overlap, meters and all. A call only takes
part when it takes and answers numbers, is quiet (it holds no object, text or list, makes no foreign call, is not
recursive and prints nothing) and weighs at least a million simple steps.
[optimizations.md](optimizations.md#calls-in-a-row-that-only-compute-a-number-run-at-once) shows one.

### A loop over different classes runs them at once

A list of a `type` holds objects of several classes, and `systems.each_update()` (or `systems.each(update_one)`,
where `update_one`'s whole body is `system.update()`) calls the same function on each. When `update` takes nothing,
answers nothing, and the compiler proves the `update`s of two classes touch nothing the other writes (the same
reckoning as calls in a row, and neither writes the list), those elements run at once on threads, and all of them
finish before the next statement. When the statement runs, it looks at the classes the list holds: only when no
two elements are of one class does it run them at once, and otherwise it runs the loop in order, as written. Two
objects of one class never run at once. A list whose classes could never run together is the plain loop, with
nothing of this in the program.

### `each_` on a long list runs in bands

Write `particles.each_step()` and nothing more. The compiler runs the walk in bands across the processors when it
proves each pass touches only its own element and the list is long enough for the bands to pay, and in order
otherwise, with the same output
([optimizations.md](optimizations.md#a-loop-whose-passes-write-only-their-own-item-runs-in-bands)). A pass may
make objects and lists of its own, keep them in its element, and read anything no pass writes (a shared
table): what one pass makes no other pass can see. Uneven elements balance on their own: the list is cut into small
chunks and a thread that finishes one takes the next
([optimizations.md](optimizations.md#a-band-that-finishes-early-takes-the-next-chunk)).

A pass may take `fallback`s, and call a singleton's function that only changes it (one that answers nothing, given
at most two numbers, `Boolean`s, enums or texts written in the source, like `journal.note("skipped")`): a band
writes those calls down and the program's own thread makes them after the loop, in the list's order, so the
singleton ends exactly as running in order leaves it, with no lock
([optimizations.md](optimizations.md#a-call-that-only-changes-a-singleton-waits-for-the-join)). Write the singleton
plainly: the compiler picks how threads share it. A `--trace-fallbacks` build prints a banded loop's fallback lines
when the loop ends, in order.

### A loop over a list's positions runs in bands

```gdscript
func reshape(shapes: List<Shape>) {
    var index = 0
    while index < shapes.count() {
        var shape = shapes[index]
        var worker = Worker(shape)
        shapes[index] = worker.transformed()
        index = index + 1
    }
}
```

A `while` that steps a counter by one to a list's `count()`, takes the item at that position first and steps the
counter last runs in bands too, when every pass writes only its item, what the item alone reaches (its lists, the
objects in them) and what the pass made (the `Worker` and its scratch lists), and writes back only its own slot.
Whether two items share something a pass writes is known only when the loop runs, so the loop first checks that
each item, and everything a pass writes through it, is held once; anything shared runs the whole loop in order.
A pass may take `fallback`s, as in `each_` walks. This is how the compiler's own passes over its functions spread across the
processors ([optimizations.md](optimizations.md#a-loop-over-a-lists-positions-runs-in-bands)).
`--optimization-report` lists every such loop heavy enough for bands that still runs in order, under "Loops run in
order", with the first thing its passes do that keeps them in order and the function that does it
([compiler.md](compiler.md#read-what-was-not-optimised)).

### The first crash stops the program

Work that runs at once may crash, and the first crash that happens stops the program at once: its report is
printed and nothing else runs. When two pieces of work would both crash, either report may be the one printed,
and exactly one is. The report ends with one frame line marked `started`, naming the line where the program wrote
the call that ran apart.

### No lock to write

Work the compiler runs at once never shares anything one of them writes, so it needs no lock, and there is no
lock, thread-local, atomic or scheduler class for a program to name (`Lock`, `Atomic<Integer>`, `ThreadLocal<Integer>`
and `Scheduler` are unknown, as any misspelt class is); naming anything in `Spite.Internal` is a compile error.
A program that starts a thread counts references with atomic operations where another thread may count them at
the same time, and every other program keeps the plain, cheaper counts: the code around a loop that runs in bands
counts plainly, since no band runs while it does
([optimizations.md](optimizations.md#plain-counts-where-no-other-thread-counts-a-class)).

## What the compiler does at a wait

A wait the compiler lets other work run past (a frame's call, below, or a read in a row) is a **state machine** the
compiler writes while compiling. Every function that can reach a wait is compiled a second time as a resumable
version: its locals and parameters live in a small frame on the heap instead of on the stack, and every wait
inside it is a numbered point the function can return from and later jump back to. Starting the work makes that
frame and runs it to its first wait; when the wait is over, the program's event loop runs it again from where it
stopped. There is no stack per piece of work, no stack switching and nothing to ship that a WebAssembly build
could not carry:

| The program writes | While it waits |
|---|---|
| `program.sleep(duration)` | the state machine returns, and is run again once its time has come |
| `Console.read_line()`, reading or writing a `File`, a `Tcp.Server`'s `accept`, a TCP connection's `connect`, `read_line`, `read_as_string` or `read_as_buffer`, a UDP `receive` | the one blocking system call runs on a helper thread of the program's thread pool, and the state machine is run again, on the program's thread, once it has returned |

Code that is not inside a state machine waits where it is: its wait runs the event loop itself, so the state
machines keep going around it until it is done. Only one piece of Spite code runs on the program's thread at a
time, and a state machine only stops at one of these points, so no two pieces of Spite code touch the program's
state at the same time. The helper thread runs the system call and nothing else.

**A network read always waits.** A TCP or UDP read has one form, and it waits for what it answers
([standard_library.md](standard_library.md#tcp)): there is no "nothing yet" for a game loop to poll every frame.

**When blocking is faster, the compiler blocks.** A program with no state machine has no event loop and no helper
threads: every wait is the plain blocking call. Inside a program that has them, a wait is the plain call whenever
nothing else could run, and a program whose only work runs in bands or `Parallel`s (no `Concurrent`, no frame's
call, no read in a row) has no state machine to run meanwhile, so every wait in it is the plain call. It is the same rule as everywhere else in Spite: you write what you mean, and the compiler
picks the fast way to do it.

### A wait in a frame does not hold the frame

A **frame loop** is a `while` whose passes sleep until the next frame: it calls `program.sleep`, itself or through
a function of its own object. The work such a loop runs often waits, for a file, a socket or a database, and a
frame should not stop while it does. So, in the loop and in every function of its own object it calls, a call
that waits and answers nothing, made on another object of the program (`report.fetch()`, `systems[index].update()`),
is started by the compiler as a state machine of its own: it runs until its first wait, the frame goes on, and
the rest of it runs while the loop sleeps. Nothing in the program says so.

```gdscript title=frame_waits/report_file.spite
var program = Program()
var text = ""

func fetch() {
    var a_hundred_milliseconds = Duration(100, 'milliseconds')
    program.sleep(a_hundred_milliseconds)
    var file = File(".spite/documentation_report.txt")
    var read = file.read_as_string()
    crash read
    text = read
}
```
```gdscript title=frame_waits/frame_waits.spite entry
var console = Console()
var program = Program()
var report = ReportFile()
var frames = 0

func FrameWaits() {
    load "../spite_standard_library/windows"
    var one_millisecond = Duration(1, 'milliseconds')
    var file = File(".spite/documentation_report.txt")
    crash file.write_string("sales are up")
    while report.text.is_empty() {
        frames = frames + 1
        if frames == 1 {
            report.fetch()
        }
        program.sleep(one_millisecond)
    }
    console.print("loaded '{report.text}' while frames kept running:", frames > 1)
}
```
```output
loaded 'sales are up' while frames kept running: true
```

Written plainly, `report.fetch()` would have held the first frame for a tenth of a second, and the loop would have
seen the text on its first test. Here it returns at its sleep, and the loop draws frames until the text is there.

- **What stays the same.** The call does everything it did, in the order it is written; only when its waits end
  has moved, to later frames. A crash inside it halts the program with its own report, at its own line, as a crash
  anywhere does. A call whose function never waits is the plain call.
- **What it waits for.** Anything a call waits for: a sleep, a file, a socket. It moves on wherever the program
  waits, which in a frame loop is the frame's sleep.
- **How many at once.** Each call site keeps at most eight in flight. The ninth, made while eight from the same line
  still wait, waits in place for the oldest of them, as the plain call would have waited for itself: the frame is
  held, never a call dropped.
- **None is lost.** A started call nobody waits for still runs to its end: the next time its line is reached, the
  ones that finished are let go, and the program does not end until every one has finished.
- **The frame's own waits stay.** Only a call on another object is started. The loop's sleep, and anything that
  waits in a function of the loop's own object, is the frame's pace and waits where it is written, and so does a
  call whose answer is used.
- **No write is lost.** The call and the frame now take turns at their waits, so a value one of them read before
  a wait may be out of date after it. When either side would write such a value back (`var seen = count`, a wait,
  then `count = seen + 1`), the call is not started: it waits where it is written, as it always did, and
  `--optimization-report` lists it under "Waits that hold the frame". The value can be a number, a `Boolean`, an
  enum value or text; a parameter read before the wait and stored after it counts, so does one statement that
  reads an attribute, waits and writes it (`total = total + waited()`), and so does handing such a value to a change
  of a list or dictionary. Writing into an object the call made itself is no hazard, since the frame cannot see
  it. Read the value again after the wait and the call is started.
- **Which calls are started.** The receiver is an attribute of the frame loop's class, a path of attributes from
  it, or an item of a list attribute read by a name or a number, and its class is an ordinary class of the program,
  not a singleton. A call with arguments, or whose answer is used, or made on the loop's own object, waits where it
  is written. The frame loop may reach `program.sleep` through any depth of calls to functions of its own class.

## The REPL answers at the waits

A `--repl-port` build (see [repl.md](repl.md)) serves its commands **on the program's own thread, at the same
waits**. The socket thread only reads a command and hands it over; the scheduler answers it the next time the
program waits, and hands the answer back. Every command therefore sees the program between two steps, never in
the middle of one: a frame loop that ends in `program.sleep` is answered between frames, a tool waiting on
`Console.read_line` is answered while it waits, and once the entry constructor returns, the program waits for
nothing but commands until a client sends `exit`. None of this is written by the program.

```gdscript title=frame_loop/frame_loop.spite entry
var console = Console()
var program = Program()
var frame = 0
var running = true
var _frames_to_run = 200

func FrameLoop() {
    load "../spite_standard_library/windows"
    var ten_milliseconds = Duration(10, 'milliseconds')
    while running and frame < _frames_to_run {
        frame = frame + 1
        program.sleep(ten_milliseconds)
    }
    console.print("stopped")
}
```
```output
stopped
```

This session runs against the program above while its loop runs:

```wire frame_loop
$ spite --connect=4000 --command="program.running"
{"ok":true,"value":"true","type":"Boolean"}

# Answered between two frames: the loop sees the change at its next test.
$ spite --connect=4000 --command="program.running = false"
{"ok":true,"value":"false","type":"Boolean"}

$ spite --connect=4000 --command="exit"
{"ok":true,"value":"","type":""}
```

A program that never waits is answered too. In a `--repl-port` or `--hot-reload` build, and only there, the
compiler ends every pass of every `while` in the program's own code with a **check point**: one load of a flag
that the REPL's thread (or the file watcher) raises when a command (or a changed file) is waiting; only then does it
answer it there, on the program's thread, between two passes of the loop. A busy loop is served between two passes the way a frame loop is served between
two frames:

```gdscript title=busy_loop/busy_loop.spite entry
var console = Console()
var passes: Long = 0
var running = true
var _passes_to_run: Long = 300000000

func BusyLoop() {
    load "../spite_standard_library/windows"
    while running and passes < _passes_to_run {
        passes = passes + 1
    }
    console.print("stopped after at least one pass:", passes > 0)
}
```
```output
stopped after at least one pass: true
```

```wire busy_loop
# The loop never waits: the command is answered at the end of a pass.
$ spite --connect=4000 --command="program.running = false"
{"ok":true,"value":"false","type":"Boolean"}

$ spite --connect=4000 --command="exit"
{"ok":true,"value":"","type":""}
```

A build without those flags has no check points at all, and its C is byte for byte what it would be if check
points did not exist. The standard library's own loops have none either, so a command waits
for a library call to return.

## What it costs

Everything on this page is chosen per program, from what the compiler finds in it, and a program in which it finds
nothing carries none of it; the compiler's side of each is on
[optimizations.md](optimizations.md#concurrency-machinery-only-where-it-is-used).

| The program | Carries at run time |
|---|---|
| has nothing the compiler overlaps | nothing: no scheduler, no state machine, no helper thread, no thread pool, plain reference counts; every wait is the plain blocking call |
| has a wait the compiler lets other work run past (a frame's call, two reads in a row) | the scheduler and its event loop; a second, resumable copy of each function that work reaches that waits (the plain copy is dropped when nothing calls it); one heap frame per waiting call made inside it; the thread pool, whose threads run each blocking call made while something else could run (a new thread is started only when none is waiting, and stays in the pool); atomic reference counts for the classes another thread can count |
| has calls in a row, a loop over classes or a loop in bands that the compiler runs at once | the thread pool, one start per call or band handed to it and a small, fixed number of allocations |
| is built with `--repl-port` or `--hot-reload` | the scheduler, and one check-point call at the end of every pass of every loop of its own code; nothing of either in any other build |

What is never there: a stack per piece of work, a stack switch, a runtime that a WebAssembly build could not carry.

---

Next: [Standard library](standard_library.md), the classes every program can name, and what each one does.
