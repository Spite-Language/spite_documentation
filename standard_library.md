# Standard library

The standard library is ordinary Spite, a folder of its own that a program loads by its path, and a program reads it the way it reads its own code:
every class a program can name (`String`, `List`, `Integer`, `File`) is a file there, and
`--final-classes` prints each one as the program uses it. Its values are reference counted like every other
object ([memory.md](memory.md)). When an operation cannot succeed it says so in its type rather than crashing:
an index or a key that is not there reads as `T?`, a file that cannot be read answers `null`, and text that does
not parse as a number answers `null`.

`common` holds what every operating system shares; `windows`, `linux` and `mac`
reopen the classes each system does differently, and a program loads the one it is compiled for
([foreign_libraries.md](foreign_libraries.md#each-operating-system-reopens-what-it-changes)).

**Use the library's class instead of making your own.** A class of your own named like one of the library's, a
`Geometry.Vector3` or a `Color` in a folder, is a compile error naming the library's class to use, and what it lacks
is added by reopening it ([A class name means one class](classes_and_files.md#a-class-name-means-one-class)).

**A program carries only the classes it uses.** The library is part of every program's
source, but a production build keeps only what `main` can reach: a program that never makes a `FileSystemWatcher`,
a `Tcp.Client` or a `Process` has none of their code, and none of the operating-system functions only
they call is looked up when the program starts ([optimizations.md](optimizations.md#tree-shaking)).
What a class costs when it is used is what its Spite does, and each section below says where that is more than a
call. An inspectable build (`--repl`, `--repl-port`, `--hot-reload`, `--development`) keeps everything, so the
REPL can look at any of it.

## What is in it

| Class | What it is | Page |
|---|---|---|
| `String` | immutable text | [below](#string) |
| `Integer`, `Long`, `Float`, `Double`, `Boolean`, ... | numbers, as classes, with their maths (`square_root()`, `sine()`, `Float.Pi()`, ...) | [values_and_types.md](values_and_types.md#numbers-are-classes), [maths](values_and_types.md#maths-functions) |
| `Nothing`, `Anything` | what a function returns when it returns nothing; the empty `type` every class fits | [functions_and_operators.md](https://github.com/Spite-Language/spite_spec/blob/master/specs/functions_and_operators.md#calling-one) |
| `List<T>`, `Dictionary<Key, Value>` | containers, and the member templates | [collections.md](collections.md) |
| `Console` | the terminal: print, read a line | [below](#console) |
| `File`, `Directory` | files and folders | [below](#read-and-write-a-file) |
| `FileSystemWatcher` | the paths that changed under a file or a folder, told by the operating system | [below](#watch-files-and-folders) |
| `Process` | run another program | [below](#run-a-process) |
| `Program` | this program: what it was started with, exit, sleep, environment variables, its own path | [below](#program) |
| `Clock` | elapsed time for measuring, and the wall clock | [below](#clock) |
| `Benchmark<T>` | one timed run of a function: what it answered and how long it took | [time.md](time.md#measuring-a-piece-of-work) |
| `Test` | runs every test function of a namespace, each class built with its constructor as the setup | [testing.md](testing.md) |
| `Instant`, `Duration`, `Date`, `Time`, `DateTime`, `Period`, `TimeZone`, `TimeZones` | exact time, the calendar, time zones as presentation, ISO 8601 text (read with `var read: Instant? = text` and the rest) | [time.md](time.md) |
| `Environment`, `Build` | run-time and compile-time settings | [programs.md](programs.md) |
| `Reload` | how many reloads a `--hot-reload` program has swapped in, and which classes they rebuilt | [repl.md](repl.md#knowing-what-a-reload-rebuilt) |
| `JsonWriter<T>`, `JsonReader<T>` | any value to JSON text and back | [json.md](json.md) |
| `Buffer` | bytes, and the numbers and text laid out in them, in either byte order, read at a position | [below](#bytes-are-a-buffer) |
| `ForeignBytes` | memory a C library hands out, read and written by position with every access checked | [below](#bytes-are-a-buffer) |
| `BinaryWriter<T>`, `BinaryReader<T>` | any value to compact bytes (a `Buffer`) and back, for Spite programs talking to each other and for files | [json.md](json.md#write-and-read-bytes) |
| `Vector2`, `Vector3`, `Vector4`, `Matrix3`, `Matrix4`, `Quaternion`, `AxisAlignedBox`, `Plane`, `Frustum`, `Ray`, `Color`, `CubicBezier`, `Easing`, `Noise` | game maths: points, directions, transforms, rotations, culling and picking, colours in the web's formats, curves, easings and noise | [game_maths.md](game_maths.md) |
| `IpAddress` | an IPv4 or IPv6 address, read from text, resolved from a host name, written back as text | [below](#ipaddress) |
| `Tcp.Server`, `Tcp.Client`, `Tcp.Connection` | TCP over IPv4 and IPv6: listen, accept, connect, lines and bytes | [below](#tcp) |
| `Udp.Server`, `Udp.Client`, `Udp.Datagram` | UDP datagrams over IPv4 and IPv6 | [below](#udp) |
| `Http.Server`, `Http.Client`, `Http.Request`, `Http.Response` | HTTP/1.1, connections kept alive | [below](#http) |
| `WebSocket.Client`, `WebSocket.Server`, `WebSocket.Connection`, `WebSocket.Message` | WebSocket (RFC 6455) over an HTTP upgrade: text and binary messages both ways, ping and close | [below](#websocket) |
| `Base64`, `Deflate`, `Zlib`, `Gzip` | bytes as base64 and base64url text; bytes compressed as raw DEFLATE, zlib and gzip | [below](#bytes-base64-compression-hashes-and-passwords) |
| `Sha256`, `SecureRandom`, `Argon2` | SHA-256 and HMAC-SHA256; bytes from the operating system's secure random source; password hashes as PHC strings | [below](#bytes-base64-compression-hashes-and-passwords) |
| `DynamicLibrary` | call a native library | [foreign_libraries.md](foreign_libraries.md) |
| `Spite.Class`, `Spite.Function`, ... | reflection | [reflection.md](reflection.md) |

The other files of the library are the machinery these are built from: `NumberText` and the time-zone readers. They are ordinary classes a program can read and the REPL can inspect, which a program
has no reason to call. Memory, the event loop and the threads the compiler runs work on are not classes of
the library: the compiler writes them ([memory.md](memory.md#memory-belongs-to-the-compiler)).

## What belongs in the standard library

Anything that no company owns belongs here: a protocol, a file format, an algorithm, a piece of maths. HTTP, TLS,
SHA-256 and HMAC, Argon2, secure random bytes, base64 and gzip are standard, and the library grows to cover them
as programs need them. A client for a branded product (a database such as MongoDB or Postgres, a vendor's API) is
never standard: it is a package, kept by whoever maintains it, and a program loads it pinned to a commit like any
other repository
([packages.md](packages.md#loading-a-repository-pinned-to-a-commit)). The `windows`, `linux` and
`mac` folders of the library are not brand packages: they are how a standard class is written for one system.

The library has TCP and UDP over IPv4 and IPv6, HTTP/1.1 with kept-alive connections and WebSocket over it
([below](#tcp)), SHA-256 and HMAC, Argon2id, secure random bytes, base64 and base64url, and DEFLATE, zlib and
gzip ([below](#bytes-base64-compression-hashes-and-passwords)). TLS belongs to it as well.

## `String`

Immutable. Text of up to 15 bytes is kept inside the `String` value itself and allocates nothing; longer text is
one reference-counted block ([optimizations.md](optimizations.md#short-text-lives-inside-the-string)), and none of
it changes what a program means. A value is placed inside written text, `"hello {name}"`, and two values join
with `+` ([values_and_types.md](values_and_types.md#string)). `==`, `!=`, `<` and `>` compare by content.

A `String` answers where it can: `text[index]` is the whole character that starts at byte `index`, a `String?`
that is `null` past the end, before the start or inside a character of several bytes, and `index_of` answers an
`Integer?`, `null` when the text is absent, and `slice(start, end)` answers a `String?`, `null` for a range outside
the text or backwards, never a shorter text that would read like the one asked for. The one member that halts is
`byte_at`, on a byte below 0 or past the end, since it answers a number and a `0` there reads like a real byte; it
names the index and the length. Reading a number is the one that can find nothing to answer, so
each number's conversion from text answers a `T?`: `null` when the text is not a number that fits the type.
Assigning text to a number calls it, so it is allowed only into a `T?`: `var age: Integer? = "42"` is `42`, and
`var age: Integer = "42"` is an error. Every member,
with what it answers at the edges, is in [the specification](https://github.com/Spite-Language/spite_spec/blob/master/specs/standard_library.md#the-string-class).

```gdscript title=string_members/string_members.spite entry
var console = Console()

func StringMembers() {
    load "../spite_standard_library/windows"
    var line = "  ada, grace, linus  "
    var names = line.trim().split(", ")
    var first = names.first()
    crash first
    var shouted = first.upper_case()
    var count: Integer? = "3"
    crash count
    var missing = line.index_of("barbara")
    var first_length = first.length()
    var tail = first.slice(1, first_length)
    crash tail
    var second = first[1]
    crash second
    console.print(shouted, count, missing, tail, second)
    var file_lines = "one\ntwo".lines()
    var line_count = file_lines.count()
    var renamed = line.replace("ada", "Ada").trim()
    console.print(line_count, renamed)
}
```
```output
ADA 3 null da d
2 Ada, grace, linus
```

## Paths

A `Path` is a place in the file system. It is made from text by a conversion, so a program writes the text and the
compiler casts it wherever a `Path` is wanted: `File("levels/forest.txt")` and `var level: Path = "levels/forest.txt"`
both read the text as a path ([conversions](classes_and_files.md#constructors-conversions-and-presets)). Reading it
tidies it: `.` and empty pieces go, `..` takes back the piece before it, both separators become `/`, and a trailing
separator goes, so `"./a/b/../c/"` is `a/c`, and a path that starts at the root never climbs above it.

| Member | Result | Notes |
|---|---|---|
| `name` | `String` | the last piece: `d.txt` |
| `extension` | `String` | after the name's last dot, `txt`; empty when there is none, and for a name that starts with a dot (`.env`) |
| `parent` | `Path` | everything but the last piece; `.` when there is none |
| `is_rooted` | `Boolean` | starts at the root (`/`) or a drive (`C:`) |
| `joined(other)` | `Path` | `other` inside this one; a rooted `other` is itself |
| `written` | `String` | the text as it was given, untidied |
| `==`, `to_string()` | | two paths are equal when their tidied text is |

`File`, `Directory`, `move_to`, `FileSystemWatcher`'s `watch_folder` and `watch_file`, `Environment.load_from_file`
and `Process`'s `working_directory` all take a `Path` (`tests/paths`). They hand the system the path as it was
written, and `File(...).path` reads it back that way.

## Read and write a file

`File(path)` is a value, a path, and several may exist at once.

| Member | Result | Notes |
|---|---|---|
| `path` | `String` | |
| `to_string()` | `String` | the path, so a `File` prints as its path and a crash on one names it ([failure.md](failure.md#what-a-crash-reports)) |
| `file_name` | `String` | the last piece of the path: `File("levels/forest.txt").file_name` is `forest.txt` |
| `read_as_string()` | `String?` | `null` when the file cannot be read, or its bytes are not UTF-8 |
| `write_string(text)` / `append_string(text)` | `Boolean` | replaces the content / adds to its end |
| `exists()` / `remove()` | `Boolean` | |
| `move_to(path)` | `Boolean` | renames or moves it ([below](#rename-or-move-a-file-or-folder)) |
| `byte_count()` | `Long?` | how many bytes it holds; `null` when it cannot be opened |
| `last_written()` | `Instant?` | when it was last written ([time.md](time.md)); `null` when it does not exist |
| `read_as_buffer()` | `Buffer?` | the whole file; `null` when it cannot be read, and an empty buffer for an empty file |
| `read_buffer_at(position, count)` | `Buffer?` | up to `count` bytes starting `position` bytes in: fewer at the end, none past it; `null` when it cannot be opened |
| `write_buffer(buffer)` | `Boolean` | replaces the content with the buffer's bytes |
| `append_buffer(buffer)` | `Long?` | adds the buffer's bytes to the end; answers the position they start at, `null` when they could not all be written |

A file is one holder with two reads: `read_as_string()` when it is text and `read_as_buffer()` when it is bytes,
and the two writes match them. Nothing about a file halts the program: every write, `remove()` and `move_to(path)`
answers whether it happened, and the answer must be used, so the shortest write is one line that says what a
failure means: `fallback file.write_string(text)` to carry on without it, `crash file.write_string(text)` when it is a
bug ([failure.md](failure.md#an-answer-is-used)).

```gdscript title=file_tasks/file_tasks.spite entry
var console = Console()

func FileTasks() {
    load "../spite_standard_library/windows"
    var log_file = File(".spite/documentation_demo_log.txt")
    crash log_file.write_string("first line")
    crash log_file.append_string(", second line")
    var log_exists = log_file.exists()
    console.print("exists", log_exists)
    var content = log_file.read_as_string()
    crash content
    console.print("content", content)
    var removed = log_file.remove()
    console.print("removed", removed)
    log_exists = log_file.exists()
    console.print("exists after remove", log_exists)
}
```
```output
exists true
content first line, second line
removed true
exists after remove false
```

`read_as_string()` is a `String?`: narrow it with `if content { }`, `fallback content` (which returns the function's
default) or `crash content` (which halts), exactly like any other `T?` ([failure.md](failure.md)). This entry
constructor uses `crash`, because `fallback` is not allowed in a constructor.

**Bytes.** A cache or an asset file is bytes, not text, and bytes are a `Buffer`: `read_as_buffer()` reads the
whole file, `read_buffer_at(position, count)` reads from any position, which is what seeking is for, and
`append_buffer(buffer)` says where a record landed, which is what an index needs. A `Buffer` reads and writes the
numbers in it ([below](#bytes-are-a-buffer)). Every call opens and closes the file, so read a large file in
one call and walk the buffer rather than record by record.

```gdscript title=byte_records/byte_records.spite entry
var console = Console()
var _first_record_value: Long = 1111
var _second_record_value: Long = 2222

func ByteRecords() {
    load "../spite_standard_library/windows"
    var store = File(".spite/documentation_byte_records.bin")
    var record = Buffer(8)
    record.append_long_little_endian(_first_record_value)
    crash store.write_buffer(record)
    var next = Buffer(8)
    next.append_long_little_endian(_second_record_value)
    var second_at = store.append_buffer(next)
    crash second_at
    var read_back = store.read_buffer_at(second_at, 8)
    crash read_back
    var second = read_back.read_long_little_endian_at(0)
    crash second
    var size = store.byte_count()
    crash size
    console.print("the second record starts at", second_at, "and holds", second, "of", size, "bytes")
}
```
```output
the second record starts at 8 and holds 2222 of 16 bytes
```

### Lock a file

`file.lock()` waits until no other process holds a lock on the file, and answers a `LockedFile` you hold until you
call `unlock()` or let it go. It is how two programs take turns on one file, such as a cache both write. If the
program ends or crashes the system releases the lock, so it is never left held. `lock()` answers `null` when
the path cannot be opened or the system refuses the lock. There is nothing to write for the operating system: the same call works on Windows,
Linux and macOS. The rules are in [the specification](https://github.com/Spite-Language/spite_spec/blob/master/specs/standard_library.md#system-classes).

### Rename or move a file or folder

`move_to(path)` is one function for both, on a `File` and on a `Directory`: a new name in the same folder renames
it, a path in another folder moves it there. It answers `true` when it moved, and the value then names the new
path, so the same `File` keeps working. It never replaces anything: when a file or folder is already at `path` it
answers `false` and changes nothing, so a move can never lose what was there. It answers `false` too when there is
nothing to move, or when the system cannot move it (a folder to another drive).

```gdscript title=moving_files/moving_files.spite entry
var console = Console()

func MovingFiles() {
    load "../spite_standard_library/windows"
    var folder = Directory(".spite/documentation_moving")
    crash folder.create()
    var draft = File(".spite/documentation_moving/draft.txt")
    crash draft.write_string("notes")
    var renamed = draft.move_to(".spite/documentation_moving/final.txt")
    console.print(renamed, draft.file_name)
    var keeper = File(".spite/documentation_moving/keeper.txt")
    crash keeper.write_string("keep me")
    var replaced = draft.move_to(".spite/documentation_moving/keeper.txt")
    var kept = keeper.read_as_string()
    crash kept
    console.print(replaced, kept)
    crash draft.remove()
    crash keeper.remove()
}
```
```output
true final.txt
false keep me
```

### A large file

`read_as_buffer()` answers the whole file as a `Buffer`, and whether the bytes are read or the file is mapped into
memory is the compiler's choice, never the program's: a program says what it reads, not how. A file too large for one
`Buffer` (past 2 GB) is read in pieces with `read_buffer_at(position, count)`: each call opens, seeks and closes the
file, which measured about 50 microseconds a call on Windows, so a loop reading a megabyte at a time spends almost
nothing on it (512 MB in 169 ms, against 572 ms in 64 KB pieces, warm cache). Read a megabyte or more per call;
there is no separate reader to open and close.

**Independent reads overlap.** Two or more untyped `var name = file.read_as_string()` in a row (or `socket.read_line()`),
each on a name or an attribute and none naming an earlier one, are started together and all finished before the next
statement: the compiler starts each but the last as a state machine of its own, so the program waits once for the
slowest instead of once per file. Nothing is written for it, and nothing after the reads can see a difference:
a file written by the next statement is written after every read has finished
([concurrency.md](concurrency.md#reads-in-a-row-overlap)). What it costs is a frame on
the heap for each read but the last and the event loop that finishes them, and only in a program that reads this
way ([optimizations.md](optimizations.md#concurrency-machinery-only-where-it-is-used)). A read or write of a
`File`, `console.read_line()`, a `Tcp.Server`'s `accept()`, a connection's `read_line()`, `read_as_string()` and
`read_as_buffer()`, and `program.sleep()` are the library's waits: inside a state machine the compiler writes, each lets other work run.
Everything else, `Process.run()` included, blocks the thread that calls it.

## List a directory

| Member | Result | Notes |
|---|---|---|
| `path` | `String` | |
| `folder_name` | `String` | the last piece of the path: `Directory("levels/forest").folder_name` is `forest` |
| `entries()` | `List<Directory.Entry>` | every folder and file inside it, as `Directory` and `File` values, each kind sorted by name with a merge sort (`n log n`), so a folder of thousands of files lists quickly |
| `exists()` / `create()` | `Boolean` | |
| `move_to(path)` | `Boolean` | renames or moves it with everything inside ([below](#rename-or-move-a-file-or-folder)) |

```gdscript title=directory_tasks/directory_tasks.spite entry
var console = Console()

func DirectoryTasks() {
    load "../spite_standard_library/windows"
    var target = Directory(".spite/documentation_demo_dir")
    crash target.create()
    var target_exists = target.exists()
    console.print("exists", target_exists)
    var examples = Directory("examples")
    var listed = examples.entries()
    crash listed
    var has_hello = listed.filter_directories().map_folder_names().contains("hello")
    console.print("has hello", has_hello)
}
```
```output
exists true
has hello true
```

`entries()` answers a `List<Directory.Entry>?`: `null` when the folder does not exist or cannot be opened, so a
missing folder is never read as an empty one. Names are text on every system: `café.txt` lists as `café.txt`, and a
folder holding a name Windows stores as broken UTF-16 lists as `null` rather than as bytes that are not its name. Its member templates keep one kind:
`folder.entries().filter_files()` is a `List<File>` and `filter_directories()` a `List<Directory>`
([a list of a union](metaprogramming.md#member-templates)).

## Walk a directory tree

A `Directory` has a `path`, the way a `File` does, and `entries()` lists what is inside it as a
`List<Directory.Entry>`: each entry is a `Directory` or a `File` whose `path` is already joined to its parent's,
so a `switch` tells them apart and a folder is walked by calling the same function again.

```gdscript title=directory_walk/directory_walk.spite entry
var console = Console()

func DirectoryWalk() {
    load "../spite_standard_library/windows"
    var root = Directory(".spite/documentation_walk")
    var inner = Directory("{root.path}/inner")
    crash root.create()
    crash inner.create()
    var top = File("{root.path}/top.txt")
    crash top.write_string("top")
    var deep = File("{inner.path}/deep.txt")
    crash deep.write_string("deep")
    walk(root)
}

func walk(folder: Directory) {
    var entries = folder.entries()
    crash entries
    var index = 0
    while index < entries.count() {
        var entry = entries[index]
        switch entry {
            Directory: {
                console.print("folder", entry.path)
                walk(entry)
            }
            File: console.print("file", entry.path)
        }
        index = index + 1
    }
}
```
```output
folder .spite/documentation_walk/inner
file .spite/documentation_walk/inner/deep.txt
file .spite/documentation_walk/top.txt
```

Folders come first, then files, each sorted by name, and `.` and `..` are never listed.

## Watch files and folders

`FileSystemWatcher()` is told by the operating system which paths changed, so nothing reads the disk over and over.
Making one takes no arguments and starts nothing: no thread and no job exist until something is watched. One watcher
watches any number of files and folders, and `watch_for_changes` takes either, so a folder or a file a program
already holds is passed as it is.

| Member | Result | Notes |
|---|---|---|
| `watch_for_changes(target: FileSystemWatcher.Target)` | `Boolean` | `Target` is the union of `Directory` and `File`: a `File`, or a `Directory` with everything below it; `false` when there is nothing there to watch |
| `changes()` | `List<String>` | never waits: the paths changed since the last call, each once, or none |
| `wait_for_changes()` | | blocks this thread until `changes()` has something to answer |

A change is reported once the watcher has seen none for 100 ms, so a burst (a save that writes a file in pieces,
a checkout that touches a hundred) comes back as one list, each path once, in the order they first changed.
Each path is the `path` of the watched `Directory` or `File` joined with what is below it, with `/` between. A
folder is reported when it is created, removed or renamed, not when what is inside it changes: the files inside
are reported instead. The 100 ms are counted from when the watcher sees the change, which is in a call, so a
program that calls `changes()` once a frame sees a save about 100 ms after it happened.

```gdscript title=watch_folder/watch_folder.spite entry
var console = Console()
var program = Program()
var _most_tries = 250

func WatchFolder() {
    load "../spite_standard_library/windows"
    var folder = Directory(".spite/documentation_watch")
    crash folder.create()
    var watcher = FileSystemWatcher()
    crash watcher.watch_for_changes(folder)
    crash File("{folder.path}/level.txt").write_string("three goblins")
    var changed = watcher.changes()
    var tries = 0
    var twenty_milliseconds = Duration(20, 'milliseconds')
    while changed.is_empty() and tries < _most_tries {
        program.sleep(twenty_milliseconds)
        changed = watcher.changes()
        tries = tries + 1
    }
    var changed_paths = changed.join(", ")
    console.print("changed:", changed_paths)
    var again = watcher.changes()
    var again_count = again.count()
    console.print("changed since:", again_count)
}
```
```output
changed: .spite/documentation_watch/level.txt
changed since: 0
```

`changes()` suits a program that already runs a loop, a game once a frame, a tool between steps. A thread with
nothing else to do calls `wait_for_changes()` and then `changes()`, which is what `--hot-reload` does
([repl.md](repl.md#how-it-works)); on the program's main thread it would stop everything else, waiting work the
compiler started included, until something changes.

Each system's folder asks its own kernel: `ReadDirectoryChangesW` on the folder, with its sub-folders, on Windows;
an `inotify` watch on each folder on Linux, adding one for each new folder; and a `kqueue` entry on each file and
folder on macOS, adding new files when their folder changes. A file is watched through its folder on Windows and
Linux, so a save that replaces the file is still seen. Nothing of it is in a program that never calls `FileSystemWatcher()`:
not the code, and not the lookups of the system's functions ([optimizations.md](optimizations.md#tree-shaking)).

## Run a process

```gdscript title=process_tasks/process_tasks.spite entry
var console = Console()

func ProcessTasks() {
    load "../spite_standard_library/windows"
    var listing = Process("echo", ["build", "finished"])
    var code = listing.run()
    crash code
    console.print("exit code", code)
    var output = listing.read_output_as_string()
    crash output
    var trimmed_output = output.trim()
    console.print("output", trimmed_output)
}
```
```output
exit code 0
output build finished
```

| Member | Result | Notes |
|---|---|---|
| `Process(command, arguments)` | | `arguments` is a `List<String>`, each one argument of the child, quoted for you (below) |
| `working_directory` | `Path?` | the folder the child runs in; `null`, the default, is the program's own |
| `environment_variables` | `Dictionary<String, String>` | variables set for the child alone, on top of the program's own: `process.environment_variables["LOG"] = "1"` |
| `with_errors` | `Boolean` | `false`; `true` reads the child's error stream into the output too, as the shell's `2>&1` |
| `keeps_errors_apart` | `Boolean` | `false`; `true` reads the child's error stream on its own, for `read_errors_as_string()` |
| `time_limit` | `Duration?` | `null`, the default, waits as long as the child takes; past the limit the child, and everything it started, is stopped and `run()` answers `null` |
| `run()` | `Integer?` | runs it through the system's shell, waits for it and answers its exit code (`null` when it could not start or was stopped at its `time_limit`, so no exit code is made up for it; on Linux and macOS, a child a signal ended answers 128 plus the signal's number, as a shell reports it); captures standard output as bytes on Windows, where byte `0x1A` does not end the output |
| `read_output_as_string()` | `String?` | what it wrote to its standard output; `null` before `run()`, after a run that could not start, or when those bytes are not UTF-8 |
| `read_output_as_buffer()` | `Buffer?` | the same output as the bytes it wrote, for a child that writes bytes; `null` before `run()` or after a run that could not start |
| `read_errors_as_string()` | `String?` | what it wrote to its error stream, when `keeps_errors_apart` is `true`; `null` otherwise, before `run()`, or after a run that could not start |
| `read_errors_as_buffer()` | `Buffer?` | the same error stream as bytes |
| `was_stopped` | `Boolean` | `true` after a run its `time_limit` stopped; what it wrote until then can still be read |
| `run_attached()` | `Integer?` | runs it with the program's own terminal, so what it writes and reads is the user's, and answers its exit code, read as `run()` reads it; `null` when the system could not start it |

Each argument reaches the child whole, as one argument, whatever it holds. On Linux and macOS each is put in single
quotes for the shell, so the child's `argv` holds exactly the text given. On Windows a program reads its own
command line, so an argument is quoted the way Windows splits one back apart (the rules of `CommandLineToArgvW`):
an argument with no space, tab, quote or `& | < > ^` is written as it is; one with them is put in double quotes,
with a `"` inside written `\"` and the backslashes before a quote doubled; and one shaped `key=value` quotes only
its value, so `-script=a b` reaches the child's command line as `-script="a b"` (the form Unreal and other
programs that read `-key=value` themselves expect) and still splits back into `-script=a b`
(`conformance/stage4/process_settings`). A `%` is still read by the Windows shell.

`working_directory` and `environment_variables` are set by the shell line itself, before the command: a
`cd` into the folder and one assignment per variable, so a relative command or path is read from that folder, and
a folder that does not exist fails the run with the shell's exit code. The output is read as it arrives, up to 64 KiB at a time, and each
piece appended in place, so reading 50 MB takes as long as the child takes to write it. It is kept as bytes, and
read as text only when asked, so a child that writes bytes is never forced through a `String`.

The output includes the child process's own trailing newline, which is why the example above calls `.trim()`.
What the child writes to its error stream goes to the program's own error stream, unless `with_errors` reads it into the output or `keeps_errors_apart` reads it on its own:

```gdscript title=process_limits/process_limits.spite entry
var console = Console()
var _patience_seconds = 300

func ProcessLimits() {
    load "../spite_standard_library/windows"
    var timed = Process("cmd", ["/c", "echo", "answer"])
    timed.keeps_errors_apart = true
    timed.time_limit = Duration(_patience_seconds, 'seconds')
    var code = timed.run()
    if not code {
        console.print("stopped", timed.was_stopped)
        return
    }
    var output = timed.read_output_as_string()
    var errors = timed.read_errors_as_string()
    crash output and errors
    var trimmed_output = output.trim()
    var error_bytes = errors.length()
    console.print(code, trimmed_output, "errors:", error_bytes)
}
```
```output
0 answer errors: 0
```

On Windows the output is the bytes the child wrote, so a line ends `\r\n` there. `run()`
is not one of the library's waits: it blocks the thread that calls it, and other waiting work does not run
meanwhile ([which calls wait](#read-and-write-a-file)).

## `Console`

`Console()` is a singleton: the same instance everywhere, so every file holds `var console = Console()`, and
`Console().print(...)` is an error naming it ([classes_and_files.md](classes_and_files.md#singletons)).

| Member | Does |
|---|---|
| `print(...values)` | each value's `to_string()`, separated by a space, then a line break |
| `write(...values)` | the same without the line break |
| `error(...values)` | like `print`, to the error stream |
| `debug(...values)` | each value's `to_debug()`, separated by a space, then a line break |
| `read_line()` | one line of input without its line break, however long, as a `String?`: `null` at the end of the input, and for a line whose bytes are not UTF-8 (the next call reads the line after it) |

**You never flush.** The compiler chooses how printed output reaches its destination, and whatever it chooses,
three things hold: a line is never split, nor mixed with a line another thread prints at the same time; everything
printed is out before the program exits and before a crash report; and text written before the program reads
input, a prompt, is visible before the read. `print`, `error` and `debug` end a line and hand it to the operating
system at once, whether the output is a terminal, a file or a pipe, so a server's log redirected to a file shows
each line when it happens rather than when the program ends. `write` leaves its text for the next line end, the
next read of input or the end of the program
([the rule and its measurement](https://github.com/Spite-Language/spite_spec/blob/master/specs/standard_library.md#system-classes)). There is no `flush()` to call.

A printed line is the one write that answers nothing: if the system refuses it (a closed pipe, a terminal that
went away), the line is lost, since the place a failure would be reported is the output that just failed.

`print`, `write` and `error` are ordinary functions of `Console`, taking
`...values: List<Printable>`, where `Printable` is a `type` that requires `to_string(): String`. Every number,
`Boolean`, `String`, enum value, `Spite.Class` and `Spite.Namespace` answers it, and a class of yours
prints once it declares `to_string()`. Passing one that does not is a compile error naming `to_string`; a `List`
or `Dictionary` has none either, so show it with `debug` or `join` it first
([the exact errors](https://github.com/Spite-Language/spite_spec/blob/master/specs/standard_library.md#system-classes)).

```gdscript title=printable_doc/ticket.spite
enum Stage {
    'boarding'
}

var code = ""
var seats = 0

func Ticket(starting_code: String, starting_seats: Integer) {
    code = starting_code
    seats = starting_seats
}

func to_string(): String {
    return "{code} for {seats}"
}
```
```gdscript title=printable_doc/printable_doc.spite entry
var console = Console()

func PrintableDoc() {
    load "../spite_standard_library/windows"
    var ticket = Ticket("A12", 3)
    var boarding: Ticket.Stage = 'boarding'
    console.print("next:", ticket, boarding, true, 2.5, ticket.class)
}
```
```output
next: A12 for 3 boarding true 2.5 Ticket
```

`debug` is for reading a program's state rather than for its output. It takes
`...values: List<Debuggable>` (`to_debug(): String`), and every value answers that without writing anything: a
class shows its name and every attribute, nesting into the attributes that are classes, a `List` shows `[a, b]`, a
`Dictionary` `{"key": value}`, text is quoted, an enum value is written `'like_this'`, and an absent
`T?` is `null`. An object already being shown further up is written `Name {...}`, so a cycle ends. A class that
declares its own `to_debug()` is shown by it wherever it appears, and a class nothing asks to show gets none:
the automatic one is `Spite.Debug<$value_type>`, compiled only for the classes a
program debugs.

```gdscript title=debug_doc/crew.spite
var captain = ""
var sailors = List<String>()
var ship: Ship? = null
```
```gdscript title=debug_doc/ship.spite
var name = ""
var crew: Crew? = null

func Ship(starting_name: String) {
    name = starting_name
}
```
```gdscript title=debug_doc/debug_doc.spite entry
var console = Console()

func DebugDoc() {
    load "../spite_standard_library/windows"
    var crew = Crew()
    crew.captain = "Ana"
    crew.sailors.append("Bo")
    var ship = Ship("Gull")
    crew.ship = ship
    ship.crew = crew
    console.debug(crew, 3, "done")
    ship.crew = null
}
```
```output
Crew { captain: "Ana", sailors: ["Bo"], ship: Ship { name: "Gull", crew: Crew {...} } } 3 "done"
```

```gdscript title=console_input_doc/console_input_doc.spite entry
var console = Console()

func ConsoleInputDoc() {
    load "../spite_standard_library/windows"
    console.write("name? ")
    var name = console.read_line()
    if name {
        console.print("hello", name)
    } else {
        console.print("no input")
    }
}
```
```output
name? no input
```

## `Program`

`Program()` is this running program, a singleton bound once as `var program = Program()`.

| Member | Does |
|---|---|
| `exit(code)` | flushes what was printed and ends the process with `code` |
| `sleep(duration)` | waits the `Duration`'s whole milliseconds; other waiting work runs meanwhile ([concurrency.md](concurrency.md)). A duration under a millisecond, zero or negative only lets other work have its turn |
| `environment(name)` | the process environment variable, as a `String?`: `null` when it is not set |
| `executable_path()` | the path of the running executable, as the operating system gives it, as a `String?`: `null` when the system refuses or the path is not UTF-8 |
| `working_directory()` | the folder the program runs in, as a `String?`: `null` when the system refuses or the path is not UTF-8 |
| `live_allocations()` | how many allocations are alive |
| `live_bytes()` | how many bytes the live allocations asked for |

A program that calls `live_allocations()` or `live_bytes()` counts every allocation and free it makes, as a
`--debug-memory` build does; one that calls neither carries none of the counting.

The entry constructor returning normally is exit code `0`.

```gdscript title=program_basics/program_basics.spite entry
var console = Console()
var program = Program()

func ProgramBasics() {
    load "../spite_standard_library/windows"
    var missing = program.environment("SPITE_DOCUMENTATION_UNSET_VARIABLE")
    if missing {
        console.print("set to", missing)
    } else {
        console.print("not set")
    }
    var one_millisecond = Duration(1, 'milliseconds')
    program.sleep(one_millisecond)
    console.print("slept")
}
```
```output
not set
slept
```

## `Clock`

`Clock()` is a singleton with two readings:

| Member | Does |
|---|---|
| `elapsed(): Duration` | the monotonic clock, a [`Duration`](time.md) from an arbitrary start: subtract two readings to measure |
| `now(): Instant` | the wall clock, as an exact [`Instant`](time.md): show it through a time zone |

Each system reads its own clock, in its own folder of the library (`QueryPerformanceCounter`,
`GetSystemTimeAsFileTime` on Windows, `clock_gettime` with `CLOCK_MONOTONIC` and
`CLOCK_REALTIME`), through `DynamicLibrary`. The operating system writes a reading into a slot in the calling function's frame
([placement](https://github.com/Spite-Language/spite_spec/blob/master/specs/memory.md#placement-the-compiler-decides-where-memory-lives)),
and Windows' counter frequency, fixed at boot, is read once when `Clock()` is first made. A reading is a `Duration`,
two `Long`s, exact to the nanosecond on every system; the resolution is the hardware's (100 ns for Windows' usual
10 MHz counter, 1 ns on Linux and macOS). `clock.elapsed() - started` is the time since `started`, and
`taken.total('microseconds')` is it in whole microseconds.

```gdscript title=clock_basics/clock_basics.spite entry
var console = Console()
var clock = Clock()
var program = Program()

func ClockBasics() {
    load "../spite_standard_library/windows"
    var five_milliseconds = Duration(5, 'milliseconds')
    var started = clock.elapsed()
    program.sleep(five_milliseconds)
    var waited = clock.elapsed() - started
    var at_least = Duration(4, 'milliseconds')
    console.print("waited at least 4 ms:", waited >= at_least)
}
```
```output
waited at least 4 ms: true
```

To time one function rather than a stretch of code, `Benchmark(work)` reads the clock on both sides of a single
call and keeps what the function answered beside the `Duration` it took
([Measuring a piece of work](time.md#measuring-a-piece-of-work)).

## `IpAddress`

An `IpAddress` is an IPv4 or an IPv6 address and nothing else: a port is always a number beside it. TCP and UDP
take one, and a datagram says who sent it with one.

```gdscript title=ip_address_forms/ip_address_forms.spite entry
var console = Console()

func IpAddressForms() {
    load "../spite_standard_library/windows"
    var read: IpAddress? = "2001:DB8:0:0:0:0:0:7"
    crash read
    var is_ipv6 = read.is_ipv6()
    console.print("{read} {is_ipv6}")
    var mapped: IpAddress? = "::ffff:10.0.0.1"
    var plain: IpAddress? = "10.0.0.1"
    crash mapped
    crash plain
    var same = mapped == plain
    console.print("{mapped} {same}")
    var name: IpAddress? = "localhost"
    crash name
    console.print("localhost is first {name}")
    var nowhere: IpAddress? = "no-such-host.invalid"
    if not nowhere {
        console.print("a name that resolves to nothing is null")
    }
    var resolved = "localhost".resolve_ip_addresses()
    var first = resolved[0]
    crash first
    console.print("localhost resolves first to {first}")
}
```
```output
2001:db8::7 true
10.0.0.1 true
localhost is first 127.0.0.1
a name that resolves to nothing is null
localhost resolves first to 127.0.0.1
```

| Member | Does |
|---|---|
| `var read: IpAddress? = "text"` | reads dotted IPv4 (`"192.168.1.20"`) or IPv6 (`"::1"`, `"2001:db8::7"`, `"::ffff:10.0.0.1"`, `"fe80::1%4"`), or resolves a host name (`"localhost"`, `"example.com"`) to its first address, IPv4 first, waiting in place; `null` when the text is neither, or names nothing. An address from a name keeps the name, which `Http.Client` writes as the `host` header and `WebSocket.Client` in its handshake |
| `"host".resolve_ip_addresses(): List<IpAddress>` | asks the system for every address a host name (or an address written as text) names, the IPv4 ones first; empty when it names none. It waits in place |
| `is_ipv4(): Boolean`, `is_ipv6(): Boolean` | which kind it is |
| `==`, `!=` | the same address or not (its `equals(other)`) |
| `to_string(): String` | the address as RFC 5952 writes it: lowercase, the longest run of zero groups as `::`, an IPv6 zone after a `%` |
| `IpAddress()` | `0.0.0.0` |

An IPv4 address inside `::ffff:` is that IPv4 address, so a datagram a dual-stack server receives from `127.0.0.1`
comes from `127.0.0.1`. There is no constructor that takes text, and no network call takes text: text becomes an address through
`to_ip_address()`, which can answer `null`, and every call that says where to go takes an `IpAddress`. Once the compiler casts text where an `IpAddress` is wanted and folds a
literal it can read (spite_compiler#228, #229), `client.connect("127.0.0.1", 8080)` needs nothing written; until
then a program narrows `var read: IpAddress? = "127.0.0.1"` first.

## TCP

TCP is three classes over IPv4 and IPv6, the same on Windows (winsock), Linux and macOS: a `Tcp.Server` listens
and accepts, a `Tcp.Client` connects, and each end of a connection talks the same way. What a server accepts is a
`Tcp.Connection`; a client holds one and talks through it. `--repl-port` and `spite --connect` are written with them,
and so is a game server.

```gdscript title=tcp_echo/tcp_echo.spite entry
var console = Console()
var server = Tcp.Server()
var _echo_port = 47401

func TcpEcho() {
    load "../spite_standard_library/windows"
    crash server.listen_locally(_echo_port)
    var client = Tcp.Client()
    crash client.connect_locally(_echo_port)
    var peer = server.accept()
    crash peer
    crash client.write_line("ping")
    var heard = peer.read_line()
    crash heard
    crash peer.write_line("{heard} pong")
    var answer = client.read_line()
    crash answer
    console.print(answer)
    client.close()
    var after = peer.read_line()
    if not after {
        console.print("the client left:", peer.closed)
    }
    peer.close()
    server.close()
}
```
```output
ping pong
the client left: true
```

| `Tcp.Server` | Does |
|---|---|
| `listen_locally(port)`, `listen_everywhere(port)` | listens on `127.0.0.1`, or on every interface of both IPv4 and IPv6; `false` when it cannot |
| `listen_at(address: IpAddress, port)` | listens on the one interface an address names (a host name's, through `var read: IpAddress? = "localhost"`, its first address, IPv4 first); `false` when it cannot |
| `accept(): Tcp.Connection?` | waits for the next client; `null` when the server is not listening |
| `close()` | stops listening |

| `Tcp.Client` | Does |
|---|---|
| `connect(address: IpAddress, port)` | connects to an address (a host name's through `var read: IpAddress? = "example.com"`); `false` when nobody answers |
| `connect_locally(port)` | connects to `127.0.0.1` |

| A `Tcp.Client` and a `Tcp.Connection` | Do |
|---|---|
| `read_line(): String?` | waits for a whole line, without its line break; `null` when the connection ended first |
| `read_as_buffer(): Buffer?` | waits until at least one byte has arrived and answers what arrived; `null` once the connection is closed |
| `read_as_string(): String?` | the same read as text, finishing a character the network cut at its end; `null` once closed, or when the bytes are not UTF-8 |
| `write_line(text): Boolean`, `write_string(text): Boolean`, `write_buffer(buffer): Boolean` | sends all of it, waiting while the system's buffer is full |
| `closed: Boolean` | `true` once the other end has closed the connection (or it broke, or `close()` was called) |
| `close()` | closes it |

**Every read waits.** There is one read per kind, and it waits for what it answers or for the end: inside a
state machine the compiler writes, `accept`, `read_line`, `read_as_string` and `read_as_buffer` are points it returns
from, so other work runs meanwhile ([concurrency.md](concurrency.md#what-the-compiler-does-at-a-wait)). A frame
loop that serves connections keeps running because the compiler parks the wait (spite_compiler#218, #219), not
because the program asks whether anything has arrived. Resolving a host name and connecting always wait in place.

**A closed peer is an answer, not a failure.** A read answers `null` when the connection has ended, and sets
`closed`; `read_line()` answers `null` too for a line whose bytes are not UTF-8, and the next call reads the line
after it, so a peer's bytes never ride inside a `String`. A write to a closed connection sends nothing, answers `false`, and sets `closed` too; nothing crashes.
Every read answers a buffer of its own, as a `Udp.Datagram`'s does, and the numbers in it are read from it
([below](#bytes-are-a-buffer)). UDP, HTTP and WebSocket are [below](#udp).

## UDP

UDP is a `Udp.Server`, open on a port a client knows, and a `Udp.Client`, open on a port the system picks, both
over IPv4 and IPv6, the same on Windows, Linux and macOS. One `receive()` is one datagram, a `Udp.Datagram` that
holds its bytes and who sent them, read as text or as a `Buffer` once it is in hand, so nothing received is lost
by asking for the wrong one.

```gdscript title=udp_ping/udp_ping.spite entry
var console = Console()
var _ping_port = 47402

func UdpPing() {
    load "../spite_standard_library/windows"
    var server = Udp.Server()
    crash server.open_locally(_ping_port)
    var client = Udp.Client()
    crash client.open()
    var ping = "ping".to_buffer()
    var server_address: IpAddress? = "127.0.0.1"
    crash server_address
    crash client.send_to(server_address, _ping_port, ping)
    var arrived = server.receive()
    crash arrived
    var text = arrived.read_as_string()
    crash text
    var sender = arrived.sender_address
    var sender_port = arrived.sender_port
    console.print("{text} from {sender}")
    var pong = "pong".to_buffer()
    crash server.send_to(sender, sender_port, pong)
    var answer = client.receive()
    crash answer
    var answer_text = answer.read_as_string()
    crash answer_text
    console.print(answer_text)
}
```
```output
ping from 127.0.0.1
pong
```

| `Udp.Server` | Does |
|---|---|
| `open_locally(port)`, `open_everywhere(port)` | opens on `port` of `127.0.0.1`, or of every interface of both IPv4 and IPv6; port `0` is one the system picks; `false` when the port is taken |
| `open_at(address: IpAddress, port)` | opens on the one interface an address names; `false` when the port is taken |

| `Udp.Client` | Does |
|---|---|
| `open()` | opens on a port the system picks, on every interface of both IPv4 and IPv6 |

| A `Udp.Server` and a `Udp.Client` | Do |
|---|---|
| `port: Integer` | get-only: the port it is open on, the one the system gave it after `open()` (asked of the system with `getsockname`), so a client can tell a server where to answer; reading it when it is not open halts, since it has none, and a program that wants the number after `close()` keeps it in a `var` before closing, which says it means to |
| `send_to(address: IpAddress, port, buffer): Boolean` | sends one datagram; `false` when the socket cannot reach the address (an IPv6 one from `open_locally`) or the system refuses it |
| `receive(): Udp.Datagram?` | waits for the next datagram; `null` when it is closed |
| `close()` | closes it |

| `Udp.Datagram` | Does |
|---|---|
| `read_as_string(): String?` | the bytes as text, `null` when they are not UTF-8 |
| `read_as_buffer(): Buffer` | the bytes as they are |
| `sender_address: IpAddress`, `sender_port: Integer` | get-only: who sent it, to answer with `send_to`; it travels with the datagram, so two threads receiving on one socket never see each other's sender |

A datagram can arrive empty, which is a list of no bytes, not `null`. UDP itself neither orders nor repeats lost
datagrams; reliable channels are the program's.

## HTTP

`Http.Server` and `Http.Client` speak HTTP/1.1 over `Tcp.Client` and `Tcp.Connection`, keeping a connection open for the next exchange. A client writes the requested host and includes a nonstandard port in the `host` header, with IPv6 addresses bracketed before the port.
A message is an `Http.Request` (`method`, `path`, `version`, `headers`, a body) or an `Http.Response` (`status`,
`headers`, a body); `header(name)` reads a header whatever its case and `set_header(name, value)` writes one, and
`reason()` is the response's reason phrase. A body is bytes, one holder with two reads: `read_body_as_string()`
(a `String?`, `null` when the body is not UTF-8) and `read_body_as_buffer()`, written with `write_body_string(text)`
or `write_body_buffer(buffer)`, so an image or a compressed body is never forced through a `String`.

| Member | Does |
|---|---|
| `Http.Server.listen_locally(port)`, `listen_everywhere(port)` | listens, like `Tcp.Server` |
| `Http.Server.next_request(): Http.Request?` | waits for the next well-formed request, on a new connection or on one kept open; a malformed one is answered `400 Bad Request` and its connection closed, and a body over `largest_body` (1 MiB) counts as malformed; `null` when the listener fails or is closed |
| `Http.Server.respond(request, response): Boolean` | writes the response with its `content-length`, and keeps the connection open for the client's next request unless the client asked to close it; `false` when the response could not be written whole |
| `Http.Server.close()` | stops listening and closes every connection kept open |
| `Http.Client.send(address: IpAddress, port, request): Http.Response?` | sends the request with `host` (the name the address was resolved from, or the address itself) and `content-length` over a connection kept open to that host and port, or a new one, and reads the response, whether its body is sized, chunked or ends when the connection closes; `null` when nobody answers or the response is malformed |
| `Http.Client.close()` | closes the connections it keeps open |

**Connections are kept alive.** HTTP/1.1 keeps a connection open between exchanges, and both sides do: a server
answers `connection: keep-alive` and waits on every open connection and on its listener at once, and a client
keeps the connection after a response and sends its next request to the same host and port over it. A connection
closes when either side says `connection: close` (a request that sets that header gets its answer and then the
close), when an HTTP/1.0 client does not ask for `keep-alive`, when a response's body ends with the connection,
and when it has been idle too long: a server closes one after `idle_limit`, a `Duration` (5 seconds), without a request,
and keeps at most `largest_waiting` (64) open, closing each further one after its answer; a client keeps at most
`largest_idle` (8). A kept connection the server has closed meanwhile is noticed when the client's next request
gets no answer at all, and that request is sent once more over a new connection. Requests on one connection are
answered in order, and requests a client sent before reading the answers (pipelining) are read one at a time.
A response to `HEAD`, and a `204` or `304`, carries no body. A server still refuses a request with a chunked body.

Inside a state machine the compiler writes, `next_request` and `send` wait the way a `Tcp.Connection` does, so a server serves while its program
runs ([concurrency.md](concurrency.md#what-the-compiler-does-at-a-wait)). A server reads one request at a time:
while a client is slow to send the rest of a request, the other connections wait for it.

## WebSocket

A WebSocket keeps one connection open in both directions and carries messages over it, text or bytes, from
either side at any time: what a browser game or a chat speaks once its page has loaded. A connection starts as an
HTTP request, so the server is built on an `Http.Server`, the way an Express app serves both from one port:
`WebSocket.Server(http)` takes the server the program already has. The program keeps its own HTTP loop, asks
`asks_to_upgrade(request)`, and `accept(request)` answers a `WebSocket.Connection`; every other request is the
program's to answer.

```gdscript
var http = Http.Server()
var sockets = WebSocket.Server(http)

func serve() {
    var request = http.next_request()
    crash request
    if not sockets.asks_to_upgrade(request) {
        var page = Http.Response()
        page.write_body_string("hello")
        fallback http.respond(request, page)
        return
    }
    var connection = sockets.accept(request)
    fallback connection
    var message = connection.receive()
    while message {
        if message.is_text() {
            var text = message.read_as_string()
            crash text
            fallback connection.send_string("you said {text}")
        } else {
            var bytes = message.read_as_buffer()
            fallback connection.send_buffer(bytes)
        }
        message = connection.receive()
    }
}
```

`accept` answers `null` when the upgrade is malformed, after answering it itself (`400 Bad Request`, or `426 Upgrade
Required` for a version other than 13), so a request is never answered twice. A client asks with an
`Http.Request`, whose `path` and headers (an `origin`, a cookie, the subprotocols it speaks in
`sec-websocket-protocol`) go out with the handshake:

```gdscript
var _chat_server_port = 8080

func talk() {
    var socket = WebSocket.Client()
    var request = Http.Request()
    request.path = "/chat"
    var server_address: IpAddress? = "127.0.0.1"
    crash server_address
    crash socket.connect(server_address, _chat_server_port, request)
    fallback socket.send_string("hello")
    var answer = socket.receive()
    crash answer
    var text = answer.read_as_string()
    crash text
    console.print(text)
    socket.close()
}
```

Both sides talk through the same members, the client's and a `WebSocket.Connection`'s, and every message
received is a `WebSocket.Message`, one holder with two reads: `is_text()`, `read_as_string()` (a `String?`,
`null` for a binary message) and `read_as_buffer()`.

| Member | Does |
|---|---|
| `WebSocket.Server(http)` | the upgrades of requests `http.next_request()` hands out |
| `asks_to_upgrade(request): Boolean` | whether the request asks for a WebSocket (its `upgrade` header is `websocket`) |
| `accept(request): WebSocket.Connection?` | answers the request with `101 Switching Protocols` and keeps its connection; `null` after refusing a malformed upgrade; a request that does not ask to upgrade halts |
| `WebSocket.Client().connect(address: IpAddress, port, request): Boolean` | connects like `Tcp.Client.connect`, writing the name the address was resolved from (or the address) as the `host`, sends the handshake with the request's path and headers, and checks the server's answer; `false` when nobody answers or the answer is not a WebSocket one |
| `send_string(text): Boolean`, `send_buffer(buffer): Boolean` | sends one message, a text or a `Buffer`; `false`, sending nothing, once the connection has ended or `close()` was called |
| `send_ping(bytes): Boolean` | sends a ping carrying up to 125 bytes |
| `receive(): WebSocket.Message?` | waits for the next whole message, text or bytes; `null` once the connection has ended |
| `close()` | starts the closing handshake with code 1000; messages already on their way still arrive through `receive()`, which answers `null` once the other side has answered the close |
| `closed: Boolean` | `true` once the connection has ended, and on a client before `connect` has opened it |
| `close_code: Integer?` | how it ended: the code of the closing handshake, `null` while it is open |
| `close_reason: String` | the text the other side's close carried, often empty |
| `last_pong: Buffer?` | what the latest pong carried, `null` until one arrives |
| `largest_message: Integer` | on a `WebSocket.Connection`: the longest message `receive()` takes, 16 MiB unless set |

**What arrives is checked, and a broken connection says so.** Pings are answered with a pong while `receive()`
waits, and a message sent in pieces arrives whole. Anything the protocol forbids ends the connection instead of
being skipped: the side that sees it sends a close with the code that names the problem, and its `receive()`
answers `null` with `close_code` set to that code. The codes are the protocol's own:

| `close_code` | Means |
|---|---|
| `1000` | a normal close, by either side |
| `1001` | the other side went away (a connection dropped without `close()` sends it) |
| `1002` | a frame the protocol forbids: one a client did not mask (or a server did), a reserved bit or opcode, a piece of a message that was never started, a ping or close too long or in pieces, a close code nobody may send |
| `1005` | the other side closed without giving a code |
| `1006` | the connection broke without a close at all |
| `1007` | a text message, or a close's reason, that is not valid UTF-8 |
| `1009` | a message longer than `largest_message` |
| `1003`, `1008`, `1010` to `1014`, `3000` to `4999` | whatever the other side closed with |

Inside a state machine the compiler writes, `receive()`, `accept` and `connect` wait the way a `Tcp.Connection` does
([concurrency.md](concurrency.md#what-the-compiler-does-at-a-wait)). `wss`, WebSocket over TLS, belongs here as well.

## Bytes are a `Buffer`

Bytes a program reads or writes are a `Buffer`, whoever hands them over: a `File`, a `Tcp.Connection`, a `Udp.Datagram`, a
`WebSocket.Message`, an HTTP body, a `Process`'s output, a `BinaryWriter`. A file format or a network message is
numbers laid out in bytes, so a `Buffer` reads and writes those numbers, in the byte order the format uses, at the
position the format puts them.

A `Buffer` is still a run of bytes, and everything a list of bytes does it does the same way: `count()`,
`is_empty()`, `buffer[index]` (a `Byte?`), `buffer[index] = byte`, `append(byte)`, `truncate(count)`, `clear()`,
`copy()`, and the templates and passed functions a list is walked with
([collections.md](collections.md#listt)). It keeps the bytes themselves in one block, so a read is one load and there
is no list hiding inside it. How much room it keeps is the library's and the compiler's.

**There is no list of bytes.** `List<Byte>` is a compile error wherever it is written, and wherever the compiler
would make one: a run of bytes has one class, so there is one way to hold it, and it is the class that can read
the numbers in it. Write `Buffer(size)` instead
([the rule](https://github.com/Spite-Language/spite_spec/blob/master/specs/standard_library.md#bytes-are-a-buffer)).

**Byte order is said in each name.** A number of two bytes or more is read and written through a function that
names its order after the number: `_little_endian`, the order of every machine Spite compiles for and of Spite's own
binary format, or `_big_endian`, which puts the most significant byte first (PNG, network order). So a call says what
it reads, one buffer may hold both orders, each read where its format says, and the order of every read is known
while compiling. A `byte` and a `tiny` are one byte and have no order. A buffer is made with a size,
`Buffer(size)`, for the bytes the program knows it will write (`Buffer(0)` when nothing is known): the size is a
suggestion for room, never a limit, and the buffer grows past it as always. The one place the room matters is a
foreign function that writes into it: pass `Buffer(4096)`, then `take_written(count)` says how many bytes it
wrote ([foreign_libraries.md](foreign_libraries.md)).

**Every read says where.** A buffer has no reading position of its own: each read names its position, and a
program walking a message keeps its own, `var at = 0`, moved past what it read. There is one way to read a number.

| Member | Result | Notes |
|---|---|---|
| `read_<number>_at(position)` | `<number>?` | the number whose bytes start at `position`; `null` unless every byte is in the buffer |
| `append_<number>(value)` | | adds the number's bytes at the end |
| `write_<number>_at(position, value)` | | replaces the bytes at `position`; a position whose bytes are not all in the buffer halts |
| `read_buffer_at(position, count)` | `Buffer?` | a new buffer of the `count` bytes at `position`; `null` unless every byte is there |
| `append_buffer(other, start, count)` | | copies `count` bytes of `other`, from `start`, to the end; a run not inside `other` halts |
| `read_string_at(position, count)` | `String?` | the text the `count` bytes at `position` spell, `null` when they are not valid UTF-8 |
| `read_as_string()` | `String?` | the text all the bytes spell, `null` when they are not valid UTF-8 ([below](#bytes-base64-compression-hashes-and-passwords)) |

`<number>` is `tiny`, or one of `short`, `unsigned_short`, `integer`, `unsigned_integer`, `long`, `unsigned_long`,
`float` or `double` followed by `_little_endian` or `_big_endian`, so the functions are `read_integer_little_endian_at(0)`,
`read_unsigned_short_big_endian_at(4)`, `append_double_little_endian(value)`, and so on; one byte anywhere is
`buffer[position]` and one more at the end is `append(byte)`. A `List` has none of these functions, whatever it holds: calling one on a list is a compile
error that says to keep the bytes in a `Buffer`.

A read answers a `T?` because the bytes came from outside: narrow it, or prove it. One bound proves every `_at` read
under it: after `crash body + 10 <= header.count()`, a read of width `w` at `body + k` with `k + w <= 10` is a plain
number, and in a loop over the buffer (`while position + 15 < records.count()`, stepping `position` by up to 16) each
read inside the window is one load, with no test at all
([proofs.md](proofs.md#a-proven-count-or-bound-proves-a-read-of-a-width)). An index built from loop counters,
`row * stride + column`, is proven by one `crash height * stride <= pixels.count()` before the loops
([proofs.md](proofs.md#a-proven-count-or-bound-proves-a-read)).

```gdscript title=byte_numbers/byte_numbers.spite entry
var console = Console()
var _first_record = 1200

func ByteNumbers() {
    load "../spite_standard_library/windows"
    var message = Buffer(10)
    message.append_unsigned_short_big_endian(2)
    message.append_integer_big_endian(_first_record)
    message.append_integer_big_endian(-35)
    var records = read_records(message)
    console.print("records:", records)
    var missing = message.read_long_big_endian_at(6)
    console.print("a read past the end is null:", not missing)
}

func read_records(message: Buffer): Integer {
    var at = 0
    var count = message.read_unsigned_short_big_endian_at(at)
    crash count
    at = at + 2
    var first = message.read_integer_big_endian_at(at)
    crash first
    at = at + 4
    var second = message.read_integer_big_endian_at(at)
    crash second
    console.print(count, "records:", first, second)
    return count
}
```
```output
2 records: 1200 -35
records: 2
a read past the end is null: true
```

A whole value (a class, a `type`, a list of them) is not read number by number: `BinaryReader<T>` reads it from a
buffer and `BinaryWriter<T>` writes it into one, the way `JsonReader<T>` and `JsonWriter<T>` do with text
([json.md](json.md#write-and-read-bytes)). Their format is Spite's own; a format someone else defined is read with
the buffer's numbers, as above. The rules, the edge cases and the error texts are in
[the specification](https://github.com/Spite-Language/spite_spec/blob/master/specs/standard_library.md#bytes-are-a-buffer).

**Memory C hands out.** A foreign library sometimes answers an address and a size, such as a GPU buffer mapped for the
program to fill. `ForeignBytes(address, count)` wraps it: `count()`, `foreign[position]` and `foreign[position] =
value` for one byte, the same `read_<number>_at` and `write_<number>_at` functions, with their byte order in the name, as a `Buffer`
(each read a `T?`, each write halting outside the memory), `read_buffer_at(position, count): Buffer?` and
`write_buffer_at(position, buffer)`. It has no position of its own. The memory stays C's: `ForeignBytes` never frees
it, and it is only as alive as the library says it is.

## Bytes: base64, compression, hashes and passwords

These classes take and answer a `Buffer`, and `text.to_buffer()` gives a text's bytes as one. `bytes.read_as_string()`
turns bytes back into text: it answers a `String?`, `null` when the bytes are not valid UTF-8 (a stray continuation
byte, a cut sequence, an overlong form, a surrogate or a value past U+10FFFF), so bytes from outside are never
taken for text they do not spell (`conformance/stage6/utf8_text`). Each is written in Spite, so a program that uses none of them
carries none of them.

| Class | Member | Does |
|---|---|---|
| `Base64` | `encode(bytes): String`, `encode_url(bytes): String` | RFC 4648 base64 with `=` padding; base64url (`-` and `_`) without it, as JWTs write it |
| | `decode(text): Buffer?`, `decode_url(text): Buffer?` | the bytes back; `null` for a character outside the alphabet, a length no encoding has, missing padding in `decode`, or set bits after the last byte |
| `Deflate`, `Zlib`, `Gzip` | `compress(bytes): Buffer` | raw DEFLATE (RFC 1951), zlib (RFC 1950, Adler-32) or gzip (RFC 1952, CRC-32 and size) |
| | `decompress(bytes): Buffer?` | the bytes back, with no length given in advance; `null` for a truncated or malformed stream, a checksum or size that does not match, or bytes after the end |
| `Sha256` | `hash(bytes): Buffer` | the 32-byte SHA-256 digest (FIPS 180-4) |
| | `hmac(key, message): Buffer` | HMAC-SHA256 (RFC 2104), a key longer than 64 bytes hashed first |
| `SecureRandom` | `random_buffer(count): Buffer?` | bytes from the operating system's secure source: `BCryptGenRandom` on Windows, `getrandom` on Linux, `getentropy` on macOS; `null` when the source gives none, and a negative `count` crashes |
| `Argon2` | `hash(password): String?` | `null` when no random bytes came for the salt; otherwise a new Argon2id (RFC 9106) hash as a PHC string, `$argon2id$v=19$m=65536,t=3,p=1$<salt>$<tag>`: 64 MiB, 3 passes, 1 lane, a 16-byte random salt and a 32-byte tag |
| | `verify(password, stored): Boolean?` | whether the password is the one `stored` was made from, for `argon2id`, `argon2i` and `argon2d` PHC strings of version 19 with any cost; `null` when `stored` is not such a string, so a damaged record is not taken for a wrong password; the tags are compared in constant time |
| | `derive(password, salt, time_cost, memory_cost, lanes, tag_length): Buffer` | the raw Argon2id tag, `memory_cost` in KiB |

A hash takes about 150 ms in an ordinary build and a few seconds in a `--debug-memory` one, whose arithmetic is
checked; it waits in place, so a server runs it on the thread pool. The `Argon2` hash tree-shakes to Blake2b and
the fill of its memory, both private to the class.

---

Next: [Lists, dictionaries and member templates](collections.md), the containers every program uses and how they are extended.
