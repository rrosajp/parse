Installation
------------

.. code-block:: pycon

    pip install parse

Usage
-----

Parse strings using a specification based on the Python `format()`_ syntax.

   ``parse()`` is the opposite of ``format()``

The module is set up to only export ``parse()``, ``search()``, ``findall()``,
and ``with_pattern()`` when ``import *`` is used:

>>> from parse import *

From there it's a simple thing to parse a string:

.. code-block:: pycon

    >>> parse("It's {}, I love it!", "It's spam, I love it!")
    <Result ('spam',) {}>
    >>> _[0]
    'spam'

Or to search a string for some pattern:

.. code-block:: pycon

    >>> search('Age: {:d}\n', 'Name: Rufus\nAge: 42\nColor: red\n')
    <Result (42,) {}>

Or find all the occurrences of some pattern in a string:

.. code-block:: pycon

    >>> ''.join(r[0] for r in findall(">{}<", "<p>the <b>bold</b> text</p>"))
    'the bold text'

If you're going to use the same pattern to match lots of strings you can
compile it once:

.. code-block:: pycon

    >>> from parse import compile
    >>> p = compile("It's {}, I love it!")
    >>> print(p)
    <Parser "It's {}, I love it!">
    >>> p.parse("It's spam, I love it!")
    <Result ('spam',) {}>

("compile" is not exported for ``import *`` usage as it would override the
built-in ``compile()`` function)

The default behaviour is to match strings case insensitively. You may match with
case by specifying `case_sensitive=True`:

.. code-block:: pycon

    >>> parse('SPAM', 'spam', case_sensitive=True) is None
    True

.. _format():
  https://docs.python.org/3/library/stdtypes.html#str.format


Format Syntax
-------------

A basic version of the `Format String Syntax`_ is supported with anonymous
(fixed-position), named and formatted fields::

   {[field name]:[format spec]}

Field names must be a valid Python identifiers, including dotted names;
element indexes imply dictionaries (see below for example).

Numbered fields are also not supported: the result of parsing will include
the parsed fields in the order they are parsed.

The conversion of fields to types other than strings is done based on the
type in the format specification, which mirrors the ``format()`` behaviour.
There are no "!" field conversions like ``format()`` has.

Some simple parse() format string examples:

.. code-block:: pycon

    >>> parse("Bring me a {}", "Bring me a shrubbery")
    <Result ('shrubbery',) {}>
    >>> r = parse("The {} who {} {}", "The knights who say Ni!")
    >>> print(r)
    <Result ('knights', 'say', 'Ni!') {}>
    >>> print(r.fixed)
    ('knights', 'say', 'Ni!')
    >>> print(r[0])
    knights
    >>> print(r[1:])
    ('say', 'Ni!')
    >>> r = parse("Bring out the holy {item}", "Bring out the holy hand grenade")
    >>> print(r)
    <Result () {'item': 'hand grenade'}>
    >>> print(r.named)
    {'item': 'hand grenade'}
    >>> print(r['item'])
    hand grenade
    >>> 'item' in r
    True

Note that `in` only works if you have named fields.

Dotted names and indexes are possible with some limits. Only word identifiers
are supported (ie. no numeric indexes) and the application must make additional
sense of the result:

.. code-block:: pycon

    >>> r = parse("Mmm, {food.type}, I love it!", "Mmm, spam, I love it!")
    >>> print(r)
    <Result () {'food.type': 'spam'}>
    >>> print(r.named)
    {'food.type': 'spam'}
    >>> print(r['food.type'])
    spam
    >>> r = parse("My quest is {quest[name]}", "My quest is to seek the holy grail!")
    >>> print(r)
    <Result () {'quest': {'name': 'to seek the holy grail!'}}>
    >>> print(r['quest'])
    {'name': 'to seek the holy grail!'}
    >>> print(r['quest']['name'])
    to seek the holy grail!

If the text you're matching has braces in it you can match those by including
a double-brace ``{{`` or ``}}`` in your format string, the same escaping method
used in the ``format()`` syntax.


Format Specification
--------------------

