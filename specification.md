---
layout: default
title: TAP 14 specification
---

# NAME

TAP14 - The Test Anything Protocol v14

## SYNOPSIS

TAP, the Test Anything Protocol, is a simple text-based interface between testing modules a test harness. TAP started life as part of the test harness for Perl but now has implementations in C/C++, Python, PHP, Perl and probably others by the time you read this.
This document describes version 14 of TAP. Go to TAP to read about previous versions.

## THE TAP14 FORMAT

TAP14's general format is:

```
   TAP version 14
   1..N
   ok 1 Description # Directive
   # Diagnostic
     ---
     message: 'Failure message'
     severity: fail
     data:
       got:
         - 1
         - 3
         - 2
       expect:
         - 1
         - 2
         - 3
     ...
   ok 47 Description
   ok 48 Description
   more tests....
```

For example, a test file's output might look like:

```
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

In this document, the "harness" is any program analyzing TAP output. Typically this will be Perl's runtests program, or the underlying TAP::Harness-runtests> method.
A harness must only read TAP output from standard output and not from standard error. Lines written to standard output matching /^(not)?ok\b/ must be interpreted as test lines. After a test line a block of lines starting with '---' and ending with '...' will be interpreted as an inline YAML document providing extended diagnostic information about the preceding test. All other lines must not be considered test output.

## TESTS LINES AND THE PLAN

### The version
To indicate that this is TAP14 the first line must be

```
    TAP version 14
```

### The plan
The plan tells how many tests will be run, or how many tests have run. It's a check that the test file hasn't stopped prematurely. It must appear once, whether at the beginning or end of the output.
The plan is usually the first line of TAP output (although in future there may be a version line before it) and it specifies how many test points are to follow. For example,

```
    1..10
```

means you plan on running 10 tests. This is a safeguard in case your test file dies silently in the middle of its run. The plan is optional but if there is a plan before the test points it must be the first non-diagnostic line output by the test file.
In certain instances a test file may not know how many test points it will ultimately be running. In this case the plan can be the last non-diagnostic line in the output.
The plan cannot appear in the middle of the output, nor can it appear more than once.

### The test line
The core of TAP is the test line. A test file prints one test line test point executed. There must be at least one test line in TAP output. Each test line comprises the following elements:

-    ok or not ok

This tells whether the test point passed or failed. It must be at the beginning of the line. /^not ok/ indicates a failed test point. /^ok/ is a successful test point. This is the only mandatory part of the line.
Note that unlike the Directives below, ok and not ok are case-sensitive.

-    Test number

TAP expects the ok or not ok to be followed by a test point number. If there is no number the harness must maintain its own counter until the script supplies test numbers again. So the following test output

```
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

-    Description
Any text after the test number but before a # is the description of the test point.

```
    ok 42 this is the description of the test
```

Descriptions should not begin with a digit so that they are not confused with the test point number.
The harness may do whatever it wants with the description.

-    Directive
The test point may include a directive, following a hash on the test line. There are currently two directives allowed: TODO and SKIP. These are discussed below.

To summarize:
-    ok/not ok (required)
-    Test number (recommended)
-    Description (recommended)
-    Directive (only when necessary)

### YAML blocks
If the test line is immediately followed by an indented block beginning with /^\s+---/ and ending with \^\s+.../ that block will be interpreted as an inline YAML document. The YAML encodes a data structure that provides more detailed information about the preceding test.
The YAML document is indented to make it visually distinct from the surrounding test results and to make it easier for the parser to recover if the trailing '...' terminator is missing.
For example:

```
    not ok 3 Resolve address
     ---
     message: "Failed with error 'hostname peebles.example.com not found'"
     severity: fail
     data:
       got:
         hostname: 'peebles.example.com'
         address: ~
       expected:
         hostname: 'peebles.example.com'
         address: '85.193.201.85'
     ...
```

## DIRECTIVES

