---
layout: post
title: Assignment Conversion
tags: R
comments: true
---

The R language has two common methods of assignment. The `=` and `<-` binary
operators are semantically equivalent in most cases, so which to use largely
comes down to convention for the particular project. At times it may be
desirable to convert between the two styles.

Converting all instances of `<-` to use `=` assignment is fairly straightforward.
`<-` only performs assignment and is unique, so one can simply do a regular
expression replacement for all occurrences of `<-`. The regex uses a [negative
lookbehind](http://www.regular-expressions.info/lookaround.html#lookbehind) to
avoid corner cases such as user defined `%<-%` operator or the `<<-` operator
and should work in nearly every case.

<script src="https://gist.github.com/jimhester/96539a09055402d194002e8d3e2ea172.js?file=changeArrowAssign.R"></script>

Converting all instances of `=` to use `<-` is more challenging. `=` can also
occur in the equality operators `==`, `>=`, `<=` as well as named arguments in
function calls `fun(x = 1)`. It is not possible to use a regular expression to
disambiguate all cases for this transformation.

Fortunately we can use information provided by R's parser to handle this issue
for us. The 
[`assignment_linter`](https://github.com/jimhester/lintr/blob/master/R/assignment_linter.R#L3-L17)
in the [lintr](https://github.com/jimhester/lintr#readme) uses this method, so
we can discriminate the assignment `=` from the argument `=`. So we can lint
the file using the `assignment_linter` to find the locations of all `=`
assignments, and convert them to `<-` assignments using a little Rcpp.

<script src="https://gist.github.com/jimhester/96539a09055402d194002e8d3e2ea172.js?file=changeEqualAssign.R"></script>

This gives us a robust method of performing this replacement without error for
any number of files. A similar linter and replacement function could be written
to replace the regular expression approach for modifying arrow assignment.
