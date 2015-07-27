# |DRAFT-TITLE|

* [Editor's copy](https://|THIS-USER|.github.io/iceTrav/)
* [Working Group Draft] (https://tools.ietf.org/html/draft-jennings-behave-rtcweb-firewall)


## Contributing

Be aware that all contributions to the specification fall under the "NOTE WELL"
terms outlined below.


## Building the Draft

Formatted text and HTML versions of the draft can be built using `make`.

```sh
$ make
```

This requires that you have the necessary software installed.  There are several
other tools that are enabled by make, check the Makefile for details, including
links to the software those tools might require.


## Installation and Setup

Mac users will need to install
[XCode](https://itunes.apple.com/us/app/xcode/id497799835) to get `make`, see
[this answer](http://stackoverflow.com/a/11494872/1375574) for instructions.
Some of the makefile targets need GNU make 4.0, which Apple doesn't ship yet;
sorry, but if you want those, you can
[download](https://www.gnu.org/software/make/) and build a copy for yourself.

Windows users will need to use [Cygwin](http://cygwin.org/) to get `make`.

All systems require [xml2rfc](http://xml2rfc.ietf.org/).  This requires [Python
2.7](https://www.python.org/).  The easiest way to get `xml2rfc` is with `pip`.

Using a `virtualenv`:

```sh
$ virtualenv --no-site-packages venv
# remember also to activate the virtualenv before any 'make' run
$ source venv/bin/activate
$ pip install xml2rfc
```

To your local user account:

```sh
$ pip install --user xml2rfc
```

Or globally:

```sh
$ sudo pip install xml2rfc
```

xml2rfc depends on development versions of [libxml2](http://xmlsoft.org/) and
[libxslt1](http://xmlsoft.org/XSLT).  These packages are named `libxml2-dev` and
`libxslt1-dev` (Debian, Ubuntu) or `libxml2-devel` and `libxslt1-devel` (RedHat,
Fedora).

If you use markdown, you will also need to install `kramdown-xml2rfc`, which
requires Ruby and can be installed using the roby package manager, `gem`:

```sh
$ gem install kramdown-xml2rfc
```


## NOTE WELL

Any submission to the [IETF](https://www.ietf.org/) intended by the Contributor
for publication as all or part of an IETF Internet-Draft or RFC and any
statement made within the context of an IETF activity is considered an "IETF
Contribution". Such statements include oral statements in IETF sessions, as
well as written and electronic communications made at any time or place, which
are addressed to:

 * The IETF plenary session
 * The IESG, or any member thereof on behalf of the IESG
 * Any IETF mailing list, including the IETF list itself, any working group
   or design team list, or any other list functioning under IETF auspices
 * Any IETF working group or portion thereof
 * Any Birds of a Feather (BOF) session
 * The IAB or any member thereof on behalf of the IAB
 * The RFC Editor or the Internet-Drafts function
 * All IETF Contributions are subject to the rules of
   [RFC 5378](https://tools.ietf.org/html/rfc5378) and
   [RFC 3979](https://tools.ietf.org/html/rfc3979)
   (updated by [RFC 4879](https://tools.ietf.org/html/rfc4879)).

Statements made outside of an IETF session, mailing list or other function,
that are clearly not intended to be input to an IETF activity, group or
function, are not IETF Contributions in the context of this notice.

Please consult [RFC 5378](https://tools.ietf.org/html/rfc5378) and [RFC
3979](https://tools.ietf.org/html/rfc3979) for details.

A participant in any IETF activity is deemed to accept all IETF rules of
process, as documented in Best Current Practices RFCs and IESG Statements.

A participant in any IETF activity acknowledges that written, audio and video
records of meetings may be made and may be available to the public.
