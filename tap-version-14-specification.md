---
layout: default
title: TAP 14 specification
---

<!-- SPDX-License-Identifier: Artistic-2.0 -->

## Goals of This Specification

* Document the observed behavior of widely used TAP implementations.
* Add no features that are not already in wide usage across multiple
  implementations.
* Explicitly allow what is already allowed, deny what is already denied.
* Provide updated and clearer guidance for new TAP implementations.

# Name

TAP14 - The Test Anything Protocol v14

## Synopsis

TAP, the Test Anything Protocol, is a simple text-based interface between
testing modules and test harness. TAP started life as part of the test
harness for Perl but now has implementations in C/C++, Python, PHP, Perl,
JavaScript, and probably others by the time you read this.  This document
describes version 14 of TAP.

The key words _must_, _must not_, _required_, _shall_, _shall not_,
_should_, _should not_, _recommended_, _may_, and _optional_ in this
document are to be interpreted as described in [RFC
2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Changes From TAP13 Format

TAP14 is largely backwards compatible with TAP13.  That is, TAP14
is designed to be reasonably parseable by any compliant TAP13
Harness, and TAP14 Harnesses should be able to reasonably
interpret the output of TAP13 Producers.

The following changes have been made to the specification, which
are covered in much more detail below.

* Change TAP version line to `14`.
* Add child tests as 4-space indented TAP streams, with a
  trailing test point and leading comment line.
* Formalize the following conventions:
    * 2-space indentation for YAML diagnostics.
    * plans always start at `1`.
    * escaping of `\` and `#` characters.
    * prefixing of test point descriptions with `" - "`.
* Increased clarity regarding parsing rules, based on behavior of
  extant TAP13 implementations in popular programming languages.

## TAP14 Format

TAP14's general grammar is:

```ebnf
TAPDocument := Version Plan Body | Version Body Plan
Version     := "TAP version 14\n"
Plan        := "1.." (Number) (" # " Reason)? "\n"
Body        := (TestPoint | BailOut | Pragma | Comment | Anything | Empty | Subtest)*
TestPoint   := ("not ")? "ok" ((" " Number (" -")? | " -") (" " Description)?)? (" " Directive)? "\n" (YAMLBlock)?
Directive   := " # " ("todo" | "skip") (" " Reason)?
YAMLBlock   := "  ---\n" (YAMLLine)* "  ...\n"
YAMLLine    := "  " (YAML)* "\n"
BailOut     := "Bail out!" (" " Reason)? "\n"
Reason      := [^\n]+
Pragma      := "pragma " [+-] PragmaKey "\n"
PragmaKey   := ([a-zA-Z0-9_-])+
Subtest     := ("# Subtest" (": " SubtestName)?)? "\n" SubtestDocument TestPoint
Comment     := ^ (" ")* "#" [^\n]* "\n"
Empty       := [\s\t]* "\n"
Anything    := [^\n]+ "\n"
```

Note that the above is intended as a rough "pseudocode" guidance for
humans.  It is not strict EBNF.  Detailed parsing instructions for each
element can be found in the sections below.

The Version is the line `TAP version 14`.

The Body is a collection of lines representing a test set.

The Plan reports on the number of tests included in the Body.

A Subtest indicates a nested TAP Document that is included by the parent
TAP Document.  It is a TAP Document where each line is indented by 4 spaces.

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

## Harness Behavior

In this document, the "harness" is any program analyzing TAP output.

Typically this will be a test framework's runner program, but may also be a
programmatic parser, parent test object, or test result reporter.

In a typical TAP implementation, tests are programs that ouput TAP data
according to this specification.  The "test harness" reads TAP output from
these programs, and handles it in some way.

A harness that is collecting output from a test program _should_ read and
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

## Document Structure

TAP14 producers _must_ encode TAP data using the UTF-8 encoding.

Harnesses _should_ interpret all TAP streams using UTF-8.  Harnesses _may_
provide a mechanism for using other encodings explicitly, if needed for
backwards compatibility.

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

Its basic grammar is:

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

`#` and `\` characters may be escaped in Plan reason, and if so, _should_
be unescaped prior to being presented to the user.  See "Escaping" below.

### Test Points

The core of TAP is the "Test Point". A test file prints one test point
executed. There must be at least one test point in TAP output. Each test
point comprises the following elements:

In summary:

- Test Status: `ok`/`not ok` (required)
- Test number (recommended)
- Description (recommended, prefixed by `" - "`)
- Directive (only when necessary)

#### Test Status: `ok` or `not ok`

This tells whether the test point passed or failed. It must be at the
beginning of the line. `/^not ok/` indicates a failed test point.
`/^ok/` is a successful test point. This is the only mandatory part of
the line.

Note that unlike the Directives below, `ok` and `not ok` are
case-sensitive.

#### Test Point ID

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

This test output is _not_ a successful test run:

```tap
TAP version 14
1..6
not ok
ok
not ok
ok
ok
```

Five tests are shown, but the plan indicated that there would be 6.
Furthermore, tests 1 and 3 are explicitly failing.  Perl's
`Test::Harness` will report:

```
FAILED tests 1, 3, 6
Failed 3/6 tests, 50.00% okay
```

Test Points _may_ be output in any order, but any Test Point ID
provided _must_ be within the range described by the Plan.

This is valid TAP and a successful test run:

```tap
TAP version 14
1..3
ok 2
ok 3
ok 1
```

This is not a successful test run. Even though there are 3 Test Points,
the Test Point ID 4 is outside the stated Plan range.

```tap
TAP version 14
1..3
ok 2
ok 4
ok 1
```

Test Point IDs _should_ be unique within the TAP Document.  Harnesses _may_
warn about repeated Test Point IDs or treat them as a test failure, but
_must not_ treat a Test Point with a re-used ID as a non-TAP line.

#### Description

Any text after the test number but before a `#` is the description of
the test point.

```tap
ok 42 - this is the description of the test
```

Descriptions _should_ be separated from the Test Point Status and Test
Point ID by the string `" - "`, in order to prevent confusing a numeric
description with a Test Point ID.  Harnesses _must_ treat Test Points
identically whether the description starts with `" - "` or not.

For example, these two test points _must_ be treated identically by a
Harness:

```tap
ok 1 this is fine
ok 1 - this is fine
```

Harnesses _should not_ consider a leading `" - "` to be a part of the
description reported to a user.

#### Directive

Directives are special notes that follow the first unescaped `#` on the
Test Point line that is preceded and followed by whitespace.  Only two are
currently defined: `TODO` and `SKIP`.

Directives are not case sensitive.  That is, Harnesses _must_ treat `#
SKIP`, `# skip`, and `# SkIp` identically.

Harnesses _may_ support additional platform-specific Directives.  Future
versions of this specification may codify additional Directives with
defined semantics.

Unrecognized Directives _must not_ be treated as test failure, or an
invalid TAP line.  Harnesses _should_ include any unrecognized directives
in the Test Point description.

Note that escaped `#` characters are not to be treated as delimiters for
Directives.  See "Escaping" below.

##### Whitespace Around Directive Delimiter

Earlier versions of the TAP specification were not explicit about
whitespace requirements regarding directives.  The following rules maximize
compatibility between TAP14 producers and harnesses:

1. Producers _must_ output directives with at least one space character
   preceding the `#` in a directive, as well as at least one space
   character between the `#` and the directive name.

2. Harnesses _must not_ treat escaped `#` characters as directive
   delimiters.

3. Harnesses _may_ accept directive delimiters where the `#` is not
   preceded by whitespace, but _should_ warn that such output is
   non-conformant with the TAP14 specification.

If harnesses choose to parse directives without whitespace before and after
the `#`, then they ought to consider the impact if test descriptions
contain URLs and/or may be coming from TAP producers that are not diligent
about escaping `#` characters.  This should be done only to the extent
necessary for backwards compatibility with existing TAP producers.

For example:

```tap
TAP version 14

# MUST be treated as a SKIP test
ok 1 - must be skipped test # SKIP

# MUST NOT be treated as a SKIP test
ok 2 - must not be skipped test \# SKIP

# MAY be treated as a SKIP test, but SHOULD warn about it
ok 3 - may skip, but should warn# skip
ok 4 - may skip, but should warn #skip
ok 5 - may skip, but should warn#skip
```

##### Backwards Compatibility and Parsing Notes

For backwards compatibility with earlier incarnations of TAP, Harnesses
_must_ accept additional non-whitespace characters following the
literal strings `"SKIP"` or `"TODO"`.  Everything after `(TODO|SKIP)\S*\s+`
is the reason.  For example, this is supported, and shows a test with 2
`skip` tests: one with no reason given, and the other with an explanation.

```tap
TAP version 14
1..2

# skip: true
ok 1 - do it later # Skipped

# skip: true
# skip reason: "only run on windows"
ok 2 - works on windows # Skipped: only run on windows
```

For broad compatibility with as many harnesses as possible, as well as
future-proofing their output, Producers _should_ always report `SKIP` and
`TODO` tests using only the directive names ("SKIP" or "TODO"), optionally
followed by a space and a reason.  Future versions of this specification
may drop support for `\S*` characters following directive names.

Thus, the regular expression for directives is:

```
/\s+#\s*(SKIP|TODO)\S*\s+([^\n]*)/
directive type = $1
reason = $2
```

More examples of parsing directives:

```tap
TAP version 14

# skip: true
# skip reason: "this test is skipped"
# description: ""
ok 1 # skip this test is skipped

# skip: false
# description: "not skipped: https://example.com/page.html\#skip is a url"
ok 2 not skipped: https://example.com/page.html#skip is a url

# skip: true
# skip reason: "case insensitive, so this is skipped"
# description: ""
ok 3 - #SkIp case insensitive, so this is skipped
```

##### `TODO` tests

If the directive starts with `# TODO`, the test is counted as a todo test,
and any text after `TODO\S*\s+` is the explanation.

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

##### `SKIP` tests

If the directive starts with `# SKIP`, the test is counted as a skipped
test, and the text after `SKIP\S*\s+` is the explanation.

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

If a Test Point is followed by a 2-space indented block beginning with the
line `  ---` and ending with the line `  ...`, separated from the Test
Point only by comments or whitespace, then the block of lines between these
markers will be interpreted as an inline YAML Diagnostic document according
to version 1.2 of the [YAML specification](https://yaml.org).

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
preceded by zero or more characters of whitespace, are comments.

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
TAP version 14
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

Pragmas _must not_ include Comments, Directives, or other characters other
than those specified above.

### Blank Lines

For the purposes of this specification, a "blank" line is any line
consisting exclusively of zero or more whitespace characters (ie,
`/^[ \t]+$/`).

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

to standard output. Any message after these words _must_ be presented by
the Harness as the reason why testing must be stopped.  For example:

```tap
Bail out! MySQL is not running.
```

`#` and `\` characters may be escaped in `Bail out!` messages, and if so,
_should_ be unescaped prior to being presented to the user.  See
"Escaping" below.

```tap
# reason for stopping: # and \ are not supported
Bail out! \# and \\ are not supported
```

The words "Bail out!" are case insensitive.

### Anything Else

Any line that is not a valid version, plan, test point, YAML diagnostic,
pragma, a blank line, or a bail out is invalid TAP.

A Harness _may_ silently ignore invalid TAP lines, pass them through to its
own stderr or stdout, or report them in some other fashion.  However,
Harnesses _should not_ treat invalid TAP lines as a test failure by
default.

## Escaping

Sometimes a user may include a `#` character in a Test Point description,
Plan comment, Bailout reason, or TODO/SKIP reason.

In order to distinguish this from a directive or other sort of comment, the
`#` character may be escaped with a backslash `\` character.  To include a
literal `\` character, a double-`\` may be used.  No other characters may
be escaped in this way; a `\` that precedes any character other than `\` or
`#` will be interpreted as a literal `\` and included in the result data.

Providers _should_ escape any `#` or `\` that is present in the Test Point
description or directive reason sections.

For example:

```js
// using node-tap
const t = require('tap')
t.pass('hello # \\ world', { todo: 'escape # characters with \\' })
// outputs:
// ok 1 - hello \# \\ world # TODO escape \# characters with \\
```

Harnesses _must_:

1. Treat any `\\` as a literal `\`, but ignore it for the purpose of
   escaping a `#` character.
2. Treat any `\#` as a literal `#`, but ignore it for the purpose of
   delimiting a directive, provided the `\` was not escaped.
3. Treat any unescaped `#` as a literal `#` in a description or reason
   field, if doing so would not cause it to be treated as a delimiter.

### Escaping Examples

The following are examples of escaping `#` and `\` characters, and
including unescaped `#` characters in the Test Point description or `TODO`
reason.

Note that the specific fields `description`, `todo`, and `todo reason` are
not normative, and shown for illustration purposes only.  Harnesses
_should_ present this data to their consumers in whatever manner is
appropriate for their language and context.

```tap
TAP version 14

# description: hello
# todo: true
ok 1 - hello # todo

# description: hello # todo
# todo: false
ok 2 - hello \# todo

# description: hello
# todo: true
# todo reason: hash # character
ok 3 - hello # todo hash \# character
# (assuming "character" isn't a known custom directive)
ok 4 - hello # todo hash # character

# description: hello \
# todo: true
# todo reason: hash # character
ok 5 - hello \\# todo hash \# character
# (assuming "character" isn't a known custom directive)
ok 6 - hello \\# todo hash # character

# description: hello # description # todo
# todo: false
# (assuming "description" isn't a known custom directive)
ok 7 - hello # description # todo

# multiple escaped \ can appear in a row
# description: hello \\\# todo
# todo: false
ok 8 - hello \\\\\\\# todo

1..8
```

## Subtests

Subtests provide a way to nest one TAP14 stream inside another.  This is
useful in a variety of situations.  For example:

1. A Harness parses a collection of TAP documents, providing output which
   is also in TAP format.

    ```tap
    TAP version 14
    1..2

    # Subtest: foo.tap
        1..2
        ok 1
        ok 2 - this passed
    ok 1 - foo.tap

    # Subtest: bar.tap
        ok 1 - object should be a Bar
        not ok 2 - object.isBar should return true
          ---
          found: false
          wanted: true
          at:
            file: test/bar.ts
            line: 43
            column: 8
          ...
        ok 3 - object can bar bears # TODO
        1..3
    not ok 2 - bar.tap
      ---
      fail: 1
      todo: 1
      ...
    ```

2. A test framework Producer provides an API for grouping assertions about
   a related subject.  For example:

    ```js
    import t from 'tap'
    t.ok(true, 'true is ok')
    t.test('this is a subtest', subtest => {
      subtest.pass('this is fine')
      subtest.fail('this is not fine')
      subtest.end()
    })
    ```

    which produces the output:

    ```tap
    TAP version 14
    ok 1 - true is ok
    # Subtest: this is a subtest
        ok 1 - this is fine
        not ok 2 - this is not fine
        1..2
    not ok 2 - this is a subtest
    1..2
    ```

Subtests are designed with graceful fallback for TAP13 harnesses in mind.

Since TAP13 specifies that non-TAP output _should_ be ignored or provided
directly to the user, and indented Test Points and Plans are non-TAP
according to TAP13, only the terminating correlated test point will be
interpreted by most TAP13 Harnesses.  Thus, they will usually report the
overall subtest result correctly, even if they lack the details about the
results of the subtest.

Since several TAP13 parsers in popular usage treat a repeated Version
declaration as an error, even if the Version is indented, Subtests _should
not_ include a Version, if TAP13 Harness compatibility is desirable.

### Bare Subtests

In its simplest form, a Subtest is introduced by a 4-space indented valid
TAP line.  That is, a Test Point, Version, Plan, or (if Pragmas are
supported) Pragma, prefixed by 4 space characters.

For example:

```tap
TAP version 14
    ok 1 - subtest test point
    1..1
ok 1 - subtest passing
1..1
```

Note that, because the Version is optional in subtests, and the plan may
occur after all test points, the first item in a subtest may be a further
subtest.  Harnesses must thus treat any multiple of 4-space indentation is
multiple levels of nested subtest:

```tap
TAP version 14
        ok 1 - nested twice
        1..1
    ok 1 - nested parent
    1..1
ok 1 - double nest passing
1..1
```

The first test point at the parent level is the correlated Test Point for
the subtest, and terminates the Subtest.

### Commented Subtests

A Subtest may be introduced with a comment at the parent test indentation
level, which defines the expected name of the terminating correlated Test
Point.

This comment has the form `^# Subtest(: .*)$`.  Everything after the `: `
is the Subtest Name.  For example:

```tap
# Subtest: <name>
```

or

```tap
# Subtest
```

A Commented Subtest with a Subtest Name _must_ be terminated by a Test
Point with a matching Description.  If no Subtest Name is present, then the
terminating Test Point _must not_ include a description.

For example:

```tap
TAP version 14

ok 1 - in the parent

# Subtest: nested
    1..1
    ok 1 - in the subtest
ok 2 - nested

# Subtest: empty
    1..0
ok 3 - empty

# Subtest
    ok 1 - name is optional
    1..1
ok 4

1..4
```

Commented Subtests are encouraged, as they provide the following benefits:

* Easier for humans to read.  For example:

    ```tap
    TAP version 14
                1..1
                ok 1 - hmm, what level is this?
    ```

    vs:

    ```tap
    TAP version 14
    # Subtest: level 1
        # Subtest: level 2
            # Subtest: level 3
                1..1
                ok 1 - clearly level 3
    ```

* Additional strictness around matching the Test Point description to
  Subtest Name can catch errors and detect accidentally interleaved output.

### Subtest Bailouts

Harnesses _must_ treat a Bailout in a nested Subtest as a bailout for the
entire test run.

For backwards compatibility with TAP13 Harnesses, Producers _should_ emit a
`Bail out!` line at the root indentation level whenever a Subtest bails
out.

### Subtest Pragmas

Any Pragmas set in a Subtest affect _only_ the parsing of the Subtest.
Harnesses _must not_ allow Pragmas set in Subtests to affect the behavior
of the parser with respect to the parent TAP Document.

For example, given a Harness where a `strict` Pragma will cause it to treat
any non-TAP as an error:

```tap
TAP version 14
pragma -strict
# Subtest: child test
    1..1
    pragma +strict
    ok 1
ok 1 - child test
!!This is not valid TAP content!!
1..1
```

In this TAP document, the non-TAP content _must not_ be treated as a test
failure, because the `strict` Pragma setting at the parent level was false.

### Subtest Parsing/Generating Rules

1. The subtest TAP document is indented 4 spaces.
2. Subtests _must_ be a valid TAP document, meaning that they cannot be
   entirely empty.  At minimum, they must include a `1..0` line to indicate
   that no Test Points are expected.  This implies the following:
    1. Subtests can be nested within subtests by indenting another 4
       characters for each level of nesting.
    2. YAML Diagnostics are indented 2 spaces with respect to the Test
       Point they are associated with.  So, for example, a YAML Diagnostic
       block for a Test Point in a nested subtest would be indented 6
       spaces (4 + 2).  In a subtest nested within another subtest, YAML
       diagnostics would be indented 10 spaces (4 + 4 + 2).
3. Subtests _should not_ emit a Version line, if compatibility with TAP13
   Harnesses is desirable.
4. The subtest is terminated by a single Test Point line in the parent TAP
   document, which follows the indented TAP Document.  This is the
   Correlated Test Point, and reflects the pass/fail status of the Subtest.
    1. Producers _must_ communicate the intended status of the subtest
       (pass/fail/todo/skip/etc.) by assigning these semantics to the
       correlated Test Point.
    2. Harnesses _should_ treat the entire subtest as either a pass or fail
       based on the status of the correlated Test Point, but _may_ treat
       the subtest as a failure if they would consider the nested subtest
       TAP Document a test failure.
5. Harnesses _should_ treat otherwise valid TAP that is indented anything
   other than a multiple of 4 spaces as non-TAP.
6. Providers _should not_ emit a Version for the Subtest TAP Document.
7. Providers _should_ emit a Subtest Comment if the name of the subtest is
   known at the outset.
8. If Subtest Comment is provided, Harnesses _should_ continue the subtest
   until a matching test point (same name, or no name in either) at parent
   level is found, treating any other un-indented lines as non-TAP, and
   fail the test if a matching test point is not found.
9. If Subtest Comment is not provided, Harnesses _must_ treat the next Test
   Point at the parent level as the end of the subtest, and treat any
   intervening lines indented less than the subtest level as non-TAP.
10. Harnesses _should_ treat unterminated Subtests as non-TAP.
11. A Bailout in a subtest TAP document _must_ abort the entire process,
    exactly as if it had occurred in the top-level TAP document.
12. Any lines at the parent indentation level that occur between the
    Subtest Introduction and a valid Subtest Termination _must_ be treated
    as non-TAP output.

------

## Examples

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

## Procrastination Considered `ok`

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

## Bugs

Feature requests and bug reports should be [raised on
GitHub.](https://github.com/TestAnything/Specification/issues/new)

## Authors

The TAP 14 Specification is authored by [Isaac Z.
Schlueter](https://github.com/isaacs), as a result of much discussion on
the [TestAnything
Specification](https://github.com/TestAnything/Specification) project, with
considerable input, encouragement, feedback, and suggestions from:

* [Matt Layman](https://github.com/mblayman)
* [Leon Timmermans](https://github.com/Leont)
* [Bruno P. Kinoshita](https://github.com/kinow)
* [Chad Granum](https://github.com/exodist)

## Acknowledgements

Special thanks to [Jonathan Kingston](https://github.com/jonathanKingston)
for his efforts to keep TAP flourishing and bring implementors together.

The TAP13 Specification was written by Andy Armstrong with help and
contributions from Pete Krawczyk, Paul Johnson, Ian Langworth and Nik
Clayton, based on the original TAP documentation by Andy Lester, based on
the original `Test::Harness` documentation by Michael Schwern.

The basis for the TAP format was created by Larry Wall in the original test
script for Perl 1. Tim Bunce and Andreas Koenig developed it further with
their modifications to `Test::Harness`.

## Copyright

Copyright 2015-2022 by Isaac Z. Schlueter <i@izs.me> and Contributors.

Copyright 2003-2007 by Michael G Schwern <schwern@pobox.com>, Andy Lester
<andy@petdance.com>, Andy Armstrong <andy@hexten.net>.

This specification is released under the [Artistic License
2.0](https://spdx.org/licenses/Artistic-2.0)
