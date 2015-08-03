---
layout: post
title: Parsing gff files with Rcpp for use with ggbio
tags: R
comments: true
---

The [ggbio][ggbio] package is a great tool to use for visualizing next generation
sequencing data.  It combines the graphing power of the [ggplot2][ggplot2] package
with the biology aware data structures in bioconductor.  The package includes
support for plotting genes in the standard databases supplied by bioconductor,
which works well for heavily studied organisms such as human and mouse.

If you are interested in a less well annotated organism, there is no prebuilt
database to pull from.  In this case often the gene annotation is described as
a general feature format (gff) file.  This specification has gone through
a number of revisions through the years, the most current of which is [gff3][gff3].

Unfortunately, there is no current function to import gff3 into the GRangeList
format which is used by ggbio to plot genes.  This specification also is
somewhat complex to parse in R due to there being multiple levels of
relationships and optional fields.

The [genomeIntervals][genomeIntervals] package contains a function to read gff3 files, however
this creates a "Genome_intervals_stranded" object, not the GRangesList we need
for ggbio.  The [easyRNASeq][easyRNASeq] package has a function to convert the
Genome_intervals_stranded object into a GRangesList, but that seems like
a large unrelated dependency, it would probably be easier to just parse and
create the object directly as a GRangesList.  Fortunately using Rcpp, some tips
from the above packages and some convenient helper functions it is actually
fairly straightforward.

First we create a helper function to do string tokenizing

{% highlight cpp %}
vector<string> inline split_string(const string &source, const char *delimiter = " ", bool keep_empty = false) {
    vector<string> results;

    size_t prev = 0;
    size_t next = 0;

    while ((next = source.find_first_of(delimiter, prev)) != string::npos) {
        if (keep_empty || (next - prev != 0)) {
            results.push_back(source.substr(prev, next - prev));
        }
        prev = next + 1;
    }
    if (prev < source.size()) {
        results.push_back(source.substr(prev));
    }

    return results;
}
{% endhighlight %}

Then the c++ parsing code to convert the gff file to a data frame.  Note we
have to store the entire file in memory before constructing the data frame to
determine the number of columns due to the optional attributes.


{% highlight cpp %}
// [[Rcpp::export]]
Rcpp::DataFrame gff2df(std::string file, const char *attribute_sep = "=") {
  CharacterVector records;
  vector< vector < string > > column_strings(FIELD_SIZE);
  vector< map< string, string > > attribute_list;
  set< string > attribute_types;
  ifstream in(file.c_str());
  string rec;
  int count=0;
  while(getline(in,rec)){
    if(rec.at(0) != '#'){ //not a comment line
      vector< string > strings = split_string(rec,"\t");
      for(uint i = 0;i<strings.size()-1;++i){
        column_strings[i].push_back(strings.at(i));
      }
      vector< string > attribute_strings = split_string(strings.at(strings.size()-1), ";");
      map< string, string> line_attributes;
      for(uint i = 0;i< attribute_strings.size();++i){
        vector< string > pair = split_string(attribute_strings.at(i), attribute_sep);
        line_attributes[pair.at(0)] = pair.at(1);
        if(attribute_types.find(pair.at(0)) == attribute_types.end()){
          attribute_types.insert(pair.at(0));
        }
      }
      attribute_list.push_back(line_attributes);
    }
    else if(rec.find("##FASTA") != string::npos){
      break;
    }
    count++;
  }
  Rcpp::DataFrame result;
  for(uint i = 0;i<FIELD_SIZE;++i){
    result[FIELDS[i]]=column_strings.at(i);
  }
  for(set< string >::const_iterator it = attribute_types.begin(); it != attribute_types.end(); ++it){
    vector< string > column_data;
    for(vector< map<string, string > >::const_iterator vec_it = attribute_list.begin();vec_it != attribute_list.end();++vec_it){
      if(vec_it->count(*it) == 1){
        column_data.push_back(vec_it->at(*it));
      }
      else{
        column_data.push_back("NA");
      }
    }
    result[*it]=column_data;
  }
  return(result);
}
{% endhighlight %}

This gives us the file as a dataframe in R.  We then need to transform the data
frame into a GRangeList object for ggbio.  One problem with constructing the
data frame in the C++ code the way that I did it is that all the columns are
created as strings, even though a number of the columns are numeric, and the
others can probably be factors.  Luckily using the all_numeric function from
the [Hmisc][Hmisc] package will do the test and conversion for us.


{% highlight r %}
#from Hmisc library
all_numeric = function (x, what = c("test", "vector"), extras = c(".", "NA")) {
  what <- match.arg(what)
  old <- options(warn = -1)
  on.exit(options(old))
  x <- sub("[[:space:]]+$", "", x)
  x <- sub("^[[:space:]]+", "", x)
  xs <- x[! x %in% c("", extras)]
  isnum <- !any(is.na(as.numeric(xs)))
  if (what == "test"){
    isnum
  }
  else if (isnum){
    as.numeric(x)
  }
  else {
    x
  }
}
{% endhighlight %}

