# Memory

Spite uses **reference counting**, JavaScript-like. A scalar (a number, a `Boolean`, an enum value) is a plain
value, copied wherever it goes. Everything else (a class instance, `List<T>`, `Dictionary<Key, Value>`, `String`, a
union, an object literal) is a **reference**: assigning it, passing it, storing it in a field, a list or a
dictionary, and returning it all share the exact same object. There is no reference syntax to write: a reference
is the default. When the last reference to an object goes (a scope ends, a field is overwritten, an element is
removed), the object's `drop()` runs, if it has one, its own references are released, and it is freed. There is
no garbage collector and no pause.

What that costs while the program runs is a count in every object and one addition or subtraction each time a
reference is kept or let go (plain arithmetic, atomic only in a program that starts a thread) and nothing
else: no collector, no runtime to ship. A singleton is not counted at all, and a number, a `Boolean` or an enum
value is never an object.

## No lifetimes, and no value classes

Some languages keep memory safe by making you describe who owns what and for how long, and the compiler checks the
description: a great deal of annotation that has nothing to do with what the program is for. Spite gets the same
safety from a smaller set of tools: reference counting, a value that may be missing is a `Monster?` that must be
narrowed with `fallback` or `crash` before it is used.
Where the safe form needs a rule, the compiler has it. You do not write lifetimes.

Some languages also ask you to decide, for each type, whether it is a value or a reference. In Spite that is the
compiler's question, never yours. There are only classes, and every object behaves as a shared object. The compiler
chooses registers, the stack, inline storage or the heap for each use, whichever keeps that behaviour cheapest, and
work it could have shown to be unnecessary and did anyway is a bug in the compiler, not something you tune by
splitting a type in two.

## Do: know that sharing is visible

Two names can hold the same object. A mutation through either one shows up through the other, because there is
only ever one object:

```gdscript title=sharing_basics/box.spite
var label = "unnamed"

func Box(starting_label: String) {
    label = starting_label
}
```
```gdscript title=sharing_basics/sharing_basics.spite entry
var console = Console()

func SharingBasics() {
    load "../spite_standard_library/windows"
    var original = Box("a")
    var alias = original
    alias.label = "b"
    console.print("original label", original.label)
}
```
```output
original label b
```

