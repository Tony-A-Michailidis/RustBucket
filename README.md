# Rust Bucket

A small programming language where every keyword comes from the garage, with a
built-in OBD-II scan tool and a garage of simulated cars to point it at.

Yes, the name is close to one that's taken. This one has more actual rust.

The whole thing is a single self-contained HTML file: tokenizer, parser,
interpreter, playground UI and car simulator, no build step and no
dependencies. Open it in a browser and press Run.

```
plug "commuter"

spec codes = scan()
lap i from 1 to len(codes) {
  honk codes[i - 1] + "  " + explain(codes[i - 1])
}
```

```
P0420  Catalyst system efficiency below threshold (bank 1)
P0171  System too lean (bank 1)
```

## Running it

Open `rust-bucket-playground.html` in any modern browser. That's the entire
install. Nothing is sent anywhere, and the page works offline apart from the
Google Fonts link in the `<head>`.

Pick an example from the chips along the top, edit the source on the left, and
press **Run** or **Ctrl/Cmd+Enter**. Output appears on the right, errors come
back with a line number.

## Why the keywords are what they are

The design rule was that a keyword only earns its place if the car meaning
explains the code meaning. `shift` changes the value you're in, the way you
change the gear you're in. A `lap` is a counted trip around a circuit, which is
exactly a for-loop. You `brake` to stop early, and you `swerve` when the road
ahead is blocked. Nothing is themed for the sake of it: `spec` and `build` would
read fine in a language with no jokes in it.

## Language reference

### Statements

| Rust Bucket | Meaning |
| --- | --- |
| `spec x = 5` | Declare a value in the current scope |
| `shift x = 6` | Reassign an existing value. Also `shift xs[0] = 6` |
| `honk expr` | Print a line |
| `check c { } swerve { }` | If / else. Chain with `swerve check` |
| `lap i from 1 to 5 { }` | Counted loop, both ends inclusive |
| `cruise c { }` | While loop |
| `brake` | Leave the enclosing loop |
| `build f(a, b) { }` | Define a function |
| `deliver expr` | Return from a build |
| `plug "commuter"` | Connect the scan tool to a simulated car |

### Values

| Form | Notes |
| --- | --- |
| `42`, `3.14` | Numbers are doubles |
| `"text"` | Strings. `\n` is the only escape |
| `green`, `red` | True and false |
| `stalled` | No value |
| `[3.36, 2.07, 1.43]` | A convoy (list). Zero-indexed: `gears[0]` |

### Operators

Arithmetic `+ - * / % ^`, where `^` is exponentiation and binds tighter than
`*`. Comparison `== != < > <= >=`. Logic spelled out: `and`, `or`, `not`, with
short-circuit evaluation.

`+` concatenates when either side is a string, and joins two convoys. Division
or modulo by zero is an error rather than infinity.

Truthiness: `red`, `stalled`, `0`, `""` and the empty convoy are false.
Everything else is true.

Comments run from `//` to the end of the line.

### Built-in functions

Maths and text:

| Function | Does |
| --- | --- |
| `sqrt(x)`, `cbrt(x)` | Square and cube roots |
| `abs(x)`, `floor(x)` | Magnitude, round down |
| `min(a, b)`, `max(a, b)` | Smaller, larger |
| `round(x)`, `round(x, n)` | Round, optionally to n decimals |
| `len(xs)` | Length of a convoy or string |
| `add(xs, v)` | Append to a convoy, returns it |
| `text(v)` | Render any value as a string |
| `pad(v, width)` | Right-pad so columns line up |

### Grammar

```ebnf
program    = { statement } ;
statement  = "spec" NAME "=" expr
           | "shift" postfix "=" expr
           | "honk" expr
           | "plug" expr
           | "check" expr block [ "swerve" ( block | check ) ]
           | "lap" NAME "from" expr "to" expr block
           | "cruise" expr block
           | "build" NAME "(" [ NAME { "," NAME } ] ")" block
           | "deliver" [ expr ]
           | "brake"
           | expr ;
block      = "{" { statement } "}" ;
expr       = or ;
or         = and { "or" and } ;
and        = equality { "and" equality } ;
equality   = compare { ( "==" | "!=" ) compare } ;
compare    = term { ( "<" | ">" | "<=" | ">=" ) term } ;
term       = factor { ( "+" | "-" ) factor } ;
factor     = unary { ( "*" | "/" | "%" ) unary } ;
unary      = ( "-" | "not" ) unary | power ;
power      = postfix [ "^" unary ] ;
postfix    = primary { "(" [ args ] ")" | "[" expr "]" } ;
primary    = NUMBER | STRING | NAME | "green" | "red" | "stalled"
           | "(" expr ")" | "[" [ args ] "]" ;
```

Statements have no terminator. Newlines are whitespace, so the grammar is
unambiguous without them, and a statement can span as many lines as it likes.

## The OBD-II layer

Every car sold in the US since 1996, and in Europe since 2001 (petrol) or 2004
(diesel), exposes the same diagnostic interface. Rust Bucket models it. The
commands map onto real OBD-II modes so a program written here would port onto a
real adapter with the simulator swapped out underneath.

