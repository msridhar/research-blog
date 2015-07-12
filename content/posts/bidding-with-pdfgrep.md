+++
date = "2015-07-12T06:36:38-07:00"
title = "Paper Bidding With pdfgrep"

+++

I have a bunch of paper bidding to do for
[POPL](http://conf.researchr.org/home/POPL-2016), and in discussing
with others I got a great tip from Eran Yahav on using
[pdfgrep](https://pdfgrep.org/) to find papers of interest.  I thought
I'd quickly write up how I'm using it.  First, I downloaded all the
submissions.  If your conference is using
[HotCRP](http://www.read.seas.harvard.edu/~kohler/hotcrp/), you can
conveniently do a search for `-conflict:me` to get a list of all
papers for which you do not have a conflict, and then download them
using the links at the bottom.  Once you have all the papers in a
folder, you can run a search like so:

    # find papers mentioning types
    pdfgrep -cH types *.pdf | grep -v ":0"

I run with `-c` to only show which files match and the number of
matches---usually I want to open the full PDF to get proper context
for the actual matches.  The above search can be a bit
slow, but fortunately most of us have multicores, and the search is
easily parallelized using
[GNU Parallel](http://www.gnu.org/software/parallel/):

    parallel 'pdfgrep -cH types {}' ::: *.pdf | grep -v ":0"

On my MacBook Pro, this searches around 250 PDF files in less than 20
seconds.  I installed pdfgrep and GNU Parallel using MacPorts, but I'm
guessing Homebrew or direct installation will also work fine.