Most often a straight format-less ``{}`` will suffice where a more complex
format specification might have been used.

Most of `format()`'s `Format Specification Mini-Language`_ is supported:

   [[fill]align][sign][0][width][.precision][type]

The differences between `parse()` and `format()` are:

- The align operators will cause spaces (or specified fill character) to be
  stripped from the parsed value. The width is not enforced; it just indicates
  there may be whitespace or "0"s to strip.
- Numeric parsing will automatically handle a "0b", "0o" or "0x" prefix.
  That is, the "#" format character is handled automatically by d, b, o
  and x formats. For "d" any will be accepted, but for the others the correct
  prefix must be present if at all.
- Numeric sign is handled automatically.  A sign specifier can be given, but
  has no effect.
- The thousands separator is handled automatically if the "n" type is used.
- The types supported are a slightly different mix to the format() types.  Some
  format() types come directly over: "d", "n", "%", "f", "e", "b", "o" and "x".
  In addition some regular expression character group types "D", "w", "W", "s"
  and "S" are also available.
- The "e" and "g" types are case-insensitive so there is not need for
  the "E" or "G" types. The "e" type handles Fortran formatted numbers (no
  leading 0 before the decimal point).

===== =========================================== ========
Type  Characters Matched                          Output
===== =========================================== ========
l     Letters (ASCII)                             str
w     Letters, numbers and underscore             str
W     Not letters, numbers and underscore         str
s     Whitespace                                  str
S     Non-whitespace                              str
d     Integer numbers (optional sign, digits)     int
D     Non-digit                                   str
n     Numbers with thousands separators (, or .)  int
%     Percentage (converted to value/100.0)       float
f     Fixed-point numbers                         float
F     Decimal numbers                             Decimal
e     Floating-point numbers with exponent        float
      e.g. 1.1e-10, NAN (all case insensitive)
g     General number format (either d, f or e)    float
b     Binary numbers                              int
o     Octal numbers                               int
x     Hexadecimal numbers (lower and upper case)  int
ti    ISO 8601 format date/time                   datetime
      e.g. 1972-01-20T10:21:36Z ("T" and "Z"
      optional)
te    RFC2822 e-mail format date/time             datetime
      e.g. Mon, 20 Jan 1972 10:21:36 +1000
tg    Global (day/month) format date/time         datetime
      e.g. 20/1/1972 10:21:36 AM +1:00
ta    US (month/day) format date/time             datetime
      e.g. 1/20/1972 10:21:36 PM +10:30
tc    ctime() format date/time                    datetime
      e.g. Sun Sep 16 01:03:52 1973
th    HTTP log format date/time                   datetime
      e.g. 21/Nov/2011:00:07:11 +0000
ts    Linux system log format date/time           datetime
      e.g. Nov  9 03:37:44
tt    Time                                        time
      e.g. 10:21:36 PM -5:30
===== =========================================== ========

The type can also be a datetime format string, following the
`1989 C standard format codes`_, e.g. ``%Y-%m-%d``. Depending on the
directives contained in the format string, parsed output may be an instance
of ``datetime.datetime``, ``datetime.time``, or ``datetime.date``.

.. code-block:: pycon

    >>> parse("{:%Y-%m-%d %H:%M:%S}", "2023-11-23 12:56:47")
    <Result (datetime.datetime(2023, 11, 23, 12, 56, 47),) {}>
    >>> parse("{:%H:%M}", "10:26")
    <Result (datetime.time(10, 26),) {}>
    >>> parse("{:%Y/%m/%d}", "2023/11/25")
    <Result (datetime.date(2023, 11, 25),) {}>


Some examples of typed parsing with ``None`` returned if the typing
does not match:

.. code-block:: pycon

    >>> parse('Our {:d} {:w} are...', 'Our 3 weapons are...')
    <Result (3, 'weapons') {}>
    >>> parse('Our {:d} {:w} are...', 'Our three weapons are...')
    >>> parse('Meet at {:tg}', 'Meet at 1/2/2011 11:00 PM')
    <Result (datetime.datetime(2011, 2, 1, 23, 0),) {}>

