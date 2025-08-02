# test-runner
a compact, single-dependency test runner that keeps it simple.

the strengths of this library lie in its syntactic expressiveness and compact reporting format.

* tests defined as arrays of alternating inputs and expected outputs
* supports passing named functions, context-bound methods, and labeled tests
* customizable test evaluation and output by subclassing

the reference implementation targets node.js and can also be used to test shell applications.

# reporting
the default format counts the test cases for each group.
```
ip 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38
gp 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23
gp_r3 0
ep 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15
id_bit_indices 0
reverse 0 1 2 3 4 5
involute 0 1 2
conjugate 0 1 2
```

on success:
```
double 0 1 2
```

on failure:
```
failure double 1
  inp 3
  exp 6
  out 5
```

reporting formats are customizable.

# usage
test_runner.js is the code.

## syntax
each test case is an array of the form:
* `[f, input1, expected1, ...]`
* `[["name", f], input1, expected1, ...]`
* `[[context, f], input1, expected1, ...]`
inputs can be:
* values: `f(1)` -> `1`
* arrays: `f(1, 2)` -> `[1, 2]`
output is compared using `json.stringify(...)` unless overridden.

## basic usage
the examples use coffeescript javascript syntax.

test pre-defined functions using `[function, input1, expected1, input2, expected2, ...]`
```coffeescript
{test_runner_class} = require "./test_runner"
double = (x) -> 2 * x
runner = new test_runner_class()
runner.execute [[double, 1, 2, 3, 6]]
```

## multi-argument functions
wrap input arguments in an array to pass multiple parameters
```coffeescript
add = (a, b) -> a + b
runner.execute [[add, [1, 2], 3, [4, 5], 9]]
```

## labeled tests
use `[name, function]` to custom label a test case
```coffeescript
square = (x) -> x * x
runner.execute [[["square test", square], 2, 4, 3, 9]]
```

## functions with bound context
use `[context, function]` to bind `this`
```coffeescript
obj =
  factor: 10
  scale: (x) -> x * @factor
runner.execute [[[obj, obj.scale], 2, 20]]
```

# testing shell applications
~~~
cp = require "child_process"
{test_runner_class} = require "./test_runner"

run_echo = (msg) ->
  cp.execFileSync("sh", ["-c", "echo #{msg}"]).toString().trim()

runner = new test_runner_class()
runner.execute [[run_echo, "hello", "hello", "123", "123"]]
~~~

## customizing comparison
subclass `test_runner_class` to change comparison logic
```coffeescript
class approx_runner_class extends test_runner_class
  constructor: (eps = 1e-6) ->
    super()
    @eps = eps
  equal: (a, b) ->
    if typeof a is "number" and typeof b is "number"
      math.abs(a - b) < @eps
    else
      json.stringify(a) is json.stringify(b)
  execute_tests: (tests) ->
    for [f, rest...] in tests
      name = if array.isarray(f) then f[0] else f.name
      context = if array.isarray(f) and typeof f[0] isnt "string" then f[0] else null
      results = [name]
      for i in [0...rest.length] by 2
        inp = if array.isarray(rest[i]) then rest[i] else [rest[i]]
        exp = rest[i + 1]
        out = f.apply context, inp
        ok = @equal out, exp
        results.push [ok, i / 2, name, inp, exp, out]
        break unless ok
      @options.reporter results
sqrt = math.sqrt
runner = new approx_runner_class(1e-5)
runner.execute [[sqrt, 2, math.sqrt2, 3, 1.7320508]]
```

## customizing output
override `report_compact` to format output differently
```coffeescript
class verbose_runner_class extends test_runner_class
  report_compact: (results) ->
    for [name, test_results...] in results
      console.log "== #{name} =="
      for [status, index, _, inp, exp, out] in test_results
        if status
          console.log "✓ test #{index}: passed"
        else
          console.log "✗ test #{index}: failed"
          console.log "  input:    #{json.stringify inp}"
          console.log "  expected: #{json.stringify exp}"
          console.log "  got:      #{json.stringify out}"
runner = new verbose_runner_class()
runner.execute [[double, 1, 2, 3, 7]]
```

# possible enhancements
* port to other languages

# see also
* [sc-lib/test.h](https://github.com/sph-mn/sph-sc-lib?tab=readme-ov-file#test) convenient testing for c with just a few tiny macros.
* [lib/test](https://sph.mn/computer/software/sph-lib/test.html) the guile module test-runner was inspired by