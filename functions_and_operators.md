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
is expected, never the other way round. `Nothing` is also what `.returns` reports for a function declared without
a return type. A function that is a text's or a number's (`greeting.contains`) is bound to a copy of the value.
A function value whose answer may be null answers it as null wherever it is held.

```gdscript title=function_values_doc/shouter.spite
func shout(text: String): String {
    return "{text.upper_case()}!"
}
```
```gdscript title=function_values_doc/function_values_doc.spite entry
var console = Console()

func FunctionValuesDoc() {
    load "../spite_standard_library/windows"
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
    load "../spite_standard_library/windows"
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

An attribute may hold a function value (`var on_press: Spite.Function<Nothing>? = null`, set later). One whose
default names its own class's function is bound to the object itself, which is a cycle that leaks like any other
until one side is cleared (see [memory.md](memory.md)).

**A variable never has the name of a function it can see.** Since a function's
name is already a value, a local, parameter or attribute named like a function of its class would make that name
mean two things. `var file_stem = file_stem(path)` is an error; name the result for what it holds:

```gdscript title=shadowed_function_error/shadowed_function_error.spite entry error
var console = Console()

func ShadowedFunctionError() {
    load "../spite_standard_library/windows"
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
    load "../spite_standard_library/windows"
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
var name = ""

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
    load "../spite_standard_library/windows"
    shout("a", "b")
}

func shout(...words: String) {
    console.print(words)
}
```
```diagnostic
'...words' receives the arguments as a list: write '...words: List<Type>'
```

## Anything, and plain values where a `type` is wanted

`Anything` is a built-in `type` that requires nothing, so every object fits it: `component: Anything` takes any
value and `List<Anything>()` holds any. Do not declare an empty `type` of your own. A function taking `Anything`
(or any `type`) is compiled once for each class that reaches it, so a number, a `Boolean` or an enum value passed to
it arrives as itself; stored in a `List<Anything>` it keeps its class beside it, and `if component.class == Health`
narrows it back. A value of a union passed where a `type` is wanted brings the union's members into the shape, so
a function the members share can be called on it. An enum value answers `to_string()` (its name) and `to_debug()`,
which is all a shape can ask of it, and `.class` read through the shape names the value's real class (`Integer`,
the enum). `Spite.Attribute.value`, the value an attribute holds, is typed `Anything?`. An empty `type` costs
nothing until a value is passed to it.

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
`String.functions["get_at"]` are there to read. A class without `equals` compares by identity under `==`; the
`equals` it declares must take its own class (`func equals(other: Money): Boolean`), or the compiler says so.
Number arithmetic is done in the left side's type, so a wider right side is an error that says to write the wider
operand first ([Wider arithmetic goes wider operand first](values_and_types.md#wider-arithmetic-goes-wider-operand-first));
a comparison is refused the same way, naming the comparison turned around (`'count < total' ... write the Long
first ('total > count')`).
For a class of your own, the operator is a direct call of the matching function (nothing more at run time),
and the right side casts toward the parameter's type like any other argument:

```gdscript title=operators_as_functions/money.spite
var cents = 0

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
    load "../spite_standard_library/windows"
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
var cents = 0

func Money(starting_cents: Integer) {
    cents = starting_cents
}
```
```gdscript title=operator_missing_error/operator_missing_error.spite entry error
var console = Console()

func OperatorMissingError() {
    load "../spite_standard_library/windows"
    var one = Money(1)
    var two = Money(2)
    console.print(one + two)
}
```
```diagnostic
needs it to define 'sum(other)'
```

## Setter/getter interception

Attribute access **from outside a class** also goes through operator-style functions. `person.age = 1` calls
`set_age(1)` when the class defines it (an exact function, or a template; see
[metaprogramming.md](metaprogramming.md)); `person.age` (a read) calls `get_age()` the same way. An exact
function always wins over a template. Compound assignment goes through both: `person.age = person.age + 1`
reads through `get_age()` and writes through `set_age(...)`.

```gdscript title=interception/account.spite
var balance = 0
var owner = ""

func Account(starting_balance: Integer, starting_owner: String) {
    balance = starting_balance
    owner = starting_owner
}

func set_attribute(attribute: Spite.AttributeDeclaration, value: attribute.class) {
    attributes[attribute] = value
}

func get_attribute(attribute: Spite.AttributeDeclaration): attribute.class {
    return attributes[attribute]
}

func set_owner(new_owner: String) {
    owner = new_owner.upper_case()
}

func get_owner(): String {
    return owner
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
    load "../spite_standard_library/windows"
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

Inside a class's own functions, a bare `age = 1` (no receiver) stays a plain field write when the class has an
attribute named `age`: interception only applies to `receiver.field` written from outside. A bare name that no
local and no attribute holds goes through the class's own `get_<name>()` and `set_<name>(value)`. A getter may compute and hand back a fresh
`String` or `List` as safely as it returns a field. It costs nothing hidden: an intercepted access is a direct
call, and every other access a plain field read or write.

A setter may stand without any attribute behind it: `ticket.id = 4` calls the class's own `set_id(value)` beside a
private `_id`, and a getter without an attribute answers the read. Only a function the class itself declares does
this; a template never answers a write to a name the class has no attribute for (`account.ghost = 5` is an error
saying an `Account` has no attribute or function `ghost`).

**A getter with no setter makes a read-only attribute**. That is how
reflection keeps `Spite.Class.name` from being overwritten
([reflection.md](reflection.md#reflection-is-read-only)):

```gdscript title=read_only_doc/temperature.spite
var _celsius = 0.0

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
    load "../spite_standard_library/windows"
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
    load "../spite_standard_library/windows"
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
| `a.less_than(b)` / `a.greater_than(b)` | `a < b` / `a > b` |
| `a.negate()` | `-a` |

```gdscript title=operator_by_name/money.spite
var cents = 0

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
    load "../spite_standard_library/windows"
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
'turns = 3'`), and so is a bare `get_largest()`. The exception is a class with an attribute of that name, or a local
that hides it: there a bare `turns = 3` writes the attribute itself, so the call is the only way to the setter. A
bare name that no local and no attribute holds reads through `get_<name>()` and writes through `set_<name>(value)`.
Two calls are not refused: `get_attribute`/`set_attribute` called with an attribute that is a value, such as one a
walk hands you, and a getter called with no receiver where the class holds the attribute, since the bare name
there is the raw field.

## One name, one function

There is no overloading: one name is one function. A file that declares the same name twice is an error,
`'Shop' declares 'price' twice: a later file may reopen a class and replace a declaration, but one file declares
each name once`; replacing is what a reopening file does ([packages.md](packages.md#monkey-patching-mods)). An argument is cast to its parameter's type by the ordinary
right-to-left rule ([values_and_types.md](values_and_types.md#numeric-types-and-the-casting-rule)), so a
function that takes a `Float` takes an `Integer` too:

```gdscript title=argument_casting/argument_casting.spite entry
var console = Console()

func ArgumentCasting() {
    load "../spite_standard_library/windows"
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
    load "../spite_standard_library/windows"
    var handle = Handle()
    console.print(handle.open)
}
```
```diagnostic
'drop' is called by the compiler when the last reference to an object goes, so it is declared exactly 'func drop()'
```

---

Next: [Control flow](control_flow.md), the conditions, loops and switches that steer a program.
