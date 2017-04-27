# Upgrading Perl to a Modern Version, the ActiveState Guide

Many organizations stopped upgrading Perl after the release of 5.8.8 or
5.10.1, about 8 to 11 years ago. At that point in history, major releases of
Perl could take many years to come to fruition. To make things more confusing,
point releases of a given version sometimes contained large changes (including
backwards incompatibilities) that we'd expect in major releases. For example,
5.10.1, released 20 months after 5.10, contained a number of backwards
incompatibilities and quite a few new features.

This made upgrading Perl scary and unpredictable. It also made working on the
Perl *core* challenging. What was an appropriate change for any release? When
would changes see the light of day?

To fix all that, Perl 5 pumpking Jesse Vincent instituted a new release
plan. Major releases (5.12, 5.14, etc.) would come out once a year in
April. In between, development version point releases (5.11.0, 5.11.1) would
be made once a month.

After a major release comes out (5.22.0, 5.24.0, etc.) there may be additional
point releases (5.22.1) as needed, but only for two years after the initial
release. After that period, support ends except for critical security fixes,
for which an additional year of support is provided.

Note that when I say "support" I mean that the community of people working on
Perl as a whole will work on bugs and security issues during this time
frame. This is not commercial support. If you want that, well, that's one of
the reasons to become an ActiveState customer!

Jesse's plan worked, and we've seen yearly releases in April (or May or June)
every year from 2010's 5.12.0 to 2016's 5.24.0.

The upshot of all this is that major Perl releases are much smaller and safer
than they used to be. Point releases are *extremely* safe, and we recommend
always upgrading when a new point release comes out.

But if you haven't upgraded for ages, you're still in a difficult
situation. How do you get from there to here? What should you look out for?
And why should you even upgrade at all?

## Features and Fixes

Let's tackle that last question first. Why upgrade?

The simple answer is that Perl has seen quite a few new features in the past
few years, ranging from little tweaks (the `s///r` modifier) to "this really
changes all my code for the better" (subroutine signatures). Let's take a look
at some of the highlights of the last six years, from 5.12 through 5.24.

### Y2038 Handling

If you're my age, you remember the Y2K efforts of the late 90s. While none of
the predicted catastrophes came to pass on January 1, 2000, that was in no
small part because of the countless software developers who worked to prevent
such catastrophes!

