# Proofs the compiler makes

A **proof** is a fact the compiler establishes while compiling: that a `T?` holds a value here, that an index is in
range, that an object never leaves its function, that no other thread can reach a singleton. With the fact in hand
the compiler does one of four things: it **accepts** code it would otherwise refuse, it **drops** a check it would
otherwise emit, it **chooses cheaper code**, or it **refuses** code that would go wrong. A proof is worked out while
compiling and emits nothing of its own; what it costs, if anything, is the check it leaves in place when it does not
hold.

**A proof that does not hold never fails silently.** Either the compiler refuses the code and says what to write, or
it keeps the run-time check. So each entry ends with **Falls back**: what happens when the proof does not apply, and
what you write yourself. That is the part to read before relying on a proof.

This page is a catalogue, not a second home for the rules. Each entry says in a few lines what is proven, and the
[specification](https://github.com/Spite-Language/spite_spec/blob/master/specs/README.md) states it in full; where
an entry and the specification differ, the specification decides. The optimisations built on these proofs are on
[Optimisations](optimizations.md).

Every entry has the same parts:

- **Proves**: the fact.
- **Rule**: how the compiler decides it, stated so you can predict it.
- **Buys**: the code accepted or refused, the check removed, the cost saved.
- **Falls back**: when it does not apply, and what you write instead.

Proofs that decide whether code is accepted never depend on how well the program is optimised: the same program is
accepted or refused whatever the optimisation settings.

## Which proofs apply to my code

- **Reading a `T?`** (a `Monster?`, a `[]` read, `first()`): narrow the name or the path itself with `if`,
  `fallback`, `crash`, `while` or `switch`, or leave early with `if not value { return ... }`. A condition with a call
  in it narrows nothing: put the result in a `var` and narrow that.
  [Narrowing](#narrowing-a-t), [an `if` that leaves](#an-if-that-leaves-proves-the-rest).
- **A call between your check and your read**: the proof survives unless the call may assign what you read or
  shrink a list you read. [A call keeps a proof](#a-call-keeps-a-proof-it-cannot-change).
- **Reading `list[index]`**: a bound (`index < list.count()`), a count (`list.count() >= 3`), a count kept in a
  `var`, or `not list.is_empty()` proves it. [Proven reads](#a-proven-count-or-bound-proves-a-read).
- **Reading numbers from bytes**: one bound covers a header (`crash body + 13 <= bytes.count()`).
  [A read of a width](#a-proven-count-or-bound-proves-a-read-of-a-width).
- **A tight loop over numbers**: `while index < values.count()` with `index = index + 1` last.
  [Counted loops](#a-counted-loop-reads-its-items-unchecked).
- **Dividing whole numbers**: prove the divisor against a written `0`, or divide by a constant.
  [Proven divisor](#a-proven-divisor-is-not-checked).
- **Arithmetic that could overflow**: every build checks it; a range can prove the check unneeded.
  [Overflow](#arithmetic-that-does-not-fit-halts), [ranges](#a-range-proves-arithmetic-fits).
- **A function that answers something**: end every path with `return` or a bare `crash`.
  [Every path ends](#every-path-ends-in-a-return).
- **A `switch`**: name every member or value; there is no `_:`. [Switches](#a-switch-covers-every-case).
- **Generic code asking about `$T`**: the questions fold and only the branch taken is compiled.
  [Compile-time questions](#compile-time-questions).
- **Small objects of numbers**: they live in registers or the frame when they never leave their function.
  [Frame objects](#objects-that-never-leave-their-function-live-in-the-frame).
- **Items read from a list**: an item a name holds or a call is handed is not counted while the list keeps it.
  [Items read from a list](#items-read-from-a-list).
- **Work that runs at once**: write nothing; the compiler proves which work is independent. [Threads](#threads-and-locks).

## Presence and control flow

### Narrowing a `T?`

- **Proves**: a `T?` name or path holds a value, so inside the block it is a plain `T` under the same name.
- **Rule**: the condition must be exactly a name or a path: names, `.member` and `[index]` links with no call in
  them. `if value { }` narrows its block and its `else` nothing; `fallback value` and `crash value` narrow the rest of
  the block; `while value { }` narrows the body; a `switch` case `T:` narrows its case. One check narrows every link
  of a path, never a sibling path. Narrowing tests presence, never the value: a `0`, `false` or `""` that is there is
  present.
- **Buys**: member reads, calls and stores through the value are accepted, with no further test.
- **Falls back**: reading through an unproven `T?` is refused: `this value may be null (it is a Monster?), so
  'name' cannot be read from it yet: narrow it first with 'if value { }', 'fallback value' or 'crash value'`. A
  condition with a call (`if find(name) { }`) is not a path: keep the result in a `var` and narrow the `var`.

```spite title=proofs/narrowing.spite entry
func Narrowing(): Integer {
    load "../spite_standard_library/windows"
    var names = ["ada", "grace"]
    var first = names[0]
    if first {
        return first.length()
    }
    return 0
}
```

See [Nothing fails silently](https://github.com/Spite-Language/spite_spec/blob/master/specs/failure.md#nothing-fails-silently-the-rule).

### Every side of an `and` narrows

- **Proves**: each side of an `and` holds where the whole condition holds.
- **Rule**: later sides see the earlier ones narrowed, and the right side of an `or` sees the left side's opposite.
  This holds in every condition, `while` and a plain expression included.
- **Buys**: one line proves several reads (`if node and node.ready`).
- **Falls back**: the sides are tested in order, so a later side is never read unproven.

### An `if` that leaves proves the rest

- **Proves**: after an `if` with no `else` whose block always leaves, the opposite of its condition holds.
- **Rule**: a block leaves with a `return`, a bare `crash`, a `switch` whose every case leaves, an `if` and `else`
  that both leave. Each `or` side written `not path` then narrows that path, each side read with
  the sides before it narrowed (`if not holder or not holder.weapon { return }` proves both).
- **Buys**: guards read as answers: after `if not monster { return }` the rest reads `monster`.
- **Falls back**: the rest stays unproven and a read of it is refused. Narrow it again.

### Assigning undoes a proof

- **Proves**: nothing stale is read.
- **Rule**: assigning a narrowed path, or anything it reads through (an index name included), undoes every proof
  beneath it: after `target = Monster()`, `target.weapon` is unproven again. A store through a setter undoes its
  path, and `list[i] = value` undoes what was proven below the items. Assigning a value that cannot be null keeps the
  path's own narrowing. Inside a `while`, assigning a value that may be null to a path narrowed before the loop and
  read in it is an error.
- **Buys**: a narrowed read has no run-time test.
- **Falls back**: after the assignment a read is refused until it is proven again. In a loop the error says:
  `assigning to '...' inside this loop undoes a narrowing made before the loop, and the next pass would read it
  unproven: narrow it inside the loop instead`.

### A call keeps a proof it cannot change

- **Proves**: a narrowed attribute read is still valid after a call.
- **Rule**: a narrowed attribute read is refused when a call between the check and the read may assign that
  attribute, followed through everything the call reaches. A constructor's writes to its own object do not count; a
  wait writes nothing in a program in which the compiler starts no waiting work, and anything in one where it does; a call
  through a function value reaches every function taken as a value with its signature; a `drop()` that may run is
  followed too. Inside a loop, a call anywhere in the body undoes a narrowing made before the loop for the next pass.
  A `[]` or count proof is undone by a call that shrinks the list, and a proven read that finds its list changed
  halts (see [a proven read halts outside its list](#a-proven-read-halts-outside-its-list)).
- **Buys**: no re-check after a harmless call, and a call that may null what was proven is never read past.
- **Falls back**: refused: `'keeper.target' was proven before line 12, where 'forget(...)' may assign 'target'
  again, so this read would use what the call left there, unproven: prove 'keeper.target' again after that line`.
  Prove it again after the call.

### Proving what is proven is an error

- **Proves**: nothing is checked twice.
- **Rule**: a check of a `T?` already narrowed, a `[]` read already proven (by a bound, a count or an earlier
  check), a divisor already proven, a class test the type already answers, the same call-free condition on the next
  line, or a `fallback` or `crash` that folds true is refused, naming where it was proven.
- **Buys**: no dead checks in the source.
- **Falls back**: delete the repeated check.

### A value proven present is a `T`

- **Proves**: a `T?` that can never be null holds a value wherever it is read.
- **Rule**: a local every assignment of which cannot be null, an attribute of a program class whose default and
  every assignment cannot be null, and a program function every `return` of which cannot be null. A value cannot be
  null when its type is not a `T?` or when it is one of these, so the proofs feed each other.
- **Buys**: the value is read, called on and passed as a `T` with nothing written; checking it again is the
  "already narrowed" error.
- **Falls back**: a `T?` parameter, an attribute with no default or assigned a `T?` anywhere, and what the standard
  library answers stay `T?`: narrow them as usual. In a `--hot-reload` or `--repl` build no attribute is proven,
  since a running object may hold a null an older version stored. The answer of a call worked out while compiling
  is not proven yet.

### A condition known while compiling is an error

- **Proves**: every condition written can go either way.
- **Rule**: outside a generic class and a walk, an `if` or `while` the compiler decides while compiling, and an
  `fallback` that folds false, are refused, `while true` among them. A condition on a `Build` field is kept.
- **Falls back**: remove the condition, keeping the code that runs.

### A copy made to narrow is an error

- **Proves**: one name per value.
- **Rule**: a `var` that holds a bare name or path of a `T?`, is never assigned again, whose source is not assigned
  in the function, and that is then narrowed, is refused: narrow the source itself. A snapshot taken before its
  source changes is allowed.
- **Falls back**: narrow the original name.

### A class test narrows its block

- **Proves**: inside `if value.class == Monster { }` the value is a `Monster`.
- **Rule**: it narrows on a name, a member path, or a side of an `and`. A null tests false; in a generic class a
  test that cannot hold folds to false. `!=`, the `else` and a leaving `if` narrow nothing.
- **Buys**: the member's attributes and functions are read inside the block.
- **Falls back**: a member read without the test is refused; use a `switch`.

### Every path ends in a `return`

- **Proves**: no caller gets a value nobody wrote.
- **Rule**: a path ends at a `return`, a bare `crash`, an `if` and `else` whose branches both end, a `switch` whose
  every case ends, or a folded condition whose taken branch ends. A function that answers something
  and has a path that reaches its end is refused.
- **Falls back**: refused: `'...' answers ..., but ... it reaches its end without a 'return' ...: end that path
  with 'return <value>', or with 'crash' if it cannot happen`.

### A fallback answers what its answer type builds with no arguments

- **Proves**: a guard is written once, and its answer is stated by the answer type.
- **Rule**: a `fallback` returns what the function's answer type builds with no arguments: nothing, `null`, an empty
  collection, `0`, `false`, `""`, or the object of a class made with no arguments. It is refused where the answer
  type has no such value (an enum, a class made with arguments, a singleton) and in a constructor. An `if` that only
  returns that value, or a last `if` with no `else` that only checks presence, is refused naming the `fallback`.
- **Falls back**: refused: `'fallback' returns what the answer type builds with no arguments, and a ... has none`.
  Answer a `T?`, or use `crash`.

### A switch covers every case

- **Proves**: a member, value or null added later cannot be forgotten.
- **Rule**: a switch over a union covers every member; over a `T?` of a class or union it covers the class (or every
  member) and `Null`; over an enum or a `Boolean` every value. There is no `_:`, and a switch over a whole number or
  a text is refused: choose among those with `if`s. A repeated case is an error.
- **Falls back**: refused: `this switch over '...' has no case for ...: add the cases`.

See [Switch over values](https://github.com/Spite-Language/spite_spec/blob/master/specs/control_flow.md#switch-over-values).

## Indices and numbers

### A proven count or bound proves a read

- **Proves**: `list[index]` is there, so it answers `T`, not `T?`.
- **Rule**: `index < list.count()` (also `- k`, `count() > index`, a count kept in a `var`) with any call-free
  index, `base + k < list.count()` for small `k`, and the count facts `== k`, `>= k`, `> k`, `!= 0` and
  `not is_empty()` prove the reads they cover, in `fallback`, `crash`, `if`, `while`, the right of an `and`, and after
  an `if` that leaves. A bound written with `<=` (`crash p + k <= list.count()`) proves `list[p]` to
  `list[p + k - 1]`. A proof is undone as in [assigning undoes a proof](#assigning-undoes-a-proof).
- **Buys**: a proven read is a `T`.
- **Falls back**: an unproven read is a `T?`; where a `T` is needed it is refused: `'...' may be missing, since
  every '[ ]' answers a 'T?' ...: ..., or prove the count first, 'crash list.count() > 2'`. A `Dictionary` read is
  proven only by checking that key.

```spite title=proofs/proven_read.spite entry
func Proven(): Integer {
    load "../spite_standard_library/windows"
    var values = [4, 8, 15]
    var total = 0
    var index = 0
    while index < values.count() {
        total = total + values[index]
        index = index + 1
    }
    return total
}
```

### A proven read halts outside its list

- **Proves**: a wrong or stale proof halts and never reads out of bounds.
- **Rule**: a bound proves only the top of an index, so a proven read keeps one test against the list's length.
- **Falls back**: it halts: `spite: '...' is outside its list: a bound proves only the top of an index, and this one
  is below 0 or the list changed, at ...`. The compiler drops the test where a signed test of the same values and a
  never negative index already prove it.

### A proven count or bound proves a read of a width

- **Proves**: `buffer.read_<number>_at(p)` answers the number, not a `T?`.
- **Rule**: for a number of width w (1 for `tiny` and `byte`, 2 for `short`, 4 for `integer` and `float`, 8 for
  `long` and `double`), the read is proven where `buffer[p + w - 1]` is: by a bound (`while p + 3 < bytes.count()`),
  a count, or a `<=` bound (`crash body + 10 <= header.count()` proves the reads at `body`, `body + 4` and
  `body + 8` up to a short). The position is a name, a name plus a literal, or a literal.
- **Buys**: a header or a record is read with one bound, and each read is a number.
- **Falls back**: the read is a `T?` the program narrows. A proven read still answers null below 0 or past a
  changed buffer, and then halts as above.

### An index offset by a base is proven by one guard

- **Proves**: a slice of a list is readable.
- **Rule**: the guard `base + width <= list.count()` (a `fallback` or `crash`, or the `if` that leaves on the
  opposite) with `while counter < width` proves `list[base + counter]` inside the loop, base, width and counter being
  names. Assigning `base` or `width`, or a call that shrinks the list, forgets the guard.
- **Falls back**: the read is a `T?`. The guard proves only the top, so a negative base or counter halts at the read.

### A counted loop reads its items unchecked

- **Proves**: a loop over a list needs no per-item test.
- **Rule**: inside `while index < values.count()` with the counter stepped last, the count is read once and the
  items are read with no test; the tests are made once, before the loop, where the loop's own test has not already
  proved them.
- **Buys**: a tight loop over numbers costs a load per item.
- **Falls back**: each read keeps its test.

### A proven divisor is not checked

- **Proves**: a division cannot divide by zero.
- **Rule**: a divisor that is a non-zero constant, or whose range leaves out 0 (and -1 when the dividend may be the
  smallest value), is divided with no test. `crash parts > 0` before `amount / parts` is enough.
- **Falls back**: the test stays: `spite: '...' divided by zero, at ...`.

### A divisor written as zero is an error

- **Proves**: a division that always halts never compiles.
- **Rule**: a whole-number `/` or `%` whose divisor is a constant expression equal to zero is refused.
- **Falls back**: refused: `'...' divides by zero, which always halts the program ...`.

### Arithmetic that does not fit halts

- **Proves**: no wrapped number is ever read.
- **Rule**: every whole-number `+`, `-`, `*`, unary `-`, the smallest value divided by -1 and a narrowing halts if
  the answer does not fit its type. The step of a counter under its bound carries no check, nor does constant
  arithmetic; constant arithmetic that does not fit is a compile error. Every build checks.
- **Falls back**: the check stays and halts: `spite: '...' does not fit in an Integer (...), at ...`. Where
  wrapping is the point, call `wrapping_sum`, `wrapping_subtract` or `wrapping_multiply`. `--optimization-report`
  lists every check kept.

### A range proves arithmetic fits

- **Proves**: an operation cannot overflow.
- **Rule**: every whole-number local and parameter has a range, from literals, assignments and the conditions in
  force; `count()`, `length()`, `%`, `minimum`, `maximum` and `clamp` bound their answers; loops widen counters and
  count their passes; a total assigned once per pass holds at most as many terms as there are passes. So a `Long`
  total of `Integer` terms over a list carries no check. An attribute, a list item or a call's answer has its
  type's range.
- **Buys**: an operation the ranges prove carries no overflow check.
- **Falls back**: the check stays. See
  [a range proves a check unneeded](https://github.com/Spite-Language/spite_spec/blob/master/specs/values_and_types.md#a-range-proves-a-check-unneeded).

```spite title=proofs/range.spite entry
func Ranged(): Long {
    load "../spite_standard_library/windows"
    var values = [2000000000, 2000000000, 2000000000]
    var sum: Long = 0
    var index = 0
    while index < values.count() {
        sum = sum + values[index]
        index = index + 1
    }
    return sum
}
```

### A wider operand is written first

- **Proves**: nothing is cut to fit.
- **Rule**: in arithmetic, comparisons and bitwise functions, a right side with more bits (or a decimal beside a
  whole number) is refused, unless it is a literal within the left type.
- **Falls back**: refused, naming the turned-around form: `'...' is added in Integer, since arithmetic takes the
  left side's type, and the right side is a Long, which would be cut to fit: write the Long first ...`.

### Maths on constants is worked out while compiling

- **Proves**: a maths function of known operands needs no call.
- **Rule**: a maths member whose receiver and arguments are known is answered by the compiler, a not-a-number
  answer excepted.
- **Falls back**: one call into the C library, or one instruction.

### A dictionary only read by literal keys is never made

- **Proves**: a dictionary written out holds exactly the entries written.
- **Rule**: a local made from a dictionary literal of literal keys and values, every later mention of which is
  `name[literal]`, is never assigned. A read of a key it holds is its value, still a `T?`; a read of a key it does
  not hold is known to be `null`, so `crash points["platinum"]` is a compile error.
- **Buys**: no dictionary and no hashing when every read asks a key it holds.
- **Falls back**: the dictionary is made and read as written (any other mention of it, a key in a variable, a
  write).

## Compile-time questions

### Conditions decided while compiling

- **Proves**: which branch of an `if` is taken.
- **Rule**: an `if` on a `Build` field, a codegen value, a type test, a walk's `attribute.class` or name test, a
  class test the value's type answers, or a compile-time question (and `not`, `and`, `or`, `==`, `!=` over them)
  keeps only its taken branch. Only what survives is compiled.
- **Buys**: generic code compiles only the branch its class takes.
- **Falls back**: a run-time test of a constant, removed later by folding; both branches must compile.

### A `crash` that folds false is a compile error

- **Proves**: generic code that cannot work for a class is refused for it.
- **Rule**: a `fallback` or `crash` that asks only compile-time questions folds. A `crash` that folds false in a
  function the program reaches is refused, naming the instance; a `fallback` that folds false answers its default
  and the rest of its block is not compiled.
- **Falls back**: refused: `'crash ...' always halts in ...: its condition is decided while compiling and is false`.

### Whether a function waits

- **Proves**: `function_waits` and `is_resumable` fold.
- **Rule**: waiting spreads from `sleep`, console, file and socket reads and joins, through calls, function values
  (to every function taken as a value with the same signature) and `type` dispatch, until nothing changes. Where it
  cannot tell, the answer is true.
- **Falls back**: refused when an answer would decide itself: `whether '...' can wait was answered ... before the
  functions that ask 'function_waits' were compiled ...`.

### How many arguments a function takes

- **Proves**: `$T.argument_count("<pattern>")` and a walk's `phase.argument_count()` fold to a whole number.
- **Rule**: every function a pattern matches must take the same number, and a pattern matching none is an error.
- **Falls back**: refused: `'...' has no function named '...', so 'argument_count' has nothing to count ...`, or the
  functions that fit take different numbers of arguments; ask `phase.argument_count()` in a template over them.

### Whether a function writes a parameter

- **Proves**: `function_writes_parameter(...)` and `.is_mutated` fold.
- **Rule**: a write is an attribute or item set, a call that writes, passing the parameter to a written parameter,
  or storing it where a later write could reach it. Reassigning, copying plain values out and writing a singleton
  are not writes. Where the compiler cannot tell, the answer is true.
- **Buys**: generic code skips a copy a callee never writes.

### Whether a class keeps state

- **Proves**: `$T.is_stateful` folds.
- **Rule**: true when some function other than the constructor and `drop()` writes its own object or a singleton,
  or a singleton it binds keeps state. Where it cannot tell (a function it cannot build), true.
- **Buys**: generic code shares an object nothing changes.

### The one class a `type` admits

- **Proves**: which classes can fill a `type`.
- **Rule**: with the whole program known, an attribute read through a `type` is read at the offset every class that
  fills it shares, with no class test. Not in interactive builds.
- **Falls back**: a test of the class, for a member some filler answers with a function or an offset another union
  takes.

### Tree shaking: what `main` can reach

- **Proves**: what the program uses.
- **Rule**: only what the entry reaches is kept, transitively: a function called, a member a dispatch may call, a
  function taken as a value.
- **Buys**: a program carries none of what it does not use.
- **Falls back**: it keeps too much, never too little; an inspectable build keeps everything.

## Memory

### Objects that never leave their function live in the frame

- **Proves**: an object cannot be seen outside its function.
- **Rule**: a fresh object that is never stored in memory, passed to a call that could keep it, or answered, lives
  in registers or the frame. Its class may not have a `drop()`; a class that lets go of attributes when freed is
  placed in registers only.
- **Buys**: no allocation and no count.
- **Falls back**: a heap object. `--debug-memory` shows the difference.

### Local and variadic lists in the frame

- **Proves**: a list never leaves its function.
- **Rule**: a local list made by a literal and used only through `count()` and `[]`, never assigned again, lives in
  the frame; a variadic list its callee only reads lives in the caller's frame.
- **Falls back**: a heap list.

### A function value its callee only calls lives in the frame

- **Proves**: a function value does not outlive the call.
- **Rule**: a function value passed to a function whose every mention of it is a call lives in the caller's frame,
  uncounted.
- **Falls back**: a heap function value.

### A singleton made at the start is never tested again

- **Proves**: a singleton exists wherever it is read.
- **Rule**: a singleton the entry class's attribute defaults make on every path is made before the entry function
  starts, so a function that cannot run before those defaults end reads its slot without testing that it is made.
- **Falls back**: the test stays in every function that may run before the defaults end, and for every singleton
  made only on some paths.

### A list its setup fills holds what the setup made

- **Proves**: the exact items of a list.
- **Rule**: a list attribute written by nothing but its default and filled only from the class's setup with objects
  the setup makes holds exactly those objects, in that order. A function `register(step: SomeType)` that appends to
  such a list may be called from anywhere: called only from the setup, the list is known.
- **Buys**: a loop that calls through the list's `type` is a direct call per item.
- **Falls back**: the loop as written, when `register` is called anywhere else; nothing is refused (D1781).

## Items read from a list

### An item a name holds from its list is not counted

- **Proves**: the list keeps the item alive.
- **Rule**: `var item = list[index]` with a proven read holds no count when nothing after it in its block can write
  that slot (no assignment of the list's path, no write into a list of that class, no call that may).
- **Falls back**: a counted read.

### An item written back to its own slot is the slot

- **Proves**: a write-back changes nothing.
- **Rule**: between `var item = list[index]` and `list[index] = item`, with nothing between that may change the
  name, the index, the path or the list, the write-back is not written and the read is uncounted.
- **Falls back**: a counted read and the write-back written.

### Arguments and attributes passed uncounted

- **Proves**: the caller's hold lasts the whole call.
- **Rule**: a function counts only the parameters it assigns again. An argument read from a place the call cannot
  assign (an attribute path whose attributes nothing the callee reaches assigns, a local, `this`, a singleton) is
  passed as it is. A list item used at once as a receiver or argument, or read only to be tested, and a template's
  element in a pass that cannot let go of an object, are not counted either. An attribute of a singleton that no
  function but its constructor assigns is read in place.
- **Falls back**: an argument read from a place the call may assign is counted before the call and let go after.

### Assigning a name read from a list changes only the name

- **Proves**: the list still holds the old item.
- **Rule**: a local read from a list item and then assigned, with that assignment its last mention, is refused,
  because the list keeps the old item.
- **Falls back**: write into the list (`list[index] = value`) if that is what you meant.

See [Assigning a name read from a list](https://github.com/Spite-Language/spite_spec/blob/master/specs/memory.md#assigning-a-name-read-from-a-list-changes-only-the-name).

## Threads and locks

The rules are on [Concurrency](concurrency.md) and in
[the memory model](https://github.com/Spite-Language/spite_spec/blob/master/specs/memory.md#the-memory-model).

### No other thread counts a class

- **Proves**: the objects of a class are counted by one thread only, so their counts are plain arithmetic.
- **Rule**: code that can run on another thread is every function the compiler starts on another thread (a call
  in a row, a band of a banded loop), every function those call, the constructors and attribute defaults of what they make, and, where they call a function
  value, every function the program takes as a value. A class is counted by another thread when that code names a
  value of it or of anything that holds it (an attribute, a list's or a dictionary's items, a union's member, a
  shape's filler). A union or shape whose members are not known counts every class. A program that makes a
  `ForeignCallback` counts every class atomically.
- **Buys**: plain counts for every other class whose objects' class is known where they are counted.
- **Falls back**: an atomic count wherever the class is not known while compiling, or another thread may count it.

### Which singletons a `ForeignCallback` reaches

- **Proves**: which singletons need a lock.
- **Rule**: walk the calls from every function handed to C as a `ForeignCallback`. A reached singleton is made safe for threads, an unreached one takes no lock. A reached function
  that reads or writes an attribute of a program singleton directly, from another class, reaches that singleton too.
- **Falls back**: locked.

### Read-only, atomic and locked singletons

- **Proves**: the cheapest safe form for a singleton.
- **Rule**: a singleton nothing changes after it is made takes no lock. One whose changing attributes are numbers or
  `Boolean`s touched once per function, outside loops, uses atomic operations. Any other takes its own lock for each
  function that reads `this`. A function that reads or writes no changing attribute and calls nothing that does runs
  unlocked.
- **Falls back**: the whole singleton takes its lock.

### A counted loop takes a singleton's lock once

- **Proves**: a loop of calls to one singleton cannot deadlock or interleave badly.
- **Rule**: a counted `while` that calls only one singleton's functions and plain helpers, and reaches no wait, no
  start of work and no other lock, takes the lock once for the whole loop. When every function it calls writes none
  of the singleton's state, the lock is shared with other such readers.
- **Falls back**: a lock per call.

### No attribute is read past the lock

- **Proves**: a locked singleton's state is never touched without its lock.
- **Rule**: reading or writing an attribute of a locked singleton from another class takes the singleton's lock
  around the read, or around the whole statement that assigns it, so `tally.total = tally.total + 1` adds once. An
  attribute nothing assigns after the singleton is made is read with no lock.

### A loop that can never end is an error

- **Proves**: a loop can end.
- **Rule**: every `while true` is refused, since a loop states the condition it runs under.
- **Falls back**: refused: `'while true' is decided while compiling and is always true here: ...`.

### A write to an object only this function holds must be read

- **Proves**: no lost write.
- **Rule**: a write to a local only this function holds (a constructor's result, a copy, an always-new result),
  never read after it, is refused.

### Passes that write only their own item

- **Proves**: the passes of `list.each_f()` are independent.
- **Rule**: every pass writes only its own element's number and enum attributes and reads nothing another pass
  writes.
- **Buys**: the passes run in bands on several threads.
- **Falls back**: the loop in order.

### Which calls suspend started work

- **Proves**: which functions need a resumable form.
- **Rule**: a function of waiting work the compiler starts (a frame's call, a read in a row) that can wait, and
  every waiting function it calls, is compiled as a resumable state machine.
- **Falls back**: a wait the compiler does not see as such runs the event loop in place.

### A wait in a frame is started

- **Proves**: waits in a loop overlap.
- **Rule**: in a loop that sleeps, a call with no arguments and no answer that can wait is started as a
  state machine of its own, unless either side writes back a value read before a wait.
- **Falls back**: it waits where it is written. See
  [a wait in a frame](https://github.com/Spite-Language/spite_spec/blob/master/specs/concurrency.md#a-wait-in-a-frame-does-not-hold-the-frame).

## Other refusals built on an analysis

Each reads the program while compiling and accepts or refuses code; each is stated in full in its specification, and
none has a fallback beyond the error it gives.

- **Unused is an error**: a name, parameter or attribute only written and never read
  ([Style](https://github.com/Spite-Language/spite_spec/blob/master/specs/style.md#unused-is-an-error)).
- **A constructed object must be kept and used**: a construction written as a statement of its own is refused
  ([Classes and files](https://github.com/Spite-Language/spite_spec/blob/master/specs/classes_and_files.md#a-constructed-object-must-be-kept-and-used)).
- **Singletons that make each other in a circle** are an error; a circle closed only at run time halts naming it
  ([Singleton rules](https://github.com/Spite-Language/spite_spec/blob/master/specs/classes_and_files.md#singleton-rules)).
- **Build settings are constants**: a flag's value has its field's type, an unknown flag is refused, and an `if`
  around a `load` is decided from `Build` fields and literals
  ([Programs](https://github.com/Spite-Language/spite_spec/blob/master/specs/programs.md#build-settings-build)).
- **A value fits a `type`**, and a `$T` its constraint: a class passed as a `type` must have each attribute with the
  type's type, and each function with the same parameters and answer, the type's own name standing for the class
  that fits. The refusal says which: `'...' does not fit type '...': it has no function '...'`, or that an
  attribute or argument has another type, or that a function takes a different number of arguments
  ([Inline types](https://github.com/Spite-Language/spite_spec/blob/master/specs/values_and_types.md#inline-types-and-duck-typing-in-full)).
- **A member template fits**: the member a template names exists and has the right kind (a `Boolean` for
  `filter_`, `any_` and `all_`; a number for `sum_`; a function for `each_`; a number or text for `sort_by_`), and a
  passed function takes the element
  ([Collections](https://github.com/Spite-Language/spite_spec/blob/master/specs/collections.md#standard-library-metaprogramming)).
- **A value can be written**: the `T` of a JSON or binary writer or reader holds no union, `type` or function value
  anywhere ([JSON](https://github.com/Spite-Language/spite_spec/blob/master/specs/json.md)).
- **What crosses into C**: numbers, `Boolean`s, enums, text, lists of numbers, number-only `type`s and function
  values; a callback's parameters and context are checked
  ([Foreign libraries](https://github.com/Spite-Language/spite_spec/blob/master/specs/foreign_libraries.md#what-crosses)).
- **The short form is proven the same, and the long form refused**: a `while` a member template already says, a
  chain of `if`s that is a `switch`, the same call opening every branch, a switch that is a class test and a text
  that is one hole are refused, naming the short form
  ([Control flow](https://github.com/Spite-Language/spite_spec/blob/master/specs/control_flow.md)).

Not proven anywhere, so write the check or accept the run-time report: how deep recursion goes (a stack overflow is
a [native fault report](failure.md#what-a-native-fault-reports)); unreachable code after a `return`; a reference
cycle, which leaks ([memory.md: Cycles leak](memory.md#cycles-leak)); a wider value assigned, passed or returned
into a narrower name, which wraps; and two threads writing one number attribute of an instance they share.

---

Next: [the contents](README.md), every page of the documentation.
