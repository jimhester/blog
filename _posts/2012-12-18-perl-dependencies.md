---
layout: post
title: One liner for perl dependencies
tags: R
comments: true
---

If your module is FooBar, and you are [using cpanminus](http://jimhester.com/post/setting-up-a-local-cpan-using-cpanminus-without-root-access) then

```perl
cpanm perl -MFooBar -e 'print join("\n", keys %INC),"\n"'
```

will install all the dependencies needed for that module.

This however will not work if you do not have the modules installed to run the 
script in the first place, but if you install the Devel::Modlist package it is 
as simple as

```perl
cpanm `perl -MDevel::Modlist=stdout,noversion FooBar.pl`
```