Well, there's another time-related bug swiftly approaching,
the [Year 2038 problem](https://en.wikipedia.org/wiki/Year_2038_problem). Many
programs stored their dates and times as seconds since the Unix epoch (January
1, 1970 UTC). In C this is the `time_t` type. If the epoch value is stored in
a signed 32-bit integer, this means that you have 2,147,483,647 seconds
available. That sounds like a lot, but that epoch value corresponds to January
19, 2038 at 03:14:07 UTC. That's not far from now at all!

And to make matters worse, it's easy to do calculations today that will go
well past 2038. Imagine calculating the interest and principal payments for a
30-year mortgage starting today.

Fortunately, there are many efforts to ameliorate this problem, and 64-bit
platforms today use a 64-bit integer for `time_t`. The Perl core got a head
start on this in the 5.12 release by making sure that time is always
represented with something much larger than a 32-bit integer internally,
regardless of the platform's integer size.

### Regexes and Unicode

Perl 5.14 added a number of new modifiers for regexes and substitutions. The
`/a` modifier restricts the `\s`, `\d`, and `\w` character classes to ASCII
only. This fixes a bug you might not have even known you had:

    if ( $string =~ /^\d+/ ) { ... }

Did you know that by default, `\d` will match *any* Unicode number, including
৪. That's a Bengali number four, not the Arabic 8! These sort of "look-alike"
Unicode characters have been the source of several security
vulnerabilities. The new regex modifiers make it easier to write secure code
in Perl.

### Non-Destructive Substitution

Perl 5.14 added a new `/r` modifier for substitutions. With this modifier, the
substitution operator returns a new string rather than modifying the original:

    my $new = $old =~ s/new/old/gr;

This was always possible using a much uglier idiom, but this operator is much
clearer.

### The `fc` Operator

Comparing Unicode strings case-insensitively can be tricky. Unicode has a
concept called "foldcase" which addresses this. When you apply foldcasing to
two strings you can compare them with confidence. Perl 5.16 added an `fc`
operator to help you do just that.

### Hash Randomization

The order of the items returned by `keys`, `values`, and `each` was randomized
for each call starting in 5.18. This makes Perl hashes more robust against
algorithmic complexity attacks, which can be used to cause a denial of
service.

Note that we'll cover this again later in the section on what you need to
watch out for!

### Subroutine Signatures

This was added as an experimental feature in 5.20, and is one of my favorite
additions to Perl in recent years. If you enable the signatures feature you
can write code like this:

    use feature 'signatures';
    no warnings 'experimental::signatures';
    sub run ( $self, $command, %opts = () ) { ... }

This simple feature goes a long way towards improving the readability of your
Perl code!

### Hash and (New) Array Slices

Arrays have supported a slice syntax for a long time:

    my ( $first, $second ) = @array[0, 1];

But now you can do this with hashes too:

    my %subset = %hash{ 'foo', 'bar' }

Pretty neat! But you can also do this with arrays:

    my @array = ( 'a'..'z' );
    my %int_hash = %array{ 1, 2 }
    # %int_hash is ( 1 => 'b', 2 => 'c' )

### Postfix Dereferencing

Postfix dereferencing is another new experimental feature in 5.20, and it has
since been marked as stable in 5.24. Perl has rightly been criticized for the
ugly syntax require to dereference a complex data structure. For example:

    my @elems = @{ $foo->{bar}[0]{baz} };

That outer `@{ ... }` wrapper breaks the nice left to right dereferencing
syntax of the contained expression. With postfix dereferencing, we can make
the syntax consistent (if not beautiful):

    my @elems = $foo->{bar}[0]{baz}->@*;

The difference is even more apparent when you compare the slice syntax:

    my @subset = @{ $foo->{bar}[0]{baz} }[0, 2, 3];
    @subset = $foo->{bar}[0]{baz}->@[0, 2, 3];

It's not elegant, but on balance, I find this a bit more readable than the `@{
... }` wrapper version. This works for any sort of reference, including
scalars, arrays, hashes, subs, and globs.

### Non-Capturing Regex Grouping

Are you annoyed at writing `/(?:foo|bar)/` all the time just to avoid the
speed hit of capturing in regexes? I know I am! Perl 5.22 adds a new `/n`
regex modifier which disables all capturing, so you can write `/(foo|bar)/n`
instead.

### Unicode

This isn't specific to any particular release, but rather is part of every
major release. Perl keeps new releases in sync with the Unicode specification,
meaning that if you want to be able to use the full spectrum of Unicode
characters in your code, you need to upgrade. Perl 5.10.1 used version 5.1.0
of the Unicode character database. Perl 5.24 uses version 8.0 of that database.

And of course, there have been many Unicode bugs fixed since 5.10.1. Karl
Williams, a core developer who's put a huge amount of work into Perl's Unicode
support, recommends a minimum of 5.14 for serious Unicode processing.

### And So Much More

This is a very quick skim of the updates in major releases from 5.12 through
5.24. There's a lot I didn't even mention. You can read through the perldelta
docs yourself for more details:

* [Perl 5.12.0](https://metacpan.org/pod/distribution/perl/pod/perl5120delta.pod)
* [Perl 5.14.0](https://metacpan.org/pod/distribution/perl/pod/perl5140delta.pod)
* [Perl 5.16.0](https://metacpan.org/pod/distribution/perl/pod/perl5160delta.pod)
* [Perl 5.18.0](https://metacpan.org/pod/distribution/perl/pod/perl5180delta.pod)
* [Perl 5.20.0](https://metacpan.org/pod/distribution/perl/pod/perl5200delta.pod)
* [Perl 5.22.0](https://metacpan.org/pod/distribution/perl/pod/perl5220delta.pod)
* [Perl 5.24.0](https://metacpan.org/pod/distribution/perl/pod/perl5240delta.pod)

## Land Mines Along the Way

It's not just features you get with an upgrade. You also get some backwards
incompatibilities and other potential breakage. Here are some things that you
need to watch out for.

### Smartmatch is In Limbo

The smartmatch feature was first added in 5.10.0, and later revised in
5.10.1. It was then marked as experimental in 5.18.0. In 2016 there was a
discussion about whether to revise or kill the feature that has not yet been
resolved.

Smartmatch may return as stable in a future release, but if so it will be in a
simplified form. For now, I would suggest removing the use of smartmatch
(`given`/`when` and `~~`) from any production code before upgrading your
Perl. Some alternatives include the list comprehension functions like `any`
from
[`List::Util`](https://metacpan.org/pod/List::Util),
[`Switch::Plain`](http://code.activestate.com/ppm/Switch-Plain/), and good old
`if`/`elsif`.

### Importing from `UNIVERSAL` is Forbidden

Starting with Perl 5.12, importing things like `isa` and `can` from
`UNIVERSAL` was deprecated. In 5.22 this became a fatal error. You should
always call these subroutines as class or object methods.

### Storable as a Data Interchange Format

If you're using the `Storable` module to serialize and thaw data between Perl
processes, or worse, you are saving Storable-serialized data in files, a
database, or cookies, then you need to be very careful with your upgrade
process. This module does not guarantee binary compatibility across
releases. If you pass data from a newer version of `Storable` to an older
version, the older version will always die. This module is part of the core,
so upgrading your Perl will upgrade your `Storable`.

One possibility is to upgrade all of your systems to the most recent version
*before* you upgrade your Perl. Then as you upgrade a given system, you can
re-install the latest Storable, ensuring compatibility between systems.

A better, more permanent fix, is to not use `Storable` this way at all!
`Storable` is a poor choice for data interchange. Instead, consider switching
to [`Sereal`](http://code.activestate.com/ppm/Sereal/), JSON, or YAML.

### Hash Randomization

As I already mentioned, the order of the items returned by `keys`, `values`,
and `each` was randomized starting in 5.18. This is good, but it has the
potential to break your code. In my experience, the most vulnerable code is
typically test code that looks like this:

    use Test::More;
    is_deeply( [ keys %hash ], [ 'foo', 'bar' ], ... );

This worked in older Perls because once you knew the stable order that keys
were returned in, you could hard code this in tests. Starting with Perl 5.18,
these tests will fail. Sometimes. This leads to all sorts of fun to debug
problems as you try to figure out why your test suite is failing on random
changes unrelated to the failing test.

The fix, of course, is to use `sort` whenever you get the `keys` or `values`
from a hash:

    use Test::More;
    is_deeply( [ sort keys %hash ], [ 'bar', 'foo' ], ... );

This could also come into play as an order of operations bug in non-test
code. Again, this will manifest as subtle intermittent failures.

### Assigning to `$0` Sets the Legacy Process Name

What does that mean? The short answer is that in old versions of Perl, setting
`$0` would not change the program's name in things like `top` or for the
purposes of `killall`. If you relied on being able to monitor or kill things
based on the program name at startup, you should check your code base for
assignmnent to `$0`.

On the plus, side, this is really a feature, since developers can set `$0` in
order to help the ops folks understand what they are seeing in `ps` and `top`.

### `Devel::DProf` is dead

This module was removed from the core in 5.14, and it is no longer
maintained. Use
[`Devel::NYTProf`](http://code.activestate.com/ppm/Devel-NYTProf/) for your
profiling needs instead.

### And More

As with features, there many other smaller changes. Most of these will
manifest as either warnings, compilation errors, or runtime errors.

For the warnings, it's very important that you capture all output from your
code when you test with an upgraded Perl. These warnings go direct from the
Perl interpreter to `stderr`. It's possible to intercept them, but not easy to
do this for all code consistently. Make sure that `stderr` isn't going to
`/dev/null`! The same caveat applies to catching runtime errors.

The forbidden syntax is easier to spot, since this will cause the interpreter
to die during compilation.

## Upgrading Best Practices

We recommend that you first start by trying ActivePerl 5.24.1 (or if you're in
the future, the latest stable release we offer). Upgrading to intermediate
releases one at a time is a lot of work, and a problem surfaced in 5.16 may be
fixed in 5.20 anyway.

If you have test suites (of course you have test suites) then the easiest
place to start is by running the suite under the latest Perl.

Once your tests pass and are warnings-free under the new Perl, you can then
upgrade a staging machine or two, then a few more, and so on.

Something you should be paying attention to both in the test suite and on
staging machines is the performance characteristics of your Perl code. While
recent versions of Perl include some new optimizations, they may also be
slower in some areas. These changes are hard to predict, so there's no
substitute for monitoring CPU usage, disk usage, memory usage, etc.

Once you've squashed the problems surfaced in staging, you can take the same
incremental approach to production. Upgrade one or two machines and monitor
them closely. Then upgrade a few more, and so on.

If you get really stuck with 5.24.1, you might consider falling back to an
earlier release like 5.20 as a stopgap, but upgrading to any recent release
from 5.8 or 5.10 will be a major project, so you might as well aim for all the
new shiny bits while you're at it.

Fortunately, once you've bitten the bullet and done the big leap, future
upgrades will be much easier. We recommend that you plan to upgrade your Perl
at least every other major release. Given the relatively smaller delta between
major releases these days, skipping one major release is fine. Skipping more
than one could tempt you to skip many, and suddenly you're back here, planning
a big project to update from an ancient version of Perl.

The downside to skipping *any* releases is that you may miss out on
deprecation warnings telling you about something that will go away in the next
version.

You can make upgrading simple by doing it often. That way it becomes routine
rather than an infrequent and very painful chore.

## Thanks

Thanks to the denizens of #p5p on irc.perl.org for reviewing an earlier draft
of this article and providing valuable feedback, including Joel Berger,
Christian Hansen, Dominic Hargreaves, and Karl Williamson, among others.
