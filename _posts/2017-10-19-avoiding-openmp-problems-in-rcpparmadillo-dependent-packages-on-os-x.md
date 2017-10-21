---
layout: post
title: Avoiding OpenMP problems in RcppArmadillo-dependent packages on OS X
category: R

excerpt: I recently had problems getting a package with a RcppArmadillo dependency to compile on all platforms, and here is a description of how the problem was eventually solved.
---

Recently, I was preparing a package on my Windows machine at work. Building and checking the package showed no signs of problems, and I was almost ready to submit to CRAN. But before doing so, I decided to use continuous integration via [Travis CI](https://travis-ci.org/) to also test the package on Mac and Linux. Turns out the package did *not* work as intended everywhere.

The problem occurred on OS X and was the dreaded:

    clang: error: unsupported option '-fopenmp'
    
Long story short: the problem is that the package makes use of the Armadillo C++ library for linear algebra via RcppArmadillo. Armadillo has support for OpenMP, but under an out-of-the-box OS X installation, there is *no* OpenMP support. From R version 3.4.0, R provides support for OpenMP. So essentially, R and (Rcpp)Armadillo attempts to use OpenMP, but OS X is unable to handle this ([unless you fix this yourself](http://thecoatlessprofessor.com/programming/openmp-in-r-on-os-x/)).

The solution I ended up using is having a system-dependent `Makevars` file (which sets up links to libraries), which turns on or off OpenMP support. In the following, I will briefly document the steps I took to set this up. 

# The standard `Makevars` file
The standard `Makevars` file I use for `RcppArmadillo`-dependent packages is as follows:

    PKG_CXXFLAGS = $(SHLIB_OPENMP_CXXFLAGS) 
    PKG_LIBS = $(SHLIB_OPENMP_CFLAGS) $(LAPACK_LIBS) $(BLAS_LIBS) $(FLIBS)

As this shows, there are two links here to OpenMP libraries. On Linux, we want to keep this as it is. On OS X (if there is no OpenMP support), we'd like to turn this off. In particular, the following `Makevars` allows the package to compile on OS X by turning off OpenMP:

    PKG_CXXFLAGS = -DARMA_DONT_USE_OPENMP
    PKG_LIBS = $(LAPACK_LIBS) $(BLAS_LIBS) $(FLIBS)

So how can such a system-dependent `Makevars` be created? 

# Using a `configure` script

The main part to get this all up and running is the inclusion of a [configure script](https://cran.r-project.org/doc/manuals/r-release/R-exts.html#Configure-and-cleanup). Essentially, the purpose of the configure script is to configure the `Makevars` file based on the system specifications.

First, I added the following code taken from the `RcppArmadillo` package into the main directory of my package in a file called `configure.ac` (note the name and version of the package in `AC_INIT`):

	## -*- mode: autoconf; autoconf-indentation: 4; -*-
	##
	##  Copyright (C) 2016 - 2017  Dirk Eddelbuettel for
	##  the RcppArmadillo package. Licensed under GPL-2 or later
	##  This file is a subset of the configure.ac used by
	##  RcppArmadillo, adapted to the mfbvar package by
	##  Sebastian Ankargren

	## require at least autoconf 2.61
	AC_PREREQ(2.61)

	## Process this file with autoconf to produce a configure script.
	AC_INIT([mfbvar], 0.3.0.9000)

	## Set R_HOME, respecting an environment variable if one is set
	: ${R_HOME=$(R RHOME)}
	if test -z "${R_HOME}"; then
		AC_MSG_ERROR([Could not determine R_HOME.])
	fi
	## Use R to set CXX and CXXFLAGS
	CXX=$(${R_HOME}/bin/R CMD config CXX)
	CXXFLAGS=$("${R_HOME}/bin/R" CMD config CXXFLAGS)

	## We are using C++
	AC_LANG(C++)
	AC_REQUIRE_CPP

	## Default the OpenMP flag to the empty string.
	## If and only if OpenMP is found, expand to $(SHLIB_OPENMP_CXXFLAGS)
	openmp_flag=""
	openmp_cflag=""

	## Check for broken systems produced by a corporation based in Cupertino
	AC_MSG_CHECKING([for macOS])
	RSysinfoName=$("${R_HOME}/bin/Rscript" --vanilla -e 'cat(Sys.info()[["sysname"]])')
	if test x"${RSysinfoName}" == x"Darwin"; then
	   AC_MSG_RESULT([found])
	   AC_MSG_WARN([OpenMP unavailable and turned off.])
	   openmp_flag="-DARMA_DONT_USE_OPENMP"
	else
	   AC_MSG_RESULT([not found as on ${RSysinfoName}])
	   ## Check for OpenMP
	   AC_MSG_CHECKING([for OpenMP])
	   ## if R has -fopenmp we should be good
	   allldflags=$(${R_HOME}/bin/R CMD config --ldflags)
	   hasOpenMP=$(echo ${allldflags} | grep -- -fopenmp)
	   if test x"${hasOpenMP}" == x""; then
		  AC_MSG_RESULT([missing])
		  openmp_flag="-DARMA_DONT_USE_OPENMP"
	   else
		  AC_MSG_RESULT([found])
		  openmp_flag='$(SHLIB_OPENMP_CXXFLAGS)'
		  openmp_cflag='$(SHLIB_OPENMP_CFLAGS)'
	   fi
	fi

	AC_SUBST([OPENMP_CFLAG], ["${openmp_cflag}"])
	AC_SUBST([OPENMP_FLAG], ["${openmp_flag}"])
	AC_CONFIG_FILES([src/Makevars])
	AC_OUTPUT
	
Next, remove `src/Makevars` and add `src/Makevars.in`:

	PKG_CXXFLAGS = @OPENMP_FLAG@
	PKG_LIBS= @OPENMP_CFLAG@ $(LAPACK_LIBS) $(BLAS_LIBS) $(FLIBS)
	
Now, what will happen is this: a system-specific `src/Makevars` is created based on the template file `src/Makevars.in`, where the variables `@OPENMP_FLAG@` and `@OPENMP_CFLAG@` are replaced by the appropriate values, which are in turn obtained in the `configure.ac` file. Thus, depending on which is more appropriate, either of the two `Makevars` files described in the previous section will be created.

Finally, it is also appropriate to add a `cleanup` file (also in the main directory) so that created files are removed:

	#!/bin/sh

	rm -f config.* src/Makevars
	
# Running `autoconf`
At this point, I first thought I was done and it would work. But to my surprise, it did not. What I was missing was a final but important step: to create a `configure` file from the `configure.ac` script. This is easily done on a Linux computer by the following terminal statement:

    autoconf configure.ac --output=configure
    
Next, just copy the newly-created (and incredibly long) `configure` file into the main directory of the package.

At this point, the package should compile on both Linux and OS X, with OpenMP support being dynamically enabled.

# Fixing line endings
However, there's rarely a problem fix that doesn't cause a new problem, and this is no exception. When checking the package on [win-builder](https://win-builder.r-project.org/), I received a warning about `CRLF` (Windows-style) line endings in `configure.ac` and `cleanup`, which should be `LF` (Unix-style endings). Luckily, this can easily be fixed automatically by GitHub.

First, set the default for the repository to be `LF`: 

    git config --global core.autocrlf input

Next, create `.gitattributes` in the main directory, containing the following:

    ^configure\.ac$ text eol=lf
    ^cleanup$ text eol=lf
    
This forces the line endings to always be `LF` when pushing to GitHub. (You will also need to add `^/\.gitattributes$` to `.Rbuildignore`.)