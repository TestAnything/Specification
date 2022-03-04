---
layout: default
title: TAP 14 specification
---

# STATUS: DRAFT

This is a draft document pending further editing and ratification by no
fewer than 3 widely used TAP implementations in 3 different language
communities.

This section will likely be removed prior to ratification.

## GOALS OF THIS SPECIFICATION

* Document the observed behavior of widely used TAP implementations.
* Add no features that are not already in wide usage across multiple
  implementations.
* Explicitly allow what is already allowed, deny what is already denied.
* Provide updated and clearer guidance for new TAP implementations.

# NAME

TAP14 - The Test Anything Protocol v14

## SYNOPSIS

TAP, the Test Anything Protocol, is a simple text-based interface between
testing modules a test harness. TAP started life as part of the test
harness for Perl but now has implementations in C/C++, Python, PHP, Perl,
JavaScript, and probably others by the time you read this.  This document
describes version 14 of TAP. Go to TAP to read about previous versions.

The key words _must_, _must not_, _required_, _shall_, _shall not_,
_should_, _should not_, _recommended_, _may_, and _optional_ in this
document are to be interpreted as described in [RFC
2119](https://datatracker.ietf.org/doc/html/rfc2119).

## TAP14 FORMAT

TAP14's general grammar is:

```ebnf
TAPDocument := Version Plan Body | Version Body Plan
Version     := "TAP version 14\n"
Plan        := "1.." (Number) (" # " Reason)? "\n"
Body        := (TestPoint | BailOut | Pragma | Comment | Anything | Empty)*
TestPoint   := ("not ")? "ok" (" " Number)? ((" -")? (" " Description) )? (" " Directive)? "\n" (YamlBlock)?
Directive   := "# " ("todo" | "skip") (" " Reason)?
YamlBlock   := "  ---\n" (YamlLine)* "  ...\n"
YamlLine    := "  " (Yaml)* "\n"
BailOut     := "Bail out!" (" " Reason)? "\n"
Reason      := [^\n]+
Pragma      := "pragma " [+-] PragmaKey "\n"
PragmaKey   := ([a-zA-Z0-9_-])+
Comment     := ^ (" ")* "#" [^\n]* "\n"
Empty       := [\s\t]* "\n"
Anything    := [^\n]+ "\n"
```

(Note that the above is intended as a rough "pseudocode" guidance for
humans.  It is not strict EBNF.)

The Version is the line `TAP version 14`.

The Body is a collection of lines representing a test set.

The Plan reports on the number of tests included in the Body.

For example, a test file's output might look like:

```tap
TAP version 14
1..4
ok 1 - Input file opened
not ok 2 - First line of the input valid
  ---
  message: 'First line invalid'
  severity: fail
  data:
    got: 'Flirble'
    expect: 'Fnible'
  ...
ok 3 - Read the rest of the file
not ok 4 - Summarized correctly # TODO Not written yet
  ---
  message: "Can't make summary yet"
  severity: todo
  ...
```

## HARNESS BEHAVIOR

In this document, the "harness" is any program analyzing TAP output.

Typically this will be a test framework's runner program, but may also be a
programmatic parser, parent test object, or test result reporter.

In a typical TAP implementation, tests are programs that ouput TAP data
according to this specification.  The "test harness" reads TAP output from
these programs, and handles it in some way.

A harness that is collecting output from test programs _should_ read and
interpret TAP from the process's standard output, not standard error.
(Handling of test standard error is implementation-specific.)

A Harness _should_ normalize line endings by replacing any instances of
`\r\n` or `\r` in the TAP document with `\n`.

A harness _should_ treat a test program as a failed test if:

* The TAP output lines indicate test failure, or
* The TAP output of the process is invalid in a way that is not
  recoverable, or
* The exit code of the test program is not 0 (including test programs
  killed by a fatal Unix signal).

If one or more test programs are considered failures, then a TAP Harness
_should_ indicate failure to the user in whatever means are appropriate.
For example, a command-line test runner might exit with a non-zero status
code, while a web-based continuous integration system might put a red
"FAILED" notice in the html output.

Note that some of the above guidance may not apply if the Harness is
interpreting TAP from a different sort of text stream, or for a purpose
other than actually running tests.  For example, a Harness may be used to
analyze the recorded output of past test runs and provide data about them.

## TAP DOCUMENT STRUCTURE

### Version

To indicate that this is TAP14 the first line _must_ be

```tap
TAP version 14
```

Harnesses _may_ interpret ostensibly
[TAP13](https://testanything.org/tap-version-13-specification.html) streams
as TAP14, as this specification is compatible with observed behavior of
existing TAP13 consumers and producers.  That is, they _may_ treat this as
a valid Version line while parsing TAP14:

```tap
TAP version 13
```

Harnesses _may_ treat any TAP stream lacking a version as a failed test.

### Plan

The Plan tells how many tests will be run, or how many tests have run. It's
a check that the test file hasn't stopped prematurely.

The Plan _must_ appear exactly once in the file, either before any Test
Point lines, or after all Test Point lines.  That is, if it appears after
any Test Point lines, there _must not_ be any Test Point lines following.

A Harness _must_ treat a TAP stream lacking a plan as a failed test.

The Plan specifies how many test points are to follow. For example,

```tap
1..10
```

means that either 10 test points will follow, or 10 test points are
believed to have been present in the stream already.

This is a safeguard against test data being truncated or damaged in some
other way, rendering the output unreliable.

The Plan lists the range of test point IDs that are expected in the TAP
stream.  It can also optionally contain a comment/reason prefixed by a `#`.

It's basic grammar is:

```ebnf
Plan := "1.." Number ("" | "# " Reason)
```

A plan line of `1..0` indicates that the test set was completely skipped;
no tests are expected to follow, and none should have come before.
Harnesses _should_ report on `1..0` test runs similarly to their handling
of `SKIP` Test Points, treating any comment in the Plan as the reason for
skipping.

```tap
1..0 # WWW::Mechanize not installed
```

Previous versions of TAP allowed plans to specify any two numbers, for
example, `5..8` to indicate that test points with IDs between 5 and 8 would
be run.  However, this is not widely supported.

Thus, TAP14 producers _must_ output a Plan starting with `1`.  TAP14
Harnesses _may_ allow plans starting with numbers other than 1, but if so,
they _must_ treat any Test Point IDs outside the plan range as a test
failure.

### Test Points

The core of TAP is the "Test Point". A test file prints one test point
executed. There must be at least one test point in TAP output. Each test
point comprises the following elements:

- Test Status: `ok` or `not ok`

    This tells whether the test point passed or failed. It must be at the
    beginning of the line. `/^not ok/` indicates a failed test point.
    `/^ok/` is a successful test point. This is the only mandatory part of
    the line.

    Note that unlike the Directives below, `ok` and `not ok` are
    case-sensitive.

- Test Point ID

    TAP expects the ok or not ok to be followed by an integer Test Point
    ID. If there is no number, the harness _must_ maintain its own counter
    until the script supplies test numbers again.

    For example, the following test output is acceptable:

    ```tap
    1..5
    not ok
    ok
    not ok
    ok
    ok
    ```

    and is equivalent to:

    ```tap
    1..5
    not ok 1
    ok 2
    not ok 3
    ok 4
    ok 5
    ```

    This test output is _not_ valid TAP:

    ```tap
    1..6
    not ok
    ok
    not ok
    ok
    ok
    ```

    has five tests. The sixth is missing. Test::Harness will generate

    ```
    FAILED tests 1, 3, 6
    Failed 3/6 tests, 50.00% okay
    ```

    Test Points _may_ be output in any order, but any Test Point ID
    provided _must_ be within the range described by the Plan.

    This is valid TAP:

    ```tap
    TAP version 14
    1..3
    ok 2
    ok 3
    ok 1
    ```

    This is not valid TAP:

    ```tap
    TAP version 14
    1..3
    ok 2
    ok 4
    ok 1
    ```

- Description

    Any text after the test number but before a `#` is the description of
    the test point.

    ```tap
    ok 42 - this is the description of the test
    ```

    Descriptions _should_ be separated from the Test Point Status and Test
    Point ID by the string `" - "`, in order to prevent confusing a numeric
    description with a Test Point ID.  However, the `" - "` separator is
    _optional_.

    Harnesses _should not_ consider a leading `" - "` to be a part of the
    description reported to a user.

- Directive

    The test point may include a directive, following a hash on the test
    line.  There are currently two directives allowed: `TODO` and `SKIP`.
    These are discussed below.

To summarize:

- Test Status: `ok`/`not ok` (required)
- Test number (recommended)
- Description (recommended, prefixed by `" - "`)
- Directive (only when necessary)


#### DIRECTIVES

Directives are special notes that follow a # on the Test Point line. Only
two are currently defined: `TODO` and `SKIP`. These two keywords are not
case-sensitive.

Harnesses _may_ support additional platform-specific directives.  Future
versions of this specification _may_ codify additional directives with
defined semantics.

Unrecognized directives _must_ be ignored, and treated as comments.

##### TODO tests

If the directive starts with `# TODO`, the test is counted as a todo test,
and the text after `TODO` is the explanation.

```tap
not ok 14 # TODO bend space and time
```

If the TODO has an explanation, it must be separated from TODO by a space.
These tests represent a feature to be implemented or a bug to be fixed and
act as something of an executable "things to do" list. They are not
expected to succeed.

Should a todo test point begin succeeding, the harness _may_ report it in
some way that indicates that whatever was supposed to be done has been, and
it should be promoted to a normal Test Point.

Harnesses _must not_ treat failing `TODO` test points as a test failure.

Harneses _should_ report `TODO` test points found as a list of items
needing work, if that is appropriate for their use case.

##### SKIP tests

If the directive starts with `# SKIP`, the test is counted as a todo test,
and the text after `SKIP` is the explanation.

```tap
ok 14 - mung the gums # SKIP leave gums unmunged for now
```

If the SKIP has an explanation, it must be separated from SKIP by a space.
These tests indicate that a test was not run, or if it was, that its
success or failure is being temporarily ignored.

Harnesses _must not_ treat failing `SKIP` test points as a test failure.

Harnesses _should_ report `SKIP` test points found as a list of items that
were not tested, if that is appropriate for their use case.

### YAML Diagnostics

If a Test Point is followed by a 2-space indented block beginning with
the line `  ---` and ending with the line `  ...`, separated from the Test
Point only by comments or whitespace, then the block of lines between these
markers will be interpreted as an inline YAML Diagnostic document.

The YAML encodes a data structure that provides information about the
preceding Test Point.

For example:

```tap
not ok 3 - Resolve address
  ---
  message: "Failed with error 'hostname peebles.example.com not found'"
  severity: fail
  found:
    hostname: 'peebles.example.com'
    address: ~
  wanted:
    hostname: 'peebles.example.com'
    address: '85.193.201.85'
  at:
    file: test/dns-resolve.c
    line: 142
  ...
```

Currently (March 2022) the data structure represented by a YAML Diagnostic
block has not been standardized.  TAP14 Harnesses _must_ allow any data
structures supported by their YAML parser implementation.

A future version of this specification _may_ provide guidance regarding
YAML Diagnostic fields in common usage.

### Comments

Lines outside of a YAML diagnostic block which begin with a `#` character
preceeded by zero or more characters of whitespace, are comments.

A Harness _may_ present these to the user, ignore them, or assign meaning
to certain comments.

A Harness _must not_ treat a test as a failure based on the presence of
comment lines.  That is, a Harness _must_ ignore any unrecognized comment
lines.

### Pragmas

A Pragma provides information to a Harness to control its behavior or
configure it in some way.  Each Pragma line represents a single boolean
switch which can be set to `true` or `false`.

The structure of a Pragma line is:

- `"pragma "`
- `+` (true) or `-` (false)
- key: The name of the field being enabled or disabled.  ASCII alphanumeric
  characters, `_`, and `-` are allowed.

For example:

```tap
TAP version 13
# tell the parser to bail out on any failures from now on
pragma +bail

# tell the parser to execute in strict mode, treating any invalid TAP
# line as a test failure.
pragma +strict

# turn off a feature we don't want to be active right now
pragma -bail
```

The meaning and availability of keys that may be set by Pragmas are
implementation-specific.

Harnesses _may_ choose to respond to Pragma lines, or ignore them.

Harnesses _must not_ treat unrecognized Pragma keys as a test failure, even
if they would normally treat invalid TAP as a test failure.  Harnesses
_may_ warn if a Pragma is unrecognized, or fail if the named pragma is
recognized, but cannot be set for some reason.

### Blank Lines

For the purposes of this specification, a "blank" line is any line
consisting exclusively of zero or more whitespace characters.

Blank lines within YAML blocks _must_ be preserved as part of the YAML
document, because line breaks have semantic meaning in YAML documents.  For
example, multiline folded scalar values use `\n\n` to denote line breaks.

Blank lines outside of YAML blocks _must_ be ignored by the Harness.

### Bail out!

As an emergency measure a test script can decide that further tests are
useless (e.g. missing dependencies) and testing should stop immediately. In
that case the test script prints the magic words

```tap
Bail out!
```

to standard output. Any message after these words must be displayed by the
interpreter as the reason why testing must be stopped, as in

```tap
Bail out! MySQL is not running.
```

The words "Bail out!" are case insensitive.

### Anything Else

Any line that is not a valid version, plan, test point, yaml diagnostic,
pragma, a blank line, or a bail out is invalid TAP.

A Harness _may_ silently ignore invalid TAP lines, pass them through to its
own stderr or stdout, or report them in some other fashion.  However, it
_should not_ treat invalid TAP lines as a test failure by default.

## EXAMPLES

All names, places, and events depicted in any example are wholly fictitious
and bear no resemblance to, connection with, or relation to any real
entity. Any such similarity is purely coincidental, unintentional, and
unintended.

### Common with explanation

The following TAP listing declares that six tests follow as well as
provides handy feedback as to what the test is about to do. All six tests
pass.

```tap
TAP version 14
1..6
#
# Create a new Board and Tile, then place
# the Tile onto the board.
#
ok 1 - The object isa Board
ok 2 - Board size is zero
ok 3 - The object isa Tile
ok 4 - Get possible places to put the Tile
ok 5 - Placing the tile produces no error
ok 6 - Board size is 1
```

### Unknown amount and failures

This hypothetical test program ensures that a handful of servers are online
and network-accessible. Because it retrieves the hypothetical servers from
a database, it doesn't know exactly how many servers it will need to ping.
Thus, the test count is declared at the bottom after all the test points
have run. Also, two of the tests fail. The YAML block following each
failure gives additional information about the failure that may be
displayed by the harness.

```tap
TAP version 14
ok 1 - retrieving servers from the database
# need to ping 6 servers
ok 2 - pinged diamond
ok 3 - pinged ruby
not ok 4 - pinged saphire
  ---
  message: 'hostname "saphire" unknown'
  severity: fail
  ...
ok 5 - pinged onyx
not ok 6 - pinged quartz
  ---
  message: 'timeout'
  severity: fail
  ...
ok 7 - pinged gold
1..7
```

### Giving up

This listing reports that a pile of tests are going to be run. However, the
first test fails, reportedly because a connection to the database could not
be established. The program decided that continuing was pointless and
exited.

```tap
TAP version 14
1..573
not ok 1 - database handle
Bail out! Couldn't connect to database.
```

### Skipping a few

The following listing plans on running 5 tests. However, our program
decided to not run tests 2 thru 5 at all. To properly report this, the
tests are marked as being skipped.

```tap
TAP version 14
1..5
ok 1 - approved operating system
# $^0 is solaris
ok 2 - # SKIP no /sys directory
ok 3 - # SKIP no /sys directory
ok 4 - # SKIP no /sys directory
ok 5 - # SKIP no /sys directory
```

### Skipping everything

This listing shows that the entire listing is a skip. No tests were run.

```tap
TAP version 14
1..0 # skip because English-to-French translator isn't installed
```

## Got spare tuits?

The following example reports that four tests are run and the last two
tests failed. However, because the failing tests are marked as things to do
later, they are considered successes. Thus, a harness should report this
entire listing as a success.

```tap
TAP version 14
1..4
ok 1 - Creating test program
ok 2 - Test program runs, no error
not ok 3 - infinite loop # TODO halting problem unsolved
not ok 4 - infinite loop 2 # TODO halting problem unsolved
```

## Creative liberties

This listing shows an alternate output where the test numbers aren't
provided. The test also reports the state of a ficticious board game as a
YAML block. Finally, the test count is reported at the end.

```tap
TAP version 14
ok - created Board
ok
ok
ok
ok
ok
ok
ok
  ---
  message: "Board layout"
  severity: comment
  dump:
     board:
       - '      16G         05C        '
       - '      G N C       C C G      '
       - '        G           C  +     '
       - '10C   01G         03C        '
       - 'R N G G A G       C C C      '
       - '  R     G           C  +     '
       - '      01G   17C   00C        '
       - '      G A G G N R R N R      '
       - '        G     R     G        '
  ...
ok - board has 7 tiles + starter tile
1..9
```

## BUGS

Feature requests and bug reports should be [raised on
GitHub.](https://github.com/TestAnything/Specification/issues/new)

## AUTHORS

The TAP 14 Specification is authored by [Isaac Z.
Schlueter](https://github.com/isaacs), as a result of much discussion on
the [TestAnything
Specification](https://github.com/TestAnything/Specification) project, with
considerable input, encouragement, feedback, and suggestions from:

* [Matt Layman](https://github.com/mblayman)
* [Leon Timmermans](https://github.com/Leont)
* [Bruno P. Kinoshita](https://github.com/kinow)
* [Chad Granum](https://github.com/exodist)

## ACKNOWLEDGEMENTS

The TAP 13 Specification was written by Andy Armstrong with help and
contributions from Pete Krawczyk, Paul Johnson, Ian Langworth and Nik
Clayton, based on the original TAP documentation by Andy Lester, based on
the original Test::Harness documentation by Michael Schwern.

The basis for the TAP format was created by Larry Wall in the original test
script for Perl 1. Tim Bunce and Andreas Koenig developed it further with
their modifications to Test::Harness.

## COPYRIGHT

Copyright 2015-2022 by Isaac Z. Schlueter <i@izs.me> and Contributors.

Copyright 2003-2007 by Michael G Schwern <schwern@pobox.com>, Andy Lester
<andy@petdance.com>, Andy Armstrong <andy@hexten.net>.

This specification is released under the [Artistic License
2.0](https://opensource.org/licenses/Artistic-2.0)
