---
layout: post
title: 'Perl vs Python vs Ruby: Parsing Fasta Files'
tags: R
comments: true
---

Parsing fasta files is the single most common thing one does when working with
sequencing data, so any programming language used should do the parsing
efficiently and succinctly.  Unfortunately due to the history of the format and
the length of time it has been in use it has some archaic features, line
wrapping and (sometimes) limited header size for example.

<script src="https://gist.github.com/3169859.js?file=fasta_example.fa"></script>

Many people from a biologist background who first try to parse fasta files see
this as a header followed by lines, so simply read the file line by line,
appending the sequence as they go, and checking for a new header as you read.

For example in perl:

<script src="https://gist.github.com/3169859.js?file=append.pl"></script>

This is unintuitive code, you have to either put your processing code in
a function or duplicate code to handle the last record in the file. You also
must preprocess the first header, it also requires you to define non local
header and sequence variables, which can cause logic errors.  This method is
also computationally inefficient because the sequence string has to be
repeatably resized as more lines are read in.

### Perl ###

The preferred method to read fasta files is to read them a record at a time:

<script src="https://gist.github.com/3169859.js?file=fasta.pl"></script>

This technique reads each record in one block, so no repeated string resizing
is necessary.  The program flow is also straightforward, the record is
reformatted, then processed, and the processing code is only used in one
location.  It is important to read the first '>' before entering the loop,
otherwise the first iteration of the loop will contain no information.  The
only other thing to keep in mind is to remove the new line characters from the
sequence, done here with the transliteration operator in perl.


The difference in execution time is significant:


{% highlight bash %}
append.pl Hg19.fa 24.34s user 8.80s system 99% cpu 33.153 total
fasta.pl Hg19.fa 15.34s user 1.83s system 99% cpu 17.172 total
{% endhighlight %}

Approximately a 2x speed increase between the two methods.

These tests were all done using perl, lets explore fasta reading in python and
ruby.

### Ruby ###

Ruby has been described as a mashup of perl and smalltalk, so we should be able
to use a very similar approach to the perl code, and this is indeed the case:

<script src="https://gist.github.com/3169859.js?file=fasta.rb"></script>

This code and syntax is very clean however the performance still lags behind
the perl implementation by a wide margin:


{% highlight bash %}
ruby fasta.rb Hg19.fa 22.90s user 8.68s system 99% cpu 31.578 total
{% endhighlight %}

### Python ###

Unfortunately there is no way to read until a delimiter other than newline in
python, which limits us to using the slow appending algorithm only:

<script src="https://gist.github.com/3169859.js?file=fasta.py"></script>


{% highlight bash %}
python fasta.py Hg18.fa  54.80s user 2.23s system 99% cpu 57.039 total
{% endhighlight %}

Compared to the perl implementation which ran in 17 seconds, the python
implementation is 3 times slower.

*edit*
Maximilian Haeussler brought up the point that I am using string concatenation
inside the loop, and that could be the reason for the poor python performance
due to many memory reallocations as the string grows.  This is a fair point, so
lets test and see if that is the problem.

<script src="https://gist.github.com/3169859.js?file=fasta2.py"></script>


{% highlight bash %}
python fasta2.py Hg18.fa  45.73s user 3.78s system 99% cpu 49.534 total
{% endhighlight %}

So while this method is slightly faster, it is not substantially so, and is clearly not the main cause of the poor performance.  I would gather the main reason for the slow parsing in python has to do with the sheer number of read operations which must be performed compared to the perl or ruby versions.  Because python can only read line by line the python version has to perform over 60 million reads for the Hg18 human genome, while because the ruby and perl versions only have to perform 49 total reads (there are random chromosomes in the Hg18 assembly).  This large difference in speed is likely due large difference in call overhead.

So for fasta reading python is definitely has the poorest performance of the
three languages  due to the lack of non newline delimiters.  Next post I will look into the regex performance of the three
languages on sequencing data.