And messing about with alignment:

.. code-block:: pycon

    >>> parse('with {:>} herring', 'with     a herring')
    <Result ('a',) {}>
    >>> parse('spam {:^} spam', 'spam    lovely     spam')
    <Result ('lovely',) {}>

Note that the "center" alignment does not test to make sure the value is
centered - it just strips leading and trailing whitespace.

Width and precision may be used to restrict the size of matched text
from the input. Width specifies a minimum size and precision specifies
a maximum. For example:

.. code-block:: pycon

    >>> parse('{:.2}{:.2}', 'look')           # specifying precision
    <Result ('lo', 'ok') {}>
    >>> parse('{:4}{:4}', 'look at that')     # specifying width
    <Result ('look', 'at that') {}>
    >>> parse('{:4}{:.4}', 'look at that')    # specifying both
    <Result ('look at ', 'that') {}>
    >>> parse('{:2d}{:2d}', '0440')           # parsing two contiguous numbers
    <Result (4, 40) {}>

Some notes for the special date and time types:

- the presence of the time part is optional (including ISO 8601, starting
  at the "T"). A full datetime object will always be returned; the time
  will be set to 00:00:00. You may also specify a time without seconds.
- when a seconds amount is present in the input fractions will be parsed
  to give microseconds.
- except in ISO 8601 the day and month digits may be 0-padded.
- the date separator for the tg and ta formats may be "-" or "/".
- named months (abbreviations or full names) may be used in the ta and tg
  formats in place of numeric months.
- as per RFC 2822 the e-mail format may omit the day (and comma), and the
  seconds but nothing else.
- hours greater than 12 will be happily accepted.
- the AM/PM are optional, and if PM is found then 12 hours will be added
  to the datetime object's hours amount - even if the hour is greater
  than 12 (for consistency.)
- in ISO 8601 the "Z" (UTC) timezone part may be a numeric offset
- timezones are specified as "+HH:MM" or "-HH:MM". The hour may be one or two
  digits (0-padded is OK.) Also, the ":" is optional.
- the timezone is optional in all except the e-mail format (it defaults to
  UTC.)
- named timezones are not handled yet.

Note: attempting to match too many datetime fields in a single parse() will
currently result in a resource allocation issue. A TooManyFields exception
will be raised in this instance. The current limit is about 15. It is hoped
that this limit will be removed one day.

.. _`Format String Syntax`:
  https://docs.python.org/3/library/string.html#format-string-syntax
.. _`Format Specification Mini-Language`:
  https://docs.python.org/3/library/string.html#format-specification-mini-language
.. _`1989 C standard format codes`:
  https://docs.python.org/3/library/datetime.html#strftime-and-strptime-format-codes



Result and Match Objects
------------------------

The result of a ``parse()`` and ``search()`` operation is either ``None`` (no match), a
``Result`` instance or a ``Match`` instance if ``evaluate_result`` is False.

The ``Result`` instance has three attributes:

``fixed``
   A tuple of the fixed-position, anonymous fields extracted from the input.
``named``
   A dictionary of the named fields extracted from the input.
``spans``
   A dictionary mapping the names and fixed position indices matched to a
   2-tuple slice range of where the match occurred in the input.
   The span does not include any stripped padding (alignment or width).

The ``Match`` instance has one method:

``evaluate_result()``
   Generates and returns a ``Result`` instance for this ``Match`` object.



Custom Type Conversions
-----------------------

If you wish to have matched fields automatically converted to your own type you
may pass in a dictionary of type conversion information to ``parse()`` and
``compile()``.

The converter will be passed the field string matched. Whatever it returns
will be substituted in the ``Result`` instance for that field.

Your custom type conversions may override the builtin types if you supply one
with the same identifier:

.. code-block:: pycon

    >>> def shouty(string):
    ...    return string.upper()
    ...
    >>> parse('{:shouty} world', 'hello world', {"shouty": shouty})
    <Result ('HELLO',) {}>

If the type converter has the optional ``pattern`` attribute, it is used as
regular expression for better pattern matching (instead of the default one):

