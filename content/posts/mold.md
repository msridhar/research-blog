+++
date = "2014-10-09T21:00:36-07:00"
title = "Mold at OOPSLA'14"
+++

In a couple of weeks, [Cosmin Rădoi](http://publish.illinois.edu/cos/)
will be presenting our work
[_Translating Imperative Code to MapReduce_](http://manu.sridharan.net/files/OOPSLA14Mold.pdf)
at [OOPSLA 2014](http://2014.splashcon.org/track/oopsla2014).  (Cosmin
led this project, and the key ideas and system implementation are due
to him.)  The paper describes a tool <span style="font-variant:
small-caps">Mold</span> that can automatically translate small,
sequential, imperative Java programs manipulating array-like data
structures into efficient, parallel MapReduce programs.  For example,
consider the following sequential implementation of the "word count"
program (the "Hello world!" of MapReduce):

```java
Map<String,Integer> wordCount(List<String> docs) {
  Map<String,Integer> m = new HashMap<>();
  for (int i = 0; i < docs.size(); i++) {
    String[] split = docs.get(i).split(" ");
    for (int j = 0; j < split.length; j++) {
      String w = split[j];
      Integer prev = m.get(w);
      if (prev == null) prev = 0;
      m.put(w, prev + 1);
    } 
  }
  return m;
}
```

<span style="font-variant: small-caps">Mold</span> can automatically
translate the above to the following Scala code:

```scala
docs
  .flatMap({ case (i, v9) => v9.split(" ") })
  .map({ case (j, v8) => (v8, 1) })
  .reduceByKey({ case (v2, v3) => v2 + v3 })
```

This looks a lot like the
[word count program for Apache Spark](https://spark.apache.org/examples.html)
(scroll down to "Word Count").  <span style="font-variant:
small-caps">Mold</span> provides multiple backends, so that the above
code can be run using either [Spark](https://spark.apache.org/) or the
[Scala parallel collections library](http://docs.scala-lang.org/overviews/parallel-collections/overview.html).

A big challenge here is the same difficulty faced by parallelizing
compilers, namely the need to execute loop iterations in parallel
without breaking the program.  For the Java program above, simply
running the inner loop iterations in parallel may not be safe, since
different iterations may update the count of the same word, causing a
data race.  MapReduce gets around this limitation via its
["shuffle" operation](http://en.wikipedia.org/wiki/MapReduce#Overview),
which allows for grouping inputs by some key and then applying the
reducer to different groups in parallel.  For word count, the keys are
the words themselves, so, e.g., counts can be computed in parallel for
words starting with letters 'a'--'c', 'd'--'f', etc.  Part of <span
style="font-variant: small-caps">Mold</span>'s power is in its ability
to automatically find fruitful points to introduce these "shuffle"
operations to extract more parallelism.

At a high level, <span style="font-variant: small-caps">Mold</span>
works as follows:

* <span style="font-variant: small-caps">Mold</span> first translates
  the input program into a *functional* intermediate form via
  [Array SSA](http://www.cs.indiana.edu/~achauhan/Teaching/B629/2006-Fall/CourseMaterial/1998-popl-knobe-arrayssa.pdf).
  While the correspondence between SSA and functional programming is
  [well](http://dl.acm.org/citation.cfm?doid=278283.278285)
  [known](http://dl.acm.org/citation.cfm?doid=202529.202532), our
  translation is different in that it generates `fold` operations to
  preserve the structure of loops.
* Given the functional IR, <span style="font-variant:
  small-caps">Mold</span> employs a term rewriting system to search a
  large space of semantically-equivalent programs.  The rewrite rules
  aim to introduce (parallel) map and shuffle operations wherever
  possible, exposing the parallelism in the input program.  The final
  output program is chosen based on a heuristic cost function that can
  be customized to obtain good performance for different execution
  environments (cluster, multi-core, etc.).

Thus far, we've only run <span style="font-variant:
small-caps">Mold</span> on small input programs (and some challenges
must be overcome to scale it up), but the results were promising, with
performance comparable to hand-written MapReduce programs in most
cases.  <span style="font-variant: small-caps">Mold</span>'s initial
success shows how functional representations can be useful for
parallelization even when the original program is imperative.

For all the technical details, check out
[the paper](http://manu.sridharan.net/files/OOPSLA14Mold.pdf).  Or,
even better, if you'll be at OOPSLA, attend Cosmin's talk and ask him
for details!