Then all we have to do is split the data based on seqid and grouping variable
to construct the GRangesList object we want.


{% highlight r %}
read_gff = function(file, grouping='Parent', attribute_sep="=") {
  data = data.frame(lapply(gff2df(file, attribute_sep), all_numeric, what="vector"))
  if(!grouping %in% colnames(data)){
    stop(paste(grouping, "does not exist in", file, ", valid columns are", paste(colnames(data), collapse=" ")))
  }
  stopifnot(grouping %in% colnames(data))
    lapply(split(data, factor(data$seqid)), function(df) {
      split(GRanges( ranges=IRanges( start=df$start, end=df$end),
                    seqnames=df$seqid,
                    strand=df$strand,
                    mcols = df[, !colnames(df) %in% c("seqid", "strand", "start", "end") ]
                   ),
           df[,grouping], drop=TRUE
           )
    })
}
{% endhighlight %}

This function can then be used to read and plot a gene annotation with ggbio


{% highlight r %}
library(ggbio)
{% endhighlight %}



{% highlight text %}
​Loading required package: BiocGenerics
Loading required package: parallel

Attaching package: 'BiocGenerics'

The following objects are masked from 'package:parallel':

    clusterApply, clusterApplyLB, clusterCall, clusterEvalQ,
    clusterExport, clusterMap, parApply, parCapply, parLapply,
    parLapplyLB, parRapply, parSapply, parSapplyLB

The following object is masked from 'package:stats':

    xtabs

The following objects are masked from 'package:base':

    anyDuplicated, append, as.data.frame, as.vector, cbind,
    colnames, do.call, duplicated, eval, evalq, Filter, Find, get,
    grep, grepl, intersect, is.unsorted, lapply, Map, mapply,
    match, mget, order, paste, pmax, pmax.int, pmin, pmin.int,
    Position, rank, rbind, Reduce, rep.int, rownames, sapply,
    setdiff, sort, table, tapply, union, unique, unlist, unsplit

No methods found in "RSQLite" for requests: dbGetQuery
Need specific help about ggbio? try mailing 
 the maintainer or visit http://tengfei.github.com/ggbio/

Attaching package: 'ggbio'

The following objects are masked from 'package:ggplot2':

    geom_bar, geom_rect, geom_segment, ggsave, stat_bin,
    stat_identity, xlim

{% endhighlight %}



{% highlight r %}
source('read_gff.R')
{% endhighlight %}



{% highlight text %}
​Error in file(filename, "r", encoding = encoding): cannot open the connection

{% endhighlight %}



{% highlight r %}
gff = read_gff('data/eden.gff', grouping='Name')
{% endhighlight %}



{% highlight text %}
​Error in lapply(gff2df(file, attribute_sep), all_numeric, what = "vector"): error in evaluating the argument 'X' in selecting a method for function 'lapply': Error: could not find function "gff2df"

{% endhighlight %}



{% highlight r %}
autoplot(gff[[1]])
{% endhighlight %}



{% highlight text %}
​Error in autoplot(gff[[1]]): error in evaluating the argument 'object' in selecting a method for function 'autoplot': Error: object 'gff' not found

{% endhighlight %}

Note that parsing gtf files can be done with the same code, you just need to
change the attribute seperator from '=' to ' '


{% highlight r %}
library(ggbio)
source('read_gff.R')
{% endhighlight %}



{% highlight text %}
​Error in file(filename, "r", encoding = encoding): cannot open the connection

{% endhighlight %}



{% highlight r %}
gtf = read_gff('data/Mus_musculus.GRCm38.70.gtf', grouping='gene_id', attribute_sep=' ')
{% endhighlight %}



{% highlight text %}
​Error in lapply(gff2df(file, attribute_sep), all_numeric, what = "vector"): error in evaluating the argument 'X' in selecting a method for function 'lapply': Error: could not find function "gff2df"

{% endhighlight %}



{% highlight r %}
autoplot(gtf[[1]])
{% endhighlight %}



{% highlight text %}
​Error in autoplot(gtf[[1]]): error in evaluating the argument 'object' in selecting a method for function 'autoplot': Error: object 'gtf' not found

{% endhighlight %}

The code for all of the above functions are at this [gist][gist].

[gist]: https://gist.github.com/4613075
[ggbio]: http://www.bioconductor.org/packages/release/bioc/html/ggbio.html
[ggplot2]: http://ggplot2.org/
[gff3]: http://www.sequenceontology.org/resources/gff3.html
[genomeIntervals]: http://www.bioconductor.org/packages/2.12/bioc/html/genomeIntervals.html
[easyRNASeq]: http://www.bioconductor.org/packages/release/bioc/html/easyRNASeq.html
[Hmisc]: http://cran.r-project.org/web/packages/Hmisc/index.html
