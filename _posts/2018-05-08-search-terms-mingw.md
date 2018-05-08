---
title: Search terms when compiling with only base, non-POSIX minGW
layout: single
toc: true
---

## Summary

This post can really be summarized quite simply: compiling with base minGW when the original codebase relies on POSIX runtime environments is basically about knowing what to Google.

The best Google search terms I found for solving my compiler errors was: specific variable or function causing error + a combination of 'non-posix', 'implementation', and 'equivalent'.

The worst Google search terms for solving my compiler errors was: search the actual error message. The second worst was to search the actual error message with mingw.

The rest of the post is about how I ended up with this specific minGW-POSIX-incompatibility problem in the first place, what even that problem is, my approach to dealing with it, and a more detailed explanation of tips and tricks I found.

## Introduction

I recently wanted to make use of the C++ library [SeqAn](https://www.seqan.de/) in R and figured I should start by bundling the headers together as an R package. The package is very inventively called [RSeqAn](https://github.com/compbiocore/RSeqAn) and can be downloaded from [Bioconductor](http://bioconductor.org/packages/devel/bioc/html/RSeqAn.html).

When prepping to put RSeqAn onto Bioconductor, I needed it to build and pass checks on Linux, macOS, *and* Windows. Linux and macOS was a breeze, facilitated by [Rcpp](http://www.rcpp.org/). Windows, however, was not. SeqAn has support for Windows if using Visual C++, ICC, or Clang, but not MinGW. This is unfortunate because [Rtools](https://cran.r-project.org/bin/windows/Rtools/) builds with [MinGW-w64](https://mingw-w64.org/doku.php). Thus, any package that aspires to make it onto Bioconductor (or for that matter, CRAN) needs to build and pass checks on MinGW.

My experience was very frustrating. For me it came down to three things:

1. I don't have a Windows machine, so I was testing by running it through `devtools::build_win()`, which sends your package to the [win-builder](https://win-builder.r-project.org/) system and emails you the build and check logs 30-60 minutes later.

2. The library I was repackaging makes heavy use of POSIX threads. I did not know what POSIX standards were and did not realize minGW does not provide a POSIX runtime environment.

3. I didn't understand that minGW *is* GCC, just without POSIX. A lot of the solutions I tried had to do with Win32 or MSVC. They didn't work for me.

The last point was the one that took me the longest to learn, because at first I would just Google the error message I received, like any halfway decent programmer. I would end up with messages that told me that other people experienced the same issues, but the solutions were wide and varied, from 'install this from msys' to 'this variable is Unix-only, find another way'. Once I figured out this 3rd point however, getting through all the error messages and building the package on Windows became a lot simpler. In the end it really is about knowing how to Google...

{% include base_path %}

{% capture fig_img %}
![Meme: Not sure if good at programming or great at googling]({{ basepath }}/assets/img/not_sure_if_good_at_programming.jpeg)
{% endcapture %}

<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
</figure>

Below is the approach I took, i.e. the macros I defined, the search terms I used that were helpful, the search terms I used that *were not* helpful, and any other general miscellaneous tips. I am not a C++ expert, I know just enough about references and pointers to get me in trouble. So, there might be a much smarter approach, but this one worked well enough for me.

## Defining macros

First, you need to know what compiler and/or OS is reading your code. Without it you will not be able to do anything. There is a list of [predefined macros for compilers](https://sourceforge.net/p/predef/wiki/Compilers/), so in your code whenever you want to write something in the code that is specific to the compiler or OS you just go

~~~~cpp
#ifdef __COMPILER__
// do something specific to compiler
#endif
~~~~

When it comes to mingw, the definition is `__COMPILER__ = __MINGW32__`. So every time I encountered an error while compiling, I left that code as is (because it worked on my Mac and on the Linux server) but wrapped the solution in

~~~~cpp
#ifdef __MINGW32__
// do something specific to compiler
#endif
~~~~

At first I also tried just defining code specific to OS, i.e. `#ifdef _WIN64`, however this did *not* work for me.

### A word about compound conditionals <a name="cond"></a>

There were a few instances in the code where I had to use compound conditionals, because I needed to include a header that was for both MSVC and mingw. (Could I have used a Windows OS macro instead? Perhaps... but like I said, working solution not simple solution.) There were also a few instances in which I was told by the SeqAn dev team that the code was old and unnecessary, and so that it was alright to wrap the code in a macro so that the code is only evaluated if mingw was *not* defined. (Could the SeqAn dev team have just gotten rid of this legacy code? Also perhaps...)

This is when I learned that [there is a difference between `#ifdef` and `#if defined`](https://stackoverflow.com/questions/1714245/difference-between-if-definedwin32-and-ifdefwin32), primarily that `#ifdef` is for a single condition only, and `#if defined` can do compound conditionals as well. As a result of this:

* if you use `#ifdef` or `#ifndef` you should **not** wrap the conditional variable in parentheses

* if you use `#if defined` then you **have to** wrap the conditional variable in parentheses *and* each variable `x` needs a separate `defined(x)` evaluation path.

Unfortunately I struggled with this for quite some time too, doing things like `#ifndef(__MINGW32__)` and `#if defined(STDLIB_VS || __MINGW32__)`, both of which had no effect on the code. The proper way was `#ifndef __MINGW32__` and `#if defined(STDLIB_VS) || defined(__MINGW32__)`.

## Do not search for the error itself

Searching for the error itself is not very helpful and neither is appending mingw to it. The reason why is that if the code builds and passes checks on Linux and MacOS already, then the issue is almost certainly due to mingw/Windows not having support for that particular piece of code. Additionally, if you search for the error + mingw, you do get solutions but they either require msys or are header files/implementations/patches to download and place in your codebase. Downloading msys was not an option for me, and having a bunch of random files in my codebase for the sake of mingw comptability was not appealing to me.

Furthermore, several of the threads were quite old, going back up to 10 years. This meant that the solutions were not relevant anymore, especially since there's been a lot of changes to mingw-w64 in the last few years and folks are also on the C++11 standard now. I did try several of the suggestions in these threads, and they didn't really work.

## General useful search tips and terms

### Term + either 'non-posix', 'implementation non-posix' or 'equivalent'

Since mingw is not an entirely different language, just lacking POSIX support, the most helpful things to search for are non-POSIX versions or implementations of whatever the problematic code is.

As an example, I was having a really hard time solving an error with `mkstemp` evidently not having a non-POSIX implementation:

~~~cpp
error: 'mkstemp' was not declared in this scope
     int _tmp = mkstemp(fileNameBuffer);
~~~

After googling fruitlessly with the actual error as well the error + mingw implementation I realized that what I needed was not a *mingw* implementation, but just equivalent C++ code that didn't rely on POSIX standards. Lo and behold, searching in Google for `mkstemp non-POSIX implementation` gave me [one solution](https://stackoverflow.com/questions/6036227/mkstemp-implementation-for-win32), and `mkstemp non-POSIX equivalent` gave me [another](https://stackoverflow.com/questions/7778889/what-is-the-c-standard-library-equivalent-for-mkstemp). I chose to go with the second, because I would rather write an alternative than define an implementation. It worked!

This was my alternative: <a name="mkstemp"></a>
~~~cpp
#ifndef __MINGW32__
int _tmp = mkstemp(fileNameBuffer);
(void) _tmp;
#else
tmpnam(fileNameBuffer);
std::fstream _tmp(fileNameBuffer);
#endif
~~~

### Sometimes it's about a missing header

Some of the alternative implementations require also including certain headers. If you make a change and you receive an error about the change but in a different context, *and* you don't set off a whole host of other errors, then it might just be a header issue. For example, with my solution to [mkstemp](#mkstemp), after writing it I got the error

~~~cpp
error: variable 'std::fstream _tmp' has initializer but incomplete type
     std::fstream _tmp(fileNameBuffer);
~~~

Turns out it was just because `fstream` [needed to be included](https://stackoverflow.com/questions/15080231/c-variable-stdifstream-ifs-has-initializer-but-incomplete-type).

### sys/mman.h

The exception to not downloading files and placing them in my codebase was `sys/mman.h`. This is because it seemed to be such a common error with a universally recommended solution. The other difference is that while most of my errors were about functions being out of scope because they don't have Windows implementations, this particular error was about an entire header not existing. I certainly wasn't going to redefine everything that made use of that header, so I just downloaded the [mman-win32](https://github.com/witwall/mman-win32) library recommended by everyone, placed everything from `mman.c` into `mman.h`, placed my modified `mman.h` into RSeqAn, and went on with my day.

## In conclusion...

So that is all the things I learned about macros, search terms, and mingw in order to get a working copy of SeqAn for R on Windows. It was quite a labor so I'm satisfied that I was eventually able to get a working copy. I just wanted to end with two last things I was reminded of while working on this. Despite them being common sense I think they can be easy to forget.

### Don't forget that you can always ask around!

I posted for help to the SeqAn and Bioconductor developers listservs. Several times I considered posting on StackOverflow, but eventually I always found the answer, mostly on StackOverflow. The Bioconductor developers listserv was not particularly helpful, but the SeqAn dev team was great. They offered me several points where their code was outdated and unnecessary, so that if I wrapped it in a macro that would skip the code if mingw was defined (see [A word about compound conditionals](#cond))

### If custom C++ for Rcpp, keep it POSIX-free to begin with

If you are writing custom C++ code for Rcpp and not refactoring someone else's code, then it will be really helpful to keep it POSIX-free to begin with! I am working on some C++ code for upcoming bioinformatics quality control R packages, which is why I wanted to package RSeqAn to begin with, and I am definitely keeping this in mind so that it won't run into any serious issues on Windows. Wishing my self best of luck with that one...