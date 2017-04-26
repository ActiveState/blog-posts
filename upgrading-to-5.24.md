# The ActiveState Guide to Upgrading Perl to a Modern Version

For various historical reasons, many organizations stopped upgrading Perl
after the release of 5.8.8 or 5.10.1. At that point in history, major releases
of Perl could take many years to come to fruition. To make things more
confusing, point releases of a given version sometimes contained the types of
large changes (including backwards incompatibilities) we'd expect in major
releases. For example, 5.10.1, released 20 months after 5.10, contained a
number of backwards incompatibilities and quite a few new features.

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
the reasons you become an ActiveState customer!

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
small part because of all the effort put in by countless software developers
to prevent such catastrophes!

Well, there's another time-related bug swiftly approaching,
the
[Year 2038 problem](https://en.wikipedia.org/wiki/Year_2038_problem). Historically,
many programs stored their dates and times as seconds since the Unix epoch
(January 1, 1970 UTC). If this is stored in a signed 32-bit integer, this
means that you have 2,147,483,647 seconds available. That sounds like a lot,
but that epoch corresponds to January 19, at 03:14:07 UTC. That's not far from
now at all!

And to make matters worse, it's easy to do calculations today that will go
well past 2038. Imagine calculating the interest and principal payments for a
30-year mortgage starting today.

Fortunately, there are many efforts to ameliorate this problem. The Perl core
got a headstart on this in the 5.12 release by making sure that time is always
represented with a 64-bit integer internally, regardless of the platform's
integer size.

### Pluggable Keywords

The "pluggable keywords" API is a new API added in 5.12 that makes it possible
to change how the Perl parser parses a chunk of code. This has led to a number
of interesting syntax experiments on CPAN such
as [Kavorka](https://metacpan.org/pod/Kavorka)
and [Dios](https://metacpan.org/release/Dios).

### Regexes and Unicode

Perl 5.14 added a number of new modifiers for regexes and subtitutions. The
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



### Unicode

This isn't specific to any particular release, but rather is part of every
major release. Perl keeps new releases in sync with the Unicode specification,
meaning that if you want to be able to use the full spectrum of Unicode
characters in your code, you need to upgrade. Perl 5.10.1 used version 5.1.0
of the Unicode character database. Perl 5.24 uses verson 8.0 of that database.