Directives are special notes that follow a # on the test line. Only two are currently defined: TODO and SKIP. Note that these two keywords are not case-sensitive.

### TODO tests
If the directive starts with # TODO, the test is counted as a todo test, and the text after TODO is the explanation.

```
    not ok 13 # TODO bend space and time
```

Note that if the TODO has an explanation it must be separated from TODO by a space.
These tests represent a feature to be implemented or a bug to be fixed and act as something of an executable "things to do" list. They are not expected to succeed. Should a todo test point begin succeeding, the harness should report it as a bonus. This indicates that whatever you were supposed to do has been done and you should promote this to a normal test point.

### Skipping tests
If the directive starts with # SKIP, the test is counted as having been skipped. If the whole test file succeeds, the count of skipped tests is included in the generated output. The harness should report the text after # SKIP\S*\s+ as a reason for skipping.

```
    ok 23 # skip Insufficient flogiston pressure.
```

Similarly, one can include an explanation in a plan line, emitted if the test file is skipped completely:

```
    1..0 # Skipped: WWW::Mechanize not installed
```

## OTHER LINES

### Bail out!
As an emergency measure a test script can decide that further tests are useless (e.g. missing dependencies) and testing should stop immediately. In that case the test script prints the magic words

```
    Bail out!
```

to standard output. Any message after these words must be displayed by the interpreter as the reason why testing must be stopped, as in

```
    Bail out! MySQL is not running.
```

### Diagnostics
Additional information may be put into the testing output on separate lines. Diagnostic lines should begin with a #, which the harness must ignore, at least as far as analyzing the test results. The harness is free, however, to display the diagnostics.
TAP14's YAML blocks are intended to replace diagnostics for most purposes but TAP14 consumers should maintain backwards compatibility by supporting them.

## EXAMPLES

All names, places, and events depicted in any example are wholly fictitious and bear no resemblance to, connection with, or relation to any real entity. Any such similarity is purely coincidental, unintentional, and unintended.

### Common with explanation
The following TAP listing declares that six tests follow as well as provides handy feedback as to what the test is about to do. All six tests pass.

```
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
This hypothetical test program ensures that a handful of servers are online and network-accessible. Because it retrieves the hypothetical servers from a database, it doesn't know exactly how many servers it will need to ping. Thus, the test count is declared at the bottom after all the test points have run. Also, two of the tests fail. The YAML block following each failure gives additional information about the failure that may be displayed by the harness.

```
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
This listing reports that a pile of tests are going to be run. However, the first test fails, reportedly because a connection to the database could not be established. The program decided that continuing was pointless and exited.

```
   TAP version 14
   1..573
   not ok 1 - database handle
   Bail out! Couldn't connect to database.
```

### Skipping a few
The following listing plans on running 5 tests. However, our program decided to not run tests 2 thru 5 at all. To properly report this, the tests are marked as being skipped.

```
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

```
   TAP version 14
   1..0 # skip because English-to-French translator isn't installed
```

## Got spare tuits?
The following example reports that four tests are run and the last two tests failed. However, because the failing tests are marked as things to do later, they are considered successes. Thus, a harness should report this entire listing as a success.

```
   TAP version 14
   1..4
   ok 1 - Creating test program
   ok 2 - Test program runs, no error
   not ok 3 - infinite loop # TODO halting problem unsolved
   not ok 4 - infinite loop 2 # TODO halting problem unsolved
```

## Creative liberties
This listing shows an alternate output where the test numbers aren't provided. The test also reports the state of a ficticious board game as a YAML block. Finally, the test count is reported at the end.

```
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
# Recommended Media Type Usage
When streaming TAP across a network it is MAY be possible to provide a MIME type for content negotiation.
The suggested MIME type to transfer TAP documents is `application/tap`.

## TODO

- Define the exit code of the process.
- Confirm use of bugs, authors, acknowledgements and copyright
- Remove any further wording that doesn't belong in a specification
- Add glossary of terms