.. code-block:: pycon

    >>> def parse_number(text):
    ...    return int(text)
    >>> parse_number.pattern = r'\d+'
    >>> parse('Answer: {number:Number}', 'Answer: 42', {"Number": parse_number})
    <Result () {'number': 42}>
    >>> _ = parse('Answer: {:Number}', 'Answer: Alice', {"Number": parse_number})
    >>> assert _ is None, "MISMATCH"

You can also use the ``with_pattern(pattern)`` decorator to add this
information to a type converter function:

.. code-block:: pycon

    >>> from parse import with_pattern
    >>> @with_pattern(r'\d+')
    ... def parse_number(text):
    ...    return int(text)
    >>> parse('Answer: {number:Number}', 'Answer: 42', {"Number": parse_number})
    <Result () {'number': 42}>

A more complete example of a custom type might be:

.. code-block:: pycon

    >>> yesno_mapping = {
    ...     "yes":  True,   "no":    False,
    ...     "on":   True,   "off":   False,
    ...     "true": True,   "false": False,
    ... }
    >>> @with_pattern(r"|".join(yesno_mapping))
    ... def parse_yesno(text):
    ...     return yesno_mapping[text.lower()]


If the type converter ``pattern`` uses regex-grouping (with parenthesis),
you should indicate this by using the optional ``regex_group_count`` parameter
in the ``with_pattern()`` decorator:

.. code-block:: pycon

    >>> @with_pattern(r'((\d+))', regex_group_count=2)
    ... def parse_number2(text):
    ...    return int(text)
    >>> parse('Answer: {:Number2} {:Number2}', 'Answer: 42 43', {"Number2": parse_number2})
    <Result (42, 43) {}>

Otherwise, this may cause parsing problems with unnamed/fixed parameters.


Potential Gotchas
-----------------

``parse()`` will always match the shortest text necessary (from left to right)
to fulfil the parse pattern, so for example:


.. code-block:: pycon

    >>> pattern = '{dir1}/{dir2}'
    >>> data = 'root/parent/subdir'
    >>> sorted(parse(pattern, data).named.items())
    [('dir1', 'root'), ('dir2', 'parent/subdir')]

So, even though `{'dir1': 'root/parent', 'dir2': 'subdir'}` would also fit
the pattern, the actual match represents the shortest successful match for
``dir1``.

Developers
----------

Want to contribute to parse? Fork the repo to your own GitHub account, and create a pull-request.

.. code-block:: bash

   git clone git@github.com:r1chardj0n3s/parse.git
   git remote rename origin upstream
   git remote add origin git@github.com:YOURUSERNAME/parse.git
   git checkout -b myfeature

To run the tests locally:

.. code-block:: bash

   python -m venv .venv
   source .venv/bin/activate
   pip install -r tests/requirements.txt
   pip install -e .
   pytest

----

Changelog
---------

- 1.20.2 Template field names can now contain - character i.e. HYPHEN-MINUS, chr(0x2d)
- 1.20.1 The `%f` directive accepts 1-6 digits, like strptime (thanks @bbertincourt)
- 1.20.0 Added support for strptime codes (thanks @bendichter)
- 1.19.1 Added support for sign specifiers in number formats (thanks @anntzer)
- 1.19.0 Added slice access to fixed results (thanks @jonathangjertsen).
  Also corrected matching of *full string* vs. *full line* (thanks @giladreti)
  Fix issue with using digit field numbering and types
- 1.18.0 Correct bug in int parsing introduced in 1.16.0 (thanks @maxxk)
- 1.17.0 Make left- and center-aligned search consume up to next space
- 1.16.0 Make compiled parse objects pickleable (thanks @martinResearch)
- 1.15.0 Several fixes for parsing non-base 10 numbers (thanks @vladikcomper)
- 1.14.0 More broad acceptance of Fortran number format (thanks @purpleskyfall)
- 1.13.1 Project metadata correction.
- 1.13.0 Handle Fortran formatted numbers with no leading 0 before decimal
  point (thanks @purpleskyfall).
  Handle comparison of FixedTzOffset with other types of object.
