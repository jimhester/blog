---
layout: post
title: 'Perl vs Python vs Ruby: Fasta reading using Bio packages'
tags: R
comments: true
---

Since all the languages I mentioned in my [previous
post](http://jimhester.calepin.co/perl-vs-python-vs-ruby-parsing-fasta-files.html)
have Bio packages which can parse fasta files, I did a quick comparison of the
performance of the three implementations.  Here are the implementations, they
are highly similar.

### Perl ###
<script src="https://gist.github.com/3169859.js?file=fasta-bioperl.pl"></script>

### Ruby ###
<script src="https://gist.github.com/3169859.js?file=fasta-bioruby.rb"></script>

### Python ###
<script src="https://gist.github.com/3169859.js?file=fasta-biopython.py"></script>


{% highlight bash %}
fastaLengths-bio.pl Hg19.fa 65.15s user 11.84s system 99% cpu 1:17.00 total
fastaLengths-bio.rb Hg19.fa 56.07s user 14.18s system 99% cpu 1:10.26 total
fastaLengths-bio.py Hg19.fa 46.85s user 13.11s system 99% cpu 59.969 total
{% endhighlight %}

This highlights a major implementation deficiency in the perl and ruby bio
projects for reading fasta files as the results here are the exact reverse of
the simple parsers from my previous post. This performance regression is due to
the bioperl SeqIO method attempting to identify the sequence as dna or protein
every time next_seq is called, setting the type in the SeqIO constructor brings
the perl implementation back in the lead by a fair margin.

### Perl 2 ###
<script src="https://gist.github.com/3169859.js?file=fasta-bioperl2.pl"></script>


{% highlight bash %}
fastaLengths-bio.pl Hg19.fa 38.50s user 10.76s system 99% cpu 49.267 total
{% endhighlight %}
