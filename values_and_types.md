# Values and types

## Numeric types and the casting rule

There is no cast syntax. **The right side always casts toward the left side**: in a binary operation,
assignment, a function argument (toward the parameter type), and `return` (toward the return type).

| Type | C type |
|---|---|
| `Tiny` | `int8_t` |
| `Short` | `int16_t` |
| `Integer` (default integer) | `int32_t` |
| `Long` | `int64_t` |
| `Byte` | `uint8_t` |
| `UnsignedShort` | `uint16_t` |
| `UnsignedInteger` | `uint32_t` |
| `UnsignedLong` | `uint64_t` |
| `Float` (default decimal) | `float` (32-bit) |
| `Double` | `double` (64-bit) |

The names are written in full, as every name is: `Int` or `UnsignedInt` is an error that names the spelling to
use (`'Int' is spelled 'Integer'`). `count()`, `length()` and `index_of()` always answer an `Integer`, whatever they
count.

A number is written in plain digits, and `_` may separate them for reading: `1_000_000` is a million and
`0.000_01` a hundred-thousandth, a `_` standing only between two digits: `1__000`, `1000_` and `12._5` are errors.
A decimal has digits on both sides of its point (`.5` is not a number). There is no exponent (`2.5e3` is an
error that writes `2_500` for you).