- 1.12.1 Actually use the `case_sensitive` arg in compile (thanks @jacquev6)
- 1.12.0 Do not assume closing brace when an opening one is found (thanks @mattsep)
- 1.11.1 Revert having unicode char in docstring, it breaks Bamboo builds(?!)
- 1.11.0 Implement `__contains__` for Result instances.
- 1.10.0 Introduce a "letters" matcher, since "w" matches numbers
  also.
- 1.9.1 Fix deprecation warnings around backslashes in regex strings
  (thanks Mickael Schoentgen). Also fix some documentation formatting
  issues.
- 1.9.0 We now honor precision and width specifiers when parsing numbers
  and strings, allowing parsing of concatenated elements of fixed width
  (thanks Julia Signell)
- 1.8.4 Add LICENSE file at request of packagers.
  Correct handling of AM/PM to follow most common interpretation.
  Correct parsing of hexadecimal that looks like a binary prefix.
  Add ability to parse case sensitively.
  Add parsing of numbers to Decimal with "F" (thanks John Vandenberg)
- 1.8.3 Add regex_group_count to with_pattern() decorator to support
  user-defined types that contain brackets/parenthesis (thanks Jens Engel)
- 1.8.2 add documentation for including braces in format string
- 1.8.1 ensure bare hexadecimal digits are not matched
- 1.8.0 support manual control over result evaluation (thanks Timo Furrer)
- 1.7.0 parse dict fields (thanks Mark Visser) and adapted to allow
  more than 100 re groups in Python 3.5+ (thanks David King)
- 1.6.6 parse Linux system log dates (thanks Alex Cowan)
- 1.6.5 handle precision in float format (thanks Levi Kilcher)
- 1.6.4 handle pipe "|" characters in parse string (thanks Martijn Pieters)
- 1.6.3 handle repeated instances of named fields, fix bug in PM time
  overflow
- 1.6.2 fix logging to use local, not root logger (thanks Necku)
- 1.6.1 be more flexible regarding matched ISO datetimes and timezones in
  general, fix bug in timezones without ":" and improve docs
- 1.6.0 add support for optional ``pattern`` attribute in user-defined types
  (thanks Jens Engel)
- 1.5.3 fix handling of question marks
- 1.5.2 fix type conversion error with dotted names (thanks Sebastian Thiel)
- 1.5.1 implement handling of named datetime fields
- 1.5 add handling of dotted field names (thanks Sebastian Thiel)
- 1.4.1 fix parsing of "0" in int conversion (thanks James Rowe)
- 1.4 add __getitem__ convenience access on Result.
- 1.3.3 fix Python 2.5 setup.py issue.
- 1.3.2 fix Python 3.2 setup.py issue.
- 1.3.1 fix a couple of Python 3.2 compatibility issues.
- 1.3 added search() and findall(); removed compile() from ``import *``
  export as it overwrites builtin.
- 1.2 added ability for custom and override type conversions to be
  provided; some cleanup
- 1.1.9 to keep things simpler number sign is handled automatically;
  significant robustification in the face of edge-case input.
- 1.1.8 allow "d" fields to have number base "0x" etc. prefixes;
  fix up some field type interactions after stress-testing the parser;
  implement "%" type.
- 1.1.7 Python 3 compatibility tweaks (2.5 to 2.7 and 3.2 are supported).
- 1.1.6 add "e" and "g" field types; removed redundant "h" and "X";
  removed need for explicit "#".
- 1.1.5 accept textual dates in more places; Result now holds match span
  positions.
- 1.1.4 fixes to some int type conversion; implemented "=" alignment; added
  date/time parsing with a variety of formats handled.
- 1.1.3 type conversion is automatic based on specified field types. Also added
  "f" and "n" types.
- 1.1.2 refactored, added compile() and limited ``from parse import *``
- 1.1.1 documentation improvements
- 1.1.0 implemented more of the `Format Specification Mini-Language`_
  and removed the restriction on mixing fixed-position and named fields
- 1.0.0 initial release

This code is copyright 2012-2021 Richard Jones <richard@python.org>
See the end of the source file for the license of use.
