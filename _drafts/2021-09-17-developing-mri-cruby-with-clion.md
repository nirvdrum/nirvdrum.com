---
layout: post
title: Developing MRI/CRuby with CLion
author: Kevin Menard
---

Introduction
------------

I'm a big fan of the JetBrains family of IDEs.
One of its newer entries is CLion: a cross-platform IDE for C, C++, and other systems languages such as Rust.
CLion really works best with CMake-based applications, which MRI/CRuby<a href="#footnote_1"><sup>1</sup></a> is not.
This post is a how-to guide for getting MRI up and running in CLion.


Setting the Project Up
----------------------

In order to get the most value of CLion, we want to load MRI up as a _Makefile_ project.
But, if you've just cloned the MRI repository, you won't have a _Makefile_ available.
Getting started with the MRI source can be a little daunting.
Fortunately, <a href="https://twitter.com/peterzhu2118">Peter Zhu</a> has written up <a href="https://blog.peterzhu.ca/notes-on-ruby-development/">a helpful guide for MRI development</a>.
Peter's guide will help you get started with all the prerequisites for building the MRI source and has steps on generating the project _Makefile_ using _autoconf_.


Loading a Makefile-based Project in CLion
-----------------------------------------

Once you've gotten through section 2 in Peter's guide, you should have everything you need to load MRI up a _Makefile_ project in CLion.
Start CLion up, and go to _File -> Open_ and from there, you want to point at the _GNUmakefile_ at the root of the MRI source tree.
It's important that you do **not** point at the _Makefile_ in project root because that file does not include all the _make_ tasks.

***** You'll be prompted for the "clean" task. It is actually named _clean_. ******


CLion will detect all of the available _make_ targets and set them up as run configurations.
It will also derive information from the _Makefile_ to try to guess which directories contain source files, allowing it to index their contents and establish relationship for code navigation.


General CLion Configuration
---------------------------

The MRI codebase has been in active development since 1995 and has seen some major shifts in code style and formatting, both within the project and within the wide development community.
Consequently, the codebase has some quirks around its handling of whitespace.
Historically, a single level indentation has been four space characters, but two levels of indentation used one tab character.
The new standard moving forward is to use space characters for all indentation, four spaces per level.
For already written code, tabs can be converted to spaces if other non-white space changes are being made to a particular region of code.
But, the Ruby core team does not want to make one massive change to bulk replace all tabs in the repository with spaces.

All of this is to say that you will want to update CLion's default code formatting rules.
In particular, you will want to instruct CLion to use eight spaces for one tab, as can be seen in Fig. 1.


Debugging
---------

MRI's _make_ configuration includes some tasks for firing up GDB, initializing it with helpers, and running an application.
In order to take advantage of that, you must write whatever expression you'd like to debug in a file name _test.rb_ at the root of the project.
You'd also add your GDB breakpoints in a file named _breakpoints.gdb_, also in the project root.
There are some variables you can override as well &mdash; see _common.mk_ in the project root and look for the _gdb-ruby_ task.

Additionally, the MRI source ships with a set of GDB helpers that make inspecting Ruby values much easier.
For instance, Ruby objects are typed as `VALUE` in MRI, which in turn is an `unsigned long`.
When looking at these values in GDB, they don't tell you very much:

```
(gdb) p str
$1 = 140135618298560
```

One of the GDB helper functions that MRI ships with is called `rp` and it inspects the `VALUE` instance to print out detailed information about the actual Ruby object. For instance, here we can take an otherwise opaque handle and discover that it is a literal `Regexp` object with a pattern string of `/:/` and whatever encoding is associated with encoding index `2`.

```
(gdb) rp re
(null)T_REGEXP(null): {58 ':'} len:1 (literal) encoding:2 $15 = (struct RRegexp *) 0x7f1421784588
```

While `rp` presents data about arbitrary `VALUE` handles, there are datatype-specific helpers as well, such as `rp_string`:

```
(gdb) rp_string str
{50 '2', 48 '0', 50 '2', 49 '1'} bytesize:4 (embed) encoding:1 coderange:7bit $3 = (struct RString *) 0x7f73ddbfc2c0
```

Here we can see the _str_ value is of type `T_STRING` with a value of "2021". It's small enough to be an embedded string<a href="#footnote_2"><sup>2</sup></a>, with an encoding with the index value of `1` and the string's `coderange` value is `CR_7BIT`, indicating the string consists only of ASCII-characters, which in turn implies the encoding is ASCII-compatible.

While driving the debugging process directly from _make_ is somewhat convenient, I really wanted to use CLion's graphical debugger.
I find it considerably more convenient than working with either GDB or LLDB directly.
Just like with any of the other JetBrains IDEs, you can set breakpoints by hotkeys or clicking in the gutter right from the IDE.
You can use hotkeys to drive the debugger instead of using GDB commands.
It's trivial to set up watch expressions and generally easier to see a set of values as you debug, particularly as you navigate between stack frames.
At the same time, I did not want to forgo the GDB helpers that MRI ships with. 

If you prefer to use LLDB with CLion, you'll unfortunately need to take an extra step to get things running.
As of 16-Nov-21, MRI ships with a _.gdbinit_ file, but not an _.lldbinit_ file.
Without that file, the debugger helpers that MRI ships with will not be loaded into LLDB when CLion starts the debugger.

<a name="footnote_1"></a>
<sup>1</sup>
<small>
  The reference implementation of Ruby commonly goes by two names to differentiate it from other Ruby implementations: CRuby and MRI (for Matz's Ruby Interpreter).
  Matz is a nickname for Yukihiro Matsumoto, the creator of Ruby.
  This implementation of Ruby has many more contributors to the language than Matz alone and it has advanced considerably beyond being a simple interpreter, but the MRI moniker is still in active use.
  For the rest of the post, I'll refer to the implementation as MRI.
</small>

<a name="footnote_2"></a>
<sup>2</sup>
<small>
  If you're interested in learning more about how MRI represent its data types, I'd highly recommend getting a copy of "Ruby Under a Microscope" by Pat Shaughnessy.
  For the specific example here of embedded strings, you can learn all about them from the <a href="http://patshaughnessy.net/2012/1/4/never-create-ruby-strings-longer-than-23-characters">Never Create Ruby Strings Longer than 23 Characters</a> post from his blog.
</small>