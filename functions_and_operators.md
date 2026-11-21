# Functions and operators

```gdscript
func function_name(first: Reference, second: Value): Tiny {
    return 1
}
```

The return type follows a colon, `return` is always written out, and a function with no return type returns
nothing. A function with a return type whose body can reach its end without a `return` is a compile error that names
the path that falls off. Forgetting the colon is a parse error that shows the form:

```gdscript title=missing_colon_error/missing_colon_error.spite entry error
func add_one(value: Integer) Integer {
    return value + 1
}
```
```diagnostic
a function's answer is written after a ':', as in 'func total(): Integer', and found 'Integer'
```

## What a parameter shares

A number, a `Boolean` or an enum value is copied into the parameter: assigning to it inside the function changes
the function's copy and nothing else. Everything else (a class instance, a `List`, a `Dictionary`, a union, an
object literal, a `String`) is the very object the caller holds, so a function that changes it through the
parameter changes it for the caller too. There is nothing to write at the call or in the type. There are no value
classes: when you want a copy, ask for it with `copy()` or `deep_copy()` ([memory.md](memory.md)).

Writing `&Monster` for "by reference" is a parse error that says the type alone is enough, since every class,
list and text is passed by reference already. A lone `&` anywhere else is an error too: there are no bit
operators, bits are functions on the whole numbers (`flags.bits_and(mask)`, see
[values_and_types.md](values_and_types.md#bitwise-functions)).

## Functions are values

Naming a function without calling it gives a value: the function bound to the instance it was named on, so it
runs exactly as that instance would have run it. There are no free functions and no closures. The value holds
its owner and nothing else. Its type is written like any generic, with the return last and everything before it
an argument: `Spite.Function<String, String>` takes a `String` and returns a `String`,
`Spite.Function<String, Integer, Boolean>` takes a `String` and an `Integer` and returns a `Boolean`,
`Spite.Function<Boolean>` takes nothing and returns a `Boolean`, and `Spite.Function<Nothing>` takes nothing and
returns nothing (`Nothing` is how "returns nothing" is written, so `<String>` can only mean "returns a `String`").
Calling it is `change(text)` or `change.call_function(text)`, which are the same call, with the arguments checked
like any call; and the value is also its own reflection, with `.name`, `.arguments` (all but the last type),
`.returns` (the last), `.declaration` and `.owner`, the instance it runs on. A typed value goes where a plain `Spite.Function`
is expected, never the other way round. `.returns` is exactly the declared answer, a `Spite.Class`, a `Spite.Union` or
a `Spite.Type`, and the `Nothing` union for a function declared without a return type. A function that is a text's or a number's (`greeting.contains`) is bound to a copy of the value.
A function value whose answer may be null answers it as null wherever it is held. Two function values are `==`
when they name the same function bound to equal owners, however each was made (`shouter.shout == shouter.shout`
is true, `shouter.shout == other.shout` only when `shouter == other`;
[Comparing function values](#comparing-function-values)), which is how `Observer.unsubscribe` finds the one to drop
([collections.md](collections.md#observer-tells-who-subscribed)).

```gdscript title=function_values_doc/shouter.spite
func shout(text: String): String {
    return "{text.upper_case()}!"
}
```
```gdscript title=function_values_doc/function_values_doc.spite entry
var console = Console()

func FunctionValuesDoc() {
    var shouter = Shouter()
    var shouted = apply(shouter.shout, "hi")
    var quieted = apply(quiet, "HI")
    console.print(shouted, quieted)
    var remembered = shouter.shout
    var repeated = remembered.call_function("again")
    console.print(remembered.name, repeated)
}

func apply(change: Spite.Function<String, String>, text: String): String {
    return change(text)
}

func quiet(text: String): String {
    return text.lower_case()
}
```
```output
HI! hi
shout AGAIN!
```

A function value read from a list is called where it is read: `steps[index](value)` calls the function in slot
`index`. Like every `[ ]` read it answers a value that may be missing, so an index the compiler has not proven is
the same error as for any `[ ]` read (`narrow it first with 'crash steps[index]'`...); here the loop's own condition
proves it:

```gdscript title=function_list/function_list.spite entry
var console = Console()
var steps = List<Spite.Function<Integer, Integer>>()

func FunctionList() {
    steps.append(double)
    steps.append(add_one)
    var value = 5
    var index = 0
    while index < steps.count() {
        value = steps[index](value)
        index = index + 1
    }
    console.print(value)
}

func double(number: Integer): Integer {
    return number * 2
}

func add_one(number: Integer): Integer {
    return number + 1
}
```
```output
11
```

An ordinary call, `shouter.shout("hi")`, is a direct call and builds no function value, and neither does a
function passed by name to `each`, `filter` and the other list functions; only a function held as a value costs
anything, and little: one small object and one indirect call. A program that holds none carries none. You
cannot name a function of the class itself: `Monster.is_alive` is an error, since `Monster` as a value is its
reflection; pass `monster.is_alive`, or use a member template such as `filter_is_alive()`.

An attribute may hold a function value (`var on_press: Spite.Function<Nothing>? = null`, set later). A nullable
one is narrowed through its path like any other value before it is called: `fallback button.on_press`, then
`button.on_press()`, and a local that only copied it is rewritten to that form. One whose
default names its own class's function (`var act: Spite.Function<Nothing> = rest`) is bound to the object itself.
When the program only calls it (`holder.act()`, `act.call_function()`), narrows it or clears it (`act = null`), it
holds the object without a count, so the object is freed when its last other holder goes and the value goes with
it. A call through it holds the object for the call, so a call that lets go of the object's last other holder still
runs on a live object, freed when the call returns. `copy()` and `deep_copy()` bind the copy's value to the copy. Read
for anything else (stored, passed, returned), it keeps its count, and the cycle leaks like any other until one side
is cleared (see [memory.md](memory.md#cycles-are-freed)).

```gdscript title=own_function_default_doc/holder.spite
var console = Console()
var name = "nobody"
var act: Spite.Function<Nothing> = rest

func rest() {
    console.print(name, "rests")
}

func drop() {
    console.print(name, "is freed")
}
```
```gdscript title=own_function_default_doc/own_function_default_doc.spite entry
func OwnFunctionDefaultDoc() {
    var original = Holder()
    original.name = "jay"
    var copied = original.copy()
    copied.name = "kim"
    original.act()
    copied.act()
}
```
```output
jay rests
kim rests
jay is freed
kim is freed
```

**A variable never has the name of a function it can see.** Since a function's
name is already a value, a local, parameter or attribute named like a function of its class would make that name
mean two things. `var file_stem = file_stem(path)` is an error; name the result for what it holds:

```gdscript title=shadowed_function_error/shadowed_function_error.spite entry error
var console = Console()

func ShadowedFunctionError() {
    var file_stem = file_stem("notes/today.txt")
    console.print(file_stem)
}

func file_stem(path: String): String {
    var parts = path.split("/")
    var last = parts.last()
    return last.replace(".txt", "")
}
```
```diagnostic
the variable 'file_stem' has the name of a function of this class, and a function is a value passed by its name: give the variable a name of its own
```

Written `var stem = file_stem("notes/today.txt")`, it compiles. The check covers every class, generic ones included,
and a function nothing calls.

## Variadic arguments

A last parameter written `...name: List<Type>` takes every remaining argument, written one by one at the call,
and the function receives them as an ordinary `List<Type>`. `Type` may be a class or a `type`: with a `type`,
each argument is any class that fits it, and a call on an element answers for the class it really is.

```gdscript title=variadic_doc/variadic_doc.spite entry
type Named {
    name: String
}

var console = Console()

func VariadicDoc() {
    var line = joined("-", "a", "b", "c")
    console.print(line)
    var nobody = joined("-")
    console.print("[{nobody}]")
    var aria = Guest("Aria")
    var bram = Guest("Bram")
    greet(aria, bram)
}

func joined(separator: String, ...words: List<String>): String {
    return words.join(separator)
}

func greet(...visitors: List<Named>) {
    visitors.each(say_hello)
}

func say_hello(visitor: Named) {
    console.print("hello {visitor.name}")
}
```
```gdscript title=variadic_doc/guest.spite
var name: String

func Guest(new_name: String) {
    name = new_name
}
```
```output
a-b-c
[]
hello Aria
hello Bram
```

Only the last parameter can take `...`, and it always receives a `List`. The values are passed one by one, so
passing a whole list to it is an error, and so is a `...` that is not on the last parameter or is not typed `List<Type>`; a parameter written without `...` takes a list as it is. Each call
builds the `List` its function receives, exactly as if the caller had built it, and nothing more.

Each value casts toward the element type like any argument (`Integer` values into a `...values: List<Float>` arrive
as `Float`s), and like any argument it is never a call or a construction: compute it on a line of its own and pass
the name. A function value keeps the spread, so naming `log` and calling the value with `("x", "y")` gathers them
the same way. `console.print`, `write` and `error` are ordinary variadic functions over `...values: List<Printable>`.

```gdscript title=variadic_not_a_list/variadic_not_a_list.spite entry error
var console = Console()

func VariadicNotAList() {
    shout("a", "b")
}

func shout(...words: String) {
    console.print(words)
}
```
```diagnostic
'...words' receives the arguments as a list: write '...words: List<Type>'
```

## Plain values where a `type` is wanted

Every place names what it holds ([values_and_types.md](values_and_types.md#every-place-names-what-it-holds)): there is
no type of every class, and a `type` or union with no members is refused. A function taking a `type` (or a union) is
compiled once for each class that reaches it, so a number, a `Boolean` or an enum value passed to it arrives as
itself; stored in a `List<Shown>` it keeps its class beside it, and `if shown.class == Health` narrows it back. A
value of a union passed where a `type` is wanted brings the union's members into the shape, so a function the
members share can be called on it. An enum value answers `to_string()` (its name) and `to_debug()`, which is all a
shape can ask of it, and `.class` read through the shape names the value's real class (`Integer`, the enum). Where a
value is only shown, walked or kept, the `type` says so: every class answers `to_debug()`, so
`type Shown { to_debug: Spite.Function<String> }` takes any object, numbers included.

A value reflection reads has a type the program names too: `Spite.Attribute<Value>.value` is a `Value`, which a
function taking attributes names ([reflection.md](reflection.md#a-class-and-an-instance-of-it)). Calling `to_string()`
on a value typed by a `type`, or showing it in a hole, goes through the compiler's `type Printable`
([values_and_types.md](values_and_types.md#a-hole-shows-a-value-through-its-own-to_string)): every class that reaches
it needs its own, and nothing stands in for a missing one.

A `T?` fits a `type` unnarrowed only when null can answer everything the `type` asks, which is `to_string()` and
`to_debug()`: both answer `null` for it. So `console.print(found.value)` and `console.debug(found.value)` with a
`found: Spite.Attribute<Console.Printable?>` show `null` when the attribute holds none, and so does a hole
`"{found.value}"`, or one on a `T?` of a `type` that asks for `to_string()` (`var held: Shown? = null`, `"[{thing.held}]"` shows `[null]`), while `found.value.to_string()` needs it
narrowed first. Any other `type` needs the value narrowed first, or proven present
([proofs.md](proofs.md#a-value-proven-present-is-a-t)): `var singer: Singer = maybe_rock` with a `Rock?` is "this value may be null (it is a
Rock?), but a Singer is needed: narrow it first with 'if value { }', 'fallback value' or 'crash value'".

### `null` only goes where a `T?` is wanted

`null` given to a type without `?` (assigned, passed, appended, answered or declared) is a compile error, never a
default made for it: not `0`, `false` or `""` for a number, a `Boolean` or a `String`, not the first value of an
enum, and not an object of a class or a `type`, built through its constructor or without it. Write `T?` where none
is a real answer, and narrow it before use, or give it a real value:

```spite
func packed(): Crate {
    return null
}
```
```diagnostic
'null' is given where a Crate is needed, and a Crate never holds null: write 'Crate?' where none is a real answer, otherwise give it a real Crate
```

A local a walk fills is made first, `var row = $row_type()`.

### `= null` on an attribute

An attribute whose type has no `?` never holds null, so `= null` on it is refused; the words point to the forms
that fit ([classes_and_files.md](classes_and_files.md#an-attribute-the-constructor-fills-has-no-default)):

```spite
var target: Pet = null

func Hunt(new_target: Pet) {
    target = new_target
}
```
```diagnostic
the attribute 'target' is declared 'Pet', not 'Pet?', so it never holds null: write 'var target: Pet' with no default when the constructor gives it its value, 'var target: Pet = Pet()' for a default, or 'var target: Pet? = null' for none yet, narrowed before use
```

## A function taking a union or a type is copied per argument type

A function whose parameter is a `union` or a `type`, or a generic holding one (`List<Shown>`, `List<Named>`),
is compiled once for each type a call hands it: `shown(cat)` with `func shown(named: Named)` calls a copy whose
`named` is a `Cat`, so reading `named.name` reads the cat's own attribute with no test of its class while running.
Each copy is checked as if it were written for that type, so a body that does not fit a copy is a compile error at
the line in the body, naming the copy and the call that made it. Copies whose machine code comes out the same are
one function in the executable ([optimizations.md](optimizations.md#functions-whose-machine-code-is-the-same-are-one)).

```spite
type Stocked {
    to_debug: Spite.Function<String>
}

func Armory() {
    var healths = List<Health>()
    stocked(healths)
}

func stocked(items: List<Stocked>) {
    var sword = Sword()
    items.append(sword)
}
```
```diagnostic
a Sword cannot be used where a Health is needed: a value is cast to the class a place wants through that class's conversion, and Health declares no 'Health(from: Sword)', in its copy for (List<Health>) that the call at line 7 in Armory.Armory makes
```

So a `List<Health>` goes where a `List<Stocked>` is wanted whenever the body only does what a `List<Health>` can:
`items.count()` and reading its items fit, appending a `Sword` does not. Writing through a `type` is checked the
same way: with `type Node { next: Node }`, `link(first, second)` writing `first.next = second` compiles for two
`A` objects and is refused for an `A` and a `B`, since `A.next` holds an `A`. A class naming itself
(`var parent: Person?` in `person.spite`) is a plain class and needs no copy.

Inside a copy the class is known, so `if pet.class == Dog` is decided and a `switch pet` runs only the case naming
the copy's class; the function as written is still checked for its declared types, so its body uses only what the
union or `type` offers, and every case of a `switch` is still checked there.

A value whose class is known only while running (an item of a `List<Named>`, an attribute typed `Named`) is handed
as the `type` itself and reaches the function as written. Writing a member through such a value needs every class
that fits the `type` to hold the member's own type in that attribute: "'CatKeeper' does not fit type 'Keeper': this
program assigns 'pet' through the type, which can store a Zoo.Named there, but attribute 'pet' is a Cat".

The compiler copies the functions of the program's own classes; a function of the standard library, a generic
class's function, a constructor and a parameter given a variadic list are compiled once, as written. A parameter the
body assigns, or copies into a local with no written type (`var current = node`), keeps its declared type in every
copy, since the local may later hold any value of that type. A copy that calls a function already copied on the
way to it with bigger types (a recursion that wraps its argument, `wrapped([value])`) calls that function as
written, so the copies always end; with types no bigger it gets its copy as any call does.

## Every operator is a function

Every operator is a shortcut for a function a class can define to support it, with the parameters the table shows. Any other count is an error where the operator is used: `this operator on a 'Thing' needs it to define 'sum(other)', and its 'sum' takes 0 arguments`. `equals`, `less_than` and `greater_than` are checked where they are declared instead ([Functions the compiler calls by name](#functions-the-compiler-calls-by-name)).

| Operator | Function |
|---|---|
| `a + b` | `sum(b)` |
| `a - b` | `subtract(b)` |
| `a * b` | `multiply(b)` |
| `a / b` | `divide(b)` |
| `a % b` | `remainder(b)` |
| `a == b` / `a != b` | `equals(b)` (negated for `!=`) |
| `a === b` / `a !== b` | none: asks for the very same object ([The same instance](#the-same-instance)) |
| `a < b` / `a > b` | `less_than(b)` / `greater_than(b)` |
| `a <= b` / `a >= b` | derived from the two above, no own function |
| unary `-a` | `negate()` |
| `a[x]` / `a[x] = v` | `get_at(x)` / `set_at(x, v)` |

`a[x]` is `get_at(x)`, so any class that declares `get_at` can be indexed, and since every `[]` answers a value
that may be absent, `get_at` answers a `T?`: `func get_at(index: Integer): Integer?`,
`null` when nothing is at the index, and the reader narrows it (`crash shelf[0]`, then `shelf[0] + 1`). A `get_at`
that answers a plain `Integer` is an error naming `Integer?`.

`==` and `!=` also take a value that may be missing, with nothing to narrow first: a missing value equals only
another missing one, and a chain read through one (`made.kind == "make"`) is missing when any link is, so the
comparison is simply `false` when there is no `made`. Write the comparison you mean, in a `return` as in an `if`
([Comparing needs no narrowing](failure.md#comparing-needs-no-narrowing)).

`not`/`and`/`or` stay built-in keywords, never functions, and `and`/`or` short-circuit: in `false and check()` the
call never runs. On numbers, `Boolean`, enums, `String`, `List<T>` and `Dictionary<Key, Value>` the operators are
built in (on numbers, the machine's own arithmetic; on whole numbers a zero divisor halts for `/` and for `%`).
The library declares the operators its classes answer as ordinary functions, so `Integer.functions["sum"]` and
`String.functions["get_at"]` are there to read. A class without `equals` is given one that compares its
attributes ([Every class has equals](#every-class-has-equals)); the `equals` it declares must take its own class
(`func equals(other: Money): Boolean`), or the compiler says so. Either way the right side meets that class through the ordinary casts, like any argument: `price == "10"` calls
`Price(from: String)` when `Price` declares it, and without one it is refused, never simply `false`: `a String cannot
be used where a Price is needed: a value is cast to the class a place wants through that class's conversion, and
Price declares no 'Price(from: String)'`. A value with a `to_string()` compared with a text compares as text first,
when its class takes no conversion from text ([Rune](standard_library.md#rune)), so `tag == "cost"` compares
`tag.to_string()` and `Price.attributes["cost"] == "cost"` the attribute's name.
Number arithmetic is done in the left side's type, so a wider right side is an error that says to write the wider
operand first ([Wider arithmetic goes wider operand first](values_and_types.md#wider-arithmetic-goes-wider-operand-first));
a comparison is a call on its left side like any other operator ([below](#a-comparison-is-a-call-on-its-left-side)).
For a class of your own, the operator is a direct call of the matching function (nothing more at run time),
and the right side casts toward the parameter's type like any other argument:

```gdscript title=operators_as_functions/money.spite
var cents: Integer

func Money(starting_cents: Integer) {
    cents = starting_cents
}

func sum(other: Money): Money {
    return Money(cents + other.cents)
}

func equals(other: Money): Boolean {
    return cents == other.cents
}

func greater_than(other: Money): Boolean {
    return cents > other.cents
}
```
```gdscript title=operators_as_functions/operators_as_functions.spite entry
var console = Console()
var _wallet_cents = 150
var _expected_total_cents = 200

func OperatorsAsFunctions() {
    var wallet = Money(_wallet_cents)
    var found = Money(50)
    var total = wallet + found
    console.print("total cents", total.cents)
    var two_hundred = Money(_expected_total_cents)
    var is_two_hundred = total == two_hundred
    console.print("equal", is_two_hundred)
    console.print("greater", total > wallet)
}
```
```output
total cents 200
equal true
greater true
```

Missing the function is a diagnostic that names exactly what to define:

```gdscript title=operator_missing_error/money.spite
var cents: Integer

func Money(starting_cents: Integer) {
    cents = starting_cents
}
```
```gdscript title=operator_missing_error/operator_missing_error.spite entry error
var console = Console()

func OperatorMissingError() {
    var one = Money(1)
    var two = Money(2)
    console.print(one + two)
}
```
```diagnostic
needs it to define 'sum(other)'
```

### A comparison is a call on its left side

`a == b`, `a != b`, `a < b`, `a <= b`, `a > b` and `a >= b` are calls on `a` (`equals`, `less_than`,
`greater_than`), made before anything is typed, so `b` goes into `a`'s parameter through the same casts as any
argument or assignment: numbers, `T?` values, unions and `type`s alike, and nothing is widened, cut or
judged for the comparison alone. On numbers, enums and the other built-in classes the call is the machine's own
comparison. When `b` may not fit (`count < total` with an `Integer` `count` and a `Long` `total`, `progress > -0.5`
with an `Integer` `progress`, `circle == held` with a `Circle` and a union value):

- when the swapped form fits (`total > count`, `-0.5 < progress`, `held == circle`), the compiler swaps the sides itself
  with a `fixed:` line, `'count < total' called on an Integer, which a Long on the right may not fit; swapped the sides to 'total > count'`;
- where it cannot rewrite the line (both sides call something, so swapping would change what runs first, or the line
  is in a library or generic class), the error names both ways out, narrowing by hand on its own line or swapping:
  `this value is a Long, which may not fit in an Integer: write 'var converted: Integer? = value' on its own line,
  which is null when it does not fit, and narrow that, or swap the sides ('total > count')`;
- when neither order fits (an `UnsignedLong` or an `UnsignedInteger` beside a signed number), it is the assignment's
  error alone, since swapping would only meet it again: narrow one side by hand.

A `T?` on either side needs no narrowing for `==` and `!=`: two missing values are equal, a missing one equals no
value, and a present right side goes into the left side's type as above (`color_type == encoded[0]` with an
`Integer` `color_type` compares `Integer`s).

```gdscript title=comparison_call/comparison_call.spite entry error
var console = Console()
var _huge: UnsignedLong = 7
var _below_zero = -1

func ComparisonCall() {
    compare(_huge, _below_zero)
}

func compare(big: UnsignedLong, small: Integer) {
    console.print(big == small)
}
```
```diagnostic
this value is an Integer, which may not fit in an UnsignedLong: write 'var converted: UnsignedLong? = value' on its own line
```

### Every class has equals

A class that declares no `equals` is given one: `a == b` answers whether every attribute of `a` equals the same
attribute of `b`, each compared with its own `==`, in the order the class declares them, private attributes
included. Two `Point(1, 2)` are equal, and so are two lists of equal points. An object always equals itself. The
given `equals` is called like a written one: the right side casts like any argument (`price == "10"` needs
`Price(from: String)`), and `a.equals(b)` is refused the same way, saying to write `a == b`.

- A back reference (an attribute pointing at an object that holds this one, such as a child's `parent`) is compared
  by value like any attribute, so a `deep_copy()` of a tree equals the tree.
- A cycle (a ring of nodes, a tree whose children point back at their parent) never loops: a comparison already
  under way between the same two objects on the same thread answers `true`, so two rings are equal when every node
  of one equals the node at the same place in the other.
- A singleton and an enum compare as they always did: a singleton is one object per value of its arguments, an enum
  by its value.
- A value typed by a `type` or a union compares through the class it holds: two values of the same class
  call that class's `equals` (its own or the given one, a number or a text by its value), two of different classes
  are not equal, two `null`s are equal and `null` is never equal to a value. This holds for an attribute the given
  `equals` compares and for `a == b` written on such values alike. Such a value goes on the left: a class on the
  left takes only its own class, so `circle == held` is swapped to `held == circle` with a `fixed:` line.
- A `List` equals another holding equal items in the same order, and a `Dictionary` another holding equal values
  under the same keys, so a class holding them compares what they hold.
- A function value held in an attribute compares like any function value ([Comparing function values](#comparing-function-values)).
- A `Dictionary` keyed by an object still finds each key by identity (one object, one key), whatever `equals`
  answers; a class that declares its own `equals` still cannot be a key.

The compiler answers `true` without calling anything when both sides are the same name, or when the class holds no
attributes.

### The same instance

`==` always asks `equals`, never which object a value is. `a === b` asks whether `a` and `b` hold the very same
object, not two equal ones, and `a !== b` is its opposite, the way `!=` is the opposite of `==`. They bind like
`==`, call no function and need no narrowing:

- on two objects of one class (`first === kept`); a class on one side and another class on the other is an error,
  since they are never the same object: `'===' asks whether two names hold the very same object, and a 'Crate' is
  never the same object as a 'Box'`;
- on a value typed by a `type`, on a function value (the same function on the very same owner), and on a value of
  a union whose every member is a class, beside any
  value it may hold: `part === wanted` with a `Part` from a `List<Part>`, `held === apple` with a `Fruit`, with no
  `if part.class == Label` first;
- on a `T?` side: two missing values are the same, a missing one is never the same as an object, so a list read
  (`child === root.children[0]`) needs no narrowing.

A number, a `Boolean`, a `String` or an enum is a value, not an object (a union holds only classes, so `===` always
takes one): `===` on them is an error naming the comparison to write, `'===' asks whether two names hold the very same object, and a
'Integer' is a value, not an object: compare it with '=='` (`'!='` for `!==`). Comparing against `null` is the same
error as `== null`: write `if value`, `fallback value` or `crash value`.

```gdscript title=same_instance_doc/label.spite
var text: String

func Label(starting_text: String) {
    text = starting_text
}

func name_of(): String {
    return "label {text}"
}
```
```gdscript title=same_instance_doc/same_instance_doc.spite entry
type Part {
    name_of: Spite.Function<String>
}

var console = Console()

func SameInstanceDoc() {
    var parts = List<Part>()
    var title = Label("title")
    var copied = Label("title")
    parts.append(copied)
    parts.append(title)
    var place = place_of(parts, title)
    console.print(place, copied == title, copied === title, copied !== title)
}

func place_of(parts: List<Part>, wanted: Part): Integer {
    var index = 0
    while index < parts.count() {
        var part = parts[index]
        if part === wanted {
            return index
        }
        index = index + 1
    }
    return -1
}
```
```output
1 true false true
```

`a.is_same_instance(b)` written by name is refused like every operator's function: `'is_same_instance' is what
'===' calls: write 'first === second', or 'first !== second' for 'not first.is_same_instance(second)'`.

Use `===` as little as possible: only where the program really means one object, such as checking that a
`deep_copy()` made a new object, that a back reference points at this very node, or which child of a tree an event
reached. Asking which object a value is pins it: the compiler can no longer choose to share a copy with its original or to copy where it would share
([A deep copy nothing changes is the original](https://github.com/Spite-Language/spite_compiler/blob/master/OPTIMIZATIONS.md#a-deep-copy-nothing-changes-is-the-original)),
so a class asked about anywhere is copied as written. `==` asks only the value, which a shared copy answers alike.

### Comparing function values

`a == b` on two function values answers whether they are the same function bound to equal owners: the same function
of the same class, and owners equal by the owner's `==` (its own `equals` or the given one, a number or a text by its
value). Two `reader.changed` taken from one `reader` are equal, `first.changed` and `second.changed` are equal when
`first == second`, and two different functions are never equal, whatever their owners. `!=` is the opposite.
`a === b` on two function values asks for the same function bound to the very same owner object (`first.changed ===
second.changed` is false for two equal but separate owners); two missing ones are the same. It works
wherever the function is known only at run time, so `list.index_of(handler)`, `list.contains(handler)` and
`list.remove(handler)` find an equal function value in a `List<Spite.Function<...>>`, a generic class's list
included, and a class's given `equals` compares a function value attribute the same way. Comparing keeps nothing alive: it reads both values and their
owners and lets go.

```gdscript title=function_value_compare_doc/listener.spite
var console = Console()
var name: String

func Listener(starting_name: String) {
    name = starting_name
}

func heard(amount: Integer) {
    console.print(name, "heard", amount)
}
```
```gdscript title=function_value_compare_doc/function_value_compare_doc.spite entry
var console = Console()
var handlers = List<Spite.Function<Integer, Nothing>>()

func FunctionValueCompareDoc() {
    var ada = Listener("ada")
    var bob = Listener("bob")
    var also_ada = Listener("ada")
    handlers.append(ada.heard)
    handlers.append(bob.heard)
    console.print(ada.heard == also_ada.heard, ada.heard == bob.heard)
    handlers.remove(bob.heard)
    var holds_bob = handlers.contains(bob.heard)
    var holds_ada = handlers.contains(also_ada.heard)
    console.print(holds_bob, holds_ada)
}
```
```output
true false
false true
```

## Setter/getter interception

Attribute access goes through operator-style functions, from outside a class and inside it, the constructor
included. `person.age = 1` calls `set_age(1)` (an exact function the class writes, or a template; see
[metaprogramming.md](metaprogramming.md)); `person.age` (a read) calls `get_age()` the same way. An exact
function always wins over a template. Compound assignment goes through both: `person.age = person.age + 1`
reads through `get_age()` and writes through `set_age(...)`.

```gdscript title=interception/account.spite
var balance: Integer
var owner: String

func Account(starting_balance: Integer, starting_owner: String) {
    balance = starting_balance
    owner = starting_owner
}

func set_attribute(attribute: Spite.AttributeDeclaration, value: attribute.class) {
    attributes[attribute].value = value
}

func get_attribute(attribute: Spite.AttributeDeclaration): attribute.class {
    return attributes[attribute].value
}

func set_owner(new_owner: String) {
    attributes["owner"].value = new_owner.upper_case()
}

func get_owner(): String {
    return attributes["owner"].value
}
```
`balance` has no exact `set_balance`/`get_balance`, so a template answers reads and writes to it from
outside the class. Every read calls the getter again, so a check of a getter that may write something
(`if account.next_owner { }`) proves nothing about a second read: hold it in a `var` and narrow the `var`
([failure.md](failure.md#narrowing-a-path)). `owner` has exact functions instead, which always win over a template, so every name
written through `set_owner` is capitalized.

The templates answer `account.balance = 5` and `account.balance`. Calling one with the attribute written out, `account.set_attribute('balance', 5)`, is an error that tells you to write `account.balance = 5`, since a write has one spelling; a template is called with an attribute only when the attribute is a value, such as the one a walk over the attributes hands you.

```gdscript title=interception/interception.spite entry
var console = Console()

func Interception() {
    var account = Account(100, "ann")
    account.balance = account.balance + 20
    console.print("balance", account.balance)
    account.owner = "bob"
    console.print("owner", account.owner)
}
```
```output
balance 120
owner BOB
```

### Every attribute has a setter and a getter

Every write of an attribute runs its setter and every read runs its getter, wherever it is written: outside the
class, inside it, and in the constructor. A bare `age = 1` calls `set_age(1)` and a bare `age` calls `get_age()`, and
so does `other.age` on another object of the class. An attribute the class writes no `set_age` for has one anyway,
which the compiler writes while it desugars the class, as if the class held

```spite
func set_age(value: Integer) {
    attributes["age"].value = value
}

func get_age(): Integer {
    return attributes["age"].value
}
```

A template the class declares (`set_attribute`, `get_attribute`) stands in for them where it has no exact function.
A setter that only stores its parameter and a getter that only answers the storage, the ones the compiler writes and
any the class writes that way, are compiled as the plain write and read: they cost nothing, keep what a check proved,
and leave no function behind. A default (`var age = 3`) is the storage's starting value, set before the constructor
runs, and runs no setter.

```gdscript title=interception_inside/thermostat.spite
var changes = 0
var target: Integer

func Thermostat(starting_target: Integer) {
    target = starting_target
}

func set_target(value: Integer) {
    changes = changes + 1
    attributes["target"].value = value.clamp(10, 30)
}

func warm_up() {
    target = target + 15
}
```
```gdscript title=interception_inside/interception_inside.spite entry
var console = Console()

func InterceptionInside() {
    var thermostat = Thermostat(25)
    console.print(thermostat.target, thermostat.changes)
    thermostat.warm_up()
    console.print(thermostat.target, thermostat.changes)
}
```
```output
25 1
30 2
```

The constructor's `target = starting_target` ran `set_target` too, so a setter that checks or clamps a value
checks the starting one as well. A setter that reads the value it replaces reads the storage, which before the
constructor's first write holds the attribute's default; for an attribute with no default that read is refused like
any read before the constructor gives the value (`the attribute 'target' has no default, so the constructor gives it
its value, but line 5 runs 'set_target', which uses it, before it has one`), and a default the setter may read is
never removed as one the constructor replaces.

Every write outside its own setter runs the setter, so a setter that writes another attribute whose setter writes this
one back is a recursion like any other, reported at run time as a stack overflow. A getter may compute and hand back
a fresh `String` or `List` as safely as it returns a field. An intercepted access is a direct call, which the compiler
inlines like any small function.

### The storage behind the accessors

`attributes["age"].value` is the attribute's storage. Writing it is the one write that runs no setter, and reading it
runs no getter; on another object it is `other.attributes["age"].value`, and in a walk
`kept.attributes[attribute].value` (see [reflection.md](reflection.md#an-instance-indexed-by-a-declaration)). It
compiles to the plain write or read of the field.

Inside an attribute's own setter and getter a bare name is refused, written or read, since it would run that same
accessor again: the setter and the getter reach their attribute only as `attributes["age"].value`, and the compiler
rewrites a bare `age` there to that (`fixed: 'age' inside 'set_age' would run 'set_age' again: wrote
'attributes["age"].value', which reaches the storage`). The other attributes are read and written as anywhere else,
through their own accessors. A setter that compares the new value with the one it replaces reads the storage:

```gdscript title=storage_behind_the_accessors/lamp.spite
var console = Console()
var level = 0

func set_level(value: Integer) {
    if attributes["level"].value != value {
        console.print("level", attributes["level"].value, "to", value)
    }
    attributes["level"].value = value
}
```
```gdscript title=storage_behind_the_accessors/storage_behind_the_accessors.spite entry
func StorageBehindTheAccessors() {
    var lamp = Lamp()
    lamp.level = 3
    lamp.level = 3
    lamp.attributes["level"].value = 5
    lamp.level = 4
}
```
```output
level 0 to 3
level 5 to 4
```

A private attribute's functions are private too: `_open = true` calls `_set_open(true)` and a bare `_open` calls
`_get_open()`, never `set__open`.

A setter may stand without any attribute behind it: `ticket.id = 4` calls the class's own `set_id(value)` beside a
private `_id`, and a getter without an attribute answers the read. Only a function the class itself declares does
this; a template never answers a write to a name the class has no attribute for (`account.ghost = 5` is an error
saying an `Account` has no attribute or function `ghost`).

**A getter with no setter makes a read-only attribute**. That is how
reflection keeps `Spite.Class.name` from being overwritten
([reflection.md](reflection.md#reflection-is-read-only)):

```gdscript title=read_only_doc/temperature.spite
var _celsius: Float

func Temperature(starting_celsius: Float) {
    _celsius = starting_celsius
}

func get_fahrenheit(): Float {
    return _celsius * 9.0 / 5.0 + 32.0
}
```
```gdscript title=read_only_doc/read_only_doc.spite entry
var console = Console()

func ReadOnlyDoc() {
    var boiling = Temperature(100.0)
    console.print(boiling.fahrenheit)
}
```
```output
212
```

```gdscript title=read_only_error/temperature.spite
var _celsius = 0.0

func get_fahrenheit(): Float {
    return _celsius * 9.0 / 5.0 + 32.0
}
```
```gdscript title=read_only_error/read_only_error.spite entry error
func ReadOnlyError() {
    var boiling = Temperature()
    boiling.fahrenheit = 50.0
}
```
```diagnostic
'fahrenheit' is read-only
```

## Use the operator, not its function

When a class offers an operator for a function, **the operator is the only way to call it**. Writing the
function's name instead is a compile error that names the shortcut, so every program says the same thing the same
way and a reader never wonders whether `get_x()` and `.x` differ:

| Written | Error: write instead |
|---|---|
| `point.get_x()` | `point.x` |
| `point.set_x(4)` | `point.x = 4` |
| `shelf.get_at(0)` / `names.get_at("ann")` on a `List` or a `Dictionary`, and the `set_at` of both | `shelf[0]` / `names["ann"]`, and `shelf[0] = book` / `names["ann"] = 3` |
| `a.sum(b)`, `a.subtract(b)`, `a.multiply(b)`, `a.divide(b)`, `a.remainder(b)` | `a + b`, `a - b`, `a * b`, `a / b`, `a % b` |
| `a.equals(b)` / `not a.equals(b)` | `a == b` / `a != b` |
| `a.is_same_instance(b)` / `not a.is_same_instance(b)` | `a === b` / `a !== b` |
| `a.less_than(b)` / `a.greater_than(b)` | `a < b` / `a > b` |
| `a.negate()` | `-a` |

```gdscript title=operator_by_name/money.spite
var cents: Integer

func Money(starting_cents: Integer) {
    cents = starting_cents
}

func sum(other: Money): Money {
    return Money(cents + other.cents)
}
```
```gdscript title=operator_by_name/operator_by_name.spite entry error
var console = Console()

func OperatorByName() {
    var wallet = Money(150)
    var found = Money(50)
    var total = wallet.sum(found)
    console.print(total.cents)
}
```
```diagnostic
'sum' is what '+' calls: write 'wallet + found'
```

**The one exception is the function used as a value**, where no operator can stand in: `run_callback(point.get_x)`
hands over the function itself, so it is written by name. Declaring the function stays as it is: `func
get_x()`, `func sum(other)`, `func get_at(index)` are how a class offers the operator. **After a `.`, a word is
always a member name**, never a keyword, so a getter named after a keyword is read like any other: `get_return()`
is read as `thing.return`. `type` and `singleton` are the exceptions: they are reserved and never a name, after a
`.` or anywhere else.

The rule holds on every class (built in, library or your own), through a `type` or a union value (when every
member offers the operator), and inside the class itself: `negate()` inside `Duration` is refused, write `-this`.
Inside a class, a bare `set_turns(3)` is refused the same way (`'set_turns' is what writing 'turns' calls: write
'turns = 3'`), and so is a bare `get_largest()`, whether or not the class has an attribute of that name, since a bare
`turns = 3` runs the setter everywhere, the constructor included. A private one names its own form: `'_set_open' is
what writing '_open' calls: write '_open = true'`. The exception is inside the attribute's own setter and getter, where
the bare name is refused and the call is the only way to the other accessor (`get_turns()` in `set_turns`), and where a
local hides the attribute.
`get_attribute`/`set_attribute` called with an attribute that is a value, such as one a walk hands you, are not
refused either.

## One name, one function

There is no overloading: one name is one function. A file that declares the same name twice is an error,
`'Shop' declares 'price' twice: a later file may reopen a class and replace a declaration, but one file declares
each name once`; replacing is what a reopening file does ([packages.md](packages.md#monkey-patching-mods)). An argument is cast to its parameter's type by the ordinary
right-to-left rule ([values_and_types.md](values_and_types.md#numeric-types-and-the-casting-rule)), so a
function that takes a `Float` takes an `Integer` too:

```gdscript title=argument_casting/argument_casting.spite entry
var console = Console()

func ArgumentCasting() {
    var whole = 7
    var halved = half_of(whole)
    console.print(halved)
}

func half_of(value: Float): Float {
    return value / 2.0
}
```
```output
3.5
```

A class that wants to accept several kinds of value takes a `type` or a union and says so in one signature.
A parameter that is never read is an error, as every unused name is: remove it, or name it `_name` when the
signature needs it. An operator function's, a template's or a setter's parameter needs neither, since its
signature is dictated from outside ([style.md](style.md#nothing-unused)).

## Functions the compiler calls by name

Some functions are never called by the program's own code: the compiler calls them itself. `drop()` runs when the
last reference to an object goes ([memory.md](memory.md)), `to_string()` wherever an object is used as text,
`to_debug()` wherever it is shown for debugging, and `equals`, `less_than` and `greater_than` for `==`, `<` and
`>`. Each has a signature the compiler enforces, and declaring one with another is an error that writes the one
it must have, since a `drop` with a parameter could never run, an `equals` with two would leave `==` comparing
identity, and a `to_debug` answering a number would be shown as text; the parameter of `equals`, `less_than` and
`greater_than` must be the class itself, because a comparison with another type is never the one the operator calls:

| Function | Signature |
|---|---|
| `drop` | `func drop()` |
| `to_string` | `func to_string(): String` |
| `equals` | `func equals(other: Class): Boolean` |
| `to_debug` | `func to_debug(): String` |
| `less_than` | `func less_than(other: Class): Boolean` (`<` and `>=`) |
| `greater_than` | `func greater_than(other: Class): Boolean` (`>` and `<=`) |
```gdscript title=drop_with_parameter/handle.spite
var open = true

func drop(_reason: Integer) {
    open = false
}
```
```gdscript title=drop_with_parameter/drop_with_parameter.spite entry error
var console = Console()

func DropWithParameter() {
    var handle = Handle()
    console.print(handle.open)
}
```
```diagnostic
'drop' is called by the compiler when the last reference to an object goes, so it is declared exactly 'func drop()'
```

---

Next: [Control flow](control_flow.md), the conditions, loops and switches that steer a program.