| Function | OBD mode | Does |
| --- | --- | --- |
| `plug "name"` | — | Connect to a car in the simulated garage |
| `scan()` | 03 | Convoy of stored diagnostic trouble codes |
| `pending()` | 07 | Codes seen once but not yet confirmed |
| `clear()` | 04 | Erase codes, lamp off. Returns how many went |
| `mil()` | 01 | Is the malfunction indicator lamp on? |
| `sensor("rpm")` | 01 | A live reading. Accepts a PID too: `sensor("0C")` |
| `sensors()` | — | Every sensor name |
| `unit("coolant")` | — | The unit string, for printing |
| `vin()` | 09 | Vehicle identification number |
| `vehicle()` | — | A readable name for the connected car |
| `explain("P0420")` | — | What a code means |
| `causes("P0420")` | — | Common causes, most likely first |
| `wait(seconds)` | — | Let simulated time pass |

### Sensors

| Name | PID | Unit |
| --- | --- | --- |
| `rpm` | 0C | rpm |
| `speed` | 0D | km/h |
| `coolant` | 05 | °C |
| `intake` | 0F | °C |
| `load` | 04 | % |
| `throttle` | 11 | % |
| `maf` | 10 | g/s |
| `stft` | 06 | % |
| `ltft` | 07 | % |
| `o2` | 14 | V |
| `timing` | 0E | ° BTDC |
| `voltage` | 42 | V |
| `fuel` | 2F | % |

### Trouble codes

`explain()` carries a table of roughly forty common generic codes. Anything
outside the table is still decoded structurally, because the code format itself
carries meaning:

```
P 0 3 01
│ │ │ └── the specific fault: cylinder 1
│ │ └──── subsystem: 3 is ignition and misfire
│ └────── who defined it: 0 generic, 1 manufacturer-specific
└──────── system: P powertrain, B body, C chassis, U network
```

So `explain("P1234")` returns "not in the table: a manufacturer-specific
powertrain code" rather than giving up.

### The garage

| `plug` name | Car | Fault |
| --- | --- | --- |
| `clean` | 2016 Mazda MX-5 | Nothing wrong. Your baseline |
| `commuter` | 2011 Honda Accord | Running lean, tired catalytic converter, EVAP leak pending |
| `misfire` | 2008 Subaru WRX | Cylinder 3 misfiring, rough idle |
| `overheat` | 2005 Ford F-150 | Starts clean, sets P0217 as the coolant climbs |
| `cold` | 2012 Toyota Corolla | Thermostat stuck open, never reaches temperature |

The simulation is deterministic per car (seeded PRNG) and models a few things
that make diagnosis feel real:

- **Engine state evolves.** Coolant follows an exponential warm-up curve, and
  idle speed drops as the engine warms. Every `sensor()` call advances the clock
  half a second; `wait()` advances it as far as you like.
- **Codes can appear over time.** The F-150 has no stored code at key-on. It
  goes pending past 102 °C and stores past 108 °C.
- **Codes come back after clearing.** Clear the Accord and the lamp goes out.
  Four minutes of simulated driving later the faults go pending, and after
  fourteen they're stored again. Clearing a code was never the repair.
- **Faults show up in the live data.** The misfiring WRX has an idle that won't
  sit still. The lean Accord runs positive fuel trims because the computer is
  adding fuel to make up for extra air.

## Worked examples

Ten programs ship in the playground, in two groups.

**Language:** quarter-mile estimator (Hale's formula), road speed in every gear
at redline, a pit-stop strategy loop, a fuel range calculation, and lap-time
analysis over a convoy.

**OBD-II:** read and explain stored codes, dump live data and compare fuel
trims across two cars, watch a cold engine warm past its limit, diagnose a
misfire from codes plus idle stability, and clear codes then watch them return.

## How it works

One file, about 1,200 lines, no dependencies.

- **Tokenizer.** Hand-written scanner. Numbers, strings, identifiers,
  keywords, two-character operators, `//` comments, line tracking for errors.
- **Parser.** Recursive descent producing a plain-object AST. Precedence
  climbing for binary operators, postfix loop for calls and indexing.
- **Interpreter.** Tree-walking evaluator over a chain of scope objects. Builds
  are closures capturing their defining scope. Returns unwind via a thrown
  sentinel, `brake` likewise. Builds are hoisted so a program can call one
  defined below it.
- **Guards.** Execution stops after 3,000,000 steps, so a bad `cruise`
  condition reports an error instead of hanging the tab. Output is capped at
  500 lines.
- **Errors** carry a line number and are phrased in the language's own
  vocabulary: `Cannot shift "fuel" — it was never specced`.

There is no bytecode, no optimizer and no type checker. It's a tree-walker, and
for programs that fit on a screen that's the right amount of machinery.

## Roadmap

- **Real hardware.** A cheap ELM327 adapter over the Web Serial API would let
  the same programs talk to an actual car. `scan()` becomes `03`, `sensor("rpm")`
  becomes `01 0C`. Nothing in the language needs to change, which was the point
  of shaping the commands around real modes.
- **Freeze frame data**, so you can see what the engine was doing at the moment
  a code set.
- **Readiness monitors**, which is the other half of why clearing codes before
  an inspection doesn't work.
- **A `tune` block** for adjusting a running value.
- **Syntax highlighting** in the editor.

## Contributing

Issues and pull requests welcome. Good first additions: more trouble codes in
the table, more cars in the garage, or another worked example. If you add a
keyword, it has to pass the rule at the top of this file.

## Licence

MIT 
