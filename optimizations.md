# Optimisations the compiler makes on its own

The compiler makes a program faster and smaller without being asked. This page lists what it does for a program: when
each optimisation applies, when it does not, and what, if anything, you could ever notice. For almost all of them the
honest answer is: **you do not need to do anything**. Write the plain program; the compiler does the rest, and the
program means exactly what its source says. None of them changes what a program prints, returns or crashes with: only
its time and its memory.

Why the compiler does this instead of you is on [Write it plainly](write_it_plainly.md). Here is what it does.

**Every optimisation stands on a proof.** The compiler only changes how a program runs where it has proven, while
compiling, that the change cannot alter what the program prints, computes or crashes on. [Proofs](proofs.md) states
each proof in full: its rule, when it does not hold, and what to write then. Where a proof does not hold, the plain
form is generated, which is slower but never wrong.

Two rules decide what belongs here.

**Zero runtime, and everything tree-shakeable.** A program that does not use a feature carries none of it. Nothing
needs a scheduler, an interpreter or a registry shipped beside the program: the work is done while compiling
instead, and what only the run can know is the smallest check that can decide it. REPL, live reload and debugging
features may cost something while the program runs, but only in the builds that ask for them. Those builds are
slower on purpose, so measure performance only with a production build
([Measure only a production build](compiler.md#measure-only-a-production-build)).

**Hidden optimisations are good, hidden costs are bad.** Code that runs faster than you expect is a free win, so the
compiler optimises silently and never asks you to mark anything. Code that runs *slower* than you expect is the only
real surprise, so every cost that remains is written down on this page, under the optimisation it belongs to, and
`--optimization-report` lists every place in your program where one of them fell back
([compiler.md](compiler.md#read-what-was-not-optimised)).

**Internals stay ordinary objects where you inspect them.** A `--repl`, `--repl-port`, `--hot-reload` or
`--development` build is an *inspectable* build: nothing is tree-shaken, every attribute of your classes keeps its
declared place, and the optimisations that hide something (tree shaking, static singletons, folded dictionaries,
shrunk layouts, ranges that drop checks) are not applied, because code added later may read any of it. Everything
else on this page changes how fast the program runs, not what it is made of.

What you can observe at all is short: the counts `--debug-memory` prints, when a singleton's constructor runs, the
order in which a fused chain calls your member functions, the frame in which a call a frame loop started finishes,
the size of the executable, and speed. Apart from those, no optimisation changes what a program prints or computes.

The page is grouped by what the compiler is working on.

## What the program carries

### Tree shaking

After the program is built, the compiler keeps only what the entry can reach: every function nothing calls, from your
classes or the standard library, is dropped. So is every class nothing reachable uses, with its singleton slot, its
free and copy functions and its text literals. A program that never makes a `Tcp.Server`, a `Process` or a
`FileSystemWatcher` has none of their code, and a native symbol is looked up only when a function that calls it
survived.

A function nobody calls, outside a generic class, is still compiled and checked, so a mistake in it is still
reported; it just is not in the executable. An inspectable build keeps everything, so live reload has every function
to swap and the REPL can reach every class ([compiler.md](compiler.md#development-builds-and-tree-shaking)).

### Concurrency machinery only where it is used

The scheduler, the state machines, the thread pool and the table that bounds calls in a row are written for a
program only when the compiler finds work in it to run at once
([concurrency.md](concurrency.md#what-it-costs)); a program in which it finds none carries none of them, not even
an import of a thread function. The pool is made at the first piece of work the compiler starts on it (a
[call in a row](#calls-in-a-row-that-only-compute-a-number-run-at-once), a
[band](#a-loop-whose-passes-write-only-their-own-item-runs-in-bands)) and keeps its threads for the next.

### Conditions decided while compiling

A condition the compiler can answer while compiling is answered then, and only the branch taken is built: a `Build`
field, a question about a class (`$type.is_fixed_size`, whether a function waits, how many arguments it takes), a
class test the value's type answers, and `not`, `and` and `or` over them. The branch not taken is not in the program,
and in a generic class it is not even required to compile for that class: in `Store<String>` the
`value * 2` below is never built, so it is never refused.

```spite
generic $value_type

func doubled(value: $value_type): String {
    if $value_type == Integer {
        var twice = value * 2
        return "twice {twice}"
    }
    return "{value}{value}"
}
```

What you can observe: nothing but the time, and a branch that could not be built for a class is no longer refused.

### Walks over classes, values and members are unrolled

A walk whose list is known while compiling is no loop in the program: the compiler writes one call per element.
`Namespace.classes.each(walker)` makes one copy of `walker` for each class of the folder, in which the parameter is
that class (so `Runner<system>()` and `system.name` are constants). `Enum.values.each(walker)` calls `walker` once per
value, with the value as a constant. A template's plural (`fill_attributes(row)`) is one call of the template's copy
for each member it ranges over, and a walk over a function's `arguments` is unrolled when its walker needs the item
known. Nothing of a walk is left at run time: no reflection object is made, no list of classes or values is kept,
and a walker nobody walks is not compiled.

```spite
func Schedule() {
    System.classes.each(add_system)
}

func add_system(system: Spite.Class) {
    var runner = Runner<system>()
    runners.append(runner)
}
```

What you can observe: the calls come in the order of the folder's files and of the enum's values, exactly the order
the walk at run time would take. It applies when the namespace or the enum is named directly and the walker is a
function of the same class; a list of classes reached at run time is walked as an ordinary list of reflection
objects.

### A dictionary written out and only read by literal keys is folded

A local made from a dictionary literal whose keys are all text or all whole numbers written out, whose values are all
literals, and that its function only ever reads with a key written out (`points["gold"]`) is never made: each read is
the value written for that key. The read keeps its type, a `T?`, so `crash points["gold"]` and `if` narrow it as they
would, and after the `crash` the read is the value itself. A read of a key the literal does not have is known to be
`null`, so a `crash` on it is a compile error.

```spite
func points_for(golds: Integer, silvers: Integer): Integer {
    var points = {"gold": 50, "silver": 20}
    crash points["gold"]
    crash points["silver"]
    return golds * points["gold"] + silvers * points["silver"]
}
```

`points_for` makes no dictionary and hashes nothing. It does not apply to a key or a value that is not a literal, to
any other mention of the local (a key in a variable, a write, `count()`, passing it), to a dictionary held in an
attribute, or in an inspectable build, which keeps the dictionary a reader can inspect.

### A serializer's keys are folded into its code

A `JsonWriter` or `JsonReader` given a `keys` map written out where it is set holds no lookup at run time: each
attribute's copy of the write and read functions uses the text the map gives it, or the attribute's own name for one
the map leaves out. Every map set on the same serializer class must be the same, set before the serializer is
compiled and never changed through an index; otherwise the program does not compile.

### A function only a replaced call reached is not compiled

When the compiler replaces a call (a [fused chain](#a-chain-of-filter_-steps-runs-as-one-loop), a loop
[in bands](#a-loop-whose-passes-write-only-their-own-item-runs-in-bands), a folded dictionary), the function that call
reached is not compiled if nothing else reaches it, and neither is what only it called. The executable carries only
code the program can run. A constructor, `drop`, `to_string` and `to_debug` are always kept.

### A fault report is the only machinery in every program

A native fault (a null read inside a foreign library, a stack overflow) happens whatever a program uses, and a silent
end is a bug, so every program installs a handler that reports it ([failure.md](failure.md#what-a-native-fault-reports)).
It is fixed code, not a runtime system: nothing runs until a fault. Beside the executable the compiler writes a
`.functions` file listing where each function starts and ends; nothing of it is in the executable. The report reads
that file and names the function that faulted and its callers; without the file it prints only the fault line. A
`crash` or `fallback` site's condition text is not in the executable either.

## Counting

Every object is freed by counting who holds it. Most of the counting is decided while compiling, and what is left is
as cheap as the compiler can make it. You can see the effect in the allocations and frees `--debug-memory` prints:
the balance is always the same, and some of the allocations are gone.

### An argument its caller holds is passed without counting

A function's parameters are held by its caller for the whole call, so the function does not count them: it reads
them as they are and lets nothing go when it returns. The caller holds what it passes when it is a name of its own,
`this`, a singleton, or an object it has just made. A function counts a parameter only when it assigns it again, or
when it runs as a step of a state machine (its frame outlives the call that started it).

An argument read from a place the call may assign (an attribute the callee, or anything it calls, writes; a list
slot; a back reference; the answer of a wait) is counted by the caller before the call and let go after it, so the
call cannot free it under the callee. What you can observe: nothing but time. `drop()` runs at the same moment as
before.

### An attribute a call cannot assign is passed without counting

An argument or receiver that is an attribute, or a path of attributes (`settings.limits`), is passed as it is when
nothing the call can run assigns any attribute along the path. The compiler works out, for every function, the
attributes it assigns and the functions it calls, and follows the calls to the end, so the question is answered for
everything the call can reach.

```spite
var readings = List<Integer>()
var settings = Settings()

func measure(at: Integer): Integer {
    return reading_at(readings, at) + offset_of(settings.limits)
}
```

`reading_at` and `offset_of` assign neither `readings`, `settings` nor `Settings.limits`, so both are passed
uncounted. It does not apply when the call, or anything it reaches, assigns an attribute on the path (or one of that
name through a shape), calls a function value, runs a `drop()` that may assign it, waits, or hands a function to C
that may call it back; a path through a back reference is always counted. What you can observe: nothing but time.

### A name holding an item or an attribute is not counted

`var held = this.part`, or `var point = points[position]` once the read is proven, does not count `held` when
nothing after it in its block can write the slot it came from: no assignment of an attribute on its path, no call
that may write one, and no write into a list of that class. The name is then read as it is and lets go of nothing
when its block ends. This is what lets a list's templates (`contains`, `index_of`, `join`, `copy` and the others
handed a function that cannot write the list) read their items without counting them, and a loop that reads
`var point = points[position]` and only reads its attributes neither counts nor lets go of each point.

An `append` is not such a write: it fills the empty slot past the end, so it never replaces an item a name holds. A
loop that reads `var item = items[index]` and appends to another list of the same class holds each item without
counting it.

It does not apply when the name is assigned again, when something after it in the block may write the slot, or in a
step of a state machine. What you can observe: nothing but time.

### An item passed to a call that cannot change its list is not counted

`buckets[customer].append(order)`, after a `crash buckets[customer]` that proves the read, passes the bucket as it
is in its slot, with no count taken, when nothing the call can run writes an item of a list of that class and the
list itself is a path nothing assigns. The same holds for an item passed as an argument: `length_of(tracks[at])`,
after `crash tracks[at]`. It does not apply to an item read without its `crash`, to a call that may write a list of
that class or reach a `drop()` that does, to a call through a function value, or in a function that waits.

### Storing an object into a list counts it only when it changes the slot

Writing an object into a list's slot compares it with what the slot holds. When it is the same object, nothing is
counted and nothing is written.

```spite
var held = names[at]
crash held
names[at] = held
```

### An item written back to the slot it came from is not written

```spite
func step() {
    var index = 0
    while index < particles.count() {
        var particle = particles[index]
        particle.left = particle.left + particle.speed
        particles[index] = particle
        index = index + 1
    }
}
```

`particles[index] = particle` writes that very name back into the very slot it was read from, with nothing between
that could replace the slot. The slot holds that object already, so the write, its test of the index and its
comparison are left out. It applies in one block to a list of the library, read from a name, `this` or attributes
nothing assigns, and to a call of a function whose whole body writes an attribute back into a list's slot when the
call before it on the same receiver ended with the matching read. A write in another block, through another path or
index, or after something that may write the list is made as written.

### A list item read only to test it is not counted

`list[index] is present` on a `List` of items that cannot be null is the test
`index < count`: the item is never read, so it is never counted and let go. `crash velocities[entity] and
healths[entity]` is two such tests. It does not apply to a list of nullable items (an item there can be missing
inside the bounds) or to a `Dictionary`.

### A singleton a function answers is not counted

A singleton lives from its first use to the program's end, so its count never decides when it goes. A function whose
declared answer is a singleton class returns it without counting it, and a local declared from a singleton and never
assigned again is the singleton itself, read as it is. It does not apply to a function whose declared answer is a
union, a shape or a nullable, or to a local that is assigned again.

### Who holds an object is decided while compiling

The compiler works out, for the whole program, which parameters a function *keeps*: a parameter nothing assigns,
whose last read is a place that keeps it (an attribute, the slot past the end of a list, the answer, or a parameter
another function keeps). Its class must free quietly: no `drop()` in it or in anything it holds. A call of such a
function hands it the count: an object the caller just made goes in as it is and is not let go after the call, and
the place that keeps it does not count it again.

```spite
func one_round(round: Integer): Integer {
    var items = List<Item>()
    var index = 0
    while index < 50000 {
        var owner = Owner((index + round) % 90)
        var item = Item(index % 1000, owner)
        items.append(item)
        index = index + 1
    }
    return items.sum_price()
}
```

`Item` keeps its `new_owner` and `append` keeps its value, so the loop counts neither: each object is made with its
one count, the one its holder keeps. The same holds for a local whose last read is a place that keeps it.

An attribute a constructor assigns first holds nothing yet, so it is stored without reading what it held and letting
that go; so is the slot past a list's end that `append` writes. It does not apply to a name read again later, read in
a `while` condition or inside an `if`, to a class with a `drop()` in what it holds, or to a back reference, a union,
a shape or a nullable. An object is freed at the same moment as before or earlier; nothing it holds has a `drop()`,
so that moment cannot be seen.

### An object that outlives the function is not counted in it

What a singleton the entry class makes holds, and an attribute of it that never changes, is held by the program for
the whole run. A function that keeps such a value in a local need not count it. A count taken and let go with
nothing between them that could let anything go (an item read into a row and let go when the row goes, an argument
counted for an inlined call) is removed together.

### An attribute that only holds an item of its holder's list is not counted

An attribute of a singleton that is only ever given an item of a list the same singleton holds
(`open = pages[at]`, proven by its `crash`) holds that item without counting it: the list holds it already. Its
default is then never made either, when every read of it comes after a write. This needs a class that frees quietly
and every write of the list to be an `append` or the matching write-back, and it does not apply to a program that
makes work on other threads.

```spite
singleton

var pages = List<Page>()
var open = Page(0)
var at = 0

func take(index: Integer) {
    at = index
    crash pages[at]
    open = pages[at]
}
```

### Letting go of an object of a known class is direct

Where the class of an object is known while compiling (an attribute or a list item with a declared class), letting it
go takes the count down in place and calls the class's own free function directly; only an object of an unknown class
looks its free function up from its class number. The release is written where it stands, not as a call.

### Plain counts where no other thread counts a class

A count is a plain addition and subtraction unless the program can run code on another thread: one in which the
compiler runs work on another thread, or that makes a `ForeignCallback`, which C may call from a thread of its own. Only then are counts atomic, and
even then only for the classes another thread can count. The compiler finds every class whose objects code on another
thread can name, hold or reach ([no other thread counts a class](proofs.md#no-other-thread-counts-a-class)), and every
count of any other class stays plain, in the whole program. A count whose class is not known while compiling (a union, a shape, a
function value) stays atomic in a program with threads, and a program that makes a `ForeignCallback` counts every
class atomically.

## Where objects live

### Objects that never leave their function live in registers or the frame

After inlining, the compiler follows every object from where it is made. An object that never leaves its function (it
is not passed to a call that keeps it, stored in memory, answered or waited on) is never allocated:

1. One object at a time (a loop that makes one row, one copy or one buffer each pass and lets it go before the next):
   every attribute the function touches is a register.
2. Several objects alive together: each gets its own registers.
3. Otherwise, when the class lets go of nothing when freed and is at most 4096 bytes, the object is room in the
   function's frame, and its counts are dropped.

An object answered by `return` is made at the return from the registers, so a function that builds its answer through
a loop of temporary objects makes one object, not one per pass.

```spite
func simulate(steps: Integer): Offset {
    var position = Offset(0.0, 0.0, 0.0)
    var velocity = Offset(1.0, 0.5, 0.25)
    var step = 0
    while step < steps {
        var moved = velocity.scaled(0.001)
        position = position + moved
        step = step + 1
    }
    return position
}
```

`scaled` and `+` are inlined, `moved` and every `position` but the answer live in registers, and the one object made
is the answer, at the `return`.

It does not apply to an object whose class has a `drop`, is held by back references, to one of more than 32
attributes, or to anything in a function that waits. What you can observe: `--debug-memory` counts fewer allocations
and frees.

### An object let go before anything reads it is never made

A row declared `var row: $row_type = null` holds a new object for each attribute until something assigns it. Once the
fill is inlined, each of those objects would be made and let go with nothing reading it in between. Such an object is
never made, when its class frees nothing but its own memory.

### A local list of known size lives in the frame

A list literal kept in a local (`var weights = [4, 3, 2, 1]`) that the function only counts and reads by index is
never made: its items are written into room in the function's frame. No list, no buffer and no count is allocated.

```spite
func largest_remainder(round: Integer): Integer {
    var remainders = [round % 10, round % 7, round % 13, round % 3]
    var weights = [4, 3, 2, 1]
    var biggest = 0
    var index = 0
    while index < remainders.count() {
        crash weights[index]
        var weighted = remainders[index] * weights[index]
        if weighted > biggest {
            biggest = weighted
        }
        index = index + 1
    }
    return biggest
}
```

It applies to a literal of numbers of at most 4096 bytes, never assigned again, whose every use is `count()` or
`list[index]`. A list passed to a call, answered, kept in an attribute, appended to or held by a function that
waits is made as before, and so is a list of objects or texts. An index outside the list still stops the program with
the same report.

### Objects of one class sit together

A `List` of a class of the program whose objects are only ever put into it fresh and read back from it holds the
objects themselves, one after another in its storage, instead of a pointer to each. `append` copies the new
object's bytes into the next slot and `list[index]` is the slot's address, so a walk over the list reads memory in
order and loads no pointer.

The compiler reads the whole program and holds a list this way when the element class has no `drop()`, is not held by
a back reference, holds only numbers, `Boolean`s, enums or objects it holds inside it, and every use of the list is
`append` of a freshly made object, `count()`, a read for an attribute or a call of the class, or a template over its
attributes.

```spite
func sum_points(): Long {
    var total: Long = 0
    var position = 0
    while position < points.count() {
        var point = points[position]
        total = total + point.across + point.down
        position = position + 1
    }
    return total
}
```

When nothing ever needs one of the items whole, the storage goes one step further and holds **one array per
attribute**: `points[position].down` is a load from the `down` array, so a loop that reads one attribute reads only
that array. The program's code is the same either way.

What you can observe: time and memory only; the list's storage is larger by the objects it now holds, and an object
appended that is not made for the append is made, copied into its slot and let go of, so `--debug-memory` counts it.

### An object only its holder holds lives inside it

When an attribute's object can only ever be held by the one object that holds it, its bytes are laid inside the
holder. `Item` grows by the size of an `Owner`, `item.owner` is the address of that room, and writing it copies the
numbers of the new `Owner` into the room. An `Owner` made only to be handed to `Item` is then made in registers, so
making an item is one allocation instead of two.

```spite
var price = 0
var owner = Owner(0)

func Item(new_price: Integer, new_owner: Owner) {
    price = new_price
    owner = new_owner
}
```

It needs a class of plain numbers with no `drop()`, an attribute written only by the holder's defaults or by a
constructor from a parameter it keeps, handed a new object by every caller, and read only as a part of it
(`item.owner.age`). A read kept anywhere (a list, another attribute, an answer), compared, passed to a function or
copied, or a second holder for one object, keeps the attribute a counted pointer. If a write into the room is ever
given an object that is held elsewhere, the compile stops saying so.

### Each class is laid out by how the program uses it

A class is one class to the program and several layouts to the compiler. The program sees `Player` with its forty
attributes; the compiler lays out each object with what the program uses.

- **Untouched attributes have no slot.** An attribute no built function reads or writes, whose default is made of
  literals and the library's collections and texts, takes no space in its object, its default is not run, and nothing
  releases it. A default that calls anything else keeps its slot, so it still runs once per object.
- **Attributes nothing writes are rebuilt where read.** An attribute no function writes always holds what its default
  made. A number, a `Boolean` or an enum is the default itself (a constant when it is one literal); a text or table is
  built where it is read, when it cannot be told apart. A singleton is the singleton itself. A table rebuilt where
  read is built at each read, so a function that reads it in a loop builds it on every call. An attribute written
  anywhere, read through a union, or whose default is not pure keeps its slot.
- **Slots are dense and aligned.** The rest of each class is packed by size: 8 bytes (references, enums, `Long`,
  `Double`), then 4, 2 and 1, each at its natural alignment, so a `Boolean` costs one byte, not eight. Declared order
  holds within a size, so attributes declared together stay on one cache line.
- **Attributes read together through a union share one block.** When code reads a member through a union and every
  class of the union holds that member with the same type, it sits at the same offset in every one of them, so the
  read is one load with no test of the object's class.

```spite
union Combatant {
    Player
    Monster
}

func hits(fighter: Combatant): Integer {
    return fighter.attack - fighter.defense
}
```

`attack` and `defense` sit at the same two offsets in `Player` and `Monster`, so `hits` reads two numbers and never
asks which class it was given. The union block does not apply to a member a class answers with a function, a `T?`
number or a back reference.

What you can observe: objects are smaller, and a read through a union takes one load. Reflection still lists every
declared attribute, because it lists the declaration, not the slots.

### `String`'s words cost a text nothing

`String`'s irregular and uncountable words are attributes a program changes by reopening `String`. Every text is a
`String`, so the compiler never stores them in a text: one that is read is rebuilt wherever it is read. A text is 32
bytes whether the program inflects words or not. A program that writes one of them, or whose reopened default is more
than literals, is refused, because every text would then carry the tables and making the tables makes texts.

### Short text lives inside the `String`

A text is a `String`, a `Buffer` and the bytes. Every text is one allocation: the three sit together in one block. A
written text, a join and a first append get room for 47 bytes or for the whole text if that is longer, so a short
text never needs a block of its own for its bytes. When the bytes outgrow the room they move to storage of their own,
and the room inside is not used again. Freeing the `String` frees the block.

It does not apply to a program whose `String` keeps an attribute other than its buffer and length, or whose `String`
or `Buffer` has a `drop()`: it then makes the `String` and its `Buffer` apart. What you can observe: one allocation
per text and the time.

### Small blocks come from the program's own free lists

An object, a list's storage, a text and a task frame of up to 1016 bytes come from the program's own free lists, not
from a call into the system: a freed block goes back on the list of its size and is reused first. The size is
worked out while compiling when it is a constant. In a program with threads every thread keeps its own lists, so
no allocation takes a lock, and a block freed on another thread goes back to its owner's list.

Memory freed into a list stays with the program, so a program's peak of small blocks of one size is what it keeps.
Nothing is silent: freeing the same memory twice stops with `the program freed the same memory twice`, and asking for
more than the system has stops with `the program ran out of memory`. A program that allocates nothing carries none of
it, and only a `--debug-memory` build counts blocks.

### Defaults the constructor replaces are never made

A constructor that assigns an attribute before anything can read it makes its objects with a copy of the class's
defaults that leaves that attribute out, so the default is never made.

```spite
var values = List<Integer>()
var label = "none"

func Holder(start: Integer) {
    values = List<Integer>()
    values.append(start)
    label = "made"
}
```

`Holder(start)` makes neither the default list nor the default text. It applies to an attribute the constructor
assigns in its own body outside any `if` or loop before any statement that reads it or passes `this` on, when the
default is quiet: a constant, a text, or an object whose making assigns only its own attributes and whose class has
no `drop()`. What you can observe: fewer allocations.

### A deep copy is written per class

The compiler writes each class's `copy()` and `deep_copy()` when the class declares none: a new object made with no
defaults, with every attribute assigned from the original. Only a class whose attributes can reach the class itself
again copies through a table that remembers every object copied, so a graph is copied with its shape; every other
class's copy is a plain sequence of copies and stores. A class that declares its own `deep_copy()` is copied through
it.

### A list of lists cleared and filled with new lists keeps their room

```spite
func group() {
    buckets.clear()
    var made = 0
    while made < 32768 {
        var bucket = List<Order>()
        buckets.append(bucket)
        made = made + 1
    }
}
```

A `clear()` of a list of lists followed by a counted loop that only makes a new empty list and appends it empties
every inner list in place and keeps its storage, so the next fill appends into lists that already have the room
they grew to. An inner list held anywhere else is replaced by a new one, and every `drop()` runs in the same order.
It does not apply when anything else is between the `clear()` and the loop, or the loop does more than make and
append.

## Text and bytes

### Text joined in one piece

`"{first} meets {second} at the gate."` is one join, not a chain of pairs. The compiler works out every piece first,
left to right as written, adds up their lengths, makes the text once with exactly that room, and copies each piece's
bytes into it. A written piece is copied straight from the executable's data. A join written as calls
(`first.sum(second)`) is a call, and a program that replaces `String.sum` by reopening `String` keeps calling its
own. What you can observe: one allocation for the whole join.

### A number joined into text is written in place

Inside a join, an `Integer` or a `Long` piece is written as digits straight into the text being joined: the compiler
counts the digits before the text is made, so no text is made for the number. The digits are the ones `to_string()`
writes. It does not apply to a number that is the whole text (`"{index}"`) or a program that replaces
`Integer.to_string()`.

### Text appended to itself grows in place

`built = "{built}item {index},"`, `text = text + piece` and `joined = joined.sum(separator)` append to the text they
assign. When the text is held only by that name, the pieces are appended to it where it is, and the room is grown
once if they do not fit (at least doubled, so a loop of appends copies each byte a constant number of times). When
something else holds the text too, it is copied first and the name is given the copy, so what the other holder sees
never changes.

```spite
func words(count: Integer): String {
    var text = ""
    var index = 0
    while index < count {
        text = "{text}word {index} "
        index = index + 1
    }
    return text
}
```

It does not apply to a piece that names the text itself, a `String?` or a symbol, or an assignment to an attribute or
a list slot. What you can observe: the time, and a text's room, which can be up to twice its length.

### Bytes are copied in one block

The library's text and bytes functions (`slice`, `sum`, `replace`, `split`, `trim`, `upper_case`, `to_buffer`,
`Buffer.append_buffer` and the rest) copy runs of bytes as one block copy, not a call per byte. `equals`,
`starts_with`, `contains`, `index_of`, `split` and `replace` compare eight bytes at a time and look for a match only
where the first byte matches. A number's `to_string()` counts its digits, makes room once and writes them from the
last.

### A number read from bytes is one load

`Buffer`'s big-endian reads and writes (`read_integer_big_endian_at` and the rest, of every width) are one load or
store of that width with the bytes swapped, and a little-endian one is one load. The public read still tests the
bounds and answers `null` outside them.

### A number's own value is its register

A number is read through the bytes of its room, and the room is made only when its bytes are needed. A read of the
number's own width at offset 0 is the number itself: no `Buffer` is made, so `index.to_string()`, `var read: Long? = count`
and every other function of the numbers cost no allocation. `to_buffer()`, and a read at another offset or width,
still copies the bytes into a `Buffer`.

### A dictionary hashes a key once, cheaply

A `Dictionary` keeps its keys and values in two lists in the order the keys were first set, and over them a table of
slots, never more than half full. Every `get_at`, `set_at`, `contains` and `remove_at` hashes the key once and walks
the slots from the one the hash names; a key is compared only where its stored hash bits already match. Growing the
table moves each slot by the bits it keeps, so no key is hashed again, and an empty dictionary has no table.

A `String` key is hashed by its bytes eight at a time. A key that is not text (a whole number, a `Boolean`, an
enum, an object) is hashed from its bits by one multiply, with no text made for it; an object is hashed by its
identity, and a `Float` or `Double` is hashed after `0.0` and `-0.0` become one key and every not-a-number becomes
one key. What you can observe: the time, one allocation for the table once it holds a key, and `keys()`, `values()`
and `to_debug()` answering in insertion order as always. A read written twice (`if counts[word] { total = total +
counts[word] }`) still looks the key up twice.

## Arithmetic and checks

Every whole-number `+`, `-` and `*` halts when its answer does not fit its type, every read of a list halts outside
it, and every division halts on a zero divisor. The compiler leaves out each check it can prove cannot fire.

### Arithmetic a range proves is not checked

Where the compiler proves while compiling that an operation always fits, the check is left out and the operation is
one machine instruction. The proof is a range for each operand
([a range proves arithmetic fits](proofs.md#a-range-proves-arithmetic-fits)):

- a literal is its own range; a local or parameter of a whole-number type of at most four bytes has its type's range
  until a `var` or an assignment gives it its value's; a `Long` has no range until one is given to it;
- `count()` of a list or a dictionary, and `length()` of a `String`, run from 0 to 2147483647;
  `%`, `/`, `minimum`, `maximum`, `clamp`, `to_long` and `to_integer` are bound by their operands;
- a comparison narrows the local it compares inside a `while` or an `if` whose condition says so, on each side of an
  `and`, after a `fallback` or a `crash` of it, and after an `if` that leaves;
- the two branches of an `if` join, and before a `while` every local its body assigns takes its type's range, except
  a counter only ever stepped up by constants, which keeps its floor and, under `counter < bound`, is at most the
  bound plus its steps;
- a total assigned once in a loop (`total = total + term`) holds at most as many terms as the loop has passes. So a
  `Long` total of `Integer` terms over a list's items keeps no check.

```spite
func total(values: List<Integer>): Long {
    var sum: Long = 0
    var index = 0
    while index < values.count() {
        sum = sum + values[index]
        index = index + 1
    }
    return sum
}
```

`index + 1` is unchecked: `index` runs from 0 to 2147483646 under the loop's condition. `sum + values[index]` is
unchecked too: the loop runs at most 2147483647 passes, each adding an `Integer`, which a `Long` always holds.

An operand that is an attribute, a list item or a call's answer has its type's range, so an operation on two of them
keeps its check. An inspectable build proves no ranges, since code may run between two steps. What you can observe:
`--optimization-report` lists every check kept, with the ranges that did not prove it.

After inlining, the compiler also proves bounds on single values: a constant its value, a comparison 0 to 1, a
remainder by a positive constant `k` from `-(k - 1)` to `k - 1`, a mask with a constant from 0 to the mask, and a
sum, difference or product of bounded operands. Arithmetic a function receives from an inlined callee is checked
again in the caller, and this removes the check there when the bounds fit.

```spite
func score(value: Integer): Integer {
    return value * 2 + bonus
}
```

Inlined where `value` is `index % 100` and `bonus` is the constant 3, neither the product nor the sum is checked.

### Maths on constants is worked out while compiling

A `+`, `-` or `*` on whole numbers whose operands are each one value, read from constants, locals given a constant and
other such operations, is worked out while compiling and written as its answer. An answer that does not fit its type
keeps the operation, which halts as it would have. After inlining, the same holds for every operation left with
constant operands, and a branch on a constant becomes a jump.

```spite
func area(): Integer {
    var width = 12
    var height = width * 3
    return width * height + 4
}
```

`area` returns 436 with no arithmetic left. A division, a call and a decimal are handled by
[calls on constants are run while compiling](#calls-on-constants-are-run-while-compiling).

### Arithmetic by 0 and 1 is no arithmetic

A remainder by 1, and a product or a bitwise and with 0, are the constant 0, whatever the other side is; a division or
product by 1, and a sum, difference, or, exclusive or or shift by 0, are the other side. What follows folds too.

### A proven read tests only its bounds

`values[index]` where a bound proves the read (`while index < values.count()`) is written in place for a list whose
items are not nullable: one unsigned comparison of the index with the list's length, which halts the same way when
the index is below 0 or the list changed, and one load of the item. `count()` of a list, and
`length()` of a `String`, is one load. A list of `T?` items, of unions or of function values still calls `get_at`,
and so does a list whose `get_at` or `count` a program replaces by reopening the class. What you can observe:
nothing but time; the message of a read outside its list is the same.

### A bound a signed test already proved is not tested again

A read of a list's item tests its index against the count, unsigned, so a negative index fails too. When that read
comes after a signed `index < count` of the same two values (the loop's own test) with nothing writing either value
in between, and the index is never negative, the test is always true and is left out: the read goes straight to the
item. The same holds after an unsigned test that already passed, so `crash values[pick]` then `values[pick]` tests the
index once.

```spite
func add_up(values: List<Float>): Float {
    var total = 0.0
    var index = 0
    while index < values.count() {
        total = total + values[index]
        index = index + 1
    }
    return total
}
```

`values[index]` is the load alone: the loop's test already proved the index is inside the list. An index the tests
cannot prove is still tested and still halts with the same report.

### A proven divisor is not checked

A division or remainder whose divisor's range holds neither 0 nor -1 (`crash parts > 0` before `amount / parts`) is
written with no zero test and no -1 path, only the division. A division of 32-bit numbers divides in 32 bits, which is
faster than 64, whether it is checked or not. A divisor the range cannot prove is tested as before and halts with the
same report; `!= 0` proves nothing, since a range cannot leave out a middle value.

```spite
func shares(amounts: List<Integer>, parts: Integer): Long {
    crash parts > 0
    var total: Long = 0
    var index = 0
    while index < amounts.count() {
        var amount = amounts[index]
        total = total + amount / parts + amount % parts
        index = index + 1
    }
    return total
}
```

### Dividing by a constant is a multiplication

A division or remainder by a constant that is neither 0 nor -1 carries no checks, since they cannot fail. By a power
of two it is a shift or a mask (with the rounding toward zero that `/` has); by any other constant it is a
multiplication by a number worked out while compiling, a shift and a correction, exact for every value. A
multiplication by a power of two that is not checked is a shift.

```spite
func sum_of_remainders(): Long {
    var total: Long = 0
    var index: Long = 0
    while index < 1000 {
        total = total + index % 7
        index = index + 1
    }
    return total
}
```

`index % 7` is no division: a multiplication, two shifts, two additions and a subtraction.

### A division and a remainder of the same two numbers are one division

A division followed at once by the remainder of the same two numbers (or the reverse), with a divisor that is not a
constant, is one division: the processor answers both together. A checked divisor is tested once, and the one that
stops names the one written first, which is the one that would have stopped first.

```spite
func digits(value: Integer, base: Integer): Integer {
    crash base > 1
    var rest = value
    var total = 0
    while rest > 0 {
        var digit = rest % base
        rest = rest / base
        total = total + digit
    }
    return total
}
```

### A remainder by a power of two tested against zero is a mask

`x % 2^k` read only by `== 0` or `!= 0` is `x` with its low `k` bits kept: a whole number is divisible by `2^k`
exactly when those bits are zero. `if index % 2 == 0` is a single test.

### Arithmetic checks cost one jump on the hot path

A checked `+`, `-` or `*` jumps on the processor's overflow flag to code that writes the report, which sits after the
function's blocks, out of the path that runs. A division's zero check jumps out the same way. A 32-bit sum the
compiler has proven cannot overflow is one instruction.

### The maths functions are one call into the C library

`sine()`, `cosine()`, `tangent()`, the arc functions, `power`, `exponential`, the logarithms, `floor`, `ceiling`,
`truncate` and `round` on `Float` and `Double` are one call to the C library's function, with the operands in the
registers the system passes them in; `%` on decimals is the library's exact remainder, and `square_root()` is one
instruction. The executable imports the C library only when it calls one of them, and only the functions it calls.
The answers are the C library's, bit for bit.

## Functions and loops

### Small functions are inlined, and a function called in a loop more so

A call to a small function that waits for nothing (an accessor, a list's `get_at` and `set_at`, a tiny helper, a read
of a singleton) is replaced by a copy of the function's body. A call inside a loop is inlined when the callee is
written in the program and is up to 96 instructions long, or up to 400 when that call is the only call to it, so a
body run millions of times pays no call, its answer lands in the caller's registers, and an object it makes and
answers (a `Matrix4` product, a transformed `Vector3`) is
[kept in registers](#objects-that-never-leave-their-function-live-in-registers-or-the-frame) instead of on the heap.
A singleton's function, whose body claims and lets go of the singleton's lock, is copied with that lock.

```spite
func transform_all(model: Matrix4<Float>, point: Vector3<Float>): Float {
    var total = 0.0
    var step = 0
    while step < 1000000 {
        var moved = model.transform_point(point)
        total = total + moved.x
        step = step + 1
    }
    return total
}
```

`transform_point` is written into the loop, and the `Vector3` it answers is three registers: the loop makes no object.

It does not apply to a function that waits, makes a foreign call or starts a thread, to a function with a loop of its
own (that loop is [run in lanes](#a-counted-loop-over-numbers-runs-eight-at-a-time) or in bands where it stands), to a
caller that is already very large, or to a buffer's growth, which is the cold path of every append. What you can
observe: nothing in the output. A crash in inlined arithmetic still names the function it was written in, and the
executable holds the callee's body once per call site.

### A crash's report is a cold call

The code that reports a failed `crash` runs only once, as the program halts, so it is moved out of the function into a
function of its own that the guard calls. The passing path is unchanged and the function holding the guard is as small
as it was without it, so it is inlined. The report reads the same: the frames name the guard's function and its
callers as before.

### A call that returns at once on a test is tested in place

A call inside a loop to a function too large to inline, whose first block only reads and compares and then returns
nothing on one side, makes that test in the caller: the call runs only on the other side. A list's private `_reserve()` called
by every `append` is a compare with its room, and the call is made only when the list grows.

### The class of an object made in the function is known

An object made in the function has its class written once, right after it is made, and nothing writes it again. A test
of the object's class (a function taking a `type`, given a `Crate` or a number boxed for it, chooses what to run by its
class) is decided while compiling and the other sides are gone before objects are placed. A function taking a `type`
is compiled for the classes it is given, and a number, `Boolean` or enum value is never boxed except where it
travels as a shape.

```spite
func measure_all(steps: Integer): Double {
    var total: Double = 0.0
    var crate = Crate(1.0, 2.0, 3.0)
    var step = 0
    while step < steps {
        crate.width = step % 7
        total = total + doubled(crate)
        total = total + doubled(step % 10)
        step = step + 1
    }
    return total
}
```

Once `doubled` is inlined, its test of its argument's class is decided for `crate` and for the number, so each call is
plain arithmetic with no box made.

### A function value whose function is known is called directly

A call through a function value is a plain call when every function the value can hold is the same one: the value is
made and called in one function and the compiler follows its copies. A plain call can be inlined, so a function value
made and called in one loop costs what the function's body costs. A call that passes a closure the caller knows to a
function that calls it is given its own copy of that function, evaluated knowing the closure, so the function given
to `each`, `count` or any walk runs inside the walk with no call through a pointer. A function passed to a template
by name (`names.each(say_hello)`) is never made into a function value at all. The function value still describes its
arguments when asked. At most 64 such copies are made in a program, and the copy is kept only when knowing the
closure made a call direct or decided a test.

```spite
func turn_all(points: List<Point>) {
    var round = 0
    while round < 200 {
        points.each(turn)
        round = round + 1
    }
}
```

`points.each(turn)` calls a copy of the list's `each` in which `turn` is inlined.

### A walk over a list its setup filled calls each item directly

A class whose setup fills a list with objects it makes there, and nothing else ever changes the list, holds exactly
those objects in that order; a loop over that list that calls a function through the list's `type` on each item is
written as one direct call per item, with no test of its class. This is how a stage runs its passes: its setup
registers them, and its `run` calls them.

```spite
type Step {
    applied(Integer): Integer
}

var steps = List<Step>()

func Steps() {
    var doubling = Doubling()
    register(doubling)
    var adding = Adding()
    register(adding)
}

func register(step: Step) {
    steps.append(step)
}

func run(number: Integer): Integer {
    var current = number
    var index = 0
    while index < steps.count() {
        var step = steps[index]
        current = step.applied(current)
        index = index + 1
    }
    return current
}
```

`run` becomes `Doubling.applied` then `Adding.applied`. It applies when the list is read only by the one function that
appends to it and by such counted loops, is filled only from the class's setup, and each class's function has the
parameters and answer of the `type`'s ([a list its setup fills holds what the setup made](proofs.md#a-list-its-setup-fills-holds-what-the-setup-made)).
A `register` called anywhere else, or anything else touching the list, leaves the loop as written, testing each
item's class; nothing is refused.

### A chain of `filter_` steps runs as one loop

`lamps.filter_is_lit().sum_brightness()` makes no list in between: when a member template is called on the answer of
`filter_` (or a chain of them), the compiler writes one function on the first list's class from the last template's
body with its loop wrapped in an `if` per `filter_`, and calls that. The last step may be `filter_`, `count_`, `any_`,
`all_`, `sum_`, `each_`, `map_` or `find_by_`, and a `map_` may stand in the middle too:
`items.filter_is_active().map_owners().filter_is_adult().sum_age()` is one loop over `items` that names each item's
owner in the loop, with no list of owners made.

```spite
var total = lamps.filter_is_lit().sum_brightness()
```

calls `is_lit` then `brightness` on the first lamp, then on the second, and allocates no list for the filtered lamps.
What you can observe: the order. Each element's members run before the next element's. A passed function
(`filter(f)`) and a chain on a `Dictionary`'s values run step by step.

### Short loops on known counts are unrolled

A loop that is not nested around another loop, and whose number of passes the compiler can count from what it knows
at the loop's start, is written out as that many copies of its body when it runs at most 16 passes and the copies
hold at most 1024 instructions. Each copy then knows the counter's value, so the reads and tests on it fold: a walk
over a list of three known items becomes three straight pieces of code, each with its item known. A loop that ends
early on a value only the run knows is still unrolled, each copy keeping that test.

```spite
func match_into(entity: Integer, found: List<Integer>): Boolean {
    var index = 0
    var all = true
    while all and index < headers.count() {
        crash headers[index]
        var place = entity % headers[index]
        found[index] = place
        all = place >= 0
        index = index + 1
    }
    return all
}
```

With `headers` known to be `[1, 2, 3]`, the loop is three copies: `entity % 1` is 0, the others are divisions by
constants, and every bound check is gone. What you can observe: the time, and a larger executable.

### What a loop does not change is computed before it

An instruction in a loop whose operands are all written outside the loop is moved before it and runs once instead of
once per pass: constants, unchecked arithmetic, addresses, and loads of an attribute or a list's count and storage
nothing in the loop writes. In a program with threads a load another thread may write never moves, so a loop
waiting for another thread (`while not gate.is_open() { }`) still sees each new value, and a loop that tests an object
keeps the load where it is.

### An attribute a loop updates lives in a register until the loop ends

A loop that makes no call or allocation and writes an attribute keeps it in a register for the whole loop: the
attribute is loaded once before, each read is the register, and it is stored once at each way out. Two attributes
never share memory, so a store to one cannot change what a read of another reads. In a program with threads this
applies to the attributes of a singleton that takes its lock and to classes no other thread touches.

```spite
func advance() {
    var step = 0
    while step < 1000 {
        angle = angle + speed * 0.001
        if angle > 6.28318 {
            angle = angle - 6.28318
            turns = turns + 1
        }
        step = step + 1
    }
}
```

`angle` and `turns` live in registers for the thousand passes, `speed` is read once before the loop, and `angle` and
`turns` are written once, after it. A loop that stops the program halfway leaves the attribute as it was before the
loop, which nothing can read afterwards.

### A counted loop over numbers runs eight at a time

A `while` loop that steps a counter by one up to a bound, and whose passes only read and write numbers at the
counter's place in lists of numbers (a `List<Float>`, a `List<Integer>`, a list of objects kept as one array per
attribute), runs most of its passes in the processor's vector registers: eight 32-bit numbers (or four 64-bit ones)
per instruction. A vector loop is put in front of the loop, and the loop itself is left to run whatever passes
remain.

```spite
func scale_list(from: List<Float>, into: List<Float>, offset: Float) {
    var index = 0
    while index < from.count() {
        into[index] = from[index] * 1.5 + offset
        index = index + 1
    }
}
```

It applies when the test is `counter < bound` on a bound the loop does not change, the body is one straight run whose
only branches are the tests that halt the program, every value is a sum, difference, product, quotient or square root
of decimals, a sum, difference, product or bit operation of whole numbers that cannot overflow, a comparison or a
conversion, and nothing it computes is read after the loop except the counter and sums.

A few tests run each time the loop is entered: the vector loop runs only passes that cannot halt (the pass that would
halt runs in the loop as written and halts there, naming what it names today), a processor without the vector
instructions runs the loop as written, and a loop that writes one list and reads another tests that the two do not
overlap. A program with no such loop carries none of it. What you can observe: only time. Every number is computed by
the same operation in the same order as in the loop as written, so it is the same to the last bit.

A running total (`total = total + term`) is kept one partial total per lane and added at the end. For whole numbers
this is the same answer in any order. A sum that keeps its overflow check is added in wider lanes beside the sum of the
terms' sizes, and if the total could have overflowed along the way the vector loop's work is thrown away and the
loop as written runs every pass, halting where it would have halted. A decimal total is split into lanes only where
the compiler proves that no addition rounds: every term is a multiple of one power of two no larger than a bound,
and the total the loop starts with is such a multiple too. A term that may round (a division, a square root, a decimal
read from a file) keeps the loop's order, so the answer is always the same to the last bit.

### An `if` that only adds or picks a value has no branch

An `if` with no `else` whose body only adds to a running number, or sets a variable to a value, is run without a
jump: the test becomes a mask and the amount is cut by the mask, and a value picked is one conditional move. A test the
processor cannot guess (`if point.down >= round` over points in no order) then costs nothing when it is wrong.

```spite
var biggest = 0
var index = 0
while index < 4 {
    var weighted = remainders[index] * weights[index]
    if weighted > biggest {
        biggest = weighted
    }
    index = index + 1
}
```

It applies to a short body that only computes. A body that calls, stores, divides, makes or lets go of anything, or
that the test guards from a fault, keeps its jump. The body's sums still halt on overflow exactly when they would
have.

### A loop whose passes write only their own item runs in bands

`orbits.each_advance()` on a `List` of a class of the program runs on several threads at once, one band of the list
for each processor, when every pass writes only its own element: the compiler reads `advance` and every function of
the class it calls and finds that it writes only number attributes of the element and its own locals, reads only
number attributes of the element, its locals and literals, and calls only functions of the class that take and answer
numbers. There is no parallel form to write: the program writes `each_advance()`, and when the compiler cannot prove
the pass, the loop is the loop as written.

```spite
var angle = 0.0
var speed = 0.0
var turns = 0

func Orbit(seed: Integer) {
    speed = seed % 17 + 1
}

func advance() {
    var step = 0
    while step < 1000 {
        angle = angle + speed * 0.001
        if angle > 6.28318 {
            angle = angle - 6.28318
            turns = turns + 1
        }
        step = step + 1
    }
}
```

The program's own thread runs the first band and waits for the others, so whatever comes after the loop runs after
every pass, as written.

**When it pays.** Starting a band costs a thread, so the compiler weighs the pass while compiling: a pass that
weighs little never runs in bands (it is bound by memory, not by the processor), and for a heavier pass the loop
compares the list's count with the smallest count at which bands pay and runs in order below it. A list may hold one
object twice, and two bands would then write it at once, so the loop first reads each element's reference count and
runs in order when any element is held anywhere else too.

A `while` whose body is only such a call and the step of a counter up to a literal starts the bands once: each band
runs all the passes of its own elements, one element after another, and the counter ends at the bound.

What you can observe: the time and the same output. A pass whose checked arithmetic overflows still halts the program
with the same message, but the frames it reports are those of its band, and when several passes would overflow, the
one reported may be another than the first in the list's order. Not in an inspectable build.

### Calls in a row that only compute a number run at once

```spite
var left = mixer.blend(3)
var right = mixer.blend(5)
console.print(left, right)
```

When two or more declarations in a row each take the answer of a call that only computes a number, the compiler
starts every one but the last on another thread, runs the last itself, and waits for each where its name is first
read. A call only computes a number when its arguments and answer are numbers and, through every function it calls,
it reads only its arguments and its own locals, writes only its own locals, prints and opens nothing, makes no
object and records no fallback. It may crash. The calls may be your entry class's own functions too. A later call whose
arguments read an earlier one's answer stays in order, and so does any call the compiler cannot prove. Each call must
also weigh enough, the way a pass weighs for bands, or the row runs in order.

What you can observe: the time, and which crash reports. Such a call shows nothing, so what the program prints and
answers is what running the calls one after another gives. If one crashes, the first crash that happens stops the
program at once with its report, listing the frames of the thread that crashed, and nothing after it runs; when two
would crash, either report may be the one you see, and you see exactly one. Not in an inspectable build.

## Registers and the machine code

The compiler places values in registers and shapes the machine code. Nothing of this changes what a program computes;
it is listed so the costs you might measure are explained.

### Values live in registers

Every value of a function that does not wait lives in a machine register when one is free, and in the function's
frame otherwise. Two values share a register in each other's gaps, so a value read only in a report laid out after a
loop does not hold its register through the loop. When no register is free, the value whose uses weigh least for how
long it lives is spilled: a use inside a loop weighs far more than one outside, so a value that lives across a
whole loop and is read once in it goes to the frame. A value that cannot stay in a register across a call lives in
one everywhere but around that call, so a loop whose rare path grows a buffer keeps its counter and bounds in
registers on the path it takes every time and stores them only on the way into the call. A crash report reads each
number from wherever it lives.

### A decimal lives in a decimal register

A `Float` or `Double` value of a function that does not wait lives in one of the processor's decimal registers when
every instruction that touches it is decimal maths, a comparison or conversion, a copy, a load or store, a call's
argument or answer, or a return. Decimal maths then works on those registers directly. A decimal constant read only
by decimal maths is a value of its own made once before a loop, and gives its register up when a value needs it. A
decimal comparison read only by the branch after it is the branch, with a NaN comparing false on every side.
Turning a whole number into a decimal clears the destination first, so a loop that turns its counter into decimals
several times does not run its conversions one after another.

```spite
func scaled_sum(values: List<Float>, shift: Float): Float {
    var total: Float = 0.0
    var index = 0
    while index < values.count() {
        total = total + (values[index] + shift) * 1.5 + 0.25
        index = index + 1
    }
    return total
}
```

`total` lives in a decimal register for the whole loop: each pass is two additions and a multiplication, and the item
is moved in once.

### Jumps, copies and dead values

A set of small cleanups runs on every function, and each is a plain fact about the code:

- a jump to a block that only jumps on goes to where that block jumps, and a branch its predecessor decides goes
  straight to its side;
- a copy of a value that never changes is the value, and a value nobody reads, or overwritten before it is read, is
  not computed;
- `t = a + b` followed by `x = t` is `x = a + b`, which is what turns `index = index + 1` into one addition;
- the same unchecked computation, load or address repeated in a block, or across blocks when its registers are
  written once, is made once, and a value stored is read back from its register until something could have changed
  the memory;
- a comparison tested against zero is the opposite comparison, and a comparison read only by the branch after it is
  the branch;
- a block nothing reaches leads nowhere, and straight runs of blocks are one block;
- a count taken and let go with nothing between that could let any object go is never taken.

### The machine code follows the likely path

Blocks are written along the likely path, so a jump to the next block costs nothing and a branch is inverted when its
true side comes next. Every function starts at an address that is a multiple of 16 and every loop head at a multiple of
32, the width the processor fetches each cycle, so a change elsewhere in a program does not move a hot loop across a
fetch boundary and change its time. Constants are immediates, and an address sum written only for a load or store is
the processor's addressing mode. What you can observe: the time, and an executable a few bytes larger per loop.

## Calls on constants and what a function knows

### Calls on constants are run while compiling

A call whose arguments are all known while compiling, and whose function touches nothing but its own memory, is run by
the compiler, and the call is replaced by its answer: a number becomes a constant, and an object (a `String`, a list,
any graph of objects the call made) becomes the instructions that allocate it and write its bytes, so the program
gets exactly the object the call would have made, without running the call. A maths function on known operands is
answered the same way.

```spite
func label(count: Integer): String {
    var plural = "cactus".pluralize()
    return "{count} {plural}"
}

func turn(point: Point) {
    var turned_across = point.across * 0.5.cosine() - point.down * 0.5.sine()
    point.across = turned_across
}
```

`"cactus".pluralize()` is the text `cacti` made in three allocations, with none of the inflection tables built or
searched; `0.5.cosine()` and `0.5.sine()` are constants.

It does not apply when the call, or anything it calls, reads or writes a singleton or any memory the block did not
make, calls the system or a library, starts or joins a thread, waits, would halt, or runs past a budget of
instructions. The call is then left as it was, and what is not known is never guessed. What you can observe: the time,
and fewer allocations under `--debug-memory` (a folded call's own work allocates nothing; the objects of its answer
are still allocated where it was). A program whose maths is folded gets the maths library of the machine that
compiles it; `sqrt`, `floor`, `ceil`, `trunc` and `round` are exact everywhere, so only the transcendental
functions can differ, by their last bit, between two versions of the C library.

### What a function knows is followed through all its paths

After inlining, every function is evaluated once more over all its paths at once: the compiler walks its blocks
until what it knows at the start of each block stops changing. It knows the value of every register it can tell and
the contents of every object it made or that a folded call made. Where two paths meet, what both know alike is kept.
So a list filled before a loop is still known inside it as long as nothing in the loop can write it, a read from a
known object is the number it holds, a test on known values is a jump, and a call on known values is run as above.

```spite
var steps = List<Step>()

func Checkout() {
    steps.append('discount')
    steps.append('tax')
    steps.append('discount')
}

func price(amount: Integer): Integer {
    var total = amount
    var index = 0
    while index < steps.count() {
        if steps[index] == 'discount' {
            total = total - total / 10
        }
        if steps[index] == 'tax' {
            total = total + total / 5
        }
        index = index + 1
    }
    return total
}
```

A loop that makes a `Checkout()` and calls `price` reads, after inlining, the count of a list it made itself and
three of its items: all known, so with the loop unrolled `price` is the three steps written out, each division a
multiplication by a constant. Anything that could write a known object without the compiler seeing which (passing it
to a call that is not run, storing its address where the compiler cannot see, handing it to a thread) makes the
function forget everything it knew about it from there on. A function that waits, or one that is very large, is left
as written. What you can observe: the time.

## Singletons and threads

### A singleton is made on first use, by a function of its own

A read of a singleton is a load of its slot and a test that it is made; making it (the claim of the slot, the wait
for another thread making it, the constructor, the record for its `drop()` at exit) is a function of its own, called
only when the slot is empty. A function that reads a singleton is then small enough to be inlined, and an object
that held it holds nothing but its numbers, so it lives in registers.

What you can observe: a singleton read through an attribute is made at its first read instead of when the object
holding it is made, which nothing tells apart, since making it does nothing a program sees. A singleton the entry
class makes on every path is made before the entry function starts, so a function that cannot run before that
reads its slot without testing that it is made, and the load moves out of loops. A singleton made only on some
paths is tested as before.

### A singleton that holds nothing is a static object

A singleton class of the program with no attributes, no `drop()`, no back references, and a constructor and defaults
that take nothing and do nothing is one object written into the executable's data. A read of it is the address of
that object: no slot to load, no test that it is made, no allocation and nothing let go at exit.

```spite
singleton

func doubled(value: Integer): Integer {
    return value * 2
}
```

`Doubler().doubled(21)` is a call on a constant object, or, inlined, one addition. It does not apply to a singleton
with an attribute, a `drop()`, a constructor that takes arguments or has any statement, the entry class, or any
singleton in an inspectable build. What you can observe: one allocation fewer under `--debug-memory` for each such
singleton the program uses.

### Thread safety for singletons, the cheapest safe form

In a program that hands C a `ForeignCallback`, which C may call from a thread of its own, every singleton the
function it calls can reach is made safe for threads in the first of these forms that holds for what its functions
do. A singleton no such function reaches gets none of them, and a program without one gets none at all. Work the
compiler runs at once shares no singleton one of them writes, so it never needs these.

1. **Nothing, when it never changes after it is made.** No function but its constructor assigns its attributes, no
   other class does, and every attribute is a number, a `Boolean`, an enum, an untouched `String` or another
   singleton. Its functions are compiled as in a program without threads.
2. **Atomics, when what changes is whole numbers or `Boolean`s and each function touches that state once.** A read is
   one load, a write one store, and an update (`hits = hits + amount`) retries until no other thread changed the value
   in between, computing the new value with its overflow check, so an addition that does not fit halts before
   anything is written and no two threads lose an update.
3. **Its own lock, otherwise.** Each function that reads `this` takes the singleton's lock for its whole call. The
   thread that owns the lock may take it again, a free lock is claimed with one atomic step, and a held one is
   retried after giving the processor up.

What you can observe: only the time. A reader sees a value from before or after a call, never one half-way through it,
and every update counts.

```spite
var counter = HitCounter()
var settings = Settings()

func run(): Integer {
    var index = 0
    while index < 1000000 {
        var step = settings.step_size()
        counter.record(step)
        index = index + 1
    }
    return index
}
```

With `Settings` only set in its constructor and `HitCounter.record` a single `hits = hits + amount`, `run` reads the
step in place and adds atomically, where every call would otherwise take two locks.

### A counted loop of calls to one singleton takes its lock once

A `while` outside a singleton whose condition is `counter < bound` and whose last statement is `counter = counter +
n` holds the lock of one singleton for the whole loop when every call in it is a call to that singleton's functions
answering a number, a `Boolean`, an enum or nothing (or to a plain helper that only computes), and nothing in it but
local `var`s, `if`s, plain `crash`es and arithmetic. Each call inside goes to the function's body compiled without
the lock, which is small enough to be inlined. It cannot hang: the loop ends on its own, it takes no other lock while
it holds this one, and it holds it across no wait.

```spite
var tally = Tally()
var rounds = 0

func count_up(): Integer {
    var index = 0
    while index < rounds {
        tally.add(index % 3)
        index = index + 1
    }
    return rounds
}
```

`count_up` takes `Tally`'s lock once. A singleton that would otherwise be made safe with
atomics takes its lock instead when such a loop would hold it: one lock taken per loop costs less than one
atomic step per call. What you can observe: other threads calling the singleton wait for the whole loop instead of
for each call, which is one of the orders they could already run in.

### A counted loop of reads shares its singleton's lock

A counted loop that holds a singleton's lock and calls only functions that write none of its state holds the lock
shared: any number of such loops run at once, each on its own thread. A function that changes the singleton claims
the lock as before and then waits until no reader is left, so a reader never sees it half-way through a change and a
change never runs beside a reader. Only a singleton some such loop reads makes its changing functions wait for
readers.

### A type crosses into C as its struct, in the frame

A `type` passed to a foreign function is laid out as the C struct of its attributes in room in the calling
function's frame, with each attribute at its C alignment; the call passes the room's address, and C's writes are
copied back into the object's attributes when it returns. A list of numbers passes the address of its own storage, so
C reads and writes the elements in place. A text answer is copied into a new `String`, and a null pointer stops the
program naming the function. What you can observe: none of it allocates in an ordinary function, and the object C
changed holds C's values after the call.

---

Next: [Proofs the compiler makes](proofs.md), every fact the compiler establishes while compiling.
