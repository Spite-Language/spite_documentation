# Foreign libraries and operating systems

A native library is a class, not a keyword. `DynamicLibrary(file, header, functions)` opens a `.dll`, `.so` or
`.dylib`, and `functions` declares, by hand, every function of it the program calls: a dictionary from each Spite
name to a `DynamicLibrary.FunctionDeclaration`, written where the library is bound. A call is assembled from a
declaration and run, as a [`Spite.Call`](reflection.md#a-call-built-before-it-runs) is. There is no `extern`
keyword, no binding file and no generated shim. The standard library reaches the operating system this way:
`File`, `Directory`, `Process`, `Console` and the rest are Spite over the operating system's own libraries, so
what a program can do with it, the library already does.

## Calling a function

```gdscript title=foreign_call/foreign_call.spite entry system=windows
var console = Console()
var c_runtime = DynamicLibrary(
    "ucrtbase.dll",
    "",
    {"length_of": DynamicLibrary.FunctionDeclaration("strlen", [DynamicLibrary.Argument("text", String)], Integer)},
)

func ForeignCall() {
    var call = DynamicLibrary.Call(c_runtime.functions["length_of"])
    call.arguments["text"] = "spite"
    var text_length = call.call()
    console.print(text_length)
}
```
```output
5
```

`ucrtbase.dll` is Windows' C runtime. A library is a singleton, so it is bound once beside the attributes like any
other ([classes_and_files.md](classes_and_files.md#singletons)); a program for several systems names each
system's file in a class that each system's folder reopens, as the standard library does
([below](#each-operating-system-reopens-what-it-changes)).

## Declaring a foreign function

Mortaro: "lets force wrappers to always properly declare the external usages so a moron can follow where things
come from". The `DynamicLibrary` namespace mirrors the `Spite` one, so a foreign function is declared and called
the way a Spite function is described and called:

| Class | What it is | Mirrors |
|---|---|---|
| `DynamicLibrary` | an opened library, with the `functions` it declares | `Spite.Class` and its `functions` |
| `DynamicLibrary.FunctionDeclaration(symbol, arguments, returns)` | one export: its C symbol, its arguments and its answer | `Spite.FunctionDeclaration` |
| `DynamicLibrary.Argument(name, class)` | one argument: a Spite name and its type | `Spite.Argument` |
| `DynamicLibrary.Call(declaration)` | a call assembled from a declaration, its `arguments` filled by name, run by `call()` | `Spite.Call` |

- **The key is the Spite name** the program uses (`"length_of"`); the C symbol (`"strlen"`) is written once, in
  its declaration, and nowhere else.
- **The arguments are C's**, in C's order, each a name and one of the Spite types that [cross](#what-crosses).
  One C function called with two kinds of argument (a buffer in one place, a null `Long` in another) is declared
  twice, under two names.
- **The answer is C's.** `Long` or `UnsignedLong` reads a 64-bit integer or a pointer, `Double` and `Float` a
  decimal, `String` a `const char*` copied into a `String`, any other type C's `int`, cast to it the ordinary way,
  and `Nothing` a `void` function, or one whose answer the program never reads: an answer
  [must be used](failure.md#an-answer-is-used).
- **The compiler reads the declarations while compiling.** They never change, so the dictionary is never made
  when the program runs and each call is one direct C call: no lookup, no marshalling.

A foreign function called by its C name is an error, which writes the entry to add:

```text
error: 'c_runtime.strlen' calls a foreign function by its C name, and a foreign function is called only through its declaration: add '"strlen": DynamicLibrary.FunctionDeclaration("strlen", [DynamicLibrary.Argument("text", String)], Integer)' to the functions written where 'c_runtime' is declared, with C's arguments and answer, and call it with 'var call = DynamicLibrary.Call(c_runtime.functions["strlen"])', 'call.arguments["text"] = ...', 'call.call()'
```

So following any call to where it leaves Spite takes one step: the declaration names the library, the symbol and
every type that crosses.

## Calling a declared function

`DynamicLibrary.Call(library.functions["name"])` names a declaration the compiler can read, and each
`call.arguments["argument"] = value` fills one argument, cast to its declared type; `call.call()` runs it and
answers what the declaration answers. Every argument is filled once before the call runs and is taken when it is
filled, exactly as for a `Spite.Call`; a fill nothing changes before `call()` is handed over as it is, with no copy.
What crosses is checked at the call: an argument that cannot [cross](#what-crosses), a struct whose size differs
from the header's, an answer read from a function the header declares `void`. A [fault report](failure.md#what-a-native-fault-reports)
names the line of `call.call()`.

A class usually hides its calls behind functions with Spite names of its own, so the rest of the program never
sees the library:

```gdscript
func current_directory(size: Integer, room: Buffer): Integer {
    var call = DynamicLibrary.Call(kernel.functions["current_directory"])
    call.arguments["size"] = size
    call.arguments["room"] = room
    return call.call()
}
```

## Rooms

C often writes into memory it is handed: a path, a file's bytes, text converted from one encoding to another. That
memory is a **room**, and a room is a `type` the program declares, with one field saying what C writes:

| Field | C writes | The field answers |
|---|---|---|
| `DynamicLibrary.WideText` | UTF-16 text, counted in characters | `to_string()` |
| `DynamicLibrary.Text` | bytes of text, counted in bytes | `to_string()` |
| `DynamicLibrary.Bytes` | bytes, counted in bytes | `to_buffer()` |

The declaration names how C sizes the room, a `DynamicLibrary.Sizing`, as its fourth argument:

```gdscript
type PathRoom {
    text: DynamicLibrary.WideText
}

var kernel = DynamicLibrary(
    "kernel32.dll",
    "",
    {
        "current_directory": DynamicLibrary.FunctionDeclaration(
            "GetCurrentDirectoryW",
            [DynamicLibrary.Argument("size", Integer), DynamicLibrary.Argument("room", PathRoom)],
            Integer,
            'answer_is_the_size_needed',
        ),
    },
)

func working_directory(): String? {
    var call = DynamicLibrary.Call(kernel.functions["current_directory"])
    var room = call.call()
    fallback room
    return room.text.to_string()
}
```

The program fills neither the room nor its size: the compiler makes the room, passes its size and its address,
makes it bigger and calls again when C says it was too small, and reads back what C wrote. The size is the first
whole-number argument whose name says a size (`size`, `count`, `length` or `capacity` in it) after the room, then
before it; with none so named, the first whole-number argument after the room, or the last one before it. `call()` answers the room filled, or `null` when C
answered that it failed (an answer of `0`, or below `0` for `'answer_is_what_arrived'`); a declaration that answers
`Nothing` never fails.

| `DynamicLibrary.Sizing` | C's answer | The compiler |
|---|---|---|
| `'answer_is_the_size_needed'` | what it wrote, or the size it needs when the room is short | calls again at the size needed |
| `'doubled_until_it_fits'` | what it wrote, the whole room when the room is short | doubles the room and calls again |
| `'asked_with_no_room_first'` | asked with an empty room, the size it needs | asks first, then calls with that room |
| `'answer_is_what_arrived'` | how much arrived; the program fills the size | calls once |
| `'fixed_size'` | nothing about size; the room is a `type` of numbers | calls once with the room zeroed |

A `Buffer` is made with `Buffer()`, and how much room it keeps is the compiler's: `Buffer(size)` and
`take_written` are the standard library's own, and a program writing either is told to declare a room.

### Spite names for foreign functions

Mortaro: "externally we should always use things with proper naming". A declaration's key is worked out from the
symbol by one rule, the same the error uses for the key it proposes:

1. The symbol is split into words at its capitals and underscores, in snake_case: `GetCurrentDirectoryW` is
   `get_current_directory_w`.
2. A Windows text function's last `W` or `A` goes, since the declaration says which text it takes:
   `get_current_directory`.
3. The library's common prefix goes (`vk`, `VK_`, `Vk` for Vulkan), and so does a leading `get`, which Spite
   keeps for accessors: `current_directory`.
4. A word the [style](style.md#names) table knows as an abbreviation is written in full: `SetCursorPos` is
   `set_cursor_position`.

Where the rule leaves C's own abbreviation (`strlen`, Vulkan's `Cmd`), the key is named by hand (`length_of`,
`record_draw`).

## Opening a library

- **The arguments are literals**, because the compiler reads them while compiling. The file is named exactly as
  it is on disk, extension included: a name without one is an error.
- **One instance per file and header.** `DynamicLibrary` follows the rule every singleton does
  ([classes_and_files.md](classes_and_files.md#one-instance-per-argument-values)): a singleton constructed
  with different argument values is a different instance. Every class asking for
  `DynamicLibrary("ucrtbase.dll", "", ...)` shares one library and one table of symbols, whatever functions each
  declares; the same file asked for with a header, or another file, is a second instance with its own table.
- **Only what is called is bound.** The symbols a program calls are looked up once, when the library opens, so
  every later call is one indirect call. A symbol named only by code the program never reaches is not looked up
  at all ([optimizations.md](optimizations.md#tree-shaking)), except in an inspectable build
  (`--development`, `--repl`, `--repl-port`, `--hot-reload`), which keeps everything. A missing library, or a symbol
  the library does not export that some reachable code calls, stops the program the moment the library opens,
  before any call of it has run, naming the file, or the symbol, the file it is missing from and the Spite function
  that wanted it (`the foreign function 'ring_bell' is not in its library, 'fixture.dll'`). So a typo in
  a symbol name is found when the class holding the library is first made, not on a rarely taken path.
- **A foreign call goes through the attribute or variable holding the library**, so the compiler knows which
  table binds it.

That is the whole run-time cost: opening each library once, on first use, one lookup per called symbol, and one
indirect call per call, with arguments passed as they are ([what crosses](#what-crosses)). There is no marshalling
layer, and a symbol no surviving code calls costs nothing.

## What crosses

| Spite | C | Cost |
|---|---|---|
| every number type, `Boolean` | the same C type: a number already is one, and see the widths below | none |
| `String`, as an argument | `const char*` | none: every `String` buffer already ends in a zero |
| `String`, as a declared answer | `const char*` | one copy; Spite never frees a buffer C owns |
| an enum value | its integer | none |
| `List<T>` of numbers | its element array, by address; C's writes come back into the list | none |
| `Buffer` | its bytes, by address; C's writes come back into the buffer | none |
| a `type` whose attributes are all numbers | a C struct in declaration order, by address; C's writes come back | none |
| a function value, as an argument | a C function pointer for that call ([callbacks](#calling-back-into-spite)) | a thread-local store |
| a class instance | never: its layout is the compiler's business | |

What a call answers is what its [declaration](#declaring-a-foreign-function) answers: `: Long` for a 64-bit integer
or a pointer, `: Double` for a `double`, `: String` for a `const char*` copied into a `String`, and any narrower type
for C's `int`, which the ordinary right-to-left cast takes from there. An address is a
`Long`: a number of 64 bits that C gives and takes back; Spite has no pointer type and no way to read or write the
memory an address names. What C should fill in is handed over as a `List` of numbers, a `Buffer`, a `String` or a
number-only `type`, whose own memory C writes and whose values come back; an address C returns is kept as a
`Long` and handed back to C.

With a header as the third argument, a snake_case read is a constant: `user32.mouseeventf_leftdown` is
`MOUSEEVENTF_LEFTDOWN`, an `Integer`. Without a header a library has functions and nothing else, and reading a
constant is an error that says to name the header as `DynamicLibrary`'s third argument. A constant has to come from a header that is a file the compiler can read, named
by its path from the folder it runs in; a name like `"stdlib.h"` is enough to call functions through the system's
prototypes, but reading a constant from it is an error that says so. A function that wants a struct's size, as
`SendInput` does, is given `PointPair.size`: the bytes of the struct the call passes, a `Long` the compiler knows
while it compiles. Only a `type` whose attributes are all numbers is a C struct, so `.size` on any other `type` is a
compile error, and so is passing a `type` with a `String`, a list, a dictionary or an object in it to a foreign
call: the error names the type (`a Labelled cannot cross into C`), not the field. With a header the compiler can
read, a `type` passed where the prototype takes a pointer to a struct the header declares (`POINT_PAIR*`, a pointer
`typedef` such as `LPPOINT_PAIR`, or `struct Tag*`) is compared with that struct: the struct's fields are numbers and
pointers at their natural alignment, and when the two sizes differ the call is an error naming both (`'MovePoint'
takes a 'POINT_PAIR', which 'fixture.h' declares as 8 bytes, but a PointPair is 12 bytes as a C struct: give
'PointPair' the fields of 'POINT_PAIR', in its order and at its widths`). A struct with an array, a bit field, a
nested struct or a union in it, or one only a system header declares, is not measured. C's field names are not
matched to yours: the fields of a `type` are laid out in the order you declare them, so the order is yours to get
right. The reflection members every value has (`.class`, `.attributes`, `.functions`) and
`DynamicLibrary`'s own members (`file_name`, `handle`) are never read as foreign symbols, so a `List<DynamicLibrary>`
can still `find_by_file_name` and `sort_by_file_name`.

### Argument widths

**With a header**, the call goes through the header's own prototype: the function has to be declared there, the
number of arguments is checked, and each argument is converted to its parameter's type the way C converts one --
an `Integer` literal where C wants a 64-bit `VkDeviceSize`, a `Double` where it wants a `float`. A `Long` handle goes
where C wants a pointer, since Spite has no pointer type; that integer-to-pointer conversion is the one C
complaint the call silences. A C function the header declares `void` is called the same way, and its call is a statement:
a `void` function has no answer, so reading one is a compile error (`'Forget' answers nothing, since 'fixture.h' declares it
'void', so it has no answer to read: call it on a line of its own`). Without a header the compiler cannot know, so do not
read one.

**Without a header**, the compiler knows only the Spite types, so it passes what the platform's calling convention
passes anyway:

| Spite argument | Passed as |
|---|---|
| `Tiny`, `Short`, `Integer`, `Long`, `Boolean`, an enum value | `int64_t`, sign-extended |
| `Byte`, `UnsignedShort`, `UnsignedInteger`, `UnsignedLong` | `uint64_t`, zero-extended |
| `Float` | `float` |
| `Double` | `double` |
| `String`, `List<T>` of numbers, `Buffer`, a number-only `type` | an address |

On x64 (Windows and System V) and on 64-bit ARM, every integer argument travels in a 64-bit register or an 8-byte
stack slot, and a C function reads the low bits of the width it declared, so a literal `0` reaches a `uint64_t`
parameter as a clean zero and a `-1` reaches an `int64_t` one as -1. What a missing header cannot fix: a `Double`
where C wants a `float` (write a `Float`), and on macOS for ARM, integer arguments after the eighth, which Apple
passes on the stack at their own width; name the header there.

## Calling back into Spite

Some C functions take a function: `qsort` compares with one, Windows sends a window its messages through one,
XAudio2 tells a voice its buffer ended through a table of them, and Vulkan reports what its validation found
through one. A Spite function reaches C in one of two ways, and in both the compiler writes the small function C
actually calls (a trampoline), which turns C's arguments into Spite's and calls the function on its owner. No
program writes a trampoline, and there is no new syntax.

### For the length of one call

A function value given as an argument of a foreign call is a C function pointer for that call:

```gdscript
var numbers = DynamicLibrary(
    "numbers.dll",
    "",
    {
        "sum_mapped": DynamicLibrary.FunctionDeclaration(
            "sum_mapped",
            [
                DynamicLibrary.Argument("count", Integer),
                DynamicLibrary.Argument("mapping", Spite.Function<Integer, Integer>),
            ],
            Integer,
        ),
    },
)
var scale = 3

func MappedByC() {
    var call = DynamicLibrary.Call(numbers.functions["sum_mapped"])
    call.arguments["count"] = 4
    call.arguments["mapping"] = scaled
    var total = call.call()
    console.print(total)
}

func scaled(value: Integer): Integer {
    return value * scale
}
```

`numbers.dll` exports `sum_mapped(count, function)`, which calls the function with each of `0` to `count - 1` and
adds the answers.

`sum_mapped` calls `scaled` while it runs, and that is the whole contract of the first form: C may call the function
while the foreign call runs, on the thread that made the call, as often as it likes. Any function value works, bound
to any object or held in a variable, and the call keeps it, and so its owner, alive until it returns. A C
library that keeps the pointer and calls it later, or calls it from a thread of its own, stops the program with a
message naming the function and the line that handed it over; that is what the second form is for.

### For as long as the program keeps it: `ForeignCallback`

`ForeignCallback(function, where_context)` hands C a function it keeps: `.address` is the C function pointer, and
it stays valid for as long as the program keeps the `ForeignCallback`. The function is named directly
(`order.compare`, or `compare` inside the class), and the second argument, a literal the compiler reads like a
library's file name, says how C gets the owner back to the trampoline:

| `where_context` | C calls | The owner is found | For |
|---|---|---|---|
| `'context_first'` | `function(context, ...)` | from `.context`, which the program gives C as its user data | a thread's start |
| `'context_last'` | `function(..., context)` | the same | `EnumWindows`, Vulkan's debug messenger |
| `'no_context'` | `function(...)` | because the function belongs to a singleton | a window procedure, a table of functions |

The Spite function never takes the context itself: the trampoline turns the context it is handed back into the
object the function belongs to.

**With no context, the function belongs to a singleton.** A window procedure gets the window's handle and the
message, and nothing that says which Spite object wanted it, so the compiler only accepts a function of a
singleton there, which has one owner by construction, and the singleton tells its windows apart by the handle C
passes, the way Win32 programs use `GWLP_USERDATA`:

```gdscript
singleton

var user32 = DynamicLibrary(
    "user32.dll",
    "",
    {
        "default_window_procedure": DynamicLibrary.FunctionDeclaration(
            "DefWindowProcW",
            [
                DynamicLibrary.Argument("window", Long),
                DynamicLibrary.Argument("message", UnsignedInteger),
                DynamicLibrary.Argument("word", UnsignedLong),
                DynamicLibrary.Argument("long_word", Long),
            ],
            Long,
        ),
    },
)
var windows = Dictionary<Long, Window>()
var procedure: ForeignCallback? = null

func register(): Long {
    var handling = ForeignCallback(handle_message, 'no_context')
    procedure = handling
    return handling.address
}

func handle_message(window: Long, message: UnsignedInteger, word: UnsignedLong, long_word: Long): Long {
    var found = windows[window]
    if found {
        return found.handle_message(message, word, long_word)
    }
    var call = DynamicLibrary.Call(user32.functions["default_window_procedure"])
    call.arguments["window"] = window
    call.arguments["message"] = message
    call.arguments["word"] = word
    call.arguments["long_word"] = long_word
    return call.call()
}
```

`register()`'s address goes into the `procedure` attribute of the `type` passed to `RegisterClassExW`, and every
message of every window (resize, fullscreen, IME composition, the messages Windows sends rather than posts)
arrives at `handle_message`. There is one `ForeignCallback` per such function at a time: making a second while the
first is alive stops the program, since both would be the same C function.

**A table of functions is filled by C.** A COM-style interface such as XAudio2's `IXAudio2VoiceCallback` is an
object whose first word points at a table of functions. Spite cannot write memory at an address, so the table is
built on the C side: the library exports a function that takes the `.address` of each `ForeignCallback` in the
interface's order (a declared `set_voice_started`, called with `started.address`) and keeps them in
its own table. The functions are
`'no_context'` functions of one singleton, which tells the voices apart by the object C passes back as a `Long`.

**Threads.** XAudio2 calls from its own thread, so a function handed over with `ForeignCallback` may run on a
thread C made: a function
of an ordinary object may reach only the attributes of its own that hold values, its locals and singletons, and a
singleton it reaches (the owner of a `'no_context'` function included) is made safe for threads by the compiler, a
lock or atomics as its state needs. A program that makes a `ForeignCallback` counts references
atomically. A callback never runs the scheduler: a wait on a thread that is not the program's own makes the plain
blocking call. A function value passed for one call is not checked, since it runs on the calling thread.

**Lifetime.** A `ForeignCallback` keeps its function and the function's owner alive, and C may call it for exactly
as long as the program keeps it. Keep it in the object that owns what C was given (the window class, the voice,
the debug messenger) and tell C to stop in that object's `drop()`: attributes are released after `drop()` runs,
so C has let go before the `ForeignCallback` goes. When that object is also the one whose function is handed over,
the object holding the callback and the callback holding the object would keep each other alive; the compiler
reads the callback as the back reference it is, as it does for a [cycle](memory.md#cycles-are-freed), so the
object is freed when its last other holder goes:

```gdscript
var vulkan = DynamicLibrary(
    "vulkan-1.dll",
    "",
    {
        "listen": DynamicLibrary.FunctionDeclaration(
            "listen",
            [DynamicLibrary.Argument("reporter", Long), DynamicLibrary.Argument("context", Long)],
            Nothing,
        ),
        "stop_listening": DynamicLibrary.FunctionDeclaration("stop_listening", [], Nothing),
    },
)
var _messenger: ForeignCallback? = null

func Device() {
    var reporting = ForeignCallback(report, 'context_last')
    _messenger = reporting
    listen(reporting.address, reporting.context)
}

func report(severity: Integer, data: Long): Boolean {
    return false
}

func drop() {
    stop_listening()
}

func listen(reporter: Long, context: Long) {
    var call = DynamicLibrary.Call(vulkan.functions["listen"])
    call.arguments["reporter"] = reporter
    call.arguments["context"] = context
    call.call()
}

func stop_listening() {
    var call = DynamicLibrary.Call(vulkan.functions["stop_listening"])
    call.call()
}
```

This holds when the callback is made from a function of the object itself (`report` or `this.report`) and kept
only in its attributes, directly or through the local it was made in, and nothing reads it but `.address` and
`.context`. A `ForeignCallback` the program also hands anywhere else (a list, another object, a return) keeps its
owner alive with a count as before, and the two are a cycle like any other, freed once nothing else holds them
([memory.md](memory.md#every-other-cycle-is-freed-by-a-count-that-knows-it)). An attribute holding a
`ForeignCallback` counts as read, since
keeping it is what it is for. After one is dropped, a late call from C stops the program with a message naming the
function and the line that made the `ForeignCallback`, so the owner's `drop()` must still tell C to stop first: the
check only makes forgetting loud.

**An owner is never copied.** C holds one registration, made by one object, so a `copy()` or `deep_copy()` of an
object that holds a `ForeignCallback` is a compile error at the call: a copy would share the registration, and a
deep copy would make a second callback C was never given. `copy()` is refused when the class's own attributes
hold one (`ForeignCallback`, `ForeignCallback?`, or as a type argument, `List<ForeignCallback>`); `deep_copy()`
also when any class it would reach through attributes, lists, unions or a `type` holds one, stopping at a class
that declares its own `deep_copy()`. The words name the class and the attribute; make another object, which
registers its own callback, instead:

```text
error: a copy of Listener would share the C registration in '_ticking': C calls back the one object that registered it, so make another Listener instead of copying this one
error: a deep copy of Desktop would copy the C registration in 'Listener._ticking', which it reaches through 'listeners': C was given only the one it holds, so make another Desktop instead of copying this one
```

A `copy()` of an object that holds such an owner in another attribute is allowed: both copies share the one owner.

**C types.** A trampoline's C signature is the Spite function's, one parameter at a time, each at its width
([what crosses](#what-crosses)): numbers as their C types, an address as a `Long`, and a
`Boolean` as a 32-bit integer (`BOOL`, `VkBool32`, `int`; C's one-byte `bool` is taken as a `Byte`). A callback
takes only these and returns one of them or nothing. A callback cannot take text yet: text or a struct C passes arrives as its
address, a `Long`, which the callback can keep or hand back to C, and a `String` parameter is a compile error that says so. The header does not check the trampoline
against C's declaration (C cannot name a parameter's type), so the widths are the program's to get right, the
same bargain a struct's layout is. On every 64-bit target C has one calling convention, so `CALLBACK`, `WINAPI`,
`VKAPI_PTR` and `STDMETHODCALLTYPE` need nothing.

**Cost.** A trampoline for one call is a thread-local load and an indirect call; one with a context, an indirect
call; one with no context, an atomic load and a direct call. Making a `ForeignCallback` is one allocation beside the
function value. A program that passes no function to C and makes no `ForeignCallback` carries none of it: no
trampoline, no class, no atomic counts.

## Each operating system reopens what it changes

There is no platform layer. The standard library's `common` folder holds what every system shares, and its `windows`,
`linux` and `mac` folders hold only what differs: each reopens the classes it changes, with a `DynamicLibrary` of its
own naming the system's real file. The Windows side of `Program` adds what only Windows answers, such as the
working directory:

```gdscript
type PathRoom {
    text: DynamicLibrary.WideText
}

var kernel = DynamicLibrary(
    "kernel32.dll",
    "",
    {
        "current_directory": DynamicLibrary.FunctionDeclaration(
            "GetCurrentDirectoryW",
            [DynamicLibrary.Argument("size", Integer), DynamicLibrary.Argument("room", PathRoom)],
            Integer,
            'answer_is_the_size_needed',
        ),
    },
)

func working_directory(): String? {
    var call = DynamicLibrary.Call(kernel.functions["current_directory"])
    var room = call.call()
    fallback room
    return room.text.to_string()
}
```

The standard library's own functions that call C start with `_`, so they add nothing to what a program can call.

while `common`'s `Program` keeps what every system calls the same way (`exit` through the C runtime). Today only the
Windows folder holds code: the executable is always Windows x64, and `linux` and `mac` only load `common`.
The compiler reads exactly one of these folders, the one for the
system a program is compiled for, and it loads `common` itself ([programs.md](programs.md#the-standard-library-needs-no-load)), so their functions replace
or add members by the ordinary reopening rule ([packages.md](packages.md#monkey-patching-mods)).
`--final-classes` prints each class as it came out, its system's functions included.

## Libraries written in other languages

`DynamicLibrary` does not care what a library was written in. It calls functions through the C calling convention
(the C ABI), and every systems language can export functions that way, so a mature library in C, C++, Rust, Zig or
Go is one binding away. You do not need a big ecosystem on day one: you need the library's dynamic library file
and a small Spite class around it.

| The library is written in | It exports a function to Spite with | It is built as |
|---|---|---|
| C | nothing: every function the library exports is already callable | a `.dll`, `.so` or `.dylib` |
| C++ | `extern "C"` on each function, a free function taking the object's address for each method | a shared library |
| Rust | `#[no_mangle] pub extern "C" fn` | `crate-type = ["cdylib"]` |
| Zig | `export fn` | a dynamic library (`zig build-lib -dynamic`) |
| Go | `//export` above the function, with `import "C"` | `go build -buildmode=c-shared` |

What a library hands across is what [crosses](#what-crosses): numbers, `Boolean`, text as a zero-terminated
`const char*`, arrays of numbers and number-only structs by address, and a handle as a `Long`. A Rust function
that sums an array, exported from a library built as `libmeasure.so`:

```rust
#[no_mangle]
pub extern "C" fn sum_of(values: *const i32, count: i64) -> i64 {
    let values = unsafe { std::slice::from_raw_parts(values, count as usize) };
    values.iter().map(|value| *value as i64).sum()
}
```

is declared and called from Spite exactly as a C function is, its name already snake_case:

```gdscript
var measure = DynamicLibrary(
    "libmeasure.so",
    "",
    {
        "sum_of": DynamicLibrary.FunctionDeclaration(
            "sum_of",
            [DynamicLibrary.Argument("values", List<Integer>), DynamicLibrary.Argument("count", Long)],
            Long,
        ),
    },
)

func total(values: List<Integer>): Long {
    var values_count: Long = values.count()
    var call = DynamicLibrary.Call(measure.functions["sum_of"])
    call.arguments["values"] = values
    call.arguments["count"] = values_count
    return call.call()
}
```

A C++ class, a Rust struct or a Go value never crosses as itself. The library hands out an opaque handle (a
pointer, which Spite holds as a `Long`) and a function per operation that takes it, `extern "C"` in C++,
`extern "C" fn` taking `*mut Thing` in Rust, and a `cgo.Handle` in Go. That is the shape every C library already has,
and it is the shape the binding below turns back into an object.

## A binding speaks Spite

You should not have to think about C, or any other language, to use a library. So a binding wraps the library in a
class of its own, and nothing but that class touches the `DynamicLibrary`: only Spite shapes come out of it.

The reason is that whatever can be written in Spite is written in Spite, and a foreign library is the exception that
should cost the rest of the program nothing. A C enum becomes a Spite enum with Spite names, and no C type, name or
convention is allowed to pass the binding. It also keeps the program free of any one outside language: if the
library is later written in Spite, the binding is the only thing that changes.

- **Spite names.** The functions the rest of the program calls are full words in snake_case (`open_database`, not
  `db_open_v2`), whatever the library calls them; the C name appears only in the
  [declaration](#declaring-a-foreign-function), which a binding keeps private with a leading `_`.
- **A status is a Spite enum.** A C function that answers a C enum (or a status code) answers, through the binding,
  an enum the binding writes in Spite, with Spite names, each value given its C number with `=` (`'busy' = 5`,
  [values_and_types.md](values_and_types.md#numbering-an-enums-values)). A number the enum does
  not list crashes at the boundary, so a library that grows a new value is caught where it enters, never carried
  along as a wrong answer.
- **Absence is a `T?`.** A null handle or a "not found" code becomes `null` through a `fallback` in the binding, never
  a `0` the caller has to know to compare with.
- **A handle is an object.** The binding's class holds the handle, and its `drop()` gives it back to the library, so
  the library's memory is released when the last reference goes, like every other object's.
- **Text is a `String`**, declared as the answer, never an address the caller has to read.

A binding for a library that compresses blocks, whose C side answers `0` for success, `1` when the output is too
small and `2` when the input is corrupt:

```gdscript
enum Packing {
    'packed'
    'output_too_small'
    'input_corrupt'
}

var library = DynamicLibrary(
    "libpack.so",
    "",
    {
        "create": DynamicLibrary.FunctionDeclaration("pack_create", [DynamicLibrary.Argument("level", Integer)], Long),
        "pack_block": DynamicLibrary.FunctionDeclaration(
            "pack_block",
            [
                DynamicLibrary.Argument("packer", Long),
                DynamicLibrary.Argument("input", Buffer),
                DynamicLibrary.Argument("input_count", Integer),
                DynamicLibrary.Argument("output", Buffer),
                DynamicLibrary.Argument("output_count", Integer),
            ],
            Integer,
        ),
        "destroy": DynamicLibrary.FunctionDeclaration("pack_destroy", [DynamicLibrary.Argument("packer", Long)], Nothing),
    },
)
var handle: Long = 0

func Packer(level: Integer) {
    handle = _create(level)
    crash handle != 0
}

func pack(input: Buffer, output: Buffer): Packing {
    var input_count = input.count()
    var output_count = output.count()
    var code = _pack_block(handle, input, input_count, output, output_count)
    return packing_of(code)
}

func packing_of(code: Integer): Packing {
    if code == 0 {
        return 'packed'
    }
    if code == 1 {
        return 'output_too_small'
    }
    if code == 2 {
        return 'input_corrupt'
    }
    crash
}

func drop() {
    _destroy(handle)
}

func _create(level: Integer): Long {
    var call = DynamicLibrary.Call(library.functions["create"])
    call.arguments["level"] = level
    return call.call()
}

func _pack_block(packer: Long, input: Buffer, input_count: Integer, output: Buffer, output_count: Integer): Integer {
    var call = DynamicLibrary.Call(library.functions["pack_block"])
    call.arguments["packer"] = packer
    call.arguments["input"] = input
    call.arguments["input_count"] = input_count
    call.arguments["output"] = output
    call.arguments["output_count"] = output_count
    return call.call()
}

func _destroy(packer: Long) {
    var call = DynamicLibrary.Call(library.functions["destroy"])
    call.arguments["packer"] = packer
    call.call()
}
```

The program sees a `Packer` with a `pack` that answers a `Packing`, which a `switch` reads with every value
written; `pack_create`, the numbers `0` to `2` and the handle never leave the file. Write a binding when a mature
library already does the job (compression, a database, a codec, a physics engine, a graphics API) and rewriting
it would take months; write it in Spite when it is small, or when the work is what your program is about.

## What other languages cannot hand over

Some things have no shape in the C ABI, so no binding can carry them as they are:

- **Exceptions.** A C++ exception or a Rust panic that unwinds out of an exported function is undefined behaviour
  in C, and Spite has no exceptions to receive it. The library must catch it on its side and answer a status. One
  that unwinds through a [callback](#calling-back-into-spite) toward a `catch` further out ends the program with
  a report naming the exception's code and the foreign call it came out of
  ([failure.md](failure.md#a-c-exception-through-a-spite-frame)).
- **Generics and templates.** Only a concrete function has an address: export one instantiation per type.
- **A struct returned by value**, a C union past its first member, variadic functions (`printf`) and macros that
  look like functions. Wrap them in a small exported function on the library's side.
- **Classes, traits, interfaces and closures as themselves.** They cross as a handle and functions, as above, and a
  function the library calls back is a [`ForeignCallback`](#calling-back-into-spite).
- **Another runtime's memory.** Spite never frees what the library allocated: memory the library hands out goes
  back through the library's own function, which is what a binding's `drop()` is for. A Go library brings Go's
  runtime and collector into the process with it, and a C# or Java library needs its runtime hosted, which no
  binding does for you.
- **Static libraries.** A library is loaded from its `.dll`, `.so` or `.dylib` while the program runs; a `.a` or
  `.lib` is not linked into the executable.

## When a foreign call faults

C can do what Spite cannot: read through a null pointer, write past an array, run out of stack. When it does, the
program still says so: it prints a `spite.fault` line naming the library the fault is in
(`at=fixture.dll+0x1029`), the foreign function the faulting thread is in and the line of Spite that called it
(`foreign=read_integer_at	library=...	from=native_fault_foreign.spite:13`), and then the Spite functions on the
stack. Each foreign call names itself in a slot of its own thread before it goes in, so a call another thread is
waiting in is never named, and puts back the one it found when it returns, so a foreign call made from a callback
leaves the call that called back named; how to read the whole report is in
[failure.md](failure.md#what-a-native-fault-reports).

A library that catches its own faults keeps doing so: a C `__try`/`__except` around its own code, or a handler it
registers itself, as graphics drivers do, gets the fault first, and the call returns as the library decided. Only a
fault the library leaves alone is reported ([failure.md](failure.md#a-fault-a-foreign-library-catches-itself)).

## What the compiler supplies

The members whose bodies the compiler supplies, such as opening a library and finding a symbol, are declared in
Spite as a `func` with a signature and no body, which the compiler merges into the class as its own reopening
([compiler.md](compiler.md#inspect-merged-classes)). Everything else of `DynamicLibrary` is ordinary Spite: the
`singleton` line, `file_name` and `handle` (the library's handle, a `Long`), the constructor and `drop()`.

---

Next: [Targets, the web and isomorphic classes](targets.md), what a program can be built for.