`==` on two class instances calls the class's `equals` when it has one
([functions_and_operators.md](functions_and_operators.md#every-operator-is-a-function)); otherwise it asks
whether they are the same object. A `String` always compares by content.

### A name read from a list names the item, until you assign it

`var box = boxes[0]` names the box the list holds, so `box.label = "b"` changes the box in the list. Assigning
`box` is different: `box = Box("c")` makes the name hold another box and leaves the list holding the first one.
When nothing reads `box` after that, the assignment was meant for the list, and the compiler says so:

```gdscript
var boxes = List<Box>()

func relabel(replacement: Box) {
    crash boxes[0]
    var box = boxes[0]
    box.label = "b"
    box = replacement
}
```

```
'box' was read from 'boxes[0]', so assigning it here changes only 'box', and nothing reads 'box' after:
'boxes' still holds the old item. Write 'boxes[0] = replacement' to replace the item, or give the new value a
'var' of its own
```

The same holds for a name answered by a function that returns an item of a list, such as a lookup that answers
`values[row]`: the error names the class holding the list and its function that writes it
([the rules](https://github.com/Spite-Language/spite_spec/blob/master/specs/memory.md#assigning-a-name-read-from-a-list-changes-only-the-name)).

## Do: call `copy()`/`deep_copy()` for an independent object

`copy()` makes a fresh object with the same attributes (still shared references for any attribute that is
itself a reference). `deep_copy()` recurses, giving every reference-kind attribute its own independent copy
too:

```gdscript title=copy_basics/box.spite
var label = "unnamed"

func Box(starting_label: String) {
    label = starting_label
}
```
```gdscript title=copy_basics/copy_basics.spite entry
var console = Console()

func CopyBasics() {
    load "../spite_standard_library/windows"
    var original = Box("a")
    var independent = original.copy()
    independent.label = "b"
    console.print("original label", original.label)
    console.print("independent label", independent.label)
}
```
```output
original label a
independent label b
```

`deep_copy()` copies the shape of what it reaches: an object reached twice in one copy, through a cycle, is copied
once, so a child's `parent` in the copy is the copied parent, and a back reference in the copy holds the copy of its
object when the same `deep_copy()` copied it ([the rules](https://github.com/Spite-Language/spite_spec/blob/master/specs/memory.md#the-memory-model)).

## Components in lists, handed to a system

An engine keeps each component in its own `List` and hands a system one entity's components at a time. A list
holds its items like any other reference, so `positions[0]` is the position itself: a system that writes it writes
the list's item, and the item may be kept, passed on or read after the list grows. How a list stores its items is
the compiler's choice, never yours: a list whose items are never held anywhere else may keep them inline, one block
of their attributes, and nothing the program can see changes ([collections.md](collections.md#one-list)).

### A row of items, for one call

The row is an object literal of items, and the system takes it as a `type`:

```gdscript title=row_doc/position.spite
var left = 0.0
```
```gdscript title=row_doc/velocity.spite
var across = 0.0

func Velocity(new_across: Float) {
    across = new_across
}
```
```gdscript title=row_doc/mover.spite
type Moving {
    position: Position
    velocity: Velocity
}

func update_each(moving: Moving) {
    moving.position.left = moving.position.left + moving.velocity.across
}
```
```gdscript title=row_doc/row_doc.spite entry
var console = Console()
var positions = List<Position>()
var velocities = List<Velocity>()
var mover = Mover()

func RowDoc() {
    load "../spite_standard_library/windows"
    var position = Position()
    positions.append(position)
    var velocity = Velocity(2.5)
    velocities.append(velocity)
    crash positions[0] and velocities[0]
    var tick = 0
    while tick < 4 {
        var row: Moving = {position: positions[0], velocity: velocities[0]}
        mover.update_each(row)
        tick = tick + 1
    }
    console.print(positions[0].left)
}
```
```output
10
```

A generic runner does not know the attributes of the `type` it is given, so it cannot write the literal. It fills
the row with a walk over its attributes instead, and a local of the `type` declared `= null` and filled by the walk on the
next line is a row exactly like the literal:

```gdscript title=walked_row_doc/position.spite
var left = 0.0
```
```gdscript title=walked_row_doc/velocity.spite
var across = 0.0

func Velocity(new_across: Float) {
    across = new_across
}
```
```gdscript title=walked_row_doc/mover.spite
type Moving {
    position: Position
    velocity: Velocity
}

func update_each(moving: Moving) {
    moving.position.left = moving.position.left + moving.velocity.across
}
```
```gdscript title=walked_row_doc/columns.spite
singleton

var position = List<Position>()
var velocity = List<Velocity>()

func size(): Integer {
    return position.count()
}
```
```gdscript title=walked_row_doc/runner.spite
generic $row_type

var columns = Columns()

func run(mover: Mover) {
    var index = 0
    while index < columns.size() {
        var row: $row_type = null
        fill_attributes(row, index)
        mover.update_each(row)
        index = index + 1
    }
}

func fill_attribute(attribute: Spite.AttributeDeclaration, row: $row_type, index: Integer) {
    crash columns.attributes[attribute][index]
    row.attributes[attribute] = columns.attributes[attribute][index]
}
```
```gdscript title=walked_row_doc/walked_row_doc.spite entry
var console = Console()
var columns = Columns()
var runner = Runner<Moving>()
var mover = Mover()

func WalkedRowDoc() {
    load "../spite_standard_library/windows"
    var position = Position()
    columns.position.append(position)
    var velocity = Velocity(2.5)
    columns.velocity.append(velocity)
    runner.run(mover)
    runner.run(mover)
    crash columns.position[0]
    console.print(columns.position[0].left)
}
```
```output
5
```

Every `[]` answers a `T?` and a row's attributes are items, so the template states each
read with a `crash` line before the fill: nothing the walk knows proves that `index` is inside each column. The
compiler writes the walk out where it is called: for `Runner<Moving>` the two lines become `crash
columns.position[index]`, `crash columns.velocity[index]` and `var row: Moving = {position:
columns.position[index], velocity: columns.velocity[index]}`, where each `crash` line's read is made once and
the literal uses it, so the row costs what the literal costs and `fill_attribute` is never called.

An engine that adds and removes components all the time keeps each one in a **sparse set**: a generic singleton
`Column<Position>` whose list is packed, so each entity sits at a different place in each column. The runner
works out those places first, one per attribute, and the walk's line reads each attribute from its own column at
its own place: `Column<attribute.class>().values[stored_row]`, after `var stored_row = found[attribute.index]`,
where `attribute.index` is the attribute's place in the `type`, with one `crash` line for the
place and one for the item (an index never holds another `[]` read).
A generic singleton made with no arguments is the same object every time, so `Column<Position>()` starts a path
that `crash` narrows like a name. The places are numbers in a `List`, so they may come from any `List<Integer>`:
the runner's own, or another object's, `fill_attributes(row, matcher.rows)`. The row may also hold a value made
from the id, such as an `Entity`, chosen per attribute while compiling:

```gdscript title=sparse_row_doc/position.spite
var left = 0.0
```
```gdscript title=sparse_row_doc/velocity.spite
var across = 0.0

func Velocity(new_across: Float) {
    across = new_across
}
```
```gdscript title=sparse_row_doc/entity.spite
var id = 0

func Entity(new_id: Integer) {
    id = new_id
}
```
```gdscript title=sparse_row_doc/trail.spite
var lefts = List<Float>()
var last = 0.0
var owner = -1
```
```gdscript title=sparse_row_doc/mover.spite
type Moving {
    entity: Entity
    position: Position
    velocity: Velocity
    trail: Trail
}

func update_each(moving: Moving) {
    moving.position.left = moving.position.left + moving.velocity.across
    moving.trail.lefts.append(moving.position.left)
    moving.trail.last = moving.position.left
    moving.trail.owner = moving.entity.id
}
```
```gdscript title=sparse_row_doc/column.spite
singleton

generic $component_type

var dense_of = List<Integer>()
var values = List<$component_type>()

func insert(entity: Integer, value: $component_type) {
    while dense_of.count() <= entity {
        dense_of.append(-1)
    }
    dense_of[entity] = values.count()
    values.append(value)
}

func dense_index(entity: Integer): Integer {
    crash dense_of[entity]
    return dense_of[entity]
}
```
```gdscript title=sparse_row_doc/runner.spite
generic $row_type

var found = List<Integer>()

func run(mover: Mover, entity_count: Integer) {
    var entity = 0
    while entity < entity_count {
        found.clear()
        var row: $row_type = null
        find_attributes(row, entity)
        fill_attributes(row, found, entity)
        mover.update_each(row)
        entity = entity + 1
    }
}

func find_attribute(attribute: Spite.AttributeDeclaration, row: $row_type, entity: Integer) {
    if attribute.class == Entity {
        row.attributes[attribute] = Entity(entity)
        found.append(entity)
    } else {
        var column = Column<attribute.class>()
        var dense = column.dense_index(entity)
        found.append(dense)
    }
}

func fill_attribute(attribute: Spite.AttributeDeclaration, row: $row_type, rows: List<Integer>, entity: Integer) {
    if attribute.class == Entity {
        row.attributes[attribute] = Entity(entity)
    } else {
        crash rows[attribute.index]
        var stored_row = rows[attribute.index]
        crash Column<attribute.class>().values[stored_row]
        row.attributes[attribute] = Column<attribute.class>().values[stored_row]
    }
}
```
```gdscript title=sparse_row_doc/sparse_row_doc.spite entry
var console = Console()
var runner = Runner<Moving>()
var mover = Mover()
var positions = Column<Position>()
var velocities = Column<Velocity>()
var trails = Column<Trail>()

func SparseRowDoc() {
    load "../spite_standard_library/windows"
    var first = Position()
    positions.insert(0, first)
    var second = Position()
    positions.insert(1, second)
    var slow = Velocity(1.0)
    velocities.insert(1, slow)
    var fast = Velocity(4.0)
    velocities.insert(0, fast)
    var first_trail = Trail()
    trails.insert(0, first_trail)
    var second_trail = Trail()
    trails.insert(1, second_trail)
    runner.run(mover, 2)
    runner.run(mover, 2)
    var first_steps = first_trail.lefts.count()
    var second_steps = second_trail.lefts.count()
    console.print(first_trail.owner, first_trail.last, first_steps)
    console.print(second_trail.owner, second_trail.last, second_steps)
}
```
```output
0 8 2
1 2 2
```

`find_attribute` asks about each attribute's class, and fills the row's `Entity` while it is there: that read of
`row.attributes[attribute]` is how its body says the template walks the attributes of `$row_type`. A template whose
body reads its parameter on no value is an error, even when one of its parameters is of the class it would walk.

For `Runner<Moving>` the walk is written out as `{entity: Entity(entity), position:
Column<Position>().values[found[1]], velocity: Column<Velocity>().values[found[2]], trail:
Column<Trail>().values[found[3]]}`. Each item is counted once for the row and let go at the end of the row's block,
so a call that removes an entity from its column cannot free a component under the row
([the rules](https://github.com/Spite-Language/spite_spec/blob/master/specs/memory.md#rows-walked-from-lists)).

### Arguments filled for one call

A system may take its components as arguments instead of a row, `update_each(position: Position, velocity:
Velocity)`, and a generic runner fills them with a template over the function's arguments whose plural is the
call's whole argument list ([metaprogramming.md](metaprogramming.md#walking-a-programs-structure)).
The template's line is the walked row's line, read per argument: `argument.index` is the argument's place, so
`Column<argument.class>().values[stored_row]`, after `var stored_row = rows[argument.index]`, is that component's
item. The plural stands for exactly one call the compiler writes:

```gdscript title=filled_arguments_doc/position.spite
var left = 0.0
```
```gdscript title=filled_arguments_doc/velocity.spite
var across = 0.0

func Velocity(new_across: Float) {
    across = new_across
}
```
```gdscript title=filled_arguments_doc/mover.spite
func update_each(position: Position, velocity: Velocity) {
    position.left = position.left + velocity.across
}
```
```gdscript title=filled_arguments_doc/column.spite
singleton

generic $component_type

var values = List<$component_type>()
```
```gdscript title=filled_arguments_doc/runner.spite
generic $system_type

enum Phase {
    'update'
}

var system = $system_type()
var found = List<Integer>()

func run(index: Integer) {
    found.clear()
    found.append(index)
    found.append(index)
    run_phases_each()
}

func run_phase_each(phase: Spite.FunctionDeclaration) {
    system.phase_each(made_arguments(found))
}

func made_argument(argument: Spite.Argument, rows: List<Integer>): argument.class {
    crash rows[argument.index]
    var stored_row = rows[argument.index]
    crash Column<argument.class>().values[stored_row]
    return Column<argument.class>().values[stored_row]
}
```
```gdscript title=filled_arguments_doc/filled_arguments_doc.spite entry
var console = Console()
var positions = Column<Position>()
var velocities = Column<Velocity>()
var runner = Runner<Mover>()

func FilledArgumentsDoc() {
    load "../spite_standard_library/windows"
    var position = Position()
    positions.values.append(position)
    var velocity = Velocity(2.5)
    velocities.values.append(velocity)
    runner.run(0)
    runner.run(0)
    crash positions.values[0]
    console.print(positions.values[0].left)
}
```
```output
5
```

For `Runner<Mover>` the call is written out as `system.update_each(Column<Position>().values[found[0]],
Column<Velocity>().values[found[1]])`, after the `crash` lines that narrow those reads: `update_each` writes the
columns' own items, and `made_argument` is not called for it. Called anywhere else, `made_argument` is an
ordinary function. The template may also fill a whole walked row for an argument that is a `type`
([the rules](https://github.com/Spite-Language/spite_spec/blob/master/specs/memory.md#rows-walked-from-lists)).

## `drop()` runs once, right before the object is freed

A class may define a zero-argument `func drop() { ... }` for cleanup (closing a handle, clearing a
back-reference). The compiler calls it automatically the moment the last reference goes away, never by name:

```gdscript title=drop_basics/resource.spite
var console = Console()
var name = "unnamed"

func Resource(new_name: String) {
    name = new_name
}

func drop() {
    console.print("dropped", name)
}
```
```gdscript title=drop_basics/drop_basics.spite entry
var console = Console()

func DropBasics() {
    load "../spite_standard_library/windows"
    var resource = Resource("first")
    console.print("using", resource.name)
}
```
```output
using first
dropped first
```

## Self-referential classes and unions just work

A tree, a linked list, or any other recursive structure needs nothing beyond an ordinary field, since a field of
a class type already holds a reference:

```gdscript title=tree_basics/leaf.spite
var held_value = 0

func Leaf(starting_value: Integer) {
    held_value = starting_value
}

func total(): Integer {
    return held_value
}
```
```gdscript title=tree_basics/branch.spite
union TreeNode {
    Leaf
    Branch
}

var left: TreeNode? = null
var right: TreeNode? = null

func Branch(left_node: TreeNode, right_node: TreeNode) {
    left = left_node
    right = right_node
}

func total(): Integer {
    var result = 0
    if left {
        result = result + left.total()
    }
    if right {
        result = result + right.total()
    }
    return result
}
```
```gdscript title=tree_basics/tree_basics.spite entry
union TreeNode {
    Leaf
    Branch
}

var console = Console()

func TreeBasics() {
    load "../spite_standard_library/windows"
    var first_leaf = Leaf(1)
    var second_leaf = Leaf(2)
    var small_branch = Branch(first_leaf, second_leaf)
    var third_leaf = Leaf(3)
    var tree: TreeNode = Branch(small_branch, third_leaf)
    var total = tree.total()
    console.print("total", total)
}
```
```output
total 6
```

## Cycles are freed

Two objects that hold each other, directly or through several hops, would keep each other's count above zero. You never
write a special reference for that: a parent that owns its children and a child that knows its parent are plain
references, and the compiler decides which reference of the cycle holds no count, from how the program links its
objects. A child's `parent` is the usual case: the parent makes the child with `Child(child_name, this)`, the
constructor stores the parent in `parent`, and the parent stores the child in `children`, so the compiler reads
`parent` as the back reference and the cycle is freed with its owner.

```gdscript title=back_reference_family/parent.spite
var console = Console()
var name = ""
var children = List<Child>()

func Parent(new_name: String) {
    name = new_name
}

func adopt(child_name: String) {
    var child = Child(child_name, this)
    children.append(child)
}

func drop() {
    console.print("{name} is freed")
}
```
```gdscript title=back_reference_family/child.spite
var name = ""
var parent: Parent? = null

func Child(new_name: String, new_parent: Parent) {
    name = new_name
    parent = new_parent
}
```
```gdscript title=back_reference_family/back_reference_family.spite entry
var console = Console()

func BackReferenceFamily() {
    load "../spite_standard_library/windows"
    visit()
    console.print("the family is gone")
}

func visit() {
    var parent = Parent("ada")
    parent.adopt("grace")
    var child = parent.children.first()
    crash child
    crash child.parent
    console.print(child.name, "belongs to", child.parent.name)
}
```
```output
grace belongs to ada
ada is freed
the family is gone
```

A back reference reads `null` once its object is freed, so no read is ever dangling, and the narrowing above is the
usual one. Everything above is freed when `visit` returns, which is what `--debug-memory`, below, checks: a cycle the
program links in a way the compiler cannot read keeps its counts and shows up there as leaked objects by class. Then
clear the `T?` field that closes the loop when you are done with it
([the rules](https://github.com/Spite-Language/spite_spec/blob/master/specs/memory.md#the-memory-model)).

## `--debug-memory`

`spite program --debug-memory` builds with an allocation table and prints `allocations: N frees: N` right before
the program exits. A mismatch means something leaked; when the two do not balance, it also prints a
**leaked-object summary by class name**, naming which classes' instances are still alive, which is what makes a
leaked cycle visible instead of an unexplained count. Every program on these pages is run this way, and must
balance. It also fills every byte it hands out that nobody has written yet with the same pattern, so a program that
reads memory it never wrote fails the same way on every run and in every environment, instead of reading whatever
the heap held. The table exists only in a `--debug-memory` build ([what it records](https://github.com/Spite-Language/spite_spec/blob/master/specs/memory.md#--debug-memory)).

## Memory belongs to the compiler

A program never names an address, an allocator, an allocation count or where an object lives. Memory and threads are
constructs of the language that each generator implements its own way, so nothing a program could watch ever stands in
the way of how the compiler places a value: in a register, in the frame when it never leaves its function, together with
the objects around it, or on the heap. You write the plain version, and the compiler decides ([Placement](https://github.com/Spite-Language/spite_spec/blob/master/specs/memory.md#placement-the-compiler-decides-where-memory-lives)).
No class stands for memory, and there is no list of the live instances of a class. Only
`--debug-memory`, above, reports what leaked, and it does so from outside the program.

A class that needs room for bytes holds a `Buffer`, a class of the library that keeps its bytes in one block and reads
and writes the numbers in them, and the compiler lays it out from what it proves
([standard_library.md](standard_library.md#bytes-are-a-buffer)). Bytes from outside are a `Buffer` too: a file's or a
socket's bytes, and memory a C library hands out is a `ForeignBytes`, which checks every read and write against the
size it was given. A container of your own is written over a `List` or a `Buffer`, with nothing the compiler does for
`List<T>` that it would not do for yours: the standard library's `List` is a complete one to read.

---

Next: [Metaprogramming](metaprogramming.md), writing code that writes code for every attribute of a class.
