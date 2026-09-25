# NAME

Perl::Critic::Policy::Variables::ProhibitUselessVarClearing - Do not empty a lexical that nothing reads again before it goes out of scope.

# VERSION

version 0.001

# Perl::Critic::Policy::Variables::ProhibitUselessVarClearing

A lexical goes away at the end of the block that declares it.  Perl frees it
then, and nothing that comes after the block can reach it.  So emptying one
that nothing reads again is a statement that does nothing:

```perl
if ($wanted) {
    my %secrets = lookup(%wanted);
    apply( $config, %secrets );
    %secrets = ();                  # violates: the block ends, and so does %secrets
}
```

This is a habit from C, where memory stays until something frees it.  In Perl,
the end of the scope frees it, as `perldoc perlguts` says under garbage
collection.  Clearing it by hand costs a statement, and a reader has to look
for the later read that would explain it, which does not exist.

## PROHIBITED

A statement that only empties a `my` variable, when the rest of the block that
declares the variable never reads it again:

```
%h = ();
@a = ();
$x = undef;
$hr = {};
$ar = [];
undef %h;
undef @a;
undef $x;
undef($x);
```

## ALLOWED

```perl
%h = ();  f(%h);                    # read again later
my %h;  $r = \%h;  %h = ();         # a reference can still reach it
my @a;  my $f = sub { @a };  @a = ();   # so can a closure
my %h;  for (@x) { ...; %h = () }   # the next pass reads it
our %h;  %h = ();                   # not a lexical
state %h;  %h = ();                 # lives across calls
my %h;  sub f { %h = () }           # the next call reads it
```

A mention of the variable inside a string, a regular expression or a heredoc
counts as a read, whether or not it interpolates.

## WHAT IT CANNOT SEE

Emptying what a reference points to is not reported:

```
%$hr = ();
@{$ar} = ();
undef %$hr;
```

That empties a hash or an array that other code can also hold a reference to,
and nothing in the source says whether it does.  So the statement can matter
even when nothing here reads `$hr` again.  Giving the scalar a new, empty
reference, as `$hr = {}` does, changes only the scalar, and is reported.

## WHEN THE CLEARING DOES SOMETHING

Emptying a variable runs the `DESTROY` of what it held at that moment, rather
than at the end of the block.  Releasing a lock or closing a handle before the
slow part of a block is a reason to clear a variable early.  Say so where it
happens:

```
undef $lock;    ## no critic (ProhibitUselessVarClearing) -- release the lock before the upload
```

# CONFIGURATION

None.

## METHODS

What [Perl::Critic::Policy](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3APolicy) asks of a policy, answered here rather than called
from anywhere.

### supported\_parameters

None.

### default\_severity

Low: the statement does nothing, and costs only a reader's time.

### default\_themes

`maintenance`.

### applies\_to

A plain statement, which is what an assignment or a call to `undef` is.

### violates

Reports a statement that only empties a lexical, when nothing can read the
lexical again before the end of the block that declares it.

# AUTHORS

Current Maintainers:

- George S. Baugh <george@troglodyne.net>

# COPYRIGHT AND LICENSE

Copyright (c) 2026 Troglodyne LLC

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:
The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
