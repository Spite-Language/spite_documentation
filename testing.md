# Testing

A test is a function that crashes. Its name is `test` or starts with `test_`, it takes nothing, and `crash` is the
assertion: when a test is wrong, it halts with the condition, the values of its operands and the file and line
([failure.md](failure.md#what-a-crash-reports)), which is everything needed to fix it. There is no `expect`, no
matcher and nothing to register.

```gdscript
func test_append_and_prepend_keep_order() {
    var numbers = [2, 3]
    numbers.append(4)
    numbers.prepend(1)
    crash numbers.count() == 4
    crash numbers[0] == 1
    crash numbers.last() == 4
}
```

## A test program runs `Test`

A project keeps one entry point, and its tests are a program of their own beside it that `load`s the project. The
tests sit in classes of a namespace (a folder), and the entry runs them with the library's `Test` class:
`Test(namespace)` takes the namespace and `run()` runs every test in it.

```gdscript title=test_package/checks/text_tests.spite
func test_trim_removes_the_spaces_around_text() {
    var trimmed = "  spite  ".trim()
    crash trimmed == "spite"
}

func test_split_keeps_empty_pieces() {
    var pieces = "a,,b".split(",")
    crash pieces.count() == 3
}
```
```gdscript title=test_package/test_package.spite entry
func TestPackage() {
    load "../spite_standard_library/windows"
    var tester = Test(Checks)
    tester.run()
}
```
```output
pass Checks.TextTests.test_trim_removes_the_spaces_around_text
pass Checks.TextTests.test_split_keeps_empty_pieces
2 tests, 2 passed, 0 failed
```

`run()` walks the namespace's classes, then every namespace below it, and in each class every function named
`test` or `test_...`, in the order they are declared. For each test it **builds the class with its constructor**, so
a class's constructor is the setup its tests share: whatever it makes, each test starts from it. Then it calls the
test. A test program is never part of the program it tests, so tests take no room in what a hot reload rebuilds.

```gdscript title=test_setup/bank/account.spite
var balance = 0

func deposit(amount: Integer) {
    balance = balance + amount
}

func withdraw(amount: Integer) {
    balance = balance - amount
}
```
```gdscript title=test_setup/bank/account_tests.spite
var account = Bank.Account()

func AccountTests() {
    account.deposit(100)
}

func test_withdraws() {
    account.withdraw(30)
    crash account.balance == 70
}

func test_starts_from_its_own_setup() {
    crash account.balance == 100
}
```
```gdscript title=test_setup/test_setup.spite entry
func TestSetup() {
    load "../spite_standard_library/windows"
    var tester = Test(Bank)
    tester.run()
}
```
```output
pass Bank.AccountTests.test_withdraws
pass Bank.AccountTests.test_starts_from_its_own_setup
2 tests, 2 passed, 0 failed
```

## Each test runs on its own

`run()` runs each test in a run of its own: it starts the same program again with the setting `SPITE_TEST` naming
one test (`Bank.AccountTests.test_withdraws`), and that run builds the class and calls only that test. So no test
sees what another left behind, and a failing test does not stop the others. Under each test's line comes whatever
its run printed, indented. When every test passes, the program exits with status 0; when any fails, the last line
counts it and the program exits with status 1.

**When a test fails**, its `crash` halts its own run, and the report it writes is shown under its line. Break
`test_split_keeps_empty_pieces` by expecting two pieces, and the run says:

```
pass Checks.TextTests.test_trim_removes_the_spaces_around_text
fail Checks.TextTests.test_split_keeps_empty_pieces
    spite: the program stopped at test_package/checks/text_tests.spite:8 in Checks.TextTests.test_split_keeps_empty_pieces, site 4205b4ca (pieces.count()=3)
    spite.frame	test_package/checks/text_tests.spite:8	Checks.TextTests	test_split_keeps_empty_pieces
    spite.frame	test_package/test_package.spite:4	TestPackage	TestPackage
2 tests, 1 passed, 1 failed
```

The report is enough: the file and line of the `crash` (`text_tests.spite:8`), the class and the test, and the value
of every part of the condition ([what a crash reports](failure.md#what-a-crash-reports)). There is no
expected-versus-actual prose: the test says `pieces.count() == 2`, the report says `pieces.count()=3`, and the line
is open in front of you.

**One test alone** is the same program with the setting given: `SPITE_TEST=Checks.TextTests.test_split_keeps_empty_pieces`
in the environment runs that test in place, with no report line and no count, so a failure is the bare crash
report and the status is the test's own. That is the loop for fixing a failure: open the line the report names,
fix the code or the test, run that test alone until it passes, then run them all, since a fix can break another
test. To look around inside a test while it runs, build the test program with `--hot-reload --repl-port=4000` and
set a breakpoint on the test's line ([repl.md](repl.md#breakpoints)).

**The compile in front of the run is part of the test**: a test that cannot compile reports `path:line: error: ...`
and nothing runs.

## Why tests that crash are the best tests

- **A test stops at its first broken fact.** Nothing in that test runs on top of a value already known to be
  wrong, so a failure never cascades into forty more lines that only repeat it, and each failing test hands you
  exactly one line to open.
- **There is nothing to write but the fact.** No `expect(...).toEqual(...)`, no matcher vocabulary, no message to
  keep true: `crash pieces.count() == 3` is the test, and the compiler writes the report from it.
- **The report is the memory, not a story about it.** It points at the line and shows the values there. A model
  reads `text_tests.spite:8 ... pieces.count()=3` and opens the line, which is cheaper and more exact than reading a
  paragraph of assertion output, and nothing in it was written by a person who might have been wrong.
- **The same `crash` guards production code.** A `crash` in a test and a `crash` in the program are one statement
  with one report, so every guard the program already has is a test the moment a test reaches it, and a habit
  learned in one place is right in the other.
- **What the compiler can prove is not a test at all.** A null not checked, a `switch` missing a value, a type that
  does not fit, a `crash` whose condition is known while compiling: each is a compile error before any test runs
  ([proofs.md](proofs.md)). Tests are left for what only running can show.

---

Next: [Optimisations the compiler makes on its own](optimizations.md), what the compiler does to make a program
faster and smaller without being asked.