An integer literal is an `Integer` and a decimal literal a `Float`. A whole number too big for an `Integer` is a
compile error that asks for a wider type: declare the name `Long` (`var _total: Long = 3000000000`). **A number
never silently wraps.** Arithmetic whose answer does not fit its type halts, unsigned exactly as signed, and so does a
wider value put into a narrower name ([below](#arithmetic-that-does-not-fit-halts)). Where wrapping is the point,
as in a hash or a noise function, the code says so with a function named for it: `wrapping_sum`,
`wrapping_subtract` and `wrapping_multiply` on every whole number keep the low bits of the answer:

```gdscript title=numeric_overflow/numeric_overflow.spite entry
var console = Console()
var _value_that_fits_a_byte = 200
var _large_hash: UnsignedInteger = 4000000000

func NumericOverflow() {
    load "../spite_standard_library/windows"
    var wide = _value_that_fits_a_byte
    var cut: Byte = wide
    console.print("fits", cut)
    var hash: UnsignedInteger = _large_hash
    hash = hash.wrapping_multiply(3)
    console.print("wraps to", hash)
}
```
```output
fits 200
wraps to 3410065408
```

### Arithmetic that does not fit halts

`+`, `-` and `*` on a whole number (`Tiny`, `Short`, `Integer`, `Long`, `Byte`, `UnsignedShort`,
`UnsignedInteger`, `UnsignedLong`) whose answer does not fit its type is a developer's mistake, like dividing by
zero. Every build checks each one, the one you ship included, and halts at the first that does not fit, naming
the operation, the type, the operands and the line:

```gdscript
func scaled(amount: Integer, factor: Integer): Integer {
    return amount * factor          # scaled(2000000000, 2)
}
```
```
spite: 'amount * factor' does not fit in an Integer (2000000000 * 2), at game/game.spite:17 in Game.scaled
```

A whole-number `/` or `%` by zero halts too (`'total / parts' divided by zero, at ...`), and a divisor the compiler
sees is zero (`total / 0`) is a compile error. A decimal that is not a number, or is infinite, put into a whole
number halts like any value that does not fit. Float division stays IEEE: `x / 0.0` is infinity or not-a-number.
`%` on a `Float` or a `Double` is the remainder of the division cut toward zero, so it keeps the sign of the left
side as a whole number's does: `-7.5 % 2.0` is `-1.5`, `730.5 % 360.0` is `10.5`, and `x % 0.0` is not-a-number.
A decimal prints with the fewest digits that read back as the same value, so a `Float` `0.1` prints `0.1`, never
`0.100000001`; a value that would need an exponent prints with one (`1e+21`, `1.0000000000000001e-05`), and
infinity and not-a-number print `inf` and `nan` on every system.
The same holds for `-` in front of a value (`-lowest` with the smallest `Integer`, or any unsigned value but `0`),
for the smallest signed value divided by `-1`, and for a value put into a narrower name, by assignment, as an
argument or by `return`: `var small: Tiny = count` halts when `count` is 200 (`spite: 200, an Integer, does not fit
in a Tiny, at ...`), and so does a decimal outside the whole number it is put into. A number written into a name
it does not fit (`var small: Byte = 300`) is a compile error. The compiler leaves a check out wherever it can
prove the answer fits ([proofs.md](proofs.md#arithmetic-that-does-not-fit-halts)): from the ranges your locals can
hold, a counter under its loop's bound, a remainder, a clamp, and a total that adds one term a pass over a list
([proofs.md](proofs.md#a-range-proves-arithmetic-fits)), so what remains costs little.

Which checks stay is not a guess: build with `--optimization-report=.spite/reports/name.md` and the report lists every
operation that `keeps its overflow check` with the range it saw for each side. What the compiler knows about a
number: an attribute, a list item or a call's answer can be anything its type holds; `count()` and `length()` run
from 0 to the type's top; dividing by a constant divides the range, and dividing by a name gives a result no larger in
size than the dividend; `minimum` and `maximum` with a written number bound the answer; a cast to a wider type keeps
the range. A condition narrows a number in its block with `<`, `<=`, `>`, `>=` and `==` (either way round), after a
`fallback` or `crash` that states it, on the right of an `and`, and after an `if` that leaves. An `or`, a `!=` and an
`else` narrow nothing, and neither does an `if` that leaves under an `or` (write two `if`s). A local assigned in the
branches of an `if` or a `switch` holds everything it could have been; one assigned in a loop holds what it entered
with and every value the loop gives it. A `Long` total of `Integer` terms added over a list cannot overflow, and
arithmetic on constants is worked out while compiling and never reported. Reading the report from a `--repl`,
`--repl-port` or `--hot-reload` build shows the same checks. A divisor proved non-zero stops being proved when an
attribute is assigned, when the local its path starts from is assigned, when anything is made, or when a loop starts
its next pass; the division then checks again.
The fix is a wider type written first (`Long` for a total of `Integer`s), or, when wrapping is what you meant, the
function that says so:

```gdscript title=wrapping_by_name/wrapping_by_name.spite entry
var console = Console()
var _hash_offset_basis: UnsignedLong = 1469598103934665603
var _hash_prime: UnsignedLong = 1099511628211

func WrappingByName() {
    load "../spite_standard_library/windows"
    var hash: UnsignedLong = _hash_offset_basis
    var text = "spite"
    var index = 0
    while index < text.length() {
        var code = text.byte_at(index)
        var mixed = hash.bits_exclusive_or(code)
        hash = mixed.wrapping_multiply(_hash_prime)
        index = index + 1
    }
    console.print(hash)
}
```
```output
16868922117198771000
```

### Wider arithmetic goes wider operand first

Arithmetic is done in the left side's type, and the right side is cast to it: `count * total` with an `Integer`
`count` and a `Long` `total` is an `Integer` multiply. So a right side wider than the left (more bits, or a
`Float`/`Double` under a whole number) is a compile error naming the fix: write the wider one first
(`total * count`), or, for `-`, `/` and `%`, store the left side in the wider type first. Wider means more bits:
`Tiny` and `Byte` have 8, `Short` and `UnsignedShort` 16, `Integer`, `UnsignedInteger` and `Float` 32, `Long`,
`UnsignedLong` and `Double` 64, so a `Short` under a `Byte`, a `Double` under a `Float` and an `Integer` variable
under a `Byte` are errors, while signedness alone is not wider. A literal on the right that fits the left type is
fine (`small + 1` stays a `Byte` addition); one that does not fit (`small + 300`) is the error "300 does not fit in
a Byte". A constant that overflows the `Integer` its
arithmetic is done in is an error too, instead of wrapping:

```gdscript title=wider_first/wider_first.spite entry error
var console = Console()

func WiderFirst() {
    load "../spite_standard_library/windows"
    var width = 65536
    var area: Long = 4294967296
    var scaled = width * area
    console.print(scaled)
}
```
```diagnostic
'width * area' is a multiplication in Integer, since arithmetic takes the left side's type, and the right side is a Long, which would be cut to fit: write the Long first ('area * width')
```

```gdscript title=constant_overflow/constant_overflow.spite entry error
var console = Console()

func ConstantOverflow() {
    load "../spite_standard_library/windows"
    var pixels: Long = (65536 - 120) * 65536
    console.print(pixels)
}
```
```diagnostic
'(65536 - 120) * 65536' is 4287102976, which does not fit in an Integer, the type its arithmetic is done in, so it would wrap: write the number itself, 4287102976, which is a Long
```

### Comparisons are checked the same way

A comparison casts its right side toward the left too, so an `Integer` compared with a `Float` would compare
`Integer`s: `progress > -0.5` would cut `-0.5` to `0` and answer `0 > 0`, false, where the mathematics says true.
So a comparison whose right side is wider than its left is a compile error naming the fix, as arithmetic is:
write the wider side first, with the comparison turned around:

```gdscript title=casting_edge/casting_edge.spite entry error
var console = Console()

func CastingEdge() {
    load "../spite_standard_library/windows"
    var progress: Integer = 0
    console.print("mathematically true", progress > -0.5)
}
```
```diagnostic
'progress > -0.5' compares in Integer, since a comparison takes the left side's type, and the right side is a Float, which would be cut to fit: write the Float first ('-0.5 < progress')
```

Written the other way round the comparison is done in `Float`, and answers what the mathematics does. An integer
literal that fits the left side is not wider (`small < 10` with a `Byte` `small`), and a decimal literal under a
whole number never fits:

```gdscript title=casting_fixed/casting_fixed.spite entry
var console = Console()

func CastingFixed() {
    load "../spite_standard_library/windows"
    var progress: Integer = 0
    console.print("mathematically true", -0.5 < progress)
    var price: Float = 3.0
    var quantity: Integer = 2
    console.print("total", price * quantity)
}
```
```output
mathematically true true
total 6
```

`String` converts both ways. Text is read as a number by that number class's conversion, which runs where text
meets a `T?` of the number: `var age: Integer? = "42"`. It answers `null` when the text is not a number that fits
the type, since a `0` there would read as a real zero. Text put into a plain `Integer` is an error that names the
`?` to write, and there is no `to_integer()` to call. A number becomes text with its own
`to_string()` ([below](#numbers-are-classes)), and wherever text is wanted the number is made text by itself.
The same goes for anything with a `to_string()`: a `Boolean`, an enum value (its name), a class read as a value (its
name) and an object of any class that declares `func to_string(): String`. Where a `String` is wanted (an
assignment, a `var name: String = value`, an argument, a `return`, the key of a `Dictionary<String, Value>`), such a
value is made text through it: `label.text = clicks` stores `"42"`, and `show(badge)` with `func show(text: String)`
passes `badge.to_string()`. A class with no `to_string()` is an error there: `a Pet cannot be used where a String is
needed`.

### Numbers are classes

`Integer`, `Long`, `Float`, `Double`, `Boolean` and the rest are classes in `library/` (`library/integer.spite`,
`library/double.spite`, ...), the way `String` is. Their functions are called on a value like any class's,
and inside one of them `this` is the number itself. Turning a number into text is `to_string()`, written in Spite in
`library/long.spite` and `library/double.spite`, and it is what a text hole (`"count {count}"`) calls. A program reopens a number
class the way it reopens any class ([packages.md](packages.md#monkey-patching-mods)), with a file named after it.
How wide a number is comes from its class too: each one reaches its own value through `set_this` and `get_this`, as
any class reaches an attribute through `set_x` and `get_x`, and `set_this` checks the range it holds
(`crash value >= _smallest_value and value <= _largest_value`, with `-128` and `127` named in `Tiny`), and the
compiler reads the width and the sign from that range. A program that loads no standard library has no number
classes at all: `Integer` is then an unknown type. Reading and writing `this` are the compiler's own, so
`count.get_this()` and `count.set_this(4)` are errors that tell you to write `count` and `count = 4`. There is no
type for "any number": `Number` names nothing, and a class written over numbers is generic
([below](#a-number-library-declares-the-type-it-needs)).
A reopening file:

```gdscript title=number_methods/integer.spite
func doubled(): Integer {
    return this * 2
}
```
```gdscript title=number_methods/number_methods.spite entry
var console = Console()

func NumberMethods() {
    load "../spite_standard_library/windows"
    var count = 21
    var doubled = count.doubled()
    var count_text = count.to_string()
    console.print(doubled, count_text)
    var third: Double = 1.0 / 3.0
    var bits = third.bits()
    console.print(third, bits)
}
```
```output
42 21
0.3333333333333333 4599676419421066581
```

None of this costs anything at run time: a number is still a plain value (an `Integer` is an `int32_t`), the
class gives it functions, not a header, and a function the program never calls is not emitted. A cast from one
number type to another is written by assignment, `var half: Float = count`, or by passing the value where the
other type is expected; it compiles to the one machine conversion, and there is no call form.

The same rule casts between any two classes. Where a value meets a place that wants another class (an argument, a
`return`, an assignment, a `var` with a written type), the compiler calls the wanted class's conversion from it, a
function named after the class with one parameter `from`
([classes_and_files.md](classes_and_files.md#constructors-conversions-and-presets)): `var paint: Color? = "#ff8000"`
reads the text through `func Color(from: String): Color?`, and a `Meters` passed where a `Feet` is wanted goes
through `func Feet(from: Meters): Feet?` when `Feet` declares it. The cast answers what the conversion answers, so
text read as an `IpAddress` is an `IpAddress?`, narrowed like any other `T?` before it goes where a plain `IpAddress`
is wanted. A literal is read while compiling: `var host: IpAddress = "127.0.0.1"` needs no narrowing, and
`var host: IpAddress = "not a host!"` is a compile error naming the literal and the conversion. A conversion that needs
a run to answer (one that asks the system, as `"localhost"` does, or prints) stays a `T?`. A class with no such conversion is an error naming the conversion it would need:
`a Point cannot be used where an IpAddress is needed: ... IpAddress declares no 'IpAddress(from: Point)'`.

`this` works in every class, not only numbers, to hand the object itself to something:
`registry.append(this)`. Reading your own members through it is an error, because a class already reads them by
name: write `name`, not `this.name`.

A decimal is worked out for speed, and its last bits are never a promise. A decimal literal is a `Float`, so
`tenth * 3.0` with a `Float` `tenth` multiplies in `Float` precision, and `tenth == 0.1` is `true` after `var tenth:
Float = 0.1`. Inside a loop the compiler may fuse a multiply and an add into one step and add up a sum in another
order, which is what lets a loop over decimals run several at a time; `Float.NotANumber()` and the infinities still
behave as they always do. A loop that compares decimals with `==` or `!=` keeps every step in the order written,
so what an exact comparison sees is exact.

### Bitwise functions

There are no bitwise operator symbols: `value << 3` and `value >> 3` are errors that name `value.shifted_left(count)`
and `value.shifted_right(count)`, and a lone `&`, `|`, `^` or `~` names `bits_and`, `bits_or`, `bits_exclusive_or`
and `bits_inverted`. Every whole-number class (`Tiny` to `UnsignedLong`, not `Float`, `Double`
or `Boolean`) answers them as functions, each compiled to the one C operation and inlined in an optimised build:

| Function | Answers |
|---|---|
| `shifted_left(count)` | the bits moved `count` places up, zeros coming in |
| `shifted_right(count)` | moved down: a signed type copies its sign bit in (arithmetic), an unsigned type zeros (logical) |
| `bits_and(other)`, `bits_or(other)`, `bits_exclusive_or(other)` | the bits set in both, in either, in exactly one |
| `bits_inverted()` | every bit flipped |
| `set_bit_count()`, `leading_zero_count()`, `trailing_zero_count()` | how many bits are set, and how many zeros stand above the highest set bit and below the lowest, as an `Integer` (the width, for 0) |

Each answers the receiver's type, with `other` cast to it like any argument: a `Byte` and a `Long` mask give a
`Byte`. No shift is undefined: a count of the width or more moves every bit out, and a negative count halts the
program. Calling one on a `Float` or a `Double` is an error, since they are not whole numbers; cast to a
whole number first. `bits_as_unsigned()` on a signed whole number and `bits_as_signed()` on an unsigned one keep
the bits and read them with the other sign, so a signed `-1` is `4294967295` as an `UnsignedInteger`, where storing
`-1` in an `UnsignedInteger` halts (a change of sign is checked like any narrowing; only an unsigned value stored in
a wider signed type is never checked, since it always fits). A decimal's bits are read the same way:
`Float.bits()` answers an `UnsignedInteger` and `Double.bits()` a `Long`, and `UnsignedInteger.bits_as_float()`,
`Long.bits_as_double()` and `UnsignedLong.bits_as_double()` turn them back, with nothing converted. These are what
hashes and binary formats use; each is written in place, touches no memory and allocates nothing.

```gdscript title=bitwise_basics/bitwise_basics.spite entry
var console = Console()
var _upper_four_bits_set: Byte = 240

func BitwiseBasics() {
    load "../spite_standard_library/windows"
    var flags = 12
    var mask = 10
    console.print("{flags.bits_and(mask)} {flags.bits_or(mask)} {flags.bits_exclusive_or(mask)}")
    var packed = flags.shifted_left(4).bits_or(3)
    console.print("{packed} {packed.shifted_right(4)} {packed.bits_and(15)}")
    var negative = -16
    var small: Byte = _upper_four_bits_set
    console.print("{negative.shifted_right(2)} {small.shifted_right(2)} {small.bits_inverted()}")
    console.print("{flags.shifted_left(32)} {negative.shifted_right(40)} {small.set_bit_count()}")
}
```
```output
8 14 6
195 12 3
-4 60 15
0 -1 4
```

### Maths functions

The maths a game or a renderer needs are functions of the number classes too, spelled out in full: on a `Float`
or a `Double`, `square_root()`, `sine()`, `cosine()`, `tangent()`, `arc_sine()`, `arc_cosine()`,
`arc_tangent()`, `rise.arc_tangent_over(run)` (the angle of the point `(run, rise)`, in the right quadrant),
`power(exponent)`, `exponential()`, `logarithm()` (natural), `logarithm_base_2()`, `logarithm_base_10()`,
`floor()`, `ceiling()`, `round()` (half away from zero), `truncate()`, `absolute()`, `minimum(other)`,
`maximum(other)`, `clamp(low, high)`, `is_finite()`, `is_infinite()` and `is_not_a_number()`. The whole numbers
have `absolute()`, `minimum(other)`, `maximum(other)` and `clamp(low, high)`. Each answers the receiver's type
(the three questions a `Boolean`), with the other operands cast to it like any argument, and each is the C
library's function written where it is called (`sqrtf` on a `Float`, `sqrt` on a `Double`), with no call of
Spite's own around it.

A number class also answers its special values as presets on the class itself, `Float.Pi()`, since they belong to
no one value ([classes_and_files.md](classes_and_files.md#constructors-conversions-and-presets)): `Pi()`, `Tau()`,
`EulerNumber()`, `Infinity()`, `NotANumber()`, `Largest()` and `Smallest()` on `Float` and `Double`, and
`Largest()` and `Smallest()` on every whole number. `Integer.Largest()` is 2147483647 and `Float.Smallest()` the
most negative `Float` there is. A preset builds its value on each call, and the compiler folds it. It is always
called on the class, with its PascalCase name: `angle.pi` is `'Pi' is a preset of the class Float, not of a value:
write 'Float.Pi()'`, `Float.pi()` names `Float.Pi()` the same way, and a function of a value called on the class,
`Float.square_root()`, is an error too. Another language's short name (`sqrt`, `sin`, `pow`, `min`) is an error
that gives the Spite one: `Float has no function 'sqrt': Spite spells it 'square_root'`.

```gdscript title=maths_basics/maths_basics.spite entry
var console = Console()

func MathsBasics() {
    load "../spite_standard_library/windows"
    var side: Double = 2.0
    var diagonal = side.square_root()
    console.print("{diagonal} {Float.Pi()} {Double.Pi()}")
    var speed = 7.5
    var capped = speed.clamp(0.0, 5.0)
    var ahead = speed.round()
    var behind = -2.5
    console.print("{capped} {ahead} {behind.round()} {behind.floor()} {behind.absolute()}")
    var negative = -1.0
    var root = negative.square_root()
    console.print("{root} {root.is_not_a_number()} {Integer.Largest()}")
}
```
```output
1.4142135623730951 3.1415927 3.141592653589793
5 8 -3 -3 2.5
nan true 2147483647
```

As with `/` on a float, nothing here halts: the square root of `-1` is not-a-number and the logarithm of
`0` is minus infinity, the IEEE answers. What each function answers at its edges is
[standard_library.md](standard_library.md)'s.

### Named numbers

A number that means something is given a name: a private attribute whose default is the number
(`var _days_in_400_years = 146097`), read by name where it is used. Written bare inside a function, a whole number
of three or more digits that is neither a power of ten nor a power of two is an error asking for that attribute,
and so is a decimal that is a library preset (`3.14159` names `Float.Pi()`); an attribute's default, the body of a
preset, numbers of one or two digits and text are left alone. The rule and its exact message are in
[style.md](style.md#a-number-gets-a-name). A named number costs nothing: an attribute nothing writes is read as
its default where it is read, so it compiles to the number itself. A preset may be the default
(`var _full_turn = Float.Tau()`), and the attribute then has the preset's class.

### A number library declares the type it needs

No `type` stands for every number. A class written over numbers is generic, and declares, as a `type` of its own,
the functions its codegen value must have, listing only the operators and functions it uses
([shapes](#inline-types-and-duck-typing)). The operators `+`, `-`, `*`, `/` and `%` are the functions `sum`,
`subtract`, `multiply`, `divide` and `remainder`, and `<` and `>` (with `<=` and `>=`) are `less_than` and
`greater_than`; the type's own name in them stands for the class that fits, so an `Integer` adds an `Integer` and
answers one. Every number class, `Tiny` to `Double`, has these functions, so it fits such a type the way any class
fits a type, by having its members, and `Boolean` fits none of them. The class is compiled once per number class it
is given, so the values stay plain machine numbers and nothing is boxed. `$value_type.is_number` asks, while
compiling, whether a type is a number class, which is how a walk over types tells numbers apart
([metaprogramming.md](metaprogramming.md#asking-what-a-generic-was-given)):

```gdscript title=number_type_doc/statistics.spite
generic $number_type: Measure

type Measure {
    greater_than: Spite.Function<Measure, Boolean>
}

func largest(values: List<$number_type>): $number_type {
    crash values[0]
    var best = values[0]
    var index = 1
    while index < values.count() {
        if values[index] > best {
            best = values[index]
        }
        index = index + 1
    }
    return best
}

func mean(values: List<$number_type>): Double {
    var total: Double = 0.0
    var index = 0
    while index < values.count() {
        total = total + values[index]
        index = index + 1
    }
    return total / values.count()
}
```
```gdscript title=number_type_doc/describe.spite
generic $value_type

func kind(): String {
    if $value_type.is_number {
        return "a number"
    }
    return "not a number"
}
```
```gdscript title=number_type_doc/number_type_doc.spite entry
var console = Console()

func NumberTypeDoc() {
    load "../spite_standard_library/windows"
    var scores: List<Integer> = [4, 9, 2]
    var score_statistics = Statistics<Integer>()
    var best_score = score_statistics.largest(scores)
    var mean_score = score_statistics.mean(scores)
    console.print(best_score, mean_score)
    var weights: List<Float> = [0.5, 2.5]
    var weight_statistics = Statistics<Float>()
    var heaviest = weight_statistics.largest(weights)
    console.print(heaviest)
    var byte_kind = Describe<Byte>()
    var flag_kind = Describe<Boolean>()
    var byte_text = byte_kind.kind()
    var flag_text = flag_kind.kind()
    console.print(byte_text, flag_text)
}
```
```output
9 5
2.5
a number not a number
```

`Statistics<Boolean>` is refused while compiling, since `Boolean` has no `greater_than`. An operator on a value read
as a `type` of your own that requires the operator's function, `left + right` on two values read as
`type Addable { sum: Spite.Function<Addable, Addable> }`, calls the `sum` of the left side's class, and a right side of another
class halts with `which is not one`. Through a `type` that does not require the operator's function, the operator
is an error (`'+' on a value read as 'Labeled' needs the class that fits the type, which is not known while
compiling`): take the value through a codegen value the type constrains, `generic $value_type: Addable`.

## `String`

Immutable, length-prefixed (not a bare `char*`), reference counted. A value is placed inside written text rather
than joined to it with `+`: `"hello {name}"`, where `{ }` holds one value of any type (every numeric type,
`Boolean`, and enum format themselves) and `\{` is a brace meant literally. Two values still join with `+`, and
joining written text with `+` is an error naming the form above. `==`/`!=`/`<`/`>` compare by content.

### Text escapes

A backslash writes a character a text cannot hold as itself: `\n` (line break), `\t` (tab), `\r` (carriage
return), `\\` (backslash), `\"` (double quote) and `\{` (brace). Escapes follow each other freely, so `"\\\{"` is a
backslash and a brace, and a text written inside a hole keeps its own escapes and holes. Any other character
after a backslash is that character (the formatter writes `\'` as `'`). A `{` that is not escaped opens a hole,
which ends at its matching `}`. A `{` that never closes by the end of the file (`this '{' inside text never
closes`), and a text that never closes by the end of its line (`this text never closes`), are errors.

### Values in text

A text that is one hole and nothing else, `"{clicks}"`, is an error: it is the value
itself, so the value is written directly, and where text is wanted it becomes text on its own. The error gives the line to write:

```gdscript title=lone_hole/lone_hole.spite entry error
var console = Console()
var label = ""

func LoneHole() {
    load "../spite_standard_library/windows"
    var clicks = 3
    label = "{clicks}"
    console.print(label)
}
```
```diagnostic
'"{clicks}"' is a text of one value and nothing else: assign the value directly, 'label = clicks'
```

The line depends on where the text stands: `declare the value directly, 'var shown = clicks'` for a `var` (a written
type stays: `'var shown: String = clicks'`), `return the value directly`, `pass the value directly`, `join the
value directly` beside a `+`, `look it up by the value directly` as a dictionary key, and `write its text as
'clicks.to_string()'` anywhere else. A text with a written character beside the hole, or with two holes, is
ordinary text.

```gdscript title=direct_text/direct_text.spite entry
var console = Console()
var label = ""

func DirectText() {
    load "../spite_standard_library/windows"
    var clicks = 3
    label = clicks
    var line = "clicks: {clicks}"
    console.print(label, line)
}
```
```output
3 clicks: 3
```

Building text a piece at a time costs what the pieces cost, not the text so far: `text = "{text}{piece}"` (or
`text = text + piece`) on a local variable appends in place when nothing else holds that text, and copies it
once when something does, so the other holder still sees the text it had.

```gdscript title=text_building/text_building.spite entry
var console = Console()

func TextBuilding() {
    load "../spite_standard_library/windows"
    var line = "count:"
    var index = 0
    while index < 3 {
        line = "{line} {index}"
        index = index + 1
    }
    var before = line
    line = "{line} done"
    console.print(before)
    console.print(line)
}
```
```output
count: 0 1 2
count: 0 1 2 done
```

```gdscript title=string_basics/string_basics.spite entry
var console = Console()

func StringBasics() {
    load "../spite_standard_library/windows"
    var subject = "Spite"
    var greeting = "Hello, {subject}"
    console.print(greeting)
    var greeting_length = greeting.length()
    console.print("length", greeting_length)
    var upper_case_greeting = greeting.upper_case()
    console.print("upper", upper_case_greeting)
    var mentions_spite = greeting.contains("Spite")
    console.print("contains", mentions_spite)
    var parts = greeting.split(", ")
    var parts_count = parts.count()
    console.print("parts count", parts_count)
    var joined_parts = parts.join(" - ")
    console.print("joined", joined_parts)
    var age: Integer = "42"
    console.print("parsed", age)
    var typed = "not a number"
    var not_a_number: Integer? = typed
    if not_a_number {
        console.print("parsed", not_a_number)
    } else {
        console.print("not a number")
    }
}
```
```output
Hello, Spite
length 12
upper HELLO, SPITE
contains true
parts count 2
joined Hello - Spite
parsed 42
not a number
```

Reading a number that is not there, like `"not a number"` above, answers `null`, never a `0` that would read as a
real zero. Reading with `[]` is the exception: an index or a key that may not be there answers a `T?`
([failure.md](failure.md#reading-with--answers-t)). See [standard_library.md](standard_library.md) for the full
method table.

A `type` or a `union` says what a value must have and makes no object: `Tag("favourite")` is an error, `'Tag' is a type, so it makes no object: it says what a value must have; make an object of a class that fits it`.

## `T?`: values that may be null

A number that may be null never counts as 0: narrow it before `+ - * / %`, `< > <= >=`, a unary `-`, a store or a call, whatever its width. A `Short?` added to an `Integer` is `this value may be null (it is a Short?), but an Integer is needed: narrow it first with ...`. A `Short?` widens into an `Integer?` and keeps its null.

`Monster?` is a `Monster` or `null`, and `null` exists for nothing else. A `T?` is narrowed before it is used
(`if value { } else { }`, `fallback value`, `crash value`, `while value`, or a `switch` with a `Null:` case), and
reading with `[]` answers one. A `Boolean?` is the one `T?` that is no condition, since its `false` would read as
missing; `fallback flag` alone takes one, and asks that it is there and `true`. All of it is in
[failure.md](failure.md), with the three outcomes a failure can have. Since `null` belongs to `T?` alone, `var target: Monster = null` is an error that asks for `Monster?` (none
yet) or `Monster()` (a default), and a generic class makes the default of what it is bound to with `$name()`. For a
plain value the default is its literal: `0`, `0.0`, `false` or `""`, which the error writes for you.

A `var` may be written again in the same scope, and the new one may hold another type:

```gdscript
var content = file.read_as_string()        # a String?
fallback content
var content = content.trim()               # a String
```

The new value is worked out first, so `content.trim()` reads the old `content`; then the old value is let go
(running its `drop()` if nothing else holds it); only then does the name mean the new value. A `var` in an inner block may
shadow a local, a parameter or an attribute of the same name for the length of that block; it may never shadow
a function it can see. A shadowed name that was never read is removed by the compiler and reported as `fixed:`.

## Enums

```gdscript title=enum_basics/player.spite
enum Job {
    'knight'
    'mage'
    'archer'
}

var job: Job = 'knight'

func Player(starting_job: Job) {
    job = starting_job
}

func is_knight(): Boolean {
    return job == 'knight'
}
```
```gdscript title=enum_basics/enum_basics.spite entry
var console = Console()

func EnumBasics() {
    load "../spite_standard_library/windows"
    var hero = Player('mage')
    console.print("job", hero.job)
    var hero_is_knight = hero.is_knight()
    console.print("is knight", hero_is_knight)
}
```
```output
job mage
is knight false
```

An enum declaration takes no `=` and lists one value per line, with no commas (a value may be given a number,
[below](#numbering-an-enums-values)). A value is written in single
quotes (single quotes mean an enum value and nothing else), and `'mage'` alone resolves without writing
`Player.Job`, because the parameter, the annotation, the assignment or the comparison it is used in says which
enum it belongs to. The enum itself belongs to `Player`: inside it, plain `Job`; in any other class, `Player.Job`,
never plain `Job`. Enums, unions, types and number classes take no `<>`: `Phase<Integer>` is an error that says to write
`Phase` alone.

An enum is a closed list of values, and `'knight'` is an **enum value literal**: a value the enum does not list
is a compile error naming the ones it does, and `'orange'` where no enum is expected (`var choice = 'orange'`, with
nothing to check it against) is an error too, even when exactly one enum has that value: write it where the enum
is expected. Every enum is its own class with its conversions: `value.to_string()`
is its name, `var number: Integer = value` its number, `var part: Bird.Part? = "tail"` and `var part: Bird.Part? = 7`
read one from a name or a number (`null` when no value has it), and `Part.values` lists the values. The class that declares the enum may add its own conversions, named by the enum alone
(`func Job(from: Float): Job?`); a value of that type meeting the enum goes through it, and one from `String` replaces
the compiler's reading of a name: with `func Course(from: String): Course?` answering `'soup'` for `"starter"`, the text
`"main"` is read as `null`, because the function now decides. One from `Integer` does the same for numbers. A reload
(`--hot-reload`) of a file that declares an enum replaces the enum whole, so a value added to the list is there for the
program that keeps running. At run time an enum value is a small
integer.

Every enum is its own class, and the compiler gives it two **conversions** written the way any class writes one
([classes_and_files.md](classes_and_files.md#constructors-conversions-and-presets)), as if `EnumFromText` declared
`func Course(from: String): Course?` and `func Course(from: Integer): Course?`. Like every conversion, it is applied
where a value meets the class, never called: assigning text to a `Course?` reads the value spelled that way, and a
whole number the value with that number, each `null` when the enum has none, since any value of the enum there
would read as a real one. Writing the call, `Course(name)`, is refused as a conversion called by name, and assigning
text to a plain `Course` is an error naming the `?` to write. The class that declares the enum may give it more
conversions, written in that class and named by the enum alone (`func Course(from: Float): Course?` in
`EnumFromText`); one from text or a whole number replaces the compiler's own:

```gdscript title=enum_from_text/enum_from_text.spite entry
enum Course {
    'starter'
    'soup'
    'dessert'
}

var console = Console()

func EnumFromText() {
    load "../spite_standard_library/windows"
    var names = ["dessert", "brunch"]
    var index = 0
    while index < names.count() {
        var name = names[index]
        var course: Course? = name
        if course {
            console.print(name, "is a course:", course)
        } else {
            console.print(name, "is no course")
        }
        index = index + 1
    }
}
```
```output
dessert is a course: dessert
brunch is no course
```

### Walking an enum's values

An enum is a class, so `Course` read as a value is its `Spite.Enum` ([reflection.md](reflection.md#spiteenum)),
and `Course.values` lists its values, as `Course`s, in the order the enum declares them. It is an ordinary list, walked with `each` like any other
([metaprogramming.md](metaprogramming.md#walking-a-programs-structure)).

```gdscript title=enum_walk/enum_walk.spite entry
enum Course {
    'starter'
    'soup'
    'dessert'
}

var served: Course = 'soup'
var console = Console()

func EnumWalk() {
    load "../spite_standard_library/windows"
    Course.values.each(list_course)
}

func list_course(course: Course) {
    var is_served = course == served
    console.print(course, is_served)
}
```
```output
starter false
soup true
dessert false
```

It is all decided while compiling: the walk becomes three calls, and no list of an enum's values exists at run
time. An enum is also open to the program that loads it: reopening its
class declares the enum again with the whole new list of values
([packages.md](packages.md#reopening-an-enum-replaces-it)), and a walk then walks that list.

### Numbering an enum's values

Inside a program an enum's values are only names. When something outside it already has a number for each
value (a C library, a binary file, a database, the network), give the value its number with `=` on its own line:

```gdscript title=numbered_roles/numbered_roles.spite entry
enum Role {
    'admin' = 99
    'user'
    'guest' = 7
}

var console = Console()

func NumberedRoles() {
    load "../spite_standard_library/windows"
    var role: Role = 'user'
    console.print(role, role == 'user')
}
```
```output
user true
```

The number is a whole number that fits an `Integer`, and it belongs to the declaration alone: written after a value
anywhere else it is an error. `=` because giving a value is assignment, as everywhere else. A value without a number counts on from the one
before, so `'user'` is 100, and an enum's first value without a number is 0. A number, written or counted on, that
does not fit an `Integer` is an error. Inside the program the numbers change nothing: names, comparisons, `switch`,
`Course.values` and reading from text work as for any enum, and `var number: Integer = role` gives the number
(`99` for `'admin'`). A hot reload cannot change the number of a value the running program already has: that is an
error naming the value and both numbers, and a restart renumbers it. Two values with one number are a
compile error, since the number is the value's identity outside the program:

```gdscript title=role_number_twice/role_number_twice.spite entry error
enum Role {
    'admin' = 7
    'user'
    'guest' = 8
}

var console = Console()

func RoleNumberTwice() {
    load "../spite_standard_library/windows"
    var role: Role = 'guest'
    console.print(role)
}
```
```diagnostic
'user' and 'guest' both have the number 8 in enum 'Role': a value's number is its identity, so give one of them another
```

## Unions

A tagged union. A `switch` must cover every member, and narrows the value inside each case; when every member
shares a function/attribute of the same shape, it can be called directly on the union value without a
`switch` at all (duck-typed, the same way a `type` is; see below).

A union is declared in the class that consumes it. Two classes of a program that declare one name with one content
have it moved to the entry class of the namespace holding both, the class named like its folder, which the others name
`Bunda.Pet` (see [packages](packages.md)). Two with other names or members stay apart: a
`Shelter.Pet` passes where a `Groomer.Pet` is wanted when every member of it is a member of `Groomer.Pet`, at no cost.
A caller that only hands a value on passes the member itself (`groomer.groom(cat)`) and declares nothing. Copies
with the same members are one type inside generics too: a `List<Shelter.Pet>` goes where a `List<Groomer.Pet>` is
wanted, and a function taking one copy where a function taking the other is. Inside a generic the union given may not have fewer
members than the one wanted: a `List<Shelter.Pet>` with fewer members than `Groomer.Pet` is refused where a
`List<Groomer.Pet>` is wanted, since the receiver could put into it a member the smaller union lacks (`... has fewer
members than Groomer.Pet, and the receiver could put in it a member Shelter.Pet lacks`). The class that declares a
union may give it conversions, named by the union alone (`func Job(from: String): Job?` beside `union Job`), and a
value of that type meeting the union goes through it.

```gdscript title=union_basics/player.spite
var health = 10

func Player(starting_health: Integer) {
    health = starting_health
}

func is_alive(): Boolean {
    return health > 0
}
```
```gdscript title=union_basics/monster.spite
var health = 6

func Monster(starting_health: Integer) {
    health = starting_health
}

func is_alive(): Boolean {
    return health > 0
}
```
```gdscript title=union_basics/union_basics.spite entry
union Enemy {
    Player
    Monster
}

var console = Console()

func UnionBasics() {
    load "../spite_standard_library/windows"
    var enemy: Enemy = Monster(6)
    switch enemy {
        Player: console.print("a player", enemy.health)
        Monster: console.print("a monster", enemy.health)
    }
    var enemy_is_alive = enemy.is_alive()
    console.print("alive", enemy_is_alive)
}
```
```output
a monster 6
alive true
```

Like `enum` and `type`, a `union` takes no `=` and lists one member per line. A `switch` names every
member, with no `_:` for the rest, so a member added later is an error at every switch, and two members may share
a one-line case. `enemy.class == Monster` asks which member a value is
([control_flow.md](control_flow.md#switch-over-a-union)). `Monster?` is exactly this kind of union: `Monster` and
`Null`, the class whose only value is `null`.

A union holds objects whose class a `switch` can test, so `String`, a number class, `Boolean` and a generic class
such as `List<Integer>` are refused as members: make a class of your own that keeps the value in an attribute.

A union costs nothing to hold: the value is the object itself, and a `switch` or a call on it compares the class
its header already names against the members.

## Inline types and duck typing

A `type` is a class matched by shape: any value with the same attribute names and types is accepted,
including a plain object literal:

```gdscript title=duck_typing/player.spite
var weapon = "sword"
var power = 5

func Player(starting_power: Integer) {
    power = starting_power
}
```
```gdscript title=duck_typing/duck_typing.spite entry
type Loadout {
    weapon: String
    power: Integer
}

var console = Console()

func DuckTyping() {
    load "../spite_standard_library/windows"
    var hero = Player(8)
    console.print("class of hero", hero.class)
    var hero_loadout: Loadout = hero
    console.print("from class", hero_loadout.weapon, hero_loadout.power)
    var literal_loadout: Loadout = {weapon: "bow", power: 4}
    console.print("from literal", literal_loadout.weapon, literal_loadout.power)
}
```
```output
class of hero Player
from class sword 8
from literal bow 4
```

`.class` read through the `type`-shaped variable answers the same thing: the class the value really is, read
from the object's own tag rather than from the declared type. An object literal has no class of its own, so it
answers `Object`.

This is what makes fast, JSON-shaped code possible: accept a `type`, and both a real class instance and a
plain `{ key: value }` literal work. The empty `type`, which every object fits, is built in as `Anything`
(`component: Anything`, `List<Anything>()`; see [functions_and_operators.md](functions_and_operators.md)):
never declare an empty `type` of your own. A `type`, like an `enum` and a `union`, takes no `=` and no commas.

An attribute of a `type` may itself be a union, met by that union or one of its classes; one whose type is another
`type` is met by that `type`, or by a class or a union whose classes each have its members; one whose type is a
function value is met by a function value with the same parameter types and answer. Anything else does not
fit and the error names the attribute.

A `type` of attributes only has a default like a class does: the object literal with each attribute at its own
default, whose `.class` answers `Object`. So `var target = $target_type()` in a generic class bound to such a
`type` holds a real object, and what is written through it stays written.

A `type` may require functions as well as attributes, and a member has one form: `render: Spite.Function<Integer, String>`
takes an `Integer` and answers a `String`, and `finish: Spite.Function<Nothing>` answers nothing. It names the types
it takes and returns, never the names of its parameters, because the name a class gives its own parameter does not
matter to the shape. The compiler rewrites the old `render(Integer): String` to this form with a `fixed:` line.

Any class with a function of that signature fits, which is how a list holds "any class that can render" without a
union naming every class in advance:

```gdscript title=shape_functions_doc/badge.spite
var label = ""

func Badge(new_label: String) {
    label = new_label
}

func render(width: Integer): String {
    return "[{label}] ({width})"
}
```
```gdscript title=shape_functions_doc/banner.spite
func render(columns: Integer): String {
    return "== banner {columns} =="
}
```
```gdscript title=shape_functions_doc/shape_functions_doc.spite entry
type Renderable {
    render: Spite.Function<Integer, String>
}

var console = Console()

func ShapeFunctionsDoc() {
    load "../spite_standard_library/windows"
    var badge = Badge("new")
    show(badge)
    var banner = Banner()
    show(banner)
}

func show(item: Renderable) {
    var rendered = item.render(12)
    console.print(rendered)
}
```
```output
[new] (12)
== banner 12 ==
```

`show` above is compiled twice, once for `Badge` and once for `Banner`: a function that takes a `type` gets a
copy for each class that reaches it, and in that copy `item` is that class, so `item.render(12)` is a direct call.
A number, `Boolean` or enum value passed to it arrives as the plain value. When the class is known only as the
program runs (a value read from a `List<Anything>`, say), the call tests the value's class against every class the
program ever gives that `type` and runs the matching copy. A class that never reaches the function gets no copy,
so nothing is compiled for it.

A class with an attribute holding such a function fits as well, and `item.render(12)` is written the same whichever
one met it: a `type` does not care. Only an attribute can be assigned, so a program that assigns `item.render = other`
through the `type` needs every class that fits to hold one, and a class with the function is refused (`this program
assigns 'render' through the type, and only an attribute holding a function can be assigned`). An attribute that may
hold nothing, `Spite.Function<Integer, String>?`, meets a member written with the `?`.

A `type` can also be the element of a variadic parameter, `...items: List<Renderable>`, so a call takes any number
of values of any classes that fit ([functions_and_operators.md](functions_and_operators.md#variadic-arguments)).

A `type` has no functions of its own. A required function says only what a class must have, so a call through a
`type` is always the function of the class the value really is, compiled in that class's copy: `jobs[0].finish()`
below is `Job`'s own `finish`. A `List` of a `type` answers the member templates too, over the attributes and the
argument-free functions the type names: `map_names()`, `filter_done()`, `count_done()`:

```gdscript title=shape_values_doc/job.spite
var name = ""
var done = false

func Job(new_name: String) {
    name = new_name
}

func finish() {
    done = true
}
```
```gdscript title=shape_values_doc/shape_values_doc.spite entry
type Work {
    name: String
    done: Boolean
    finish: Spite.Function<Nothing>
}

var console = Console()

func ShapeValuesDoc() {
    load "../spite_standard_library/windows"
    var jobs = List<Work>()
    var wash = Job("wash")
    jobs.append(wash)
    var dry = Job("dry")
    jobs.append(dry)
    finish_first(jobs)
    var names = jobs.map_names()
    var joined = names.join(", ")
    var finished = jobs.count_done()
    console.print(joined, finished)
}

func finish_first(jobs: List<Work>) {
    fallback jobs[0]
    jobs[0].finish()
}
```
```output
wash, dry 1
```

---

Next: [Nullable values and failure](failure.md), how a missing value and a failed operation are handled.
