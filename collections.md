# Lists, dictionaries and member templates

`List<T>` and `Dictionary<Key, Value>` are not built into the compiler. They are ordinary generic classes written in Spite
(`library/list.spite` and `library/dictionary.spite`) over `Buffer`, the room everything else is built on. The compiler keeps only the syntax: `[1, 2, 3]`, `list[index]`, and `List<T>()`. Read
those two files to see exactly what a list does; write your own container the same way
([memory.md](memory.md#memory-belongs-to-the-compiler)).

Being ordinary classes, they cost what their code costs and nothing more: a list is one heap
buffer of its elements, a program compiles only the members it calls, and one that never makes a `Dictionary`
carries none of it. Nothing runs behind them: no collector, no iterator objects, no registry of templates.

## `List<T>`

A list literal is written with commas on one line, or one entry per line with no commas:

```gdscript
var primes = [2, 3, 5, 7]
var names = [
    "ada"
    "grace"
]
var empty = List<String>()
```

| Member | Result | Notes |
|---|---|---|
| `append(value)` / `prepend(value)` | | adds to the end / to the front (index 0) |
| `insert(index, value)` | | an index out of range is clamped to the nearest end |
| `list[index]` | `T?` | may not be there, so it is narrowed ([failure.md](failure.md#reading-with--answers-t)) |
| `list[index] = value` | | calls `set_at(index, value)`; an index out of range halts, naming the line |
| `get_at(index)` | `T?` | what `list[index]` calls, `null` out of range; it is only called through `[]` ([Use the operator](functions_and_operators.md#use-the-operator-not-its-function)) |
| `remove_at(index)` | | an index out of range halts |
| `remove_swapping(index)` | | moves the last element into `index` instead of moving every later one down ; an index out of range halts |
| `remove_where(test)` / `remove_where_<member>()` | | removes every element the test is true for, in one pass, keeping the rest in order ([below](#removing-many-at-once)) |
| `truncate(count)` | | keeps the first `count` elements and releases the rest; nothing happens when `count` is out of range |
| `swap(first, second)` | | exchanges two elements; nothing happens when either is out of range |
| `remove_first()` / `remove_last()` | `T?` | removes and returns it; `null` when empty, like `first()` |
| `first()` / `last()` | `T?` | `null` when empty, like `[]`: narrow it, `crash first` or `if first { }` |
| `count()` / `is_empty()` | `Integer` / `Boolean` | `count()` is only ever the list's size |
| `contains(value)` | `Boolean` | elements that are numbers, `Boolean`, `String` or an enum only |
| `index_of(value)` | `Integer?` | the place of the first element equal to `value`, or `null` when none is; the same elements as `contains` |
| `clear()` / `reverse()` | | in place; `clear()` keeps the room the list had |
| `join(separator)` | `String` | every element becomes text: a `String`, a number, a `Boolean`, an enum value |
| `copy()` / `deep_copy()` | `List<T>` | one level, or all the way down ([memory.md](memory.md)) |

`append`, `remove_last`, `list[index]` and their kin take the same time however long the list is; `prepend`,
`insert`, `remove_first` and `remove_at` move every later element, and `contains` looks at each in turn. An empty
list has no buffer; the first element gets one of four slots, and a full buffer is resized to twice its size.

Names say where: `append` and `prepend`, never `add`; `remove_last`, never `pop`
([the rule](https://github.com/Spite-Language/spite_spec/blob/master/specs/collections.md#listt-additions)).

A list has the same members whatever it holds. Bytes from a file or a socket, and the numbers and text laid out in
them, are a `Buffer`, which still reads like a list of bytes and adds what bytes need
([standard_library.md](standard_library.md#bytes-are-a-buffer)).

## `Dictionary<Key, Value>`

A dictionary names two types: what it is keyed by, then what it holds. `Dictionary<String, Integer>` finds a count
by a name, `Dictionary<Integer, Monster>` a monster by its id, and `Dictionary<Player, Party>` a party by the player
itself. Entries keep the order they were inserted in, and `contains` and `[]` take the same time however
many keys there are, because it is a hash table over two lists. `remove_at` is the exception: it moves every later
entry down, as `remove_at` does on a list.

| Member | Result | Notes |
|---|---|---|
| `Dictionary<Key, Value>()` | | an empty one, with no table until the first key |
| `dictionary[key] = value` | | replaces the value of a key already there; it calls `set_at(key, value)`, which is only called through `[] =` |
| `dictionary[key]` | `Value?` | `null` when the key is absent; it calls `get_at(key)`, which is only called through `[]` |
| `contains(key)` | `Boolean` | |
| `remove_at(key)` | | nothing happens when the key is absent |
| `count()` | `Integer` | |
| `keys()` / `values()` | `List<Key>` / `List<Value>` | a fresh list, in insertion order |
| `copy()` / `deep_copy()` | `Dictionary<Key, Value>` | |

```gdscript title=dictionary_tasks/dictionary_tasks.spite entry
var console = Console()

func DictionaryTasks() {
    load "../spite_standard_library/windows"
    var inventory = Dictionary<String, Integer>()
    inventory["sword"] = 1
    inventory["potion"] = 4
    inventory["potion"] = 6
    var inventory_count = inventory.count()
    console.print("count", inventory_count)
    var has_shield = inventory.contains("shield")
    console.print("has shield", has_shield)
    crash inventory["potion"]
    console.print("potions", inventory["potion"])
    console.print("shields", inventory["shield"] == 0)
    var total = 0
    var values = inventory.values()
    var index = 0
    while index < values.count() {
        total = total + values[index]
        index = index + 1
    }
    console.print("total items", total)
}
```
```output
count 2
has shield false
potions 6
shields false
total items 7
```

`inventory["shield"]` is `null`, and `null` equals nothing, so `inventory["shield"] == 0` is `false`: absent is
not zero. `contains(key)` asks the question directly.

A dictionary whose entries are known when it is written is a **dictionary literal**: each entry a key, a colon and
a value, apart with commas or one per line, as a list literal's items are (the formatter puts a short one on one
line). Its keys are text or whole numbers written out. Stored where a dictionary is declared, it takes the declared
key and value types; otherwise its keys are `String` or `Integer`, as its first key is, and its value type is the
first value's:

```gdscript title=dictionary_literals/dictionary_literals.spite entry
var console = Console()

func DictionaryLiterals() {
    load "../spite_standard_library/windows"
    var plurals = {"cactus": "cacti", "foot": "feet"}
    var squares = {1: 1, 2: 4, 3: 9}
    var weights: Dictionary<String, Float> = {"feather": 0.01, "anvil": 50}
    crash plurals["cactus"]
    crash squares[3]
    crash weights["anvil"]
    console.print(plurals["cactus"], squares[3], weights["anvil"])
}
```
```output
cacti 9 50
```

A key written twice is an error (`"cactus" is written twice in this dictionary: keep one entry for each key`), and
an empty dictionary is made with `Dictionary<Key, Value>()`, since `{}` names no types.

### Any type is a key

The key type is what you look up by, and the dictionary finds a key the way that type means it:

- **numbers** by their value: `Dictionary<Integer, String>` stores and hashes the number itself, never text made
  from it, and `keys()` answers the numbers. A fraction is found by its value too, so `0.0` and `-0.0` are one key,
  and so is every `not_a_number`;
- **text** by its bytes, and a `Boolean` by whether it is `true`;
- **an enum value** as itself: `Dictionary<Color, Integer>` holds one entry for `'red'`, whatever its name is
  written as;
- **an object** as that one object: two players with the same name are two keys, and the dictionary holds each
  key, so an object cannot be freed while it is a key of one.

```gdscript title=number_keys_tasks/number_keys_tasks.spite entry
var console = Console()
var names = Dictionary<Integer, String>()

func NumberKeysTasks() {
    load "../spite_standard_library/windows"
    names[3] = "fern"
    names[11] = "moss"
    names[7] = "reed"
    names.remove_at(11)
    crash names[7]
    console.print("seven", names[7])
    var ids = names.keys()
    crash ids[0]
    var first_id = ids[0] + 100
    var joined_ids = ids.join(",")
    console.print("ids", joined_ids, "first plus 100", first_id)
}
```
```output
seven reed
ids 3,7 first plus 100 103
```

```gdscript title=party_keys/player.spite
var name = ""

func Player(new_name: String) {
    name = new_name
}
```
```gdscript title=party_keys/party_keys.spite entry
var console = Console()

func PartyKeys() {
    load "../spite_standard_library/windows"
    var ada = Player("ada")
    var other_ada = Player("ada")
    var parties = Dictionary<Player, String>()
    parties[ada] = "red team"
    crash parties[ada]
    var other_found = parties.contains(other_ada)
    console.print(ada.name, "plays for", parties[ada], "and the other ada is found:", other_found)
}
```
```output
ada plays for red team and the other ada is found: false
```

A class that declares its own `equals` says two of its objects can be equal without being one object, and a
dictionary keyed by it would miss the equal one, so it is a compile error: key the dictionary by what `equals`
compares instead, an id or a name. A key given where its type is not wanted is the same mismatch as any argument,
and the [specification](https://github.com/Spite-Language/spite_spec/blob/master/specs/collections.md#dictionarykey-value) lists every rule and error.

The key and value types are what the program means, not how it is stored: the compiler keeps a dictionary however
is fastest, as long as every lookup answers the same ([write it plainly](write_it_plainly.md#a-class-is-what-you-mean-not-how-it-is-stored)).

## One list

`List<T>` is the only list: there is no `Vector` and no `Items` to choose between. You say what you keep, and
how it is stored is the compiler's business. Numbers, `Boolean`, enums and `Memory.Address` already sit flat in
the list's buffer, and a list whose items are never held anywhere else may keep them inline, one after another
the way the processor's cache likes to read them ([spite_compiler#205](https://github.com/Spite-Language/spite_compiler/issues/205)).
Nothing a program can see changes with that choice: an item read from a list is the item itself, so a change
made through one name shows through every other.

```gdscript title=one_list/velocity.spite
var across = 0.0
var down = 0.0

func Velocity(new_across: Float, new_down: Float) {
    across = new_across
    down = new_down
}

func integrate() {
    across = across + down
}
```
```gdscript title=one_list/one_list.spite entry
var console = Console()

func OneList() {
    load "../spite_standard_library/windows"
    var velocities = List<Velocity>()
    var velocity = Velocity(1.0, 0.5)
    velocities.append(velocity)
    velocities.each_integrate()
    var total = velocities.sum_across()
    console.print(total, velocity.across)
}
```
```output
1.5 1.5
```

The rules are [in the specification](https://github.com/Spite-Language/spite_spec/blob/master/specs/collections.md#one-list).

## Removing many at once

`remove_where(test)` removes every element a function is true for, and `remove_where_<member>()` every element
whose member is true, in **one pass** that keeps the others in their order. `truncate(count)` keeps the first `count` and lets go of
the rest, and `swap(first, second)` exchanges two, which is what a pass of your own is written with when the test
is not a function of the element, such as an engine removing a column's rows by a mask of despawned entities:

```gdscript title=bulk_removal_doc/velocity.spite
var across = 0.0
var stopped = false

func Velocity(new_across: Float) {
    across = new_across
    stopped = new_across == 0.0
}
```
```gdscript title=bulk_removal_doc/bulk_removal_doc.spite entry
var console = Console()
var despawned = [false, true, false, true, false]

func BulkRemovalDoc() {
    load "../spite_standard_library/windows"
    var numbers = [4, 7, 10, 13, 16]
    numbers.remove_where(is_odd)
    numbers.truncate(2)
    var velocities = List<Velocity>()
    var entities = List<Integer>()
    var entity = 0
    while entity < 5 {
        var velocity = Velocity(1.0 * entity)
        velocities.append(velocity)
        entities.append(entity)
        entity = entity + 1
    }
    var moving = velocities.copy()
    moving.remove_where_stopped()
    var kept = 0
    var row = 0
    while row < velocities.count() {
        crash entities[row]
        var owner = entities[row]
        if not is_despawned(owner) {
            velocities.swap(row, kept)
            kept = kept + 1
        }
        row = row + 1
    }
    velocities.truncate(kept)
    entities.remove_where(is_despawned)
    var joined = numbers.join(", ")
    var across = velocities.sum_across()
    var rows = entities.count()
    var moving_count = moving.count()
    console.print(joined, moving_count, across, rows)
}

func is_odd(number: Integer): Boolean {
    return number % 2 == 1
}

func is_despawned(entity: Integer): Boolean {
    crash entity < despawned.count()
    return despawned[entity]
}
```
```output
4, 10 4 6 3
```

Each costs one walk over the collection, however many go: every element that stays is moved down at most once,
and the ones removed are released at the end ([its benchmark](https://github.com/Spite-Language/spite_benchmarks/tree/master/removing_many_at_once) measures it
against the same removal written in C). What stays keeps its order, which one `remove_swapping` per item does
not. A test that is not a function of the element alone (a mask read by row) is written as the loop above:
`swap` each row that stays down to the next free place, then `truncate`.

`remove_where` changes the collection it is called on, so it ends no chain: `names.filter(is_short).remove_where(is_long)`
is an error naming the fix, one test that says everything to remove. Like `remove_at`, all three change which
element an index names, so a proof about an element is made again after one ([the rules](https://github.com/Spite-Language/spite_spec/blob/master/specs/collections.md#removing-many-at-once)).

## Member templates: loops you do not write

A list of a class answers a family of functions named after the element's members (a function of your own is
passed instead, [below](#passing-a-function-for-each-element)). `chores.count_done()` counts
the chores whose `done` is true, `items.sum_price()` adds up their prices, and `repositories.map_names()` collects
their names. A **member** is an attribute or a function that takes no arguments (the two are the same to a
template, since reading an attribute already goes through its getter), and a template is compiled only for the
names a program calls. Text has members too: `names.map_upper_cases()` collects every name in capitals,
`names.filter_is_empty()` keeps the empty ones and `names.sort_by_length()` sorts them by length
(`conformance/stage6/text_member_templates`).

| Template | Result | The member must |
|---|---|---|
| `filter_<member>()` | `List<T>`, the elements where it is true | take nothing, return `Boolean` |
| `count_<member>()` | `Integer`, how many are true | take nothing, return `Boolean` |
| `any_<member>()` / `all_<member>()` | `Boolean` | take nothing, return `Boolean` |
| `sum_<member>()` | the member's number type | take nothing, return a number |
| `find_by_<member>(value)` | `T?`, the first element whose member equals `value` | return something comparable to `value` |
| `find_index_by_<member>(value)` | `Integer?`, the place of that element | return something comparable to `value` |
| `sort_by_<member>()` | `List<T>`, sorted ascending, stable, in `n log n` time | return a number or a `String` |
| `map_<members>()` | `List<U>`, one value per element | return a value |
| `each_<member>()` | nothing: calls it on every element | be a function that takes nothing; what it returns is discarded |
| `remove_where_<member>()` | nothing: removes the elements where it is true, keeping the rest in order | take nothing, return `Boolean` |

A member that does not fit is a compile error naming the member, what it is, and what the template needs.
`count_` on a number is one of them, and names `sum_` instead: `count()` is only ever a collection's size.
`each_` on an attribute is another: reading a value only to discard it does nothing. `map_` on a test is a third:
a `Boolean` member is a question about each element, never a value to collect, so `monsters.map_is_alive()` names
`filter_is_alive()`, `count_is_alive()`, `any_is_alive()` and `all_is_alive()` instead.

A value computed from each element is a member too. There is no `map(function)`: give the element's class a
read-only attribute that computes it (a `get_<name>()` with no setter,
[functions_and_operators.md](functions_and_operators.md#settergetter-interception)) and collect it with
`map_<name>()`. The value then has a name, every list of that class can collect it, and a chain fuses it like any
other member:

```gdscript title=computed_member/monster.spite
var health = 0

func Monster(starting_health: Integer) {
    health = starting_health
}

func get_doubled_health(): Integer {
    return health * 2
}
```
```gdscript title=computed_member/computed_member.spite entry
var console = Console()

func ComputedMember() {
    load "../spite_standard_library/windows"
    var monsters = [Monster(3), Monster(5)]
    var doubled = monsters.map_doubled_health()
    var joined = doubled.join(", ")
    console.print(joined)
}
```
```output
6, 10
```

A template costs what the loop you would have written costs: each one is a `while` over the list's buffer,
compiled into the program only when something calls it, with no function value or closure in between.
On a long list the compiler may split an `each_` walk across the thread pool
([concurrency.md](concurrency.md#each_-on-a-long-list-runs-in-bands)).

```gdscript title=list_helpers/chore.spite
var title = ""
var done = false

func Chore(new_title: String, new_done: Boolean) {
    title = new_title
    done = new_done
}

func finish() {
    done = true
}
```
```gdscript title=list_helpers/list_helpers.spite entry
var console = Console()

func ListHelpers() {
    load "../spite_standard_library/windows"
    var chores = List<Chore>()
    var write_documentation = Chore("write docs", false)
    chores.append(write_documentation)
    var ship_release = Chore("ship release", false)
    chores.append(ship_release)
    var rest = Chore("rest", true)
    chores.append(rest)
    var done_count = chores.count_done()
    console.print("done count", done_count)
    var any_done = chores.any_done()
    console.print("any done", any_done)
    var all_done = chores.all_done()
    console.print("all done", all_done)
    var titles = chores.map_titles()
    crash titles[0]
    console.print("first title", titles[0])
    chores.each_finish()
    all_done = chores.all_done()
    console.print("all done now", all_done)
}
```
```output
done count 1
any done true
all done false
first title write docs
all done now true
```

```gdscript title=list_query/item.spite
var name = ""
var price = 0
var in_stock = false

func Item(new_name: String, new_price: Integer, new_in_stock: Boolean) {
    name = new_name
    price = new_price
    in_stock = new_in_stock
}
```
```gdscript title=list_query/list_query.spite entry
var console = Console()

func ListQuery() {
    load "../spite_standard_library/windows"
    var items = List<Item>()
    var sword = Item("sword", 50, true)
    items.append(sword)
    var shield = Item("shield", 30, false)
    items.append(shield)
    var potion = Item("potion", 10, true)
    items.append(potion)
    var in_stock_count = items.filter_in_stock().count()
    console.print("in stock count", in_stock_count)
    var in_stock_price = items.filter_in_stock().sum_price()
    console.print("in stock price total", in_stock_price)
    var found = items.find_by_name("shield")
    if found {
        console.print("found", found.name, found.price)
    }
    var by_price = items.sort_by_price()
    var index = 0
    while index < by_price.count() {
        console.print(by_price[index].name, by_price[index].price)
        index = index + 1
    }
}
```
```output
in stock count 2
in stock price total 60
found shield 30
potion 10
shield 30
sword 50
```

`filter_<member>()` and `sort_by_<member>()` return a new list holding the same elements, each one referenced
once more and not copied. A `Dictionary` answers every template through its values: `inventory.sum_price()` is
`inventory.values().sum_price()`.

```gdscript title=template_mistake/repository.spite
var stars = 0
```
```gdscript title=template_mistake/template_mistake.spite entry error
var console = Console()

func TemplateMistake() {
    load "../spite_standard_library/windows"
    var repositories = List<Repository>()
    var starred = repositories.count_stars()
    console.print(starred)
}
```
```diagnostic
but 'count_' needs it to return Boolean (to add up a numeric member use 'sum_stars')
```

**A `while` that only does what a template does is an error naming the template**: a
counter walking a list from `0` to its `count()`, doing nothing with each element but what one template does
with one of its members, or what `each`, `filter`, `count`, `sum`, `find`, `any` or `all` does with a
function passed the element, on a list of anything, numbers and text included. Loops that need the index, pass
more than the element, stop early for another reason, or walk state keep their `while`. The exact shape the
compiler looks for is in [control_flow.md's rules](https://github.com/Spite-Language/spite_spec/blob/master/specs/control_flow.md#a-while-that-a-member-template-already-says).

```gdscript title=template_loop_error/item.spite
var price = 0

func Item(item_price: Integer) {
    price = item_price
}
```
```gdscript title=template_loop_error/template_loop_error.spite entry error
var console = Console()

func TemplateLoopError() {
    load "../spite_standard_library/windows"
    var items = [Item(3), Item(5)]
    var total = 0
    var index = 0
    while index < items.count() {
        total = total + items[index].price
        index = index + 1
    }
    console.print(total)
}
```
```diagnostic
this 'while' walks every element of 'items' only to add up 'price': write 'var total = items.sum_price()'
```

### A member's own function in a template's name

A member template can go one step further and ask each element's member a question of its own:
`filter_<member>_<function>(arguments)` keeps the elements whose `member` answers `function(arguments)`. The
arguments are handed on to that function, so `players.filter_name_starts_with("a")` keeps the players whose
`name` starts with `a`. It works with every member template and every function the member's class has:

```gdscript title=chained_members/player.spite
var name = ""
var score = 0

func Player(starting_name: String, starting_score: Integer) {
    name = starting_name
    score = starting_score
}
```
```gdscript title=chained_members/chained_members.spite entry
var console = Console()

func ChainedMembers() {
    load "../spite_standard_library/windows"
    var players = List<Player>()
    var anna = Player("anna", 3)
    players.append(anna)
    var bruno = Player("bruno", 5)
    players.append(bruno)
    var alex = Player("alex", 7)
    players.append(alex)
    var starting_with_a = players.filter_name_starts_with("a")
    var names = starting_with_a.map_names()
    var joined = names.join(", ")
    var with_n = players.count_name_contains("n")
    var best = players.sum_score()
    console.print(joined, with_n, best)
}
```
```output
anna, alex 2 15
```

## Member templates over an enum value

When the element has a member typed with a named enum, the
templates that ask a yes-or-no question also take one of that enum's values as their name: `filter_<value>()` keeps
the elements whose member is that value. A listing does not need one helper per kind of thing it lists; it is
filtered by its kind:

```gdscript title=post_stages/post.spite
enum Stage {
    'draft'
    'published'
    'archived'
}

var title = ""
var stage: Stage = 'draft'

func Post(new_title: String, new_stage: Stage) {
    title = new_title
    stage = new_stage
}
```
```gdscript title=post_stages/post_stages.spite entry
var console = Console()
var posts = List<Post>()

func PostStages() {
    load "../spite_standard_library/windows"
    var hello = Post("hello", 'published')
    posts.append(hello)
    var plans = Post("plans", 'draft')
    posts.append(plans)
    var old = Post("old", 'archived')
    posts.append(old)
    var published = posts.filter_published()
    var titles = published.map_titles()
    var shown = titles.join(", ")
    var drafts = posts.count_draft()
    var anything_archived = posts.any_archived()
    posts.remove_where_archived()
    var left = posts.count()
    console.print(shown, drafts, anything_archived, left)
}
```
```output
hello 1 true 2
```

`posts.filter_published()` means exactly `posts.filter_stage_is_published()` would, had `Post` a
`func stage_is_published(): Boolean { return stage == 'published' }`: one loop, one comparison of two small
integers per element, and a chain of them fuses like any other ([below](#chains-run-as-one-loop)). A directory
lists only its files the other way, by class: `directory.entries()` is a list of the union of `Directory` and `File`,
and `filter_files()` keeps the `File` items as a `List<File>` ([metaprogramming.md](metaprogramming.md#member-templates),
[standard_library.md](standard_library.md#list-a-directory)).

## Passing a function for each element

A template sees only the element and the list: `names.each_say_hello()` looks for a member `say_hello` of each
name, never for a function of the class the call is written in. A function of yours is passed as a value
instead: `names.each(say_hello)` calls `say_hello` once for every name, in order, and there is no
`say_hello_to_everyone` to write. The function is bound to whoever owns it ([functions_and_operators.md](functions_and_operators.md#functions-are-values)):
`say_hello` alone is this instance's, and `people.each(greeter.greet)` calls `greet` on `greeter`. The library's
classes are no different: `keys.filter(counts.has)` asks the dictionary `counts`, and
`words.filter(greeting.contains)` asks the text `greeting`.

`each`, `filter`, `any`, `all`, `count`, `find`, `sort_by` and `sum` each take such a function, which takes
the element as its only argument: `filter`, `any`, `all`, `count` and `find` want one returning `Boolean` (`find`
answers the first element it is true for, or `null`), `sum` one returning a number, `sort_by` one returning a
number or text. There is no `map(function)`: a value collected from each element is a member of the element
([above](#member-templates-loops-you-do-not-write)), named and collected with `map_<member>()`. This works on a list of anything (numbers included, which
have no members of their own for a template to name) and on a `Dictionary`, through its values.

```gdscript title=passed_function/greeter.spite
var console = Console()

func greet(name: String) {
    console.print("welcome, {name}")
}
```
```gdscript title=passed_function/passed_function.spite entry
var console = Console()
var greeter = Greeter()

func PassedFunction() {
    load "../spite_standard_library/windows"
    var names = ["Ada", "Grace", "Barbara"]
    names.each(say_hello)
    names.filter(is_short).each(greeter.greet)
    var short_names = names.filter(is_short)
    var joined = short_names.join(", ")
    console.print("short:", joined)
    var letters = names.filter(is_short).sum(doubled_length)
    console.print("letters, doubled:", letters)
    var first_long = names.find(is_long)
    crash first_long
    console.print("first long name:", first_long)
}

func say_hello(name: String) {
    console.print("hello, {name}")
}

func is_short(name: String): Boolean {
    return name.length() < 6
}

func is_long(name: String): Boolean {
    return name.length() > 6
}

func doubled_length(name: String): Integer {
    return name.length() * 2
}
```
```output
hello, Ada
hello, Grace
hello, Barbara
welcome, Ada
welcome, Grace
short: Ada, Grace
letters, doubled: 16
first long name: Barbara
```

A function written by name is compiled straight into the loop, so passing it costs nothing; a function value held
in a variable (`var quiet = greeter.is_quiet`, then `names.filter(quiet)`) is called through the value. A function that
does not fit is an error naming what the template needs:

```gdscript title=passed_function_mistake/passed_function_mistake.spite entry error
var console = Console()

func PassedFunctionMistake() {
    load "../spite_standard_library/windows"
    var names = ["Ada", "Grace"]
    var greetings = names.filter(say_hello)
    console.print(greetings)
}

func say_hello(name: String) {
    console.print("hello, {name}")
}
```
```diagnostic
'filter(say_hello)': 'say_hello' returns nothing, but 'filter' needs it to return Boolean
```

The template still sees only the element: a function that needs more, such as `print_statement(statement, depth)`,
is called from a `while`.

A list's own functions are passed the same way, so `numbers.each(found.append)` copies every number into `found`,
and a chain of passed functions over a list of numbers is one loop over its buffer:

```gdscript title=plain_vector/plain_vector.spite entry
var console = Console()

func PlainVector() {
    load "../spite_standard_library/windows"
    var weights = List<Float>()
    weights.append(1.5)
    weights.append(2.5)
    weights.append(4.0)
    var found = List<Float>()
    weights.each(found.append)
    var heavy = weights.filter(is_heavy).sum(double_of)
    crash weights[0]
    var first = weights[0]
    weights.append(8.0)
    var joined = found.join(", ")
    console.print(joined, heavy, first)
}

func is_heavy(weight: Float): Boolean {
    return weight > 2.0
}

func double_of(weight: Float): Float {
    return weight * 2.0
}
```
```output
1.5, 2.5, 4 13 1.5
```

## Chains run as one loop

Every template takes what the one before it gives, so they chain: `map_<members>()` turns a list of teams into a
list of their leads, `filter_<member>()` keeps some of them, and anything else finishes the chain. Read a chain
as the separate steps it is written as (that is what it means), but the compiler runs it as **one loop over
the first list**, with no list in between: `teams.filter_active().map_leads().sum_age()` visits each team once,
reads its lead, and adds the age, allocating nothing.

```gdscript title=fused_chain/person.spite
var age = 0

func Person(new_age: Integer) {
    age = new_age
}
```
```gdscript title=fused_chain/team.spite
var active = false
var lead = Person(0)

func Team(new_active: Boolean, new_lead: Person) {
    active = new_active
    lead = new_lead
}
```
```gdscript title=fused_chain/fused_chain.spite entry
var console = Console()

func FusedChain() {
    load "../spite_standard_library/windows"
    var teams = List<Team>()
    var ann = Person(34)
    var ann_team = Team(true, ann)
    teams.append(ann_team)
    var bob = Person(67)
    var bob_team = Team(false, bob)
    teams.append(bob_team)
    var ages = teams.filter_active().map_leads().sum_age()
    var active = teams.filter_active()
    var leads = active.map_leads()
    var ages_step_by_step = leads.sum_age()
    console.print(ages, ages_step_by_step)
}
```
```output
34 34
```

Only a call made directly on another template call is part of the chain: `active.map_leads()` above starts a new
one, because `active` is a list the program named and kept. The steps in the middle are `map_` (to a member that
is a class) and `filter_`; the last call can be any template, and a chain the compiler cannot write as one loop
is run step by step, which means the same. The one visible difference is order: a member function in a fused
chain runs element by element, where the steps written out would run it on every element before the next step
starts. Written as three steps, the chain above would make two lists every time it ran; fused, it makes none
([the rules](https://github.com/Spite-Language/spite_spec/blob/master/specs/collections.md#standard-library-metaprogramming) name the test that pins the count).

## How the member templates are written

The templates are ordinary Spite in `library/list.spite`, over the list's own buffer on the heap. Each one is a
template ([metaprogramming.md](metaprogramming.md#member-templates)) whose parameter is
`member: Spite.AttributeDeclaration`: it names a member of the *element*, because the body reads it on an item, and
`item[member]` reads it: the field itself, or a call to the zero-argument function. `filter_member` answers `filter_in_stock`,
`filter_is_popular` and every other `filter_<member>` call:

```gdscript
func filter_member(member: Spite.AttributeDeclaration): List<$element_type> {
    var filtered = List<$element_type>()
    var index = 0
    while index < item_count {
        var item = values.read_value(items, index)
        if item.attributes[member] {
            filtered.append(item)
        }
        index = index + 1
    }
    return filtered
}
```

The same template answers a passed function. For `names.each(say_hello)` the compiler compiles `each_member`
once more with the function's owner passed in beside the list, and `item.attributes[member]` reads as
`say_hello(item)` on that owner (or as a call through the value, for a function held in a variable), so
`filter_member` also answers `filter(is_short)` without being written twice. A chain that passes functions is still
one loop.

`_read_value(index)` reads the element in slot `index` of the list's buffer, retained, the same way a container of
your own would ([memory.md](memory.md#memory-belongs-to-the-compiler)); everything else is written in the file. A
build with `--repl` or `--repl-port` compiles every template that fits every element class of a list the loop
can reach, so `monsters.sum_health()` can be typed at the prompt; that is the one build where templates nobody
calls are in the program. A private member (`_name`) of the element gets none: a template is
`List`'s code, and a private name is read only inside its own class
([classes_and_files.md](classes_and_files.md#private-names)), so a list whose element class has `_join()` has no `each__join`
in any build.

A `Dictionary`'s templates are written the same way, in `library/dictionary.spite`: their parameter is
`member: Spite.AttributeDeclaration`, read on a `$value_type`, and each walks a copy of the values, `var listed = values()`,
so `inventory.sum_price()` is `inventory.values().sum_price()` and a member that changes the dictionary while the
template runs changes nothing the walk sees. There is no `remove_where_` among them, since removing from that copy
would remove nothing.

## Write your own member template

A program reopens `List` by putting a `list.spite` in its own folder, and a function there with a
`Spite.AttributeDeclaration` parameter named after a word of its name, read on an item (`item[member]`), becomes one
more template, exactly like the library's:

```gdscript title=list_average/list.spite
func average_member(member: Spite.AttributeDeclaration): Float? {
    var item_count = count()
    fallback item_count != 0
    var total = 0.0
    var index = 0
    while index < item_count {
        var item = _read_value(index)
        total = total + item[member]
        index = index + 1
    }
    return total / item_count
}
```
```gdscript title=list_average/score.spite
var points = 0

func Score(new_points: Integer) {
    points = new_points
}
```
```gdscript title=list_average/list_average.spite entry
var console = Console()

func ListAverage() {
    load "../spite_standard_library/windows"
    var scores = List<Score>()
    var three = Score(3)
    scores.append(three)
    var four = Score(4)
    scores.append(four)
    var average = scores.average_points()
    crash average
    console.print(average)
}
```
```output
3.5
```

What makes `average_member` a template over the element is its body: it reads `member` on an item, `item[member]`, and the item
is a `$element_type`. A parameter read on the list's own attributes (`attributes[member]`) would range over `List`'s own
members, which are its buffer, and answer nothing. A body that reads the parameter on nothing says so:

```gdscript title=unused_member_template/list.spite
func total_member(member: Spite.AttributeDeclaration): Integer {
    return 0
}
```
```gdscript title=unused_member_template/score.spite
var points = 0

func Score(new_points: Integer) {
    points = new_points
}
```
```gdscript title=unused_member_template/unused_member_template.spite entry error
var console = Console()

func UnusedMemberTemplate() {
    load "../spite_standard_library/windows"
    var scores = List<Score>()
    var three = Score(3)
    scores.append(three)
    var total = scores.total_points()
    console.print(total)
}
```
```diagnostic
'total_member' is a template over the attributes of a class, because 'member' is a word of its name, but its body reads 'member' on no value, so nothing says which class
```

---

Next: [JSON and binary](json.md), the classes that turn any value into JSON or bytes and back.
