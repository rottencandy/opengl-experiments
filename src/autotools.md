Writing a build system under Linux
==================================

Building software under Linux is usually done with `make`, a basic build
tool which, at its core, takes a set of "rules", each of which describe
how to build a "target" (a program) from its "dependencies" (source
files). Make is smart enough to figure out that if a target is more
recent than its dependencies, it doesn't need to be rebuilt, saving
time.

While it's possible to write makefiles manually, this gets rather old
rather fast. For that reason, the GNU project wrote the "autotools": a
set of scripts that generate more scripts that generate makefiles (still
with me? Okay, breath. It's not as bad as it sounds. Honest!)

The autotools consist of three major components:
- `autoconf`, which generates the "configure" script that detects where
  used libraries are and allows to reconfigure the build system;
- `automake`, which generates a makefile
- `libtool`, which adds some juice to generate libraries (shared or
  static). We won't be dealing with that one.

In addition, the `pkg-config` tool is often used in conjunction with
autotools to figure out where to find libraries. Before `pkg-config`, it
was often necessary to add large amounts of boilerplate to the
`autoconf` input file in order to look for and link in a non-system
library; while system libraries are required to be found in the default
search path for the compiler, the same is not true for additional
libraries, so every autoconf script had to look for the installation
directory of such libraries, which was a lot of work. With `pkg-config`,
however, this can be done with three lines (one for `autoconf`, and two
for `automake`).

autoconf
--------

In order to compile a simple C++-based OpenGL program, we need to have a
configure script that looks for a C++ compiler and the OpenGL libraries.
First we need to write a file `configure.ac`, the input for autoconf.
Open your favourite editor, write the following lines, and save it to a
file called `configure.ac`:

    AC_INIT([myprogram],[1.0],[you@example.com])
    AC_CONFIG_SRCDIR([main.cpp])
    AM_INIT_AUTOMAKE([foreign])
    AM_MAINTAINER_MODE([enable])

    AC_PROG_CXX

    PKG_CHECK_MODULES([gllibs], [gl glew glfw3 >= 3.0])

    AC_CONFIG_HEADERS([config.h])
    AC_CONFIG_FILES([Makefile])
    AC_OUTPUT

That seems like a lot, but it's not very hard.
`autoconf` is written in M4, an extensible macro language; M4 is very
similar in concept and operation to the C/C++ preprocessor. The input
file for `autoconf` gets processed by m4 into a working `configure`
script. In order to configure things, however, we also need to
configure a few things related to automake. This is done with extra
macros in the configure.ac file.

Whenever a macro starts with `AC`, it's an `autoconf` macro. When it
starts with `AM`, it's an automake macro. The convention is for other
packages, when they add macros, to use their own prefix; `pkg-config`
uses the prefix `PKG`.

Let's go over the lines in the configure.ac file:

- In the `AC_INIT` line, you should enter the name of your program
  (`myprogram` in the example), its version (here I've used `1.0`),
  and some address where you want bugreports to go (this doesn't have
  to be an email address).
- The `AC_CONFIG_SRCDIR` allows autoconf to verify if it's in the source
  directory. The argument to this macro should be the name of a file or
  directory that exists in the original source directory. The exact file
  doesn't matter, so long as it exists.
- In the `AM_INIT` line, we can set options for automake. In our case,
  we specify that we aren't a GNU project and thus don't need to follow
  their strict ruleset.
- The `AM_MAINTAINER_MODE` enables `automake`'s maintainer mode. With
  it, whenever you change something in the autotools input files, your
  build system will be automatically rebuilt for you the next time you
  run `make`. This ensures that your changes to `configure.ac` or
  similar aren't overlooked because you forgot to re-run `autoconf`.
- Next, we use the `AC_PROG_CXX` to look for a C++ compiler. C++ is
  rendered as CXX for historical reasons.
- The next line looks for our OpenGL libraries using pkg-config. The
  `PKG_CHECK_MODULES` macro takes two arguments:
  1. A name; this should be unique in the build system. It's up to you
     what you use for this name, but it's conventional to reuse the
     library name here. The name can only contains letters and numbers,
     however (no spaces or other characters).
  2. The pkg-config module names that you need for this particular
     library. It's possible to list multiple libraries in one go, as
     above, or you can have one `PKG_CHECK_MODULES` line per library.
     The latter is more useful if you have multiple programs to compile,
     and not all of them need all the libraries; if that isn't the case,
     however, it's better to just use a single `PKG_CHECK_MODULES` call.
     If you want to add another library, and you don't know what the
     pkg-config module name is, just run `pkg-config --list-all`, which
     will tell you. Beware, the list may be fairly long.
- Finally, the last three lines of our `configure.ac` are responsible
  for generating the output. The `AC_CONFIG_HEADERS` line tells
  `autoconf` that we want our `configure` script to generate a C/C++
  include header named `config.h` based on a template (which we will
  generate shortly) and the result of the tests it ran; the
  `AC_CONFIG_FILES` macro tells `autoconf` that our script will also
  need to combine the output of `automake` and the same test results to
  generate a Makefile; and finally, the `AC_OUTPUT` file tells
  `autoconf` to insert the stuff that actually generates those outputs.

automake
--------

Now that we've written instructions for `autoconf` to figure out where
everything is, we still need to tell `automake` what to do with that
information. If the `autoconf` input wasn't all that long, the
`automake` input is even shorter. In your favourite editor, write a file
called `Makefile.am` (not `makefile.am`!) with the following contents:

    bin_PROGRAMS = myprogram
    myprogram_SOURCES = main.cpp
    AM_CXXFLAGS = @gllibs_CFLAGS@
    AM_LDFLAGS = @gllibs_LIBS@ -lm

The first line declares that you want to install a program into a `bin`
directory (`/bin`, `/usr/bin`, or `/usr/local/bin`), i.e., available for
normal users (as opposed to `sbin_PROGRAMS`). If we want our package to
compile multiple programs into the `bin` directory, it's possible to
specify multiple programs here (just separate them with whitespace).

The second line says that in order to generate `myprogram`, the build
system needs to compile (and link) `main.cpp`. Again, if you have
multiple sources for a single program, list them all, space-separated.
Note that in `Makefile.am`, if a line gets very long, you can always use
a backslash (`\`) as the last character on a line to extend it:

    bin_PROGRAMS = myfirstprogram mysecondprogram \
        mythirdprogram

The next two lines instruct automake to pass on the information which
`autoconf`, using `pkg-config`, found about the libraries that we
instructed it to look for. These two variables, `gllibs_CFLAGS` and
`gllibs_LIBS` contain the command-line options that need to be specified
to the compiler (`AM_CXXFLAGS`/`name_CFLAGS`) and the linker
(`AM_LDFLAGS`/`name_LIBS`). Automake does a simple substitution;
whenever it finds anything between two at signs, it looks for an
`autoconf` output variable with the name that is between the two at
signs, substitutes them. In addition, it's possible to manually add
other options, such as the `-lm` we've done above (which is necessary in
order to be able to use certain &lt;cmath&gt; functions, such as `sin()`
or `fabs()`).

If you're compiling multiple programs and some need different libraries
than others, you need to use (in our example) `myprogram_CXXFLAGS` and
`myprogram_LDFLAGS` instead. In addition, it's also possible to add some
other compile- or link options to the line by just adding them 

Building
--------

Now that we've written `configure.ac` and `Makefile.am`, we're ready to
generate the build system and build everything. In the shell, ensure
that you're in the directory where your source is, and then run the
following:

    autoreconf -i
    ./configure
    make

The first line will run autoconf and automake. The `-i` argument tells
the tools to copy any missing files to the local directory.

The second line will check whether all libraries can be found, and
generate the Makefile.

If all goes well, you should now have a compiled version of your
program.


// ============================================================================
// Copyright 2015, Wouter Verhelst
// 
// This text is licensed under the Creative Commons Attribution 3.0
// Unported License. To view a copy of this license, visit
// http://creativecommons.org/licenses/by/3.0/.
//
// To the extent possible by applicable law, I hereby waive all copyright
// and related or neighboring rights to the code samples in this documents,
// and publish them into the public domain.
// ===========================================================================
